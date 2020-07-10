
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
Apache Cassandra as a backend. It is a continuation of the previous tests
with relational databases which were described in `DMTN-113`_. Those tests
showed serious scalability issues which practically rules out use of that
technology in a production setup. Scaling performance to acceptable level
will likely need a distributed system that has larger aggregate I/O throughput
and IOPS and uses RAM for faster data access.

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
used with a single node cluster. For reliable consistency Cassandra should
use replication factor three which needs at least three-node cluster, this is
not going to be optimal for performance because each node has a full set of
data.

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
interface needed few incompatible changes due to differences in the data
model between relation databases and Cassandra, all development was done on
a separate branch due to that.

Like in previous tests with Oracle clients were running at LSST verification
cluster using MPI for client communication, there was one client process per
single CCD, total 189 clients running concurrently. Clients communicate with
Cassandra cluster using Cassandra native protocol, with all three server
machines are accessible over public network (except in case of Docker test
as described below).


Partitioning
============

Partitioning and clustering indices paly very important role in Cassandra and
need to be carefully chosen based on how data are produced and accessed. For
AP pipeline main access method is based on spatial coordinates, each client
needs to read data corresponding to whole CCD region and produces new data
for the same region. Access to DIASource and DIAForcedSource is also
restricted by the history depth (nominally 12 months). For non-source data
it is obvious that we need some sort of spatial partitioning, and for sources
it likely will be a combination of the spatial and temporal partitioning.

One of the goals of partitioning in the distributed system like Cassandra is
to optimally distribute load across multiple nodes in the cluster to avoid
bottlenecks. In a context of AP pipeline with 189 independent CCD processes
it means that we want to send queries from different CCD to different
partitions. Ideally we would want one-to-one correspondence between CCDs and
partitions but that is of course impossible because CCD position on the sky
is random. The goal in that case is to optimize the size of the partition and
partitioning scheme so that on average the load is balanced across cluster
and performance is optimal.

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
which are implemented in ``sphgeom`` package. Plots below summarize
results of the Monte-Carlo. :numref:`dm-19536-avg-tile-pix.png` shows average
values of the two numbers as a function of level.
:numref:`dm-19536-hist-tile-pix.png` shows distribution of the values for
specific level. :numref:`pixels-tiles-partitioning.png` shows distributions
for number of tiles per CCD (tile) and total area of those tiles.

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
of the important goals in defining schema for APDB would to eliminate the
need to access the files that were produced long time ago (older than 12
months) so that those files could be moved to slower, less expensive storage.

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

One of the early ideas was to try to keep number of I/O operations minimal
by not flushing the data from memory during the night, forcing the flush
and compacting data during the day. Implementing this cycle in this initial
test did not show any improvements in performance with compacted data
compared to a default setup when data was compacted less frequently. Our
intermediate conclusion was that Cassandra shows no significant performance
degradation with less compacted data so default compaction policy may work
sufficiently well. Forced compaction takes significant time and there is no
reason doing it without clear benefit.

Overall performance of this initial test was unexpectedly low, after running
for 30k visits (~1 month worth of data) average read time was at the level of
3 seconds per one CCD per visit, but average store was around 7 seconds. This
did not make a lot of sense as Cassandra performance for write operations was
supposed to be much better. Also during the test we observed many cases of
client-side timeouts that point to significant performance issues that need
to be understood.

For analyzing these performance issues we instrumented our Cassandra setup
with a monitoring tool that used Cassandra JMX interface (`DM-23604`_) to
extract monitoring metrics and dump it to a file which was later ingested
into InfluxDB and exposed to Grafana. Monitoring information was also
extracted from ``ap_proto`` log files as well and saved to the same InfluxDB
so we could correlate things happening on client and server side.

Analyzing monitoring data we quickly established that the reason for poor
performance in the initial test is an over-committing of the memory. Even
though JVM was configured to leave significant amount of RAM to other
processes there were some services (notably GPFS) which also needed
significant amount of RAM and that caused intensive swapping. Reducing
memory allocation for JVM allowed us to avoid swapping and improved
performance to more reasonable level.


Java Garbage Collection
-----------------------

Second round of tests (`DM-23881`_) started with reduced JVM memory
allocation (160 GB) which eliminated swapping but we still saw frequent
timeouts on client side. Monitoring showed that on server side there were
significant delays happening during garbage collection. Apparently default
garbage collection algorithm (PArNew+CMS) used by Cassandra is not optimal
for large memory systems. Switching to to a different algorithm (G1GC)
improved GC efficiency and fixed client-side timeouts. With this setup
the test was run for 180k visits (approximately 6 months). Write performance
is improved dramatically and database operations are now dominated by
reading time which grows approximately linearly with the visit number.
:numref:`dm-23881-select-fit-1.png` summarizes read performance for all
separate tables and their total. At 180k visits total read time is
approximately 10 seconds per visit (per CCD).


.. figure:: /_static/dm-23881-select-fit-1.png
   :name: dm-23881-select-fit-1.png
   :target: _static/dm-23881-select-fit-1.png

   Select execution time as a function of visit number, "obj_select_real"
   is time for DIAObject table, "src_select_real" is for DIASource,
   "fsrc_select_real" is for DIAForcedSource, and "select_real" is the sum
   of three times. Data for visits below 60k is not included in fits.




.. _Apache Cassandra: https://cassandra.apache.org/
.. _DMTN-113: https://dmtn-113.lsst.io/

.. _DM-19536: https://jira.lsstcorp.org/browse/DM-19536
.. _DM-20580: https://jira.lsstcorp.org/browse/DM-20580
.. _DM-23322: https://jira.lsstcorp.org/browse/DM-23322
.. _DM-23604: https://jira.lsstcorp.org/browse/DM-23604
.. _DM-23881: https://jira.lsstcorp.org/browse/DM-23881
.. _DM-24692: https://jira.lsstcorp.org/browse/DM-24692
.. _DM-25055: https://jira.lsstcorp.org/browse/DM-25055




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
