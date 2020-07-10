
:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   Summary of performance studies with APDB prototype using three-node Cassandra cluster.


Introduction
============

This technical note summarizes the results of a series of tests performed with
an implementation of :abbr:`APDB (Alert Production Database)` which uses
Apache Cassandra as a backend. It is a continuation of the previous tests with
relational databases which were described in `DMTN-113`_. Those tests showed
serious scalability issues which practically rules out use of that technology
in a production setup. Scaling performance to acceptable level will likely
need a distributed system that has larger aggregate I/O throughput and IOPS
and uses RAM for faster data access.

`Apache Cassandra`_ is a well-known NoSQL database system with a number of
attractive features that can help with the implementation of such scalable
system:

  - it is an inherently distributed system with a flexible and configurable
    partitioning of the data,
  - partitioning and clustering of on-disk data can be utilized to minimize
    the number of I/O requests when reading the data,
  - all new data is stored in RAM (and fast commit log) making write
    operations extremely fast,
  - configurable replication and data consistency supports range of trade-offs
    between performance and data consistency.

These Cassandra features seem to be a close match for the requirements of the
APDB system and to validate that it can deliver necessary performance we
implemented series of tests using small-scale Cassandra cluster.


Hardware
========

It was clear before we started this round of experiments that we need a
reasonably large cluster to obtain a meaningful performance numbers. While
Cassandra can run on a single node it is clearly not an optimal setup for the
scale of the problem, and some features such as replication cannot be even
used with a single node cluster. For reliable consistency Cassandra should use
replication factor three which needs at least three-node cluster, this is not
going to be optimal for performance because each node has a full set of data.

As we did not have dedicated server hardware for APDB test we had to borrow
three machines from PDAC QServ cluster:

  - one machine from existing QServ cluster used initially qs QServ czar node
    with full name lsst-qserv-master02 (using short name master02 below),
  - two new identical machines purposed for QServ cluster purchased in 2019.

Here is the summary table with the most important specs:

    +----------+---------------------------+----------------------------+
    |          | master02                  | master03,04                |
    +==========+===========================+============================+
    | CPU      | 2 x Intel Xeon Gold 5120; | 2 x Intel Xeon Gold 5218;  |
    |          | 2 x 14 cores              | 2 x 16 cores               |
    +----------+---------------------------+----------------------------+
    | RAM      | 256 GiB                   | 256 GiB                    |
    +----------+---------------------------+----------------------------+
    | Storage  | 12 x 480 GB SATA SSD;     | 5 x 4000 GB NVMe SSD       |
    |          | RAID controller           |                            |
    +----------+---------------------------+----------------------------+

One significant difference between old master02 and new master03,04 machines
is storage system, old machine has slower SATA storage connected to a single
RAID controller. This limits performance of the storage, in particular IOPS.
Here is the approximate aggregate numbers for SSD storage as a whole on two
types of hardware:

    +-----------------+----------+-------------+
    |                 | master02 | master03,04 |
    +=================+==========+=============+
    | Read IOPS       | 15 k     | 1.5 M       |
    +-----------------+----------+-------------+
    | Write IOPS      | 12 k     | 1.1 M       |
    +-----------------+----------+-------------+
    | Read bandwidth  | 2.3 GBps | 8.5 GBps    |
    +-----------------+----------+-------------+
    | Write bandwidth | 4.5 GBps | 8.5 GBps    |
    +-----------------+----------+-------------+

There was no dedicated study of the impact of the lower IOPS rate on master02
on the performance of the whole Cassandra cluster.


Client configuration
====================

Test was using the same ``ap_proto`` application that was used in earlier
tests. ``dax_apdb`` package that implements APDB interface was extended to
support Cassandra backend using ``cassandra-driver`` Python package. APDB
interface needed few incompatible changes due to differences in the data model
between relation databases and Cassandra, all development was done on a
separate branch due to that.

Like in previous tests with Oracle clients were running at LSST verification
cluster using MPI for client communication, there was one client process per
single CCD, total 189 clients running concurrently. Clients communicate with
Cassandra cluster using Cassandra native protocol, with all three server
machines are accessible over public network (except in case of Docker test as
described below).


Partitioning
============

Partitioning and clustering indices paly very important role in Cassandra and
need to be carefully chosen based on how data are produced and accessed. For
AP pipeline main access method is based on spatial coordinates, each client
needs to read data corresponding to whole CCD region and produces new data for
the same region. Access to DIASource and DIAForcedSource is also restricted by
the history depth (nominally 12 months). For non-source data it is obvious
that we need some sort of spatial partitioning, and for sources it likely will
be a combination of the spatial and temporal partitioning.

One of the goals of partitioning in the distributed system like Cassandra is
to optimally distribute load across multiple nodes in the cluster to avoid
bottlenecks. In a context of AP pipeline with 189 independent CCD processes it
means that we want to send queries from different CCD to different partitions.
Ideally we would want one-to-one correspondence between CCDs and partitions
but that is of course impossible because CCD position on the sky is random.
The goal in that case is to optimize the size of the partition and
partitioning scheme so that on average the load is balanced across cluster and
performance is optimal.

Spatial partitioning
--------------------

For spatial partitioning we could select one of the existing schemes already
in use in LSST, e.g. HTM ot Q3C. It is interesting to compare some
characteristics of the partitioning based on these pixelization schemes. For
that we implemented a simple Monte-Carlo experiment with a selected
pixelization scheme/level and randomly placing a square CCD regions on
resulting partitioning grid. For each trial three numbers were calculated:

  - number of pixels touched by a single CCD which should corresponds to the
    number of queries that need to be executed;
  - total area of all pixels touched by single CCD, number of DIAObjects
    returned from queries is proportional to area, and it is beneficial to
    have this are to be as close as possible to a CCD size;
  - number of CCDs touched by a single pixel, this corresponds to number of
    clients querying the same partition.

For this exercise two pixelization schemes were selected  -- HTM, and MQ3C --
which are implemented in ``sphgeom`` package. Plots below summarize results of
the Monte-Carlo. :numref:`dm-19536-avg-tile-pix.png` shows average values of
the two numbers as a function of level. :numref:`dm-19536-hist-tile-pix.png`
shows distribution of the values for specific level.
:numref:`pixels-tiles-partitioning.png` shows distributions for number of
tiles per CCD (tile) and total area of those tiles.

From the plots one can conclude that MQ3C shows significantly better behavior
for pixels-per-tile value, while for tile-per-pixel value they behave
similarly. This is an expected behavior due to difference in pixel shape. 


.. figure:: /_static/dm-19536-avg-tile-pix.png
   :name: dm-19536-avg-tile-pix.png
   :target: _static/dm-19536-avg-tile-pix.png

   Average number of pixel/tile connections as a function of pixelization
   level for different pixelization schemes.

.. figure:: /_static/dm-19536-hist-tile-pix.png
   :name: dm-19536-hist-tile-pix.png
   :target: _static/dm-19536-hist-tile-pix.png

   Distributions for the number of pixel/tile connections for different
   pixelization schemes and pixelization level.

.. figure:: /_static/pixels-tiles-partitioning.png
   :name: pixels-tiles-partitioning.png
   :target: _static/pixels-tiles-partitioning.png

   Distributions for the number of pixels per tile and total pixel area for
   different pixelization schemes and pixelization level.


Temporal restriction
--------------------

Queries om DIASource table are temporally restricted to 12 months. There are
few different strategies that may be used for handling this restriction. One
of the important goals in defining schema for APDB would to eliminate the need
to access the files that were produced long time ago (older than 12 months) so
that those files could be moved to slower, less expensive storage.

Possible options for schema definition:

  - Partition by spatial index only, cluster using temporal index. This does
    not increase then number of partitions or queries but it means that old
    SSTable files have to be searched for data they don't have.
  - Partition by both spatial and temporal index. This means increasing
    number of partitions and queries. Due to Cassandra's probabilistic
    indexing feature it may still happen that some old files may be accessed.
  - Using client-controlled namespaces, probably easiest in the form of
    separate tables. Management of the namespaces will be left to client, so
    some additional logic will need to be implemented. Number of queries will
    grow depending on the granularity of the namespace "partitioning".

LAtter option is probably the one that allows precise control over which data
can be retired to slow storage without impacting performance. Few initial
tests in this study were done with temporal partitioning but most remaining
tests used separate client-controlled tables for namespaces.


Test with Cassandra on PDAC
===========================

Below is a description of many tests performed with Cassandra cluster running
on three PDAC nodes. Tests were done with different setup and configuration,
not all results of these tests are meaningful of comparable to other results
due to differences or configuration mistakes that were a part of learning
process. Cassandra configuration is quite complicated and need a deep
understanding of internal architecture, so trial and error is an essential
part of the process. There is a lot of details about the tests in
corresponding JIRA tickets, links to the ticket are included below.

Initial test
------------

The very first test (`DM-20580`_) was done mostly to study the tools,
configuration, and the behavior of the system with some specific goals:

  - check initial performance numbers
  - understand configuration and find optimal parameters for our setup
  - evaluate management tools and how they can be integrated into workflow

In thi test three nodes were configured slightly differently to take into
account difference in storage system, in particular master02 node was
allocated 96 tokens compared to 256 for each other node.

One of the early ideas was to try to keep number of I/O operations minimal by
not flushing the data from memory during the night, forcing the flush and
compacting data during the day. Implementing this cycle in this initial test
did not show any improvements in performance with compacted data compared to a
default setup when data was compacted less frequently. Our intermediate
conclusion was that Cassandra shows no significant performance degradation
with less compacted data so default compaction policy may work sufficiently
well. Forced compaction takes significant time and there is no reason doing it
without clear benefit.

Overall performance of this initial test was unexpectedly low, after running
for 30k visits (~1 month worth of data) average read time was at the level of
3 seconds per one CCD per visit, but average store was around 7 seconds. This
did not make a lot of sense as Cassandra performance for write operations was
supposed to be much better. Also during the test we observed many cases of
client-side timeouts that point to significant performance issues that need to
be understood.

For analyzing these performance issues we instrumented our Cassandra setup
with a monitoring tool that used Cassandra JMX interface (`DM-23604`_) to
extract monitoring metrics and dump it to a file which was later ingested into
InfluxDB and exposed to Grafana. Monitoring information was also extracted
from ``ap_proto`` log files as well and saved to the same InfluxDB so we could
correlate things happening on client and server side.

Analyzing monitoring data we quickly established that the reason for poor
performance in the initial test is an over-committing of the memory. Even
though JVM was configured to leave significant amount of RAM to other
processes there were some services (notably GPFS) which also needed
significant amount of RAM and that caused intensive swapping. Reducing memory
allocation for JVM allowed us to avoid swapping and improved performance to
more reasonable level.


Java Garbage Collection
-----------------------

Second round of tests (`DM-23881`_) started with reduced JVM memory allocation
(160 GB) which eliminated swapping but we still saw frequent timeouts on
client side. Monitoring showed that on server side there were significant
delays happening during garbage collection. Apparently default garbage
collection algorithm (ParNew+CMS) used by Cassandra is not optimal for large
memory systems. Switching to to a different algorithm (G1GC) improved GC
efficiency and fixed client-side timeouts, this was done after ~60k visits,
the effect is clearly seen on the plot below.

Two other significant changes at this step were:

  - avoiding spatial filtering on server side (which uses fine-grain HTM20
    index) to drastically reduce number of queries executed on server, data
    from the whole partition is now returned to client;
  - switching to MQ3C level 10 pixelization for partitioning, this reduces the
    size of the partition and amount of data per CCD that are returned to
    client (see :numref:`pixels-tiles-partitioning.png`)

With this setup the test was run for 180k visits (approximately 6 months).
Write performance is improved dramatically and database operations are now
dominated by reading time which grows approximately linearly with the visit
number. :numref:`dm-23881-select-fit-1.png` summarizes read performance for
all separate tables and their total. At 180k visits total read time is
approximately 10 seconds per visit (per CCD).

.. figure:: /_static/dm-23881-select-fit-1.png
   :name: dm-23881-select-fit-1.png
   :target: _static/dm-23881-select-fit-1.png

   Select execution time as a function of visit number, "obj_select_real"
   is time for DIAObject table, "src_select_real" is for DIASource,
   "fsrc_select_real" is for DIAForcedSource, and "select_real" is the sum
   of three times. Data for visits below 60k is not included in fits.

This test was configured without replication and equal number of tokens on
each node meaning that each node was serving one third of total data. In
production setup we will have more than one replica for hgh availability
reason. Replication configuration needs to be tested as well even though
three-node cluster is not ideal for scaling the number of replicas. Cassandra
has so-called `tunable consistency`_ which allows certain tradeoffs between
data consistency and performance, the mechanism depends on the number of
replicas in the cluster. Minimal sensible replication level for this mechanism
is three if high availability is required.

For the next test configuration was set to use two replicas though consistency
level was kept at ``ONE``. The main goal of this setup is to verify that
cluster can handle twice the throughput in I/O without degradation, and there
was a concern that replication level three in cluster of three machines is not
optimal. With this setup ``ap_proto`` ran for 100k visits. It shows the same
linear scaling for select time without degradation compared to previous test.
Store time stays approximately constant or even improves slightly over time.
:numref:`dm-23881-select-fit-2.png` shows select time as a fitted function of
visit number, :numref:`dm-23881-store-fit-2.png` is a fit of store time for
four tables (DIAObject data is stored in two separate tables). Store time is
significantly lower than select time, just as expected due to Cassandra
storing its data in memory.

.. figure:: /_static/dm-23881-select-fit-2.png
   :name: dm-23881-select-fit-2.png
   :target: _static/dm-23881-select-fit-2.png

   Select execution time as a function of visit number, labeling corresponds
   to previous plot.

.. figure:: /_static/dm-23881-store-fit-2.png
   :name: dm-23881-store-fit-2.png
   :target: _static/dm-23881-store-fit-2.png

   Store execution time as a function of visit number. Total time is higher
   than the sum of individual tables due to additional overhead on client
   side for query preparation.


Test with docker
----------------

For next series of tests (same ticket `DM-23881`_) it was decided to run
multiple Cassandra instances on a single physical machine, one instance per
storage disk. SSD storage on master02 was organized into 4 virtual disks, they
are all connected to single RAID controller which could limit overall
performance. Total number of Cassandra nodes in cluster thus equals 14
(4+5+5), each node runs in a Docker container. One complication with this
setup is that we could only map a single docket container to a public routable
interface on a host machine (Cassandra default build does not support
different port numbers) which means that only three out of 14 nodes could
serve as coordinator nodes introducing potentially uneven load into the
system.

Teh goal of this exercise was two-fold:

  - Reduce memory allocation per node which should potentially reduce garbage
    collection overhead in JVM;
  - run with replication factor three to check higher consistency level
    settings.

On client side consistency level was set to ``QUORUM`` in this case meaning
that at least two replicas have to respond for each operation before success
status is returned to client.

Like in the previous test ``ap_proto`` was left running for 100k visits with
this configuration, results are represented in two plots below. Writing
performance has improved somewhat in this case but select performance is about
50% slower compared to previous result.

.. figure:: /_static/dm-23881-select-fit-3.png
   :name: dm-23881-select-fit-3.png
   :target: _static/dm-23881-select-fit-3.png

   Select execution time as a function of visit number.

.. figure:: /_static/dm-23881-store-fit-3.png
   :name: dm-23881-store-fit-3.png
   :target: _static/dm-23881-store-fit-3.png

   Store execution time as a function of visit number.


Scylla test
-----------

There exists an alternative open source implementation of Cassandra database -
`Scylla`_. Scylla is implemented in C++ though for compatibility some pieces
(e.g. JMX) run inside separate JVM instance. CLient side "native" protocol is
100% compatible with Cassandra so that existing client code in ``ap_proto``
can run seamlessly with Scylla.

For next series of tests (`DM-24692`_) we replaced Cassandra cluster with
similarly configured Scylla cluster with three nodes. Configuration is also
mostly compatible though some options behave differently or are not supported.
Some special tuning was necessary to avoid memory swapping issues. Scylla was
running stably for most part though on the client side there were occasional
transient connection issues observed which did not cause fatal errors.

For initial test with Scylla we used single replica with the goal to establish
a baseline similar to Cassandra case. Difference with Cassandra case was in
storage configuration, Scylla does not support multiple data directories, so
all data in this test are store on a single physical disk (or one virtual disk
in case of master02). Total 150k visits were produced in this configuration.
Plot :numref:`apdb-scylla1-nb-time-select-fit.png` shows select time
dependency, which is consistent, or maybe slightly worse, than single replica
Cassandra case (:numref:`dm-23881-select-fit-1.png`). Store time
(:numref:`apdb-scylla1-nb-time-store-fit.png`) is, like in Cassandra case, is
also negligible compared to select time.

.. figure:: /_static/apdb-scylla1-nb-time-select-fit.png
   :name: apdb-scylla1-nb-time-select-fit.png
   :target: _static/apdb-scylla1-nb-time-select-fit.png

   Select execution time as a function of visit number for Scylla with single
   replica.

.. figure:: /_static/apdb-scylla1-nb-time-store-fit.png
   :name: apdb-scylla1-nb-time-store-fit.png
   :target: _static/apdb-scylla1-nb-time-store-fit.png

   Store execution time as a function of visit number for Scylla with single
   replica.

For second round of Scylla test it was configured with three replicas and
three nodes, so that each node has full set of data. All storage on each node
was merged into a single logical LVM RAID0 volume.

Plot :numref:`apdb-scylla2-nb-time-select-fit.png` shows select performance
for this test. Compared to other cases the behavior looks more complicated --
initially it grows faster, then it suddenly improves around visit 90k. That
improvement corresponds to the restart of the Scylla cluster that was
performed as a cleanup after GPFS outage. Scylla does not use GPFS so it is
not clear how any potential GPFS issues could affect Scylla. ore likely
explanation is that Scylla itself developed some inefficiencies that were
cleared after restart.

Compared to tree-replica Cassandra case Scylla performance (after restart) is
somewhat better, ~7 vs ~9 seconds at 100k visits.

.. figure:: /_static/apdb-scylla2-nb-time-select-fit.png
   :name: apdb-scylla2-nb-time-select-fit.png
   :target: _static/apdb-scylla2-nb-time-select-fit.png

   Select execution time as a function of visit number for Scylla with three
   replicas.

.. figure:: /_static/apdb-scylla2-nb-time-store-fit.png
   :name: apdb-scylla2-nb-time-store-fit.png
   :target: _static/apdb-scylla2-nb-time-store-fit.png

   Store execution time as a function of visit number for Scylla with three
   replicas.


Three-replica Cassandra test
----------------------------

For final test we repeated Cassandra test with three replicas but without
Docker, using three instances similarly to Scylla case, and with identical
storage setup. One of the goals of this test was to check how consistency
level affects performance, so part of the test was run at consistency level
``ONE`` for reading (and ``QUORUM`` for writing).

Plot :numref:`apdb-cass4-nb-time-select-fit-quorum.png` shows select timing
for the first 180k visits when consistency level for reading was set to
``QUORUM``. Performance is significantly slower than for Scylla, and also
slower than Cassandra performance with three replicas in Docker setup. Plot
:numref:`apdb-cass4-nb-time-select-fit-one.png` shows timing for last 10k
visits wen reading consistency was set to ``ONE``. For this case performance
is closer to what was seen with Scylla and earlier Docker test, but in both
those cases consistency was set to ``QUORUM``.

It is not clear why Cassandra performance differs so much between Docker setup
with 14 nodes (which is sub-optimal) and 3-node configuration. We tried to
monitor query tracing information provided by Cassandra and there seem to be
an indication that query execution on master02 takes longer than on two other
hosts, though exact numbers are hard to extract due to large number of
concurrent clients. There is also some indication that replica repair
mechanism may be responsible for some of this effect, that can be investigated
further.

.. figure:: /_static/apdb-cass4-nb-time-select-fit-quorum.png
   :name: apdb-cass4-nb-time-select-fit-quorum.png
   :target: _static/apdb-cass4-nb-time-select-fit-quorum.png

   Select execution time as a function of visit number for Cassandra with three
   replicas and read consistency level ``QUORUM``.

.. figure:: /_static/apdb-cass4-nb-time-select-fit-one.png
   :name: apdb-cass4-nb-time-select-fit-one.png
   :target: _static/apdb-cass4-nb-time-select-fit-one.png

   Store execution time as a function of visit number for Scylla with three
   replicas and read consistency level ``ONE``.


Summary
-------

It is hard to summarize all above results in one single metric. For our case
the bottleneck seem to be the execution time of the select queries, so we
chose this as a main parameter. In most cases this parameters grows linearly
with visit number, except for the case of Scylla with three replicas which has
some unexplained fluctuations. It is expected to grow with the size of the
data and due to maximum source history size of 12 months it should level off
after that time.

For summary we want to present the numbers that can be compared easily, so we
chose as a metric the time to read all table at after 180k visits (about 6
month of data). Not all above tests were ran up to 180k visits, for those we
include an estimate obtained from extrapolating fitted data.

Here is the summary table for all above tests:

+----------------------+-----------+-----------+
| Test type            | #replicas | Time, sec |
+======================+===========+===========+
| Cassandra            |     1     |   10      |
+----------------------+-----------+-----------+
| Cassandra            |     2     |   10.5    |
+----------------------+-----------+-----------+
| Cassandra w/Docker   |     3     |   17      |
+----------------------+-----------+-----------+
| Scylla               |     1     |   11      |
+----------------------+-----------+-----------+
| Scylla               |     3     |   12.5    |
+----------------------+-----------+-----------+
| Cassandra w/QUORUM   |     3     |   23      |
+----------------------+-----------+-----------+
| Cassandra w/ONE      |     3     |   15      |
+----------------------+-----------+-----------+

Data sizes
==========

For reference here is the size of the data on disk for some of the test cases:

+----------------------+-----------+---------+-----------+
| Test type            | #replicas | #visits | Size, TB  |
+======================+===========+=====================+
| Cassandra w/Docker   |     3     |  100k   |   5.8     |
+----------------------+-----------+---------+-----------+
| Scylla               |     1     |  150k   |   3.03    |
+----------------------+-----------+---------+-----------+
| Scylla               |     3     |  178k   |   10.7    |
+----------------------+-----------+---------+-----------+
| Cassandra            |     3     |  190k   |   11.1    |
+----------------------+-----------+---------+-----------+


Unresolved issues
=================

There was a lot information collected during all these tests, still there are
some question that have not been answered completely. Here are some
questions/ideas worth investigating in the future tests:

  - It is not entirely clear why select time is proportional to the data
    volume (or visit number). Naive idea is that data volume is not extremely
    large and time should probably be proportional to the number of I/O
    operations which ideally should be constant.
  - Efficiency of the client side operation was not measured. It may be
    significant (10-20% or larger) and may have various contributions, there
    may be an opportunity for optimization there too.
  - Scylla numbers look good and maybe too good, it may be possible that
    Scylla uses some shortcuts which optimize performance but may hurt
    consistency. This need to be understood if we think that Scylla is a
    viable option.
  - It is not clear how master02 storage system affects overall performance.
    There are indications that bottleneck may be there, for the future tests
    it would be better to have more uniform setup.
  - There are indications that concurrent read/writes cause "read repairs" in
    Cassandra which is a potentially costly operation, would be nice to be
    able to quantify it.

Conclusion
==========

The tests show that small-scale Cassandra cluster can provide better
performance for storing APDB data than a single-node relation database server.
Main attractive feature of Cassandra is the ability to scale performance by
simply extending existing cluster. This scalability needs to be tested in more
realistic setup than our present test cluster.

Based on this experience future tests with Cassandra can benefit from using
hardware which better matches Cassandra workload:

  - JVM seems to work better with smaller resident sets, in that respect it
    may be better to have larger number of hosts with smaller RAM that a
    single host with huge memory.
  - High IOPS seem to be critical for achieving good read performance with
    APDB data, it is advisable to have all live data on NVMe disks which
    provide much better concurrency than SATA disks.

There are a lot of unanswered questions outlined in previous section,
answering them in the future tests can improve our understanding of
bottlenecks and further improve performance.


.. _Apache Cassandra: https://cassandra.apache.org/
.. _Scylla: https://www.scylladb.com/open-source/

.. _DMTN-113: https://dmtn-113.lsst.io/

.. _DM-19536: https://jira.lsstcorp.org/browse/DM-19536
.. _DM-20580: https://jira.lsstcorp.org/browse/DM-20580
.. _DM-23322: https://jira.lsstcorp.org/browse/DM-23322
.. _DM-23604: https://jira.lsstcorp.org/browse/DM-23604
.. _DM-23881: https://jira.lsstcorp.org/browse/DM-23881
.. _DM-24692: https://jira.lsstcorp.org/browse/DM-24692
.. _DM-25055: https://jira.lsstcorp.org/browse/DM-25055
.. _tunable consistency: https://docs.datastax.com/en/cassandra-oss/3.0/cassandra/dml/dmlAboutDataConsistency.html



.. JIRA APDB tickets, time-ordered
..
.. DM-19536 	May 2019
..     Evaluate Apache Cassandra as PPDB back-end option

.. DM-22039 	Nov 2019
..     Rename dax_ppdb to dax_apdb together with all dependencies

.. DM-23214 	Jan 2020
..     Migrate Cassandra development branch to APDB

.. DM-23322 	Feb 04-07, 2020
..     Install Cassandra for APDB test on PDAC

.. DM-20580 	Feb 7-26, 2020
..     Test more realistic setup of APDB with Cassandra
..     - performance is bad, need better monitoring

.. DM-23604 	Feb-Mar 2020
..     Implement cassandra monitoring for APDB tests

.. DM-23881 	Mar-Apr 2020
..     Test Cassandra APDB implementation with finer partitioning
..     - switched from HTM to MQ3C level=10
..     - switched to G1GC
..     - Docker test with three replicas

.. DM-24692 	May 2020
..     Test Cassandra APDB with with Scylla server

.. DM-25055 	May-Jun 202
..     Test Cassandra APDB with three replicas without docker
..     - also test with different levels of consistency
