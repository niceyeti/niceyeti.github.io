# Dynamo DB

Dynamo DB is a serverless No-SQL db with no joins, just flat tables.

* scale horizontally by increasing instances
* row-based: each row contains all data for an item
* fully-managed, HA, fully-distributed
* eventually-consistent or not
* DB's are defined at the region level: you can create as many tables as you want, they will simply be confined to a region (surely with additional region options).

## Top-Level Concepts

* Key structure and indices: DynamoDB is table based, whose query-scope is limited to partition (primary) key and sort-key.
* Indexes: these overcome the limitations on key structure and querying.
* WCU/RCU and strong vs. eventual consistency: memorize the definitions and constraints on WCU/RCUs.
* Streaming: streams allow coupling with other services, all item-level modifications (create/update/delete), which can be sent to other services (Kinesis, Lambda, etc).
* Optimistic locking: Conditional Writes allow for only writing a value if some other comparison condition is met; this allows for implementing version-number based optimistic locking.

### Basics

* all data are unrelated tables
* each table has a primary key:
  * option 1: partition key (HASH): the key is diverse (well-distributed as a hash value) and unique (aka high-cardinality).
  * option 2: partition key + sort key (HASH + RANGE) combination must be unique for every item
    * example: user_id and post_timestamp
* The design and selection of a partition key and sort key should consider queryability; attributes (regular non-key columns) are NOT queryable except using indices or via brute scan-table+client-filter.
* "attributes" == columns
* maximum object size is 400kb
* attributes (columns) can be added dynamically over time

### Throughput and R/W Capacity

Provision capacity units separately for read and write.

Modes:

* provisioned (default): specify the number of r/w per second, pay for provisioned usage. "Burst" capacity is allowed temporarily.
* on-demand mode: more expensive, but pay for what you use. No need for capacity planning.

Provisioned: spec's read and write capacity separately. Exponential backoff should be use when r/w ProvisionThroughoutExceededException errors occur.

Read capacity units: these are either strongly consistent of eventually consistent (default), with separate rules for each.

* eventually consistent (default): you may retrieve stale data after writing due to replication
* strongly-consistent: set ConsistentRead=true in the api (GetItem, BatchGetItem, Query, Scan); consumes twice the RCUs.
* RCU definition: one RCU is consumed by **1 strongly-consistent read per second**, or **2 eventually-consistent reads**, for items up to **4kb**.
  * Examples:
    * 10 strongly-consistent reads/s for 4kb items: 10 RCU
    * 16 eventually consistent reads/s with item size 12kb: ceil(12kb/4kb) * 16 / (2 reads/s) = 24 RCU.
    * 10 strongly-consistent reads/s with item size of 6kb: 10 read/s / 1 read/s *ceil(6kb) / 4kb = 10* 2 = 20 RCU

Write capacity units: 1 write per second for an item up to 1kb size. If item is larger than 1kb, then more units are used:

* wcu's = items per second * ceil(item_size_kb / 1kb)
* example: 6 items per second, each of which are 4.5kb. Therefore, wcu = 6 * 5/1kb = 30 wcus
* example: 120 items per MINUTE, 2kb size: 120/60 = 2 items/s * ceil(2kb / 1) = 4 wcus

### Partitions

The partition key is input to a hashing algorithm to determine which partition it is part of. A hot partition (repetitive key) means poorly distributed data. Computing the number of partitions by RCU/WCU capacity is given by (RCUs / 3000) + (WCUs / 1000). Partitions by size is total_size / 10GB. The true number of partitions is ceil(max(partitions by capacity, partitions by size)).

The only thing to remember that RCUs and WCUs are divided evenly across partitions. Therefore, at the partition level one might receive a ProvisionedThroughputExceededError exception due to a hot partition. Remediation is given by exponential backoff.

### On-Demand Mode

No capacity planning required, unlimited WCU and RCU, and no throttling, but more expensive. You are charged in terms of reads and writes that you request/require, measured in RRU and WRU (read request units and write request units). About 2.5x more expensive.

* Capacity mode (on-demand or provisioned) can be changed as needed, up to once per day.

### Auto-Scaling

As an alternative to on-demand provisioning, you can enable auto-scaling on WCU/RCU values to scale up/down as needed, with min/max RCU/WCU values and target usage percentage specified.

### API Calls

Write:

* **PutItem**: creates or replaces existing item; consumes WCUs.
* **UpdateItem**: edit an existing item, and can be used to implement AtomicCounters (numeric attributes unconditionally incremented).
* **ConditionalWrite**: write/update/delete an item only if conditions are met (helps with concurrent access to an item).

Reads:

* **GetItem**: read based on primary key (HASH or HASH+RANGE). Eventually consistent, unless strong-consistency is specified (more RCU's consumed). **ProjectExpressions** can be used to retrieve subsets of values.
* **Query**: return items based on:
  * **KeyConditionExpression**:
    * partition key comparison must be '=' operator
    * sort key value (=, <, >, Between...)
  * **FilterExpression**:
    * additional filtering after the Query operation, before data returned
    * use only with non-key attributes
  * Returns: the number of items specified in Limit of 1MB data. Pagination available.

* **Scan**: read an entire table and then filter out data (inefficient). Returns up to 1MB. Consumes a lot of RCU.
  * **ParallelScan**: multiple workers scan multiple data segments at once; increases RCUs consumed.

* **DeleteItem**: individual items or conditionally-delete items.

* **DeleteTable**: much quicker and more efficient than calling DeleteItem on every item.

Batch ops: save in latency by reducing the number of API calls. Retry for only failed items, on failure.

* **BatchWriteItem**: up to 25 PutItem or DeleteItem, 16MB of data written, up to 400kb per item.
  * cannot update items (UpdateItem)
  * Returns UnprocessedItems for failed writes

* **BatchGetItem**: return items from one or more tables, up to 100 items and 16MB of data.
  * Returns UnprocessedKeys for failed read operations

* **PartiQL**: SQL-compatible language for DynamoDB: select, insert, update, and delete items.
  * Run queries across multiple tables, but NO JOINS
  * Run PartiQL from console, APIs, cli, or SDK

* **ConditionWrites**: for PutItem, UpdateItem, DeleteIem, and BatchWriteItem, specify an expression to determine which items should be modified:
  * exprs: attributes_exists, attribute_not_exists, attribute_type, contains (for string), IN and BETWEEN operations
    * attribute_not_exists: apply an operation if an item does not have an attribute
    * use case: attribute_not_exists(partition_key) can be use to verify an item does not exist before operation, to prevent over-writing.
    * Example:

    ```
    aws dynamodb delete-item --table-name ProductCatalog --key '{"Id": {"N": "456"}}' \
    --condition-expression "(ProductCategory IN (:cat1, :cat2) and (Price between :lo and :hi)" \
    --expression-attributes-file file://values.json
    ```

    Where values.json is:

    ```
    {
        ":cat1": {
            "S": "Sporting Goods"
        },
        ":cat2": {
            "S": "Outdoor gear"
        },
        ":lo": {
            "N": "500"
        },
        ":hi": {
            "N": "600"
        }
    }
    ```

    This query says "delete item 456 if it is in sporting goods or outdoor grear, and is between 500 and 600 in price".

### Local Secondary Index

The same primary key but an alternative sort-key for your table, consisting of one scalar attribute (column) (string, number, or binary).

* use case: queries cannot be performed on attributes (except using scan and filter on client). Local indices allow performing queries on a key and an attribute/column, similar to views in sql.

* up to 5 local indexes per table
* must be defined at table creation time
* attribute projections: keys_only, include, or all.
  * These are the attributes copied from a table into a secondary index.

### Global Secondary Index

An alternative primary key (HASH or HASH+RANGE) used to speed up queries on non-key attributes. The index key consists of scalar attributes (string, number, or binary). The GSI provisions its own WCU/RCUs, but both will be throttled.

* also has attribute projections (keys_only, include, all)
* difference from local indices: global secondary indices can be created after table creation

GSI's use their own independent RCU and WCU values, and if they are throttled due to insufficient values, then the main table to which they refer will also be throttled.

### Indexes and Throttling

GSI: if writes are throttled on GSI, then they are throttled on main table also.
    * assign WCU capacity carefully
Local secondary indices use the WCU/RCUs of the main table, and have no special throttling considerations.

### PartiQL

SQL-like syntax for DynamoDB tables:

* INSERT, SELECT, UPDATE, DELETE
* No JOIN logic
Examples:
* DELETE FROM 'demo_indexes' WHERE "user_id" = "partition_key" and "post_timestamp" == "sort-key"
* SELECT * FROM "demo_indexes" WHERE "user_id" = "partition_key" AND "post_timestamp" = "sort-key"
Note how the WHERE clause encompasses the partition and sort key.

### Optimistic Locking

Atomic counters can be implemented in attributes; likewise, Conditional Writes allow only writing values based on comparison with existing values. This directly extends optimistic locking, whereby a client request to write a value, only if its version number is an expected value:

* if the version number is expected, the update succeeds
* if the version number is wrong, the update fails

### DAX: DynamoDB Accelerator

Fully managed, HA, in-memory cache for DynamoDB.

* us latency for reads and queries
* solves the Hot Key problem: throttling that results from a hot key. DAX will not throttle.
* requires no app code changes; transparent to the api/language of DynamoDB
* other: secure, multi-AZ, 5m TTL default

The difference with elasticache is that DAX can cache objects, queries, or scans. Elasticache can be integrated with DAX, for instance to store aggregation results.

Note DAX requires configuring SGs, IAM role, VPC/subnets, and other raw components.

### Streams

Streams allow broadcasting item level events (create, update, delete) so other services can use them. Note: cannot stream to SQS.

* data retention is 24h: thus Kinesis provides durable storage
* react to changes in realtime (fan-out)
* create derivative tables, cross-region replication, etc.

You can choose the items in the stream:

* keys_only, new_image (entire new item), old_image (expired item), old and new image
* streams are made of shards
* NOTE: records are not retroactively added to a stream after adding it. This is a popular exam question.

Lambda integration: this requires using an Event Source Mapping to read from the stream, and ensure the Lambda has the appropriate IAM permissions.

* DynamoDBReadOnlyPermission
* lambda func is invoked synchronously
* configured by enabling a trigger in DynamoDB

### TTL

Automatically delete items after an expiry based on unix epoch, within 48h or execution.
Does not consume WCUs.
Deleted items will appear in queries/scans for up to 48h.
Use case: keep only relevant/current items, regulatory rules, etc.

* CloudWatch metrics will show you the number of deleted items

### CLI

Good to know for the exam:

* --projection-expression: ("only these attributes") one or more attributes to retrieve
* --filter-expression: filter items BEFORE returned to the client
Pagination:
* --page-size: (default 1000) (increases api calls under the hood) we want to retrieve the entire datasets, but each return-set is larger than supported without timeouts.
* --max-items: max number of items to show in the cli
* --starting-token: specify the NextToken parameter to retrieve the next page of items. NextToken will not appear in returned metadata for last set.

### Transactions

Coordinated all-or-nothing operations (add/update/delete) across one or more tables.
This is essentially state propagation across tables.

* ACID compliant: atomicity, consistency, isolation, and durability
* Read modes: eventual or strong consistency
* Write modes: standard, transactional
* Consumes 2x WCUs and RCUs: a read and a write for every item.

Two operations:

* TransactGetItems: one or more GetItem operations.
* TransactWriteItems: one or more PutItem, UpdateItem, or DeleteItem operations.

Use cases: financial transactions, managing orders, multiplayer games.
Patterns: transactions allow implementing transaction-log tables, by which an entire table can be reconstructed.

Usage calculations:

* ex: 3 transactional writes / s for 5kb items: 3 *5kb/1kb* 2 (transaction cost) = 30 WCUs
* ex: 5 transactions/s for 8kb items: 5 *8kb/4kb* 2 = 20 WCUs

### DynamoDB as Session State Cache

A common use case for DynamoDB and elasticache alike is for a session-state cache.

* elasticache is in-memory, whereas DynamoDB is serverless
* both are key-value stores
* only DynamoDB is autoscaling
* EFS, otoh, could also work, but fs is a different beast.
* EBS & Instance store: attached to only one EC2 instance; can be used as cache, but only locally.
* S3: higher latency, and not intended for small object

The primary usecase for DynamoDB as a session-state cache is when key-value store logic is needed, along with autoscaling, but in-memory is not required.

### Write Sharding

In certain scenarios, a natural unique id creates a hot-key/hot-shard, for example candidate_id in an election becomes less distributed as the election turns into a head to head.

The solution is to add a suffix to the partition key value:

* random suffix
* calculated suffix

I don't know this implementation, the lesson is simply to strategize for key distribution.

### Write Concurrency

Optimistic locking: Condtional Writes MUST be used to coordinate concurrent write behavior: if version number is X, accept update, otherwise fail.

Atomic Writes: allow multiple writers without coordination, as long as their actions are atomic, ie, for counters.

Batch writes: sequentially write a bunch of records.

### S3 integration: Large Objects Pattern

1) Upload large item to S3
2) Insert its data into DynamoDB as small object indexed by attributes

Can also use DynamoDB as a client of a lambda triggered directly off of S3 updates. This can be used to leverage DynamoDB's queryability features over the space of objects in the S3 bucket: search by date, list all objects, find, etc.

## Misc

Table cleanup: scan and delete items one by one. Much better, is to drop and re-create the table.
Copying: use AWS data pipeline. Alternatively, back up the table and then copy to new table.

Security: VPC endpoints can be made to conceal Dynamo to within one's cloud.

* access fully controlled by IAM
  * direct users can be given a Role limited by a Condition to specific conditions: users can be limited to row-level data
  * Batch, Get, Delete, etc can be directly limited via Policy
* encryption at rest and in transit.
* PITR available: point in time recovery, like RDS
* Global tables: multi-region, fully replicated, high perf

Testing: DynamoDB Local is available to run for develop/test.

Migration: migration services available.
