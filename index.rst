:tocdepth: 1

.. sectnum::

.. note::

   **This technote is not yet published.**

   Design document for a database of all alerts sent by the Rubin observatory.

Summary
=======

This document proposes a technical design for a database of alert packets.

At a high level, the proposal is to store raw alert packets, compressed with gzip, in an object store.
All packets can be retrieved by ID.
Queries which search the alert database can be performed by querying the PPDB to get IDs of alert packets, and then requesting the packets by ID from the object store.

Alert packets are prefixed with an identifer for the Avro schema that was used to encode them, and those schemas are also stored in the object store by ID.

In addition, to facilitate bulk access, we may store single-file archives of all alerts published each night.

This system should be straightforward to build and administer, and it should be cost effective.
It will require about 120 terabytes of space in the object store per year.

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
This amounts to about 240 terabytes per year (see "Data Sizing Calculations" below).

To be a good historical record, we need to take data durability seriously.
All writes to the Database from the Alert Production system must be fully stored before they are acknowledged, and we must be able to back up the Alert Database to protect from data loss.

The only operations that should be permitted are reads and writes; updates and deletes should not be possible in the normal course of events.
They might be necessary for corrective action if there has been a serious bug, but they certainly should not be exposed to ordinary users.

Use cases: queryable DB
^^^^^^^^^^^^^^^^^^^^^^^

The Alert Database needs to support "needle in a haystack" queries based on specific a single alert identifier.
In particular, ``get(alertId)`` ought to retrieve the single alert packet associated with a specific ``alertId``.

This should generally respond within a few seconds.

In general, we can expect recent data to be queried more often than old data for this purpose, but everything needs to be available.

Use cases: resource for developers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Users who want to test algorithms, generate simulated streams, or evaluate filters will be interested in retrieving large batches of alert packets.
They will not need to retrieve these batches very often, and it's entirely acceptable for their requests to take a long time (even days) to fulfill.

Data Sizing Calculations
------------------------

DMTN-102 :cite:`DMTN-102` provides estimates for the total volume of alerts received.
We expect a maximum of about 10 million alerts per night, with at most 80 kilobytes of data per packet.
LSE-163 :cite:`LSE-163` projects about 300 nights of observations per year.

Combining these figures, we project 800 gigabytes of alert data per night, and 240 terabytes per year of alert packet data.

Data will be compressed before storing it.
Based on experiments performed with `sample alert data <https://github.com/lsst-dm/sample_alert_info/>`__, gzip compression is likely to reduce the alert payload size by about 50%, lowering the total to about 120 terabytes per year.

Schema documents also need to be stored, but their total size will be negligible.
A single version of the schema is several hundred kilobytes.
We expect the schema to change very rarely, less than 10 times per year, so the total is likely to be well under 1 gigabyte per year.

Existing Requirements
---------------------

The database is referenced in a handful of existing project requirements documents:

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

Proposed Implementation
=======================

We can satisfy these design inputs by storing serialized Avro alert data (the same bytes sent via Kafka to brokers) in a S3-like object store, indexed by a unique alert ID.
Each alert packet corresponds to one object in the object store.

.. note::

   An alternative would be to combine many packets into a block in the object store, perhaps of about 100 alert packets.
   This might permit more efficient storage.
   Storage might be more efficient because compression would be better when storing many alerts.
   In informal experiments with simulated alert data, this requires about 20-60% less space to store.

   But this would be more complex, and make writing more difficult, as writes need to append to existing data which would require coordination between writers.
   It would also make reading more complex; a separate index would need to be maintained which translates alert packet IDs into an identifier for the block containing the alert.
   In light of these complexities, this design sticks to a simpler structure.

An object store is used because it scales well to handle many terabytes of data, and should support parallel reads and writes well.

Object Storage Scheme
---------------------

Objects will be stored under a versioned prefix, followed by the alert ID.
The versioned prefix describes the archival storage hierarchy so that it may be changed in the future.

Two types of objects will be stored: alerts and schemas:

+------------------------------------------------------------------+------------------------------+
| Key                                                              | Value                        |
+==================================================================+==============================+
| ``/alert_archive/v1/alerts/<alert_id>.avro.gz``                  | Serialized alert, in         |
|                                                                  | `Confluent Wire Format`_,    |
|                                                                  | then gzipped.                |
+------------------------------------------------------------------+------------------------------+
| ``/alert_archive/v1/schemas/<schema_id_hex>/schema.json``        | Avro schema JSON document    |
+------------------------------------------------------------------+------------------------------+

The serialized alert value is an Avro-encoded alert packet, in Confluent Wire Format, compressed with ``gzip``.
The Confluent Wire Format uses a magic byte, followed by a 4-byte schema ID, followed immediately by binary-encoded Avro data.
This entire package is compressed with ``gzip`` to save bytes at the cost of a little CPU time when reading and writing data.
Based on rudimentary experiments, this is expected to reduce storage requirements by about 50%.

In the Alert Stream, we expect consumers to fetch the schema document for an alert from a Confluent Schema Registry instance.
To avoid a dependency upon a running Confluent Schema Registry for archive operations, we should store the schema document in the alert archive, indexed by its schema ID.

Since the schema ID is a 4-byte sequence, but object keys are ASCII text, we use a hex encoding of the schema ID.

The schema document that is stored should be a single Avro ``record`` which describes the alert packet.
Referenced subschemas should be transcluded into the document, and it should be stored in Avro's `Parsing Canonical Form`_ format.

.. _Confluent Wire Format: https://docs.confluent.io/platform/current/schema-registry/serdes-develop/index.html#wire-format
.. _Parsing Canonical Form: http://avro.apache.org/docs/current/spec.html#Parsing+Canonical+Form+for+Schemas

Writing data
------------

When the alert production pipeline has computed a new alert packet, it should be careful to write it to the alert archive before publishing to a Kafka topic, to ensure that it gets archived. The overall flow should be:

 1. Compute the alert packet payload, including generating a unique alert ID which should eventually appear in the PPDB.
 2. Write the alert packet to the object store, using the alert ID as a key.
 3. Publish the alert packet to the Kafka topic that serves data to community brokers.

Reading data
------------

To read individual alert data, users access the backing alert packets through the butler, which should wrap up the object storage and provide access by alert ID.

This satisfies each of the three use cases:
 - As a **historical record**: By writing to the object store first, we can be sure that all published alerts are recorded.
   In case of Kafka downtime, we may store _more_ alerts than were recorded, but this is acceptable.

 - As a **queryable DB**: By querying the PPDB, users can search alerts by any of their fields or attributes, albeit with a one-day delay. Once they have alert IDs, they can get all underlying packets.

Alert Identifier
----------------

We need an identifier which is unique across all alerts which can be used as the key for the object store.
We can use ``alertId`` for this purpose, as defined in the PPDB.

Optional: Providing bulk access
-------------------------------

We may wish to provide bulk access to data in large chunks, like single files of all the alerts for a single night's observations.
We plan to see if there is suitable demand for this feature to justify adding it; it would approximately double the total storage requirements of the archive.

All observing is complete for a night, all the alerts that were succesfully published that night could be combined into a single Avro Object Container file, stored on an archival filesystem.
The set of published alerts can be identified by consuming from the Kafka topic.

To read bulk alert data, users can request the Avro Object Container files for particular nights.
We have not identified a particular protocol for those requests.

Limitations
===========

This design does not provide any sort of complex querying logic for data which has been stored since the last PPDB update.
Since the PPDB is updated daily, this means that the last 24 hours of data will not be indexed for complex queries.
This is acceptable, though, since the querying features of the alert database are not intended to support real-time online use cases.

Possible interaction with Alert Filtering Service
=================================================

One possible design of an alert filtering service would be to publish alert packet IDs with a small batch of useful information about the alert :cite:`DMTN-165`.
Consumers of that publication feed could decide to retrieve the full alert packet from the alert database if that small batch of useful information passed their filters.
In order to protect the object store backend and fairly use network resources, we could put a rate-limiting proxy in front of the object store.


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
