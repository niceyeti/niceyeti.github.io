# SQS

Queues provide a pattern for decoupling one's architecture and for providing
data resilience.

Beware of timing diagram complexity with queues; concurrent read/write and visibility/deletion parameters
entail cornercase complexity that should be avoided through simplified design.

### AWS SQS

The oldest AWS service, fully managed.

* unlimited throughput, unlimited number of messages in the queue
* automatic scaling
* default retention: 4 days, 14 days max
* latency: <10ms publish and receive
* limitation of 256kb msg size
* at-least-once delivery: duplicate messages are possible
* messages are manually deleted from the q
* best-effort message ordering

There are two types of queues:

* standard
* FIFO:
    1. message order is strictly preserved (MessageGroupID)
    2. duplicate messages are not introduced to the queue (MessageDeDeuplicationID)
    3. messages are grouped by message group-ID
  * Note: FIFO logic applies only to messages for a given message group ID
  * Note: you can have as many FIFO consumers as MessageGroupIDs, hence the number of message groups determines scalability.

### SendMessage, ReceiveMessages SDK API

Senders:

* Messages sent to the q are persisted until deleted.

Receivers:

* Consumers poll the q for new messages
* Receive up to 10 msgs at a time
* Manually delete items from the q using DeleteMessage api; this MUST be called to let the queue know the message was consumed and can be deleted.

Core API:

* CreateQueue, DeleteQueue
* PurgeQueue
* SendMessage (delaySeconds), ReceiveMessage, DeleteMessages
* MaxNumberOfMessages: max number of messages to return as a batch to clients
* ReceiveMessageWaitTimeoutSeconds
* ChangeMessageVisibility: change the message visibility timeout
* batch APIs: SendMessage, DeleteMessage, ChangeMessageVisibility

### ASG Integration

Q-length metrics provide integration with ASGs, such that consumers
can be scaled according to q burden.

A common architectural pattern is to place ASGs in front of and behind SQS queues:
producers scale according to their input burden, sending more messages to the queue,
and backend consumers scale according to q burden (q len).

### Security and Encryption

Access control: IAM policies control access to queue APIs.

SQS Access Policies: similar to S3 bucket policies.

* determine the accounts, users, and Roles that can access the queue
  * useful for cross-account q access: Effect (allow), Principal (some other AWS account), Action (send/receive msg, etc), resource (the queue)
  * useful for integration with other AWS services, e.g. S3: Effect (allow), Action (SendMessage), Resource (the queue ARN), Condition (string match on S3 bucket to allow q-write)

Encryption:

* in-flight https encryption
* at-rest encryption with KMS
* options: SQS-SSE default AWS-managed encryption key, or KMS that you configure.
  * SQS-SSE: entirely AWS-managed
  * KMS: keys you configure, for which you get the visibility of KMS key/api usage, etc.

### Implementation Parameters

* Visibility timeout: duration for which message will not be visible after a consumer grabs it
* Message retention period: period for which to retain messages that are not consumed.
* Delivery delay: how long to delay before publishing a message (15 mins max). Why do this? Because backing up messages means consumers can receive 'bunches' of them more at a time, reducing api calls and improving cost/efficiency, but this also seems sus.
* Maximum message size: configurable message size constraint
* Receive message wait time

Timing diagram:

```
    Message received  <-- Delay seconds --> Message visible <-- visibility timeout --> Message visible
    ^                                       v                    -                     v
    SendMessage                      ReceiveMessage              ReceiveMessage        ReceiveMessage
```

### S3 Integration Example

Reqs:

* configure policy to allow the S3 bucket to send-messages to q-xyz
* configure S3 events to send messages to the queue.

### Visibility Timeout

Messages are made temporarily not-visible after a consumer calls ReceiveMessage.
The message will become visible again after the visibility timeout.

Due to timing and failure implications:

* messages could be processed twice if visibility timeout is too low
* messages could be processed out of order
* it is up to you to define parameters that comfortably guarantee timely ordering/processing properties
* interesting: "Amazon SQS can delete a message from a queue even if a visibility timeout setting causes the message to be locked by another consumer." Ergo, locking does not guarantee the message.

### Dead Letter Queues (DLQs)

Messages are returned to the queue n times before being sent to a DLQ.
The MaxReceives threshold parameter defines how many times to re-process a message before sending to DLQ.
DLQs are useful for debugging, inspection, and recovering, since you can manually or automatically inspect the failed messages.
DLQs naturally extend the queue-pattern abstraction as a decoupling mechanism: as normal queues are used to support usecases, DLQs support the usecase of failover/debugging/inspection of messages. They support the same semantics, but for failure pathways in your architecture: just as normal queues integrate with end-value lambdas, DLQs integrate with lambdas that could implement failure/re-drive logic.

* DLQs are coupled to the source q type: FIFO queues require a FIFO DLQ, and likewise standard queues requires a standard DLQ
* redrive to source parameters: this feature allows custom handling and inspection of failed messages before returning to source q.

### Long Polling

Consumers call and then await responses from the queue for up to 20s
Long-polling should be performed by consumers to reduce latency and api calls.
This is a pseudo-subscription pattern, somewhat degenerate of real pub/sub.

* enabled at the queue level or the API level using the ReceiveMessageWaitTimeSeconds

### SQS extended client

The SQS extended client is used to send messages larger than 256kb, using an S3 bucket message proxy pattern: SQS messages contain metadata about objects and that an object occurred, and S3 is used as the data repository, all handled by the extended-client library.

* client sends metadata to queue; data is placed in S3

### FIFO Queues

Messages have a FIFO ordering guarantee.
FIFO queues subsequently have limited throughput.

* exactly-once send semantics (by removing dupes)
* messages are processed in order by consumers
* content-based configurable de-duplication parameters

De-duplication:

* de-dupe interval is 5 mins
    1. content-based: SHA256 hash of message content (default)
    2. custom: explicitly provide a message de-duplication ID

Message grouping (effectively, channels) is configured via message de-duplication in FIFO queues.

* MessageGroupID: specify this and only one consumer can receive from this group, and all messages will be in order.
  * each group ID can have a separate consumer
  * each consumer will receive items in order for that group
  * ordering across groups is not guaranteed

  NOTE: disambiguate MessageGroupID and DeduplicationID.
  MessageGroupID separates messages into channels, whereas DeduplicationID de-duplicates messages within a group.
