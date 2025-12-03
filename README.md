# ms-p2p-audit-and-compliance-service
Append-only event consumption from Kafka; optimized for streaming, low memory footprint.


# Design summary (requirements → approach)

**Requirements**

-   Append-only consumption of business events from Kafka.
    
-   Tamper-evident: compute & store SHA-256 hash for each event.
    
-   Low memory footprint: stream events; avoid buffering large in-memory batches.
    
-   Durable sinks: index in Elasticsearch (fast search) + archive immutable files to S3 / Iceberg.
    
-   Idempotent processing & at-least-once semantics (dedupe if reprocessing).
    
-   Observability and monitoring (lag, errors, throughput).
    

**Approach**

1.  Use a consumer group per environment (one or more replicas for scale). Use partition affinity to distribute work.
    
2.  Consume messages and **process each event synchronously** into a small worker pool; don't accumulate large in-memory lists.
    
3.  For archives, write events into **local rotating files** (Avro/Parquet) with limited size (e.g., 64 MB) and upload to S3 when full or after a short time window. This avoids holding many events in RAM.
    
4.  For searchability, bulk-index small batches into Elasticsearch (e.g., 500 - 2000 docs/batch).
    
5.  Compute `sha256` per canonical serialization and store it with the event metadata. Optionally sign batch manifests.
    
6.  Offset commit strategy: commit Kafka offsets **only after** the event is durably stored in sinks (S3 + ES). That gives at-least-once processing. Deduping by `eventId` on downstream read/queries ensures idempotency.
    
7.  Use a lightweight local index (RocksDB/BadgerDB/LevelDB) to store processed eventIds → S3 path for quick dedupe checks (bounded by retention, can be TTL-ed).
    
8.  Use schema registry (Avro/Protobuf) for payloads; the consumer validates schema.

