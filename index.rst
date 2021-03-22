
:tocdepth: 1

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

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
It will require about 2-3 terabytes of space in the object store per year, and about 30 gigabytes of space per year for the PostgreSQL index database.

Design Goals
============

Broadly speaking, the Alert Database will have three uses:

1. It will serve as an authoritative historical record, archiving all the alert data that the project has sent out through alert streams.
2. It will also permit science users to retrieve individual alert packets using an identifier that they got from somewhere else (for example, from a circular, or a query to the PPDB).
3. Finally, it will assist users who are developing software by providing large batches of alert data which can be used to generate a simulated stream, test online algorithms, and evaluate alert filtering techniques.

Each of these use cases has design consequences.

Design implications of being a historical record
------------------------------------------------

Because the Alert Database acts as a historical record, it must keep all data in perpetuity.
This amounts to a few terabytes per year (see "Data Sizing Calculations" below).

We must be able to back up the Alert Database to protect from data loss.

All writes to the Database from the Alert Production system must be fully stored before they are acknowledged.

The Database must always be available for writes while Alert Production is running so that it doesn't miss data.

The only operations that should be permitted are reads and writes; updates and deletes should not be possible in the normal course of events.
They might be necessary for corrective action if there has been a serious bug, but they certainly should not be exposed to ordinary users.

Bulk access to the historical archive does not need to be especially fast.

Design implications of being a queryable DB for science users
-------------------------------------------------------------

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

Design implications of being a resource for developers
------------------------------------------------------

Users who want to test algorithms, generate simulated streams, or evaluate filters will be interested in retrieving large batches of alert packets.
They will not need to retrieve these batches very often, and it's entirely acceptable for their requests to take a long time (even days) to fulfill.

Design Non-Goals
----------------

It's also important to discuss explicit non-goals.

The Alert Database does not need to be highly available for reads.
Some read downtime, like for maintenance or data compaction, is acceptable.

Data Sizing Calculations
========================

DMTN-102 :cite:`DMTN-102` provides estimates for the total volume of alerts received.
We expect a maximum of about 10 million alerts per night, each about 80 kilobytes.
LSE-163 :cite:`LSE-163` projects about 300 nights of observations per year.

Combining these figures, we project 80 gigabytes of alert data per night, and 2.4 terabytes per year.

The indexed metadata is a small fraction of that.
We propose to index by:

 - ``diaSourceId``, which is an 8-byte integer
 - ``diaObjectId``, which is an 8-byte integer
 - ``ssObjectId``, which is an 8-byte integer
 - ``healpixel``, which is an 8-byte integer
 - ``midPointTai``, which is an 8-byte floating point
 - ``filterName``, which is a string, limited to 16 bytes
 - ``programId``, which is a 4-byte integer

These total 60 bytes per alert.
These indexes point to string URIs which reference objects in the object store.
Supposing 40 bytes for the URIs, we have a 100 bytes per alert, so the index database grows at 100 megabytes per night, or about 30 gigabytes per year.

Existing Requirements
=====================

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

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
