# SNS

Send one message to many receivers, 1:n Pub/Sub.

* event producers send messages to one topic
* topics define channels for messages
* n receivers can subscribe and receive all messages
* subscribers can optionally filter topics
* up to 12.5M subscribers per topic
* 100,000 topic limit
* many, many AWS service integrations with SNS
* various subscription protocols available: email, data, etc.

Topic publish (using SDK):

* create a topic
* create a subscription
* publish to the topic

Direct publish (mobile SDK):

* create a platform app
* create a platform endpoint
* publish to the platform endpoint
* integrates with Google GCM, Apple APNS, etc.

Exam note: SNS does not support all services as clients, for example Kinesis Data Firehose cannot be set up as a subscriber.

### Security and Encryption

Encryption (the same as SQS):

* in-flight https encryption
* at-rest encryption using KMS or SSE
* client-side encryption if you want

Policies: the same as SQS.

### SNS and SQS Fan Out

Configure one SNS topic to publish to many SQS queues:

* 1:n: one topic, n subscriber queues

The motivation:

* fully-decoupled architecture (each queue can define separate responsibilities)
* data persistence, retries, other resilience gaurantees
* scalable: add more subcriber queues over time
* REQS: requires policy to allow SNS to write to SQS queues
* cross-region: possible to send messages across regions

Usecase: S3 allows sending events per one action (object-create) + prefix ("/images"). Using SNS+SQS Fanout, you can demux the event to multiple handlers instead of just one.

### SNS FIFO

SNS also has a FIFO option.

* ordering
* de-deduplication
* tight-coupling: can ONLY have SQS FIFO queues as subscribers
* limited throughput

### Filtering

JSON policy defined at SNS level to filter messages sent to a topic:

* filter -> topic
* if no filter, all messages are published to a topic
