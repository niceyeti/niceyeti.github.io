# Kinesis

Kinesis provides realtime processing of streaming data,
with replication, sharding/partitioning, and consumer/producers components.

Kinesis * services:

* Ingest realtime data (logs, IoT telemetry, etc)
* Data Streams: capture, process, and store realtime data streams
* Data Firehose: load data streams into AWS stores
* Data Analytics: analyze streams with SQL or Apache Flink
* Video Streams: capture and process

Use case: for a realtime streaming source (click streams, etc) that needs to be transformed before being sent to Redshift and S3, you would want to use Kinesis services (data streams and analytics).

### Components and Data Streams

Producer components (sdk or AWS kinesis-producer lib) generate data with:

* partition key: which partition to send data
* data blob: the data

Kinesis then implements multiple shards.

Consumer components (kinesis-client lib or sdk) receive via a partition key, sequence no. (where data was in the shard), and the data blob.

Retention:

* 1 to 365 days
* ability to re-process data (replay)
* immutability: once data is inserted into Kinesis it cannot be deleted
* ordering: data that shares the same partition goes to the same shard

Capacity modes:

* provisioned:
  * choose the number of provisioned shards
  * shards get max 1Mb/s in, 2Mb/s out
  * pay per shard, per hour
* on-demand mode: no need to provision or manage capacity
  * default $Mb/s in (or 4k records/s)
  * scales based on peak throughput of last 30 days

Security: much the same as other services like SQS.

* control access with IAM policies
* in-flight https encryption
* at-rest KMS encryption options
*

### Kinesis Client Library

The KCL lets you scale kinesis data streams by shard and EC2 instances. However, the number of EC2 instances is bound to the number of shards, because each shard can be configured as read-only for exactly one KCL instance.

### Operational Semantics

(incomplete section)

If your producers receive an ProvisionedThroughputExceededException, then the KDS should be scaled up by one of:

* increase the number of shards
* evaluate the distribution of one's partition key
* retry with exponential backoff

Use "Enhanced Fan Out" to improve read-throughout beyond 2MB/s, the default limit.

If you have a hot shard, implement shard splitting.
