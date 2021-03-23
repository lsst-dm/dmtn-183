:tocdepth: 1

.. sectnum::

.. note::

   **This technote is not yet published.**

   Design document for a database of all alerts sent by the Rubin observatory.

Summary
=======

This document proposes a technical design for a database of alert packets.

At a high level, the proposal is to store a narrow subset of the fields of alert packets in a PostgreSQL database, termed the "index."
The complete, raw alert packets can be stored in an object store; the PostgreSQL index stores string references to object IDs.
Queries work by consulting the index to find object IDs that match query predicates, then retrieving the underlying alerts from the bulk storage, and providing them back to the user.

This separation allows fast random access, and slow bulk access.
It should be straightforward to build and administer, and it should be cost effective.
It will require about 2-3 terabytes of space in the object store per year, and about 40 gigabytes of space per year for the PostgreSQL index database.

Design Inputs
=============

Use cases
---------

Broadly speaking, the Alert Database will have three uses:

1. It will serve as an authoritative historical record, archiving all the alert data that the project has sent out through alert streams.
2. It will also permit science users to retrieve individual alert packets using an identifier that they got from somewhere else (for example, from a circular, or a query to the PPDB).
3. Finally, it will assist users who are developing software by providing large batches of alert data which can be used to generate a simulated stream, test online algorithms, and evaluate alert filtering techniques.

Each of these use cases has design consequences.

Use cases: a historical record
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Because the Alert Database acts as a historical record, it must keep all data in perpetuity.
This amounts to a few terabytes per year (see "Data Sizing Calculations" below).

We must be able to back up the Alert Database to protect from data loss.

All writes to the Database from the Alert Production system must be fully stored before they are acknowledged.

The Database must always be available for writes while Alert Production is running so that it doesn't miss data.

The only operations that should be permitted are reads and writes; updates and deletes should not be possible in the normal course of events.
They might be necessary for corrective action if there has been a serious bug, but they certainly should not be exposed to ordinary users.

Bulk access to the historical archive does not need to be especially fast.

Use cases: queryable DB
^^^^^^^^^^^^^^^^^^^^^^^

The Alert Database needs to support "needle in a haystack" queries based on specific identifiers.
In particular:

 - ``get(DIASourceID)`` ought to retrieve the single alert packet associated with a specific DIASourceID.
 - ``get_for_diaobject(DIAObjectID)`` ought to retrieve all alert packets issued for a specific ``DIAObject``
 - ``get_for_ssobject(SSObject)`` ought to retrieve all alert packets issued for a specific ``SSObject``

These should generally respond within a few seconds, and should provide the full alert packet payloads in each case.
We do not expect to have many users of these APIs; we need to handle only a few requests per second.

We should also support a few "search" queries which ask for all alerts which match some broader predicates.
In particular:

 - ``cone_search(ra, dec, radius)`` ought to retrieve all alerts issued in a particular region of the sky
 - ``timerange_search(start, end)`` ought to retrieve all alerts issued in a particular time window

These can be expected to be somewhat slower than the needle-in-a-haystack queries.
Their execution time should be proportional to the size of the result set; searching in a narrow region should be fast while a large region should be slow.

We should also predict the need to support more search queries in the future.
Our set of filters should be modifiable.

In general, we should expect recent data to be queried more often than old data, so it may make sense to store it in a way that supports more rapid retrieval.

All of these queries need to be limited based on data rights, implying that it should use existing authentication frameworks.

Use cases: resource for developers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Users who want to test algorithms, generate simulated streams, or evaluate filters will be interested in retrieving large batches of alert packets.
They will not need to retrieve these batches very often, and it's entirely acceptable for their requests to take a long time (even days) to fulfill.

Data Sizing Calculations
------------------------

DMTN-102 :cite:`DMTN-102` provides estimates for the total volume of alerts received.
We expect a maximum of about 10 million alerts per night, each about 80 kilobytes.
LSE-163 :cite:`LSE-163` projects about 300 nights of observations per year.

Combining these figures, we project 80 gigabytes of alert data per night, and 2.4 terabytes per year.

The indexed metadata is a small fraction of that.
We propose to index by:

 - ``diaSourceId``, which is an 8-byte integer
 - ``diaObjectId``, which is an 8-byte integer
 - ``ssObjectId``, which is an 8-byte integer
 - ``ra``, which is an 8-byte floating point
 - ``decl``, which is an 8-byte floating point
 - ``midPointTai``, which is an 8-byte floating point
 - ``filterName``, which is a string, limited to 16 bytes
 - ``programId``, which is a 4-byte integer

These total 68 bytes per alert.
These indexes point to string URIs which reference objects in the object store.
Supposing 60 bytes for the URIs, we have a 128 bytes per alert, so the index database grows at 128 megabytes per night, or about 40 gigabytes per year.

Existing Requirements
---------------------

The database is referenced in a handful of existing project requiremnts documents:

 - OSS-REQ-0128 "Alerts" :cite:`LSE-30`:

     The Level 1 Data Products shall include the Alerts produced as part  of the nightly Alert Production.

 - OSS-REQ-0185 "Transient Alert Query" :cite:`LSE-30`:

     All published transient alerts, as well as all reprocessed historical alerts generated as part of a Data Release, shall be available for query.

 - DMS-REQ-0094 "Keep Historical Alert Archive" :cite:`LSE-61`:

     The DMS shall preserve and keep in an accessible state an alert archive with all issued alerts for a historical record and for false alert analysis.

 - DMS-PRTL-REQ-0033 "Queries on the Alerts Database" :cite:`LDM-554`:

     The Portal aspect shall provide a query interface to the Alert Database, allowing searches based on parameters which shall include, but may not be limited to, Alert ID, time of alert, position on the sky, filter, and alert characteristics.

 - DMS-PRTL-REQ-0048 "Alert Visualization" :cite:`LDM-554`:

     The Portal aspect shall provide for the users a "property sheet" for the contents of an alert packet including, but not necessarily limited to, the alert postage stamp image, the postage stamp time series, the photometric time series, the source and object information (e.g., position, brightness).

External API
============

This section lists the external APIs that are supported.
It uses Python pseudocode to describe the APIs.
These signatures should not be taken too literally; it's just a convenient language to convey the API.
The actual implementation may differ significantly, although the core functionality should be available.

The external API can be grouped into four categories which serve different use cases.

1. **Get**: these return a small result set.
   They filter down to a small number of alerts to support the "queryable DB" use case.
   These are the Get calls which are supported:

   .. code-block:: python

      def get_by_source_id(dia_source_id: int) -> AlertPacket

      def get_by_dia_object(dia_object_id: int) -> List[AlertPacket]

      def get_by_ss_object(ss_object_id: int) -> List[AlertPacket]

   These should all respond within a few seconds.

2. **Search**: these return a moderately large result set, possibly with thousands of alerts.
   They support the "queryable DB" use case as well.

   .. code-block:: python

      def search(q: Query) -> List[AlertPacket]

   A query is formed as a conjunction of constraints which are 'AND'-ed together:

   .. code-block:: py

      class Query:
          cone_search: Optional[ConeSearchConstraint]
          timerange: Optional[TimeRangeConstraint]
          filters: Optional[FilterConstraint]
          ss_objects: Optional[SSObjectConstraint]

      class ConeSearchConstraint:
          ra: float
          dec: float
          radius: float

      class TimeRangeConstraint
          start_tai: float
          end_tai: float

      class FilterConstraint:
          filters: List[str]

      class SSObjectConstraint:
          ss_objects: List[int]


   These should respond within about a minute for large queries, and within a few seconds for small ones.

   To avoid getting bogged down by serving an enormous result set, the search API will reject queries which it expects to return more than 100,000 alerts.


3. **Bulk Fetch**: this returns a very large number of alerts, possibly millions.
   It supports the "historical record" and "resources for developers" use cases.
   Because the result set is so large, it is delivered by populating a location on a shared file system; the user is alerted at an email address when the data is available.

   .. code-block:: py

      def fetch(start_jd: int, end_jd: int, destination: str, email: str) -> None

4. **Write**: this is accessible only to alert production infrastructure.

   .. code-block:: py

      def write(alerts: Iterable[AlertPacket]) -> None


Internal Implementation
=======================

The Alert Database has three components.

The **Bulk Storage** component holds the raw alert packet data, as it was delivered in alert production streams.
This is built on top of the project's S3-like object store system.

The **Index** component holds an indexed subset of alert packet fields, mapped to Bulk Storage URIs.
The Index allows for efficient lookups of alerts.

The **Frontend** component is the interface used to issue queries, and it hides the Bulk Storage and Index components from view for users.
It will probably be implemented with ``lsst.daf.Butler`` to manage interactions with the index for Get and Search queries, and custom code for Bulk Fetch and Write operations.

Queries work by first consulting the Index to get a list of matching Bulk Storage URIs.
Then, requests for the underlying alert packet data are issued to the Bulk Storage system in parallel, and the responses are streamed to the user.

Streaming the responses is important to avoid memory blowup when building a large result set.

Bulk Storage
------------

Alerts are stored in an S3-like object storage system.

Each alert packet corresponds to one object in the object store.

.. note::

   An alternative would be to combine many packets into a block in the object store, perhaps of about 100 alert packets.
   This might permit more efficient storage and retrieval.
   Storage might be more efficient because compression would be better when storing many alerts.
   Retrieval might be more efficient because it might tune the outgoing flows into a smaller number of TCP connections which get a chance to grow window sizes.

   But this would be more complex, so this design sticks to a simpler structure.

An object store is used because it is cheap, scales well to handle terabytes of data, and should support parallel retrieval reasonably well.
Object stores tend to have somewhat high latency for bulk access, but this is acceptable.

Index
-----

The Index is a Postgres table. The table's schema is this:

+----------------------------------------+----------------------------------------+
|Column                                  |Type                                    |
+========================================+========================================+
|objectURI                               |varchar(60)                             |
+----------------------------------------+----------------------------------------+
|diaSourceId                             |long                                    |
+----------------------------------------+----------------------------------------+
|diaObjectId                             |long                                    |
+----------------------------------------+----------------------------------------+
|ssObjectId                              |long                                    |
+----------------------------------------+----------------------------------------+
|sourceRA                                |double                                  |
+----------------------------------------+----------------------------------------+
|sourceDecl                              |double                                  |
+----------------------------------------+----------------------------------------+
|sourceRAErr                             |float                                   |
+----------------------------------------+----------------------------------------+
|sourceDeclErr                           |float                                   |
+----------------------------------------+----------------------------------------+
|midPointTAI                             |double                                  |
+----------------------------------------+----------------------------------------+
|filterName                              |char(16)                                |
+----------------------------------------+----------------------------------------+
|programId                               |int                                     |
+----------------------------------------+----------------------------------------+

All of these columns will get an index, except objectURI.

The Index uses Postgres because it is already in use by the project.
This will reduce development effort by allowing reuse of software, and also reduce the operational burden by standardizing on common components.

Frontend
--------

The frontend will be a Python package.
Users will have access to a cleaned-up API providing all of the non-write functions.

.. note::

   Lots more to write here; not sure how auth works?

   How does Butler work, really?


Concerns, challenges, open questions
====================================

- Parallelism of the client code's interaction with bulk storage could be tricky.
- Will the object store be fast enough to support the search query if it matches a lot of alerts?
- How will auth work?

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
