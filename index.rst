:tocdepth: 1

.. sectnum::

.. note::

   **This technote is a work-in-progress.**

Background
==========

Originally, the Consolidated Database was a way to efficiently administer a large number of independent database instances that were expected to be needed for services as well as processing.
By placing all of these instances on a single relational database management system (RDBMS) server, efficiencies of storage management and administration were expected.
Of course, centralizing so many services, some critical, creates requirements for high availability and high performance, but Oracle Real Applications Clusters (RAC) was expected to meet those needs.

Today, administration of service-local database instances is relatively simple, and the benefits of distributing these to minimize side-effects outweigh management issues.
As a result, we expect to deploy service-specific databases in Kubernetes, often of differing server technologies, whether for the Data Butler, TAP Schema, the Engineering and Facilities Database (EFD), the Alert Production Database, or Gafaelfawr.

One database instance that was to be part of the Consolidated Database has not been fully defined, however.
That database contains raw image metadata for exposures and visits.
Since the raw data product of the Observatory and its Camera is images, this metadata is critical for both operational processes and science understanding.
This raw image metadata database has come to take on the name "Consolidated Database" as it potentially consolidates information from a variety of sources, including the full range from Summit systems to Data Release Production (DRP).
This document proposes a specification for what the content of this database should be, how it should be managed, how it could be implemented, and how it might be extended.  A phased strategy for bringing it to production is also proposed.


Raw Image Metadata
==================

Raw image metadata includes information about the conditions under which the image was taken, such as timing, temperature and weather, filter, voltages, etc.
Raw versions of this information are produced by Summit systems.
Previously, the `Transformed EFD`_ has been defined as the result of a service that will derive appropriate metadata for an image and publish it along with image identifiers.
`A proposal`_ for how to do this transformation was drafted, but it is lacking at least one critical feature: relating the transformed data to raw images, rather than arbitrary time intervals.

.. _Transformed EFD: https://dmtn-050.lsst.io/#transformation
.. _A proposal: https://sqr-058.lsst.io/

Raw image metadata also includes information about the initial intended purpose of the image, which can indicate the kinds of processing that are relevant.
Some of this information comes from the Scheduler or is provided to the Camera Control System via an image type string that eventually is captured in a FITS header.

Comments by observers that are relevant to a particular image and other annotations by humans or automated systems should also be captured.
Some of this information is being captured in a "night log", whether implemented as a Confluence page or an "exposure log" system at the Summit.

Raw image metadata can also include metrics or other derived quantities resulting from processing that are nevertheless characteristic of the image, rather than of the processing itself.
This processing may occur at prompt (second to minute) timescales or at longer timescales ranging up to Data Release Production that may occur 10 years after the image was taken.

Note that there are many sources of metadata operating on a variety of latencies and that all metadata may be recalibrated, recomputed, adjusted, edited, and corrected over time.
This even applies to metadata that has been published as part of a Data Release; while we would generally apply any updates to the next year's Data Release, it is conceivable that the metadata might need to be "patched" for an existing DR.

All of this metadata is unified by its pertinence to a given image.
However, the Standard Science Visit is composed of two "snap" images with the same telescope pointing that are processed together as if they made up one single exposure of twice the length.
Accordingly, it makes sense to also summarize metadata by visit as well as raw image.
If it turns out that all visits are composed of a single image, as specified in the Alternate Science Visit definition, then these two tables collapse to one.
If other visit definitions (where combined image metadata makes sense, not just groupings of exposures for processing) end up being used, those could be stored in the same visit table or a parallel one, but this scenario seems unlikely at this point.
Visit definition for this purpose needs to be consistent across all systems that use the metadata.
For metadata about image groups that are not visits, :ref:`see below <general-dataset-metadata>` about general datasets.

In addition, we often process images in a detector-parallel fashion.
As each detector's image can be handled independently, and the available detectors could potentially change from image to image, having the ability to store metrics and metadata at the detector level is important.

This gives us three conceptual tables: one for detector-exposures, one for exposures, and one for visits, again with the latter two collapsing into one if all visits are single-exposure.
It's possible that a detector-visit table would be needed in the Standard Science Visit two-snap case.
Of course these tables are replicated for each instrument: LATISS, LSSTComCam, and LSSTCam.

Other information derived from raw images, including single-frame Source catalogs, mask planes, etc. are not metrics or metadata about an image or visit and so do not fall into the domain of the Consolidated Database.

Similarly, while metadata about calibration images is part of the Consolidated Database, master calibration images, linearizer models, and other calibration outputs are not.


Uses
====

Raw image metadata is used to report to humans, to feed back to Summit systems such as the Scheduler, and to filter images for processing.

In particular, it is of significant use for Campaign Definition.  In that regard, extending the concept to apply to metadata for other datasets such as catalogs may also be useful.

Reporting and Feedback
----------------------

All metadata from SAL events is already available as part of the EFD.
It is also available to other Commandable SAL Components (CSCs).
Some of this information is tagged with relevant exposure or visit identifiers, but much is not.
The Transformed EFD is meant to provide tagging where it is not already present.

Quality metrics derived from Prompt Processing are similarly published to the EFD.
These will always be tagged with exposure and visit identifiers.

These metrics need to be summarized and correlated at the Summit for the Scheduler, for observers, and also for Survey progress monitoring.

Campaign Definition
-------------------

The datasets used as inputs to pipelines will often need to be filtered and sometimes grouped based on properties or metrics that have been determined or computed previously.
Common examples are exclusion of images that do not meet quality metric thresholds; inclusion of images that belong to a particular science program; selection of images that meet the criteria for a co-add; or pairing of intra-focal and extra-focal detector images (within a visit or across visits) for wavefront processing.
The ultimate source of the image selection information could be a header, a Parquet table, or even something external to Science Pipelines.
This filtering/grouping may be specified at the Campaign Definition level, but ultimately the pipeline execution (graph generation) needs either to have this capability or to be able to incorporate its results.

Processing
----------

Currently "visit summary tables" are prepared during Data Release processing.
This information should be stored in the Consolidated Database.

It might make sense to retrieve visit summary data from the Consolidated Database for use in downstream pipelines, but the pipeline Middleware has no provisions at present for obtaining datasets from a non-file data source.
File exports from the Consolidated Database seem like a better way to retrieve this data, at least in the short term, even though it may be inefficient to scan through them and require more code to select the desired rows and columns.
By using file exports, there is no question of synchronization of database inserts/updates and retrievals, provenance is simplified, and scalability is assured.

External Services
-----------------

The IVOA defines two relevant data models: `Observation Data Model Core Components`_ (ObsCore), which is combined with `Table Access Protocol`_ (TAP) to form ObsTAP, describing observations that have occurred, and `Observation Locator Table Access Protocol`_ (ObsLocTAP), describing especially observations that are projected to occur in the future.
We need to serve observation data according to both of these models.

.. _Observation Data Model Core Components: https://www.ivoa.net/documents/ObsCore/20170509/index.html
.. _Table Access Protocol: https://www.ivoa.net/documents/TAP/20190927/index.html
.. _Observation Locator Table Access Protocol: https://www.ivoa.net/documents/ObsLocTAP/20210724/index.html

While these are conceived of in the IVOA documents as separate, but linked, databases, there is the potential to merge them into a single database.
However operational concerns (including frequent updates by the scheduler and maintaining a wall between public and data-rights-only information) make it fairly clear that these should be distinct.

For ObsCore, we do not need to expose Butler component datasets in the metadata model.
They can instead be exposed via IVOA DataLink services.

In addition to ObsCore, there is also the `CAOM2 data model`_ that is desirable to support as a *de facto* standard for released data products.

.. _CAOM2 data model: http://www.opencadc.org/caom2/

The Consolidated Database schema needs to be mappable to both ObsCore and CAOM2.


Architecture
============

For the ObsLocTAP service, which is specialized and distinct from other uses, a separate Summit database instance will be used.
While the information content derives from the Scheduler, it appears that the Exposure Log service already compiles this information, so it may be a more suitable basis.
The public TAP front-end for this database could be located in the cloud; it does not need to be Summit-resident.

While conceptually a single globally-accessible image metadata database could be considered desirable, resilience and scalability require multiple, distributed, communicating database instances.
In such a situation, the `CAP theorem`_ says that building such a system in a partition-tolerant and highly-available manner means that only eventual consistency can be enforced.

.. _CAP theorem: https://en.wikipedia.org/wiki/CAP_theorem

The Data Release needs are slightly different in that they are almost entirely read-only, with very rare additions.
Joining with the other Data Release tables in systems like Qserv is required.
This is better handled by using a snapshot of a subset of the live database rather than attempting to connect the live database directly.
(Note that this could still be patched or updated by taking an appropriate snapshot of the new version.)

For testing purposes, small databases will need to be instantiated, loaded, and removed.

In all cases, the database may need to be updated as different sources provide information.
At the Summit, replacement of data values seems appropriate.
In the Data Release, maintaining history would be needed.
At least some parts of the Data Release database would thus be bitemporal: the original raw numbers would always be available, and at least one revision of calibrated EFD summary data or other metadata would be available per DR.
If this is difficult to store in the Data Release database itself, it may make sense to store the raw numbers and any previous revisions in an alternate, less readily-accessible form such as a change log or backup.

Metadata is likely to contain wide fact tables with relatively limited dimensionality.
There will be many, many columns of information for each image or visit, often with only a unique image/visit identifier as the primary key.

I propose to have a hierarchical merge tree of databases.
See :ref:`the diagram <fig-consolidation-of-databases>`.

.. figure:: /_static/consolidation-of-databases.png
   :name: fig-consolidation-of-databases
   :target: ../_static/consolidation-of-databases.pdf

   "Merge tree" of databases.

The source systems at the Summit, including the EFD, will be summarized and merged into a Summit Visit Database that will include the previously-described Transformed EFD.
The same summaries will be transmitted to the US Data Facility (USDF) where they will be included in the Consolidated Database, which will also merge information from offline sources such as Parquet files.
Neither system will be the ultimate source of truth; they will be derived databases (too simple to be termed marts or warehouses) subject to update and correction.
The Data Access Centers serve read-only replicas of prompt-oriented column subsets of the Consolidated Database in conjunction with other Prompt data products as well as read-only snapshots of Data Release-relevant subsets (in particular, such subsets only include rows for visits and exposures that are part of the DR).
The branches of this flow are one-way; no database communicates "upstream".

To isolate implementation details from the users, interposing a REST API for updates in front of the low-level database implementation is desirable.
(Such an API could also support queries, although having that either as an extra layer below TAP or a parallel interface along side TAP seems undesirable.)


Butler
======

We currently have one database that tracks information about all datasets used for processing: the Butler Registry.
It would therefore seem reasonable to implement the Consolidated Database by extending that Registry database.

There are several concerns, however:

#. The schema may be more malleable than has previously been desired for the Butler Registry, with updates as new metrics are conceived, bitemporality, and instrument-specific columns.
#. We are currently planning to have different Butler repos with different Registry contents at each processing location.  The Consolidated Database, on the other hand, should be the same at each location.
#. By extending the Registry beyond ingestion requirements, to include frequent updates asynchronous from dataset creation, it may add substantial complexity to the Butler.
#. It may not be feasible to provide ObsCore and CAOM2 as views on the Registry; materialized derived tables may be necessary (e.g. to handle different requirements for specifying the geometry of regions).
#. It is infeasible to insist that all information about a dataset that might potentially be used to select or exclude it from a processing graph be preloaded into the Registry in advance of knowing that it is needed for generating a particular graph.
   Some information may come from external systems and may only be known at graph generation time.

If a way can be found to provide for Butler Registry-based graph generation while at the same time keeping the Consolidated Database outside the Butler domain, the overall system might be simplified and made more resilient.

One mechanism for doing so might be to enable the Butler graph generation code to incorporate lists of detector-exposures, exposures, detector-visits, or visits derived from the Consolidated Database.
For some uses, lists of groups of images might be useful.
These lists could be explicit lists of primary key identifiers, or, if very large, could be implemented as boolean bit-columns; they could manifest as TAGGED collections in the Butler Registry.
The lists would be presented to the Butler at graph generation time, not long in advance, but they could be persistent afterwards for provenance purposes.
As long as WHERE clause conditions combining Registry-only columns and Consolidated Database-only columns are unnecessary (which seems likely, as the Consolidated Database should generally be a superset of the Registry), this should be adequate for filtering.
By presenting a single, relatively narrow interface, the hope is that the graph generation code would require only limited changes.
At the same time, the flexibility of data sources and filtering mechanisms available to the list generation tools is maximized.
This is similar to what was proposed in `DMTN-181 <https://dmtn-181.lsst.io/>`_ as part of Campaign Management.

Another alternative would be to build a more general join engine into graph generation that can perform queries across multiple data sources.
While this may be overkill for small-scale usage of the Middleware, an engine like `Presto`_/`Trino`_ could allow federation of a wide variety of sources while operating at LSST 10-year survey scales.
This could avoid multiple ingests into the Butler Registry.
A potentially significant problem with this option is that InfluxDB, the primary repository of the EFD and lsst.verify-based metrics, does not have a Presto/Trino connector.
But linking with a SQL-based Consolidated Database would be possible.

.. _Presto: https://prestodb.io
.. _Trino: https://trino.io


Implementation
==============

Database 
--------
The Chile DevOps supported postgress instance will be used for this database. 
Schema is defined in the schema repository allowing us to generate the SQL from the description in the repo. 
Currently on 'u/ktl/add-summit-schema' branch of `SDM Schema repo <https://github.com/lsst/sdm_schemas/tree/main/yml>`


Non relational thoughts
+++++++++++++++++++++++

Given that indexing of most metadata is unlikely to produce selection ratios that are sufficiently low to offset the expense of seeks, a column store that can be rapidly scanned to select images or other datasets of interest seems like the most appropriate storage mechanism.
`Apache Cassandra`_ might be appropriate, as it is already in use for the APDB and has good scalability and distribution capabilities.
In particular, it is conceivable to have a single distributed Cassandra that would include the Summit and the Data Facilities.
Cassandra also provides the ability to add columns and column families more easily than a relational database.

.. _Apache Cassandra: https://cassandra.apache.org/_/index.html

Linking Cassandra with TAP might be difficult, however.
In addition, the Consolidated Database for raw images is likely to have only 5 million or so rows at the visit level, 10 million or so rows at the exposure level, 2 billion or so at the detector-exposure level.
Even with 1000 columns, this is only a few terabytes at most.
So an RDBMS implementation with out-of-the-box SQL/ADQL and TAP seems possible, if it can be made to scale adequately.

`MongoDB`_ offers another possibility with a very flexible schema, although its document orientation may not be ideal.
It does offer a number of index types that might be suitable, however, including a "`2dsphere`_" type that could be used in addition to HTM/HEALPix indexing.

.. _MongoDB: https://www.mongodb.com
.. _2dsphere: https://www.mongodb.com/docs/manual/core/2dsphere/

ConsDB Service
--------------

Initially A simple service is required this will:
- be deployed using Phalanx
- not, initially at least, be a CSC
- listen to a limited number of CSC events around image capture
- based on those events query the EFD to get information
- write the gather information to the tables such as exposure log in the ConsDB
- update appropriate table in ConsDB such as ObsLocTAP table to indicate an observation was made or not

This will run in async and multi-threaded mode. Updates for example are not a high priority and should not delay the exposure entry creation.  Testing can use SQLLite implying we use SQL Alchemy (1.4 or higher). 

We will commence work on this April 7 2023 in `cdb_service <https://github.com/lsst-dm/cdb_service>` repo. 

There are further considerations below. 

Phasing
-------

A phased implementation could start with the urgently-needed Summit Visit Database, loading it with the contents of the Transformed EFD.
If existing `TICK stack`_ tools are insufficient for doing this transformation, a modestly generalized framework based on the `Header Service`_ could do the summarization.
Postgres would be the initial backend.

.. _TICK stack: https://www.influxdata.com/time-series-platform/
.. _Header Service: https://dmtn-058.lsst.io

The next phase would be to replicate this to the USDF.
Following that, the visit summary tables from DRP could be loaded.
Additional data sources would be added as needed and as available.

In parallel, the Middleware Team would work to allow Consolidated Database output lists to be used in graph generation.

An evaluation of database implementations would also be done at this time to determine if Postgres should be replaced by a different backend.

.. _general_dataset_metadata:

General Dataset Metadata
------------------------

Once a raw image metadata database is defined, it makes sense to ask whether it should be extended to also include other types of images, such as co-adds, or even other types of file datasets, such as catalogs.
This is TBD and dependent on use cases.
Among other concerns, scalability to the much larger space of all datasets, increased dimensionality and complexity of dataset identification, and complex relationships between datasets would seem to make this a non-trivial extension that requires further research.


Transformed EFD
---------------

Columns in the Transformed EFD could potentially include all of the channels available in the EFD itself.
Specifically desired columns mentioned in `LSE-61 DMS-REQ-0068`_ include:

* Time of exposure start and end, referenced to TAI, and DUT1
* Site metadata (site seeing, transparency, weather, observatory location)
* Telescope metadata (telescope pointing, active optics state, environmental state)
* Camera metadata (shutter trajectory, wavefront sensors, environmental state)
* Program metadata (identifier for main survey, deep drilling, etc.)
* Scheduler metadata (visitID, intended number of exposures in the visit)

.. _LSE-61 DMS-REQ-0068: https://lse-61.lsst.io/LSE-61.pdf#page=18

Basic information is already placed in the image header at exposure (boresight, exposure time, filter).
Other information needs to be summarized from EFD information during an exposure/visit (DIMM seeing, temps, weather).

Only some metrics are composable from exposure to visit (i.e. the visit values are derivable directly from the exposure values for a two-exposure visit).
Others need to be computed separately for exposures and visits.

For channels with infrequent sampling, interpolation between points outside the exposure interval may be necessary.
The interpolation method may change over time.

For other channels that report raw values, a lookup table or other transformation may be needed to calibrate the data.
This table may of course change over time.

Some channels are expected to be computed by Prompt Processing: astrometry, PSF, zeropoint, background, and QA metrics.
Note that QA metrics submitted to `SQuaSH`_/`Sasquatch`_ via the lsst.verify interface need to be distinguished between the real data and nightly/weekly test runs.

.. _SQuaSH: https://sqr-009.lsst.io
.. _Sasquatch: https://sqr-067.lsst.io

The transformation and loading into the Summit Visit Database could occur by pulling from Kafka or InfluxDB.
A plugin per channel would determine processing and could potentially be implemented in Kapacitor or InfluxDB's Flux language or a Kafka-level stream processor.
The processor needs to know all relevant time boundaries for the exposure (and therefore visit):
 * ``startIntegration``
 * ``startShutterOpen``/``endShutterOpen``/``startShutterClose``/``endShutterClose``
 * ``endReadout``

Summit Visit Database
---------------------

The Summit Visit Database would include the Transformed EFD and additional annotations from observers obtained from the Exposure Log.
(It's not clear whether annotations from the Narrative Log are suitable, and that might be merged with the Exposure Log anyway.)
Metrics and results from the OCPS that are not part of the EFD and hence not transmitted via Kakfa can also be included at this stage.

Consolidated Database
---------------------

The Consolidated Database at the USDF would include DRP-computed data (astrometry, photometry, metrics) including the current VisitSummary datasets as well as further annotations from processing metadata.
This database would be replicated at the FrDF and UKDF for use during processing.

.. Make in-text citations with: :cite:`bibkey`.
.. Uncomment to use citations
.. .. rubric:: References
.. 
.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
