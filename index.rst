:tocdepth: 1

.. sectnum::

Abstract
========

The Consolidated Database will hold information about all the images that have been taken.
It fulfills the requirements [#metadata-reqs]_ to archive and serve metadata about the images.
It also holds pre-computed per-exposure and per-visit summaries of the Engineering and Facility Database (EFD).
It will not contain provenance or processing metadata about image data products, but metrics resulting from data processing will be stored in it.

The Consolidated Database is required [#access-req]_ to be accessible via the IVOA TAP protocol.

.. [#metadata-reqs] LSE-61 §1.2.3 DMS-REQ-0068, §4.1.4 DMS-REQ-0309; DPDD §3.1 p. 10, §3.3 footnote 30 p. 14, §4.1 p. 35, §4.3 p. 44
.. [#access-req] LSE-61 §4.1.19 DMS-REQ-0078

This document specifies what the content of this database should be, how it should be managed, how it will be implemented, and how it might be extended.  A phased strategy for bringing it to production is also proposed.

Background
==========

Originally, the Consolidated Database was a way to efficiently administer a large number of independent database instances that were expected to be needed for services as well as processing.
By placing all of these instances on a single relational database management system (RDBMS) server, efficiencies of storage management and administration were expected.
Of course, centralizing so many services, some critical, creates requirements for high availability and high performance, but Oracle Real Applications Clusters (RAC) was expected to meet those needs.

Today, administration of service-local database instances is relatively simple, and the benefits of distributing these to minimize side-effects outweigh management issues.
As a result, we expect to deploy service-specific databases in Kubernetes, often of differing server technologies, whether for the Data Butler, TAP Schema, the Engineering and Facilities Database (EFD), the Alert Production Database, or Gafaelfawr.

One database instance that was to be part of the original Consolidated Database has not been fully defined, however.
That database contains raw image metadata for exposures and visits.
Since the raw data product of the Observatory and its Camera is images, this metadata is critical for both operational processes and science understanding.
This raw image metadata database has come to take on the name "Consolidated Database" as it potentially consolidates information from a variety of sources, including the full range from Summit systems to Data Release Production (DRP).


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

The source systems at the Summit, including the HeaderService and the EFD, will be summarized and merged into a Summit Visit Database that will include the previously-described Transformed EFD.
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
But joining with a SQL-based Consolidated Database that includes appropriate summaries of the EFD would be possible.

.. _Presto: https://prestodb.io
.. _Trino: https://trino.io


Implementation
==============

PostgreSQL relational databases will be used at the Summit and the USDF to implement the Consolidated Database.
This enables the use of standard SQL for querying and the layering of TAP for science user access.
While a column store or NoSQL document-oriented database could have some advantages, the use of separate normalized tables, potentially with materialized views joining them, can provide many of the same features.

The `Sasquatch`_ REST API will be used as the insertion interface between the source systems and the Consolidated database.
This allows inserted data to be transformed into Kafka messages that are then replicated from the Summit to the USDF.
Those Kafka messages will then trigger inserts or updates to the relational tables via a Kafka connector.

.. _Sasquatch: https://sqr-067.lsst.io

Note that InfluxDB will *not* be used as the back-end.
While the stream of exposures can conceptually be viewed as a time series, the labels of the exposures (exposure and visit ids) are not time-based and yet have high cardinality.

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
Note that QA metrics submitted to Sasquatch via the lsst.verify interface need to be distinguished between the real data and nightly/weekly test runs.

The transformation will occur by doing periodic batch computations on defined time intervals, computing results for all visits ending within the interval.
This allows for simple catch-up processing, recomputation at a different location (in particular at the USDF), and recomputation at a different time (e.g. if some channel summarization method were found to need adjustment).
Stream-based processors might have lower latency, but they would not be as amenable to these recovery processes.

A configuration item per channel would determine processing.
The processor needs to know all relevant time boundaries for the exposure (and therefore visit):
 * ``startIntegration``
 * ``startShutterOpen``/``endShutterOpen``/``startShutterClose``/``endShutterClose``
 * ``endReadout``

Not all channels need to be configured to be processed in every batch computation, allowing more-rapid updates of one or a few channels if need be.


Summit Visit Database
---------------------

The Summit Visit Database would include the Transformed EFD and additional annotations from observers obtained from the Exposure Log.
Metrics and results from the OCPS that are not part of the EFD and hence not transmitted via Kakfa can also be included at this stage.

Consolidated Database
---------------------

The Consolidated Database at the USDF would include DRP-computed data (astrometry, photometry, metrics) including the current VisitSummary datasets as well as further annotations from processing metadata.
This database would be replicated at the FrDF and UKDF for use during processing.


First-Look Analysis and Feedback Functionality
==============================================

The `FAFF group report`_ points out the need for a Visit database.
The Summit Visit Database and Consolidated Database described here are intended to fulfill that need.

.. _FAFF group report:  https://sitcomtn-037.lsst.io/#the-visit-database

The Summit Visit Database will be a subset of the Consolidated Database that contains information derived solely from Summit systems.
It is appropriate for querying by other Summit systems that require near-realtime access to exposure and visit metadata.
It is also an appropriate destination for exposure and visit metadata produced by Summit systems.

The contents of the two instances will not in general be identical, even for the subset of the Consolidated Database that matches the Summit Visit Database.
The Consolidated Database is designed to be an improved version of the Summit Visit Database, containing much more information, added seconds to years later.
It is possible that some values or entries from the Summit Visit Database will be modified or updated in the Consolidated Database.
But both instances are available to Summit observers and Commissioning staff.

The use of the Sasquatch REST API will allow Rapid Analysis at the Summit to publish results into the Summit Visit Database, making them readily available after processing.
It can also allow Camera diagnostics to be published to the same database.

Using the same database server at the Summit as the current exposure and narrative log tools allows joins with their human annotations.
Additional annotations relevant to processing rather than observing can be added to the exposure log at the USDF, which will be part of the Consolidated Database.

The FAFF report describes a use case where Rapid Analysis produces a PSF FWHM which needs to be compared with the atmospheric seeing from DIMM data.
It says that these values "originate from different sources, and are not precisely synced in time."
But with the Transformed EFD component of the Summit Visit Database, the values from different sources would be combined in one database, and potentially even one table, and would, by definition, be synchronized in time (the time span of the exposure or visit).

The `FAFF requirements spreadsheet`_ (FIG-REQ-024, row 77) suggests that users should be able to add additional fields to the Visit Database during the night.
While schema changes can be handled by adding tables or columns to existing tables, such modifications would generally go through a change control process and not be highly dynamic.
Among other reasons, removing columns or tables that might be relied upon by diverse and even non-Rubin downstream systems is difficult, so additions should be well-considered.
It should be possible for systems to send arbitrary metrics and derived values to Sasquatch, however, with the only prerequisite being defining a topic name and schema for its values.
At a later date, those values can be summarized into the Consolidated Database.

.. _FAFF requirements spreadsheet: https://docs.google.com/spreadsheets/d/1bs0NNjYDLzWWENqnd_qtWWkj04gDML7ohQYnxGSwUZc/edit#gid=0


Phasing
=======

We will start by loading the information from the Header Service into an initial Summit Visit Database co-located with the exposure log and narrative log.
While there are limitations to the Header Service data, it is already summarized at the appropriate level and is available soon after the exposure readout begins.
The Summit Visit Database will be replicated to the USDF along with the existing exposure log and narrative log replicas.
This task is being worked on by DM and might take 2 weeks for KTL and WOM.

In parallel, the connection from Sasquatch/Kafka to Postgres will be implemented.
A `Kafka connector`_ already exists to do this, including the ability to do idempotent writes, so hopefully only configuration and deployment work will be needed.
This will allow Rapid Analysis results and Camera metadata to be published into the Summit Visit Database.
This task would benefit from SQuaRE expertise and might take 3 weeks for AF.

.. _Kafka connector: https://docs.confluent.io/kafka-connectors/jdbc/current/sink-connector/overview.html

After deployment of the connector at the USDF, Prompt Processing, "10 AM Processing", and DRP processing metrics can be incorporated into the Consolidated Database via this mechanism.
Each publishing client team would need to modify their systems to publish appropriately.

Next, the EFD transformation batch computation will be developed.
This is also the component that will extract visit definitions from the OODS Butler repo and use them to produce the visit table(s).
There are multiple sub-components that can be worked on here:
 * the summarization framework
 * the Butler integration to get exposure and visit timespans
 * the various summarization algorithms
 * the per-channel configuration selecting an algorithm and setting any parameters
The whole task might take 2-3 engineer-months to complete.

Meanwhile, the Middleware Team will work to allow Consolidated Database output lists to be used in graph generation.

.. Make in-text citations with: :cite:`bibkey`.
.. Uncomment to use citations
.. .. rubric:: References
.. 
.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
