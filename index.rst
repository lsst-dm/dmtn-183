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
Queries which search the alert database can be performed by querying the PPDB to get IDs of alert packets, and then requesting the packets by ID from an HTTP frontend to the object store.
A lightweight Python client library will simplify this access.

Alert packets are prefixed with an identifer for the Avro schema that was used to encode them, and those schemas are also stored in the object store by ID.

In addition, to facilitate bulk access, we may store single-file archives of all alerts published each night.

This system should be straightforward to build and administer, and it should be cost effective.
It will require about 120 terabytes of space in the object store per year.

Design Inputs
=============

Use cases
---------

Broadly speaking, the Alert Database will have two primary uses:

1. It will serve as an authoritative historical record, archiving all the alert data that the project has sent out through alert streams.
2. It will also permit science users to retrieve individual alert packets using an identifier that they got from somewhere else (for example, from a circular, or a query to the PPDB).

In addition, we aim to be flexible enough to support two possible additional use cases, without immediately committing to their development:

3. We may wish to provide bulk access to large piles of alert data (for example, several entire nights' alerts) for development and training of algorithms which process alert data.
4. We may wish to act as the low-level backend which supports an alert filtering service.

Use cases: a historical record
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Because the Alert Database acts as a historical record, it must keep all data in perpetuity.
This amounts to about 120 terabytes per year (see "Data Sizing Calculations" below).

The only operations that should be permitted are reads and writes; updates and deletes should not be possible in the normal course of events.
They might be necessary for corrective action if there has been a serious bug, but they certainly should not be exposed to ordinary users.

.. _queryable DB:

Use cases: queryable DB
^^^^^^^^^^^^^^^^^^^^^^^

The Alert Database needs to support "needle in a haystack" queries based on specific a single alert identifier.
In particular, ``get(alertId)`` ought to retrieve the single alert packet associated with a specific ``alertId``.

This should generally respond within a few seconds.

In general, we can expect recent data to be queried more often than old data for this purpose, but everything needs to be available.

Possible use case: bulk access
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Users who want to test algorithms, generate simulated streams, or evaluate filters will be interested in retrieving large batches of alert packets.
They will not need to retrieve these batches very often, and it's entirely acceptable for their requests to take a long time (even days) to fulfill.

We plan to see if there is suitable demand for this feature to justify adding it; supporting it might double our storage requirements.

Possible use case: alert filtering backend
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

DMTN-165 :cite:`DMTN-165` proposes an alert filtering service which consists of a stream of alert packet IDs and a small bit of associated data.
Consumers of that stream could decide to retrieve the full alert packet if that small bit of data passes their filters.

Fast retrieval of alert packets by ID would be required for this to work.
This is equivalent to the "`queryable DB_`" use case above, but with an additional latency requirement: alerts would need to be retrievable as soon as they are published to the filtering service.

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
We expect the schema to change very rarely, less than 10 times per year, so the total size of schema documents is likely to be well under 1 gigabyte per year.

Existing Requirements
---------------------

The database is referenced in a handful of existing project requirements documents:

 - OSS-REQ-0128 "Alerts" :cite:`LSE-30`:

     The Level 1 Data Products shall include the Alerts produced as part of the nightly Alert Production.

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
   In informal experiments with simulated alert data, this requires about 5% less space to store than compressing each alert packet separately.

   But this would be more complex, and make writing more difficult, as writes need to append to existing data which would require coordination between writers.
   It would also make reading more complex; a separate index would need to be maintained which translates alert packet IDs into an identifier for the block containing the alert.
   In light of these complexities, this design sticks to a simpler structure.

An object store is used because it scales well to handle many terabytes of data, and should support parallel reads and writes well.

Writes to the object store are handled by a Kafka consumer which copies alert packets from the main Kafka topic into the alert database.

Reads are servered with a lightweight HTTP service and accompanying client library which allow retrieval by alert ID of any packet.

This satisfies each of the primary use cases:
 - As a **historical record**: By consuming from the actual written Kafka stream, we can be sure that we are storing alert packets as they were actually sent.
   All alerts are stored in perpetuity in the database, forming a historical record.
 - As a **queryable DB**: By querying the PPDB, users can search alerts by any of their fields or attributes, albeit with a one-day delay.
   Once they have alert IDs, they can get all underlying packets.

Object storage layout
---------------------

Objects will be stored under a versioned prefix, followed by the alert ID.
The versioned prefix describes the archival storage hierarchy so that it may be changed in the future.

Two types of objects will be stored: alerts and schemas:

+------------------------------------------------------------------+------------------------------+
| Key                                                              | Value                        |
+==================================================================+==============================+
| ``/alert_archive/v1/alerts/<alertId>.avro.gz``                   | Serialized alert, in         |
|                                                                  | `Confluent Wire Format`_,    |
|                                                                  | then gzipped.                |
+------------------------------------------------------------------+------------------------------+
| ``/alert_archive/v1/schemas/<schema_id_hex>.json``               | Avro schema JSON document    |
+------------------------------------------------------------------+------------------------------+

Alert format
^^^^^^^^^^^^

Our key needs to be an identifier which is unique across all alerts.
We can use ``alertId`` for this purpose, as defined in the PPDB.

The serialized alert value is an Avro-encoded alert packet, in Confluent Wire Format, compressed with ``gzip``.

The Confluent Wire Format uses a magic byte, followed by a 4-byte schema ID, followed immediately by binary-encoded Avro data.

This entire package is compressed with ``gzip`` to save bytes at the cost of a little CPU time when reading and writing data.
Based on rudimentary experiments, this is expected to reduce storage requirements by about 50%.

Schema format
^^^^^^^^^^^^^

In the Alert Stream, we expect consumers to fetch the schema document for an alert from a Confluent Schema Registry instance.
To avoid a dependency upon a running Confluent Schema Registry for archive operations, we should store the schema document in the alert archive, indexed by its schema ID.

Since the schema ID is a 4-byte sequence, but object keys are ASCII text, we use a hex encoding of the schema ID.

The schema document that is stored should be a single Avro ``record`` which describes the alert packet.
Referenced subschemas should be transcluded into the document, and it should be stored in Avro's `Parsing Canonical Form`_ format.

.. _Confluent Wire Format: https://docs.confluent.io/platform/current/schema-registry/serdes-develop/index.html#wire-format
.. _Parsing Canonical Form: http://avro.apache.org/docs/current/spec.html#Parsing+Canonical+Form+for+Schemas

Schema updates
--------------

When a new version of the alert schema is released, the new schema should be written into the alert archive.
This can be done before any alerts are published with the new schema.

Writing data
------------

When the alert production pipeline has computed a new alert packet, it will write it to a Kafka topic, broadcasting it to brokers.
We should implement and run a consumer of this Kafka topic which copies messages into the object store.

Running as a consumer of the Kafka topic adds several seconds of latency.
This is acceptable because none of the primary use cases for the database require tight latency bounds.

Reading data
------------

To read individual alert data, users access the backing alert packets via a Python client which makes HTTP requests to a simple authenticated service.
This HTTP service needs to support two API endpoints:

1. ``GET /v1/alerts/<alert_id>`` should retrieve the alert from the object store and return it without any modification, in its original Confluent Wire Format encoding.
2. ``GET /v1/schemas/<schema_id>`` should retrieve the schema from the object store and return it without modification, in its original JSON encoding.

The client library which wraps these API calls should provide three functions:

1. ``get_alert(alert_id)`` should return a ``dict`` of deserialized alert data for the given alert.
   This function should rely upon a local cache of schema documents, stored in memory in the Python process.
2. ``get_raw_alert_bytes(alert_id)`` should return `bytes` of the underlying alert packet.
   In other words, this passes through the response from ``GET /v1/alerts/<alert_id>`` directly.
3. ``get_schema(schema_id)`` should return `bytes` of a JSON document describing the given schema.
   In other words, this passes through the response from ``GET /v1/schema/<schema_id>`` directly.

The first function, ``get_alert``, is likely to be the main API for most users.
The other two exist to power ``get_alert``, and to permit lower-level operations.

More high-level functions (for example, ones that query the PPDB to find alerts that match some predicates) may be added in the future in the client.

Optional: Providing bulk access
-------------------------------

As described above, we may choose to provide bulk access to data in large chunks, possibly with very high latency.

This could be built with a system that simply gathers a list of alert IDs from the PPDB, and then repeatedly queries the object store by alert ID, concatenating many alerts into a single Avro Object Container file that is then provided to a user through some as-yet-unspecified protocol.

This naive system would take a long time to gather data.
Optimistically estimating 10ms per alert (dominated by network roundtrip time), we would expect this to take about 28 hours to fetch all 10 million alerts for a single night's observations if they are downloaded in series without parallelization.

To make that process faster, we could precompute bulk data files by adding another Kafka consumer process which builds hourly or nightly data batches, but this would come at the cost of duplicating storage.

Limitations
===========

No complex queries for last day of data
---------------------------------------

This design does not provide any sort of complex querying logic for data which has been stored since the last PPDB update.
Since the PPDB is updated daily, this means that the last 24 hours of data will not be indexed for complex queries.
This is acceptable, though, since the querying features of the alert database are not intended to support real-time online use cases.

Alerts are published before archival
------------------------------------

Alerts are published to brokers before they are archived, which minimizes latency to the brokers.
This introduces some risk of data loss.
If the archiving Kafka consumer fails or is misconfigured, we might broadcast alerts out to brokers without ever storing them in the alert database.

We have three fallbacks, however:

1. Kafka stores messages for a configurable length of time.
   If the archivist recovers within the lifetime of messages in Kafka, we could replay historical alerts and write them into the object store.
2. We may contact downstream brokers to recover a copy of the missed alerts to store them.
3. In theory, we should be capable of reconstructing alerts entirely from the PPDB.

.. _Alert Filtering Service:

Possible interaction with Alert Filtering Service
=================================================

One possible design of an alert filtering service would be to publish alert packet IDs with a small batch of useful information about the alert :cite:`DMTN-165`.
Consumers of that publication feed could decide to retrieve the full alert packet from the alert database if that small batch of useful information passed their filters.
In order to protect the object store backend and fairly use network resources, we could put a rate-limiting proxy in front of the object store.

In order to make sure that alerts are available in the alert database before publishing one of these lightweight alert notifications, we could publish lightweight alerts directly from the same Kafka consumer which writes into the alert database's backing object store.


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
