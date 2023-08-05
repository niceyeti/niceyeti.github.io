# Other AWS Services

### SES

Send emails using SMTP or AWS SDK.
IAM integrated, as well as SNS, Lambda, etc.

### OpenSearch

Search by any field, unlike Dynamo, which only allows primary key search. Commonly used as a complement to another db.
Ingest data from Kinesis, etc.
Security through IAM and plugins.
Exposes searchability for data from other sources, such as acting
as a search index to DynamoDB data that can otherwise only be queried by key: stream DynamoDB data to OpenSearch, index it,
allow users to search on those fields, and then return the primary key by which users can get the original item in DynamoDB.

### Athena

Serverless query service to analyze data in S3 buckets and heterogeneous data, without setting up additional servers or databases.
Use SQL to query files: json, csv, Parquet, etc.
Use cases: analyze reports and business intelligence on raw data.
Performance:

* Use columnar data for cost savings (Parquet and ORC). Use Glue to convert your data to Parquet or ORC.
* Compress data for smaller retrievals (bzip, gzip, etc)
* Partition datasets in S3 by path for easy querying: /flight/parquet/year/month...
* Use larger files, > 128MB

Federated Querying: data source connectors that run on Lambda can run queries.

* Allows querying across data stored in relational, non-relational, object, and custom data sources.

### MSK: Managed Streaming for Apache Kafka

An alternative to Amazon Kinesis, fully managed Kafka on AWS:
* CRUD clusters
* MSK creates and manages Kafka brokers nodes
* Deploy the MSK cluster in your VPC
* Data is stored on EBS volumes for as long as you wnt
 MSK is serverless, for streaming data replicated across a distributed set of broker nodes.

 Differences with Kinesis:
 * 1MB default limit, but up to 10MB (Kinesis limited to 1MB)
 * Instead of streams, uses Kafka Topics with Partitions
 * Can only add partitions to a topic
 * Plaintext or in-flight encryption
 * KMS at-rest
 * Can keep the data for as long as you want

 