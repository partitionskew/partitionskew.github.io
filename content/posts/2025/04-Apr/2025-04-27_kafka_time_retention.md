+++
title = 'Kafka - Time and Retention'
summary = 'We discuss different types of timestamps associated with Kafka records and how it affects message retention'
tags = ["kafka", "time semantics"]
date = 2025-04-27
showToc = true
draft = false
+++

## Introduction

Apache Kafka is an open-source, distributed data streaming platform. Kafka data is stored durably and scalably in a multi-tier storage solution. What doesn't fit in memory is spilled to disk. What is accessed infrequently in disk is moved to cheap, bulk storage like S3 or GCS.

Today's discussion focuses on how Kafka decides how long to store data, aka retention. We review core Kafka concepts, cover key configurations, and review unexpected behaviours that can arise with the default settings and in edge cases. 

## Topics, partitions and log segments

The core abstraction in Kafka is a topic. A **topic** is a collection of messages similar to a queue. A Kafka topic is divided into partitions for scalability. A single partition can fit on a Kafka broker, aka Kafka server, and is replicated across multiple brokers for durability. 

The partition itself is further sub-divided into **log segments**. Each segment is an append-only log (AOL). New messages are added to the end of the log and the log is immutable once it is closed, aka rolling. There is only one active segment which can be read from and written to. Closed log segments are read-only.

Log rolling is determined by size and/or time. By default, a log is closed once it reaches 1 GiB. This can be tuned using the `log.segment.bytes` broker config. Alternatively, or in addition, one can also specify the `log.roll.(ms|hours)` config. By default, it is set to 7 days to ensure that a segment is closed. This is useful if the topic doesn't have much activity and won't reach the size threshold any time soon.

As a side note, segments are also closed when their indexes reach the max configured size. When you increase the segment's max size beyond ~5 GB, be sure to also increase the index's max size too or the segment could be closed before it fills up. To learn more about indexes and time-indexes, read this excellent article by Strimzi [1]. 

## Retention

To ensure that topics don't grow indefinitely in size, specify a **retention policy** based on size and/or time. Most operators specify a size-based cap to ensure predictable behaviour since some topics can be very active whereas other topics are infrequently updated. However, the two retention settings are not mutually exclusive. Time-based retentions can ensure compliance with internal requirements or external regulations which specify a minimum or maximum time for which data must be retained.

The retention configs are:
* `log.retention.bytes`: by default, there is no limit to the size of a Kafka topic
* `log.retention.(ms|minutes|hours)`: by default, 7 days

It is important to note that only closed segments are eligible for expiration. If a segment is active, it will not be considered for deletion even if it qualifies based on size and/or time.

Once a log segment has met the retention criteria, it'll be subject to the **cleanup policy**:
* delete (default): segments are deleted
* compact: only the latest message of a given key is retained

Kafka will mark eligible, closed segments for deletion. These segments will linger around for up to `log.retention.check.interval.ms` (default: 5 minutes) before Kafka finally deletes it. However, in the meantime, it is no longer readable by consumers.

Keep in mind that the retention time specifies a floor for the minimum time that a message will remain in a topic. The actual time that it takes for a message to be deleted can exceed the retention time because a segment has to be closed first before it can be marked for deletion. It then takes up to the retention check interval for the segment to actually be deleted.

## Timestamp Types

The question naturally is: how does Kafka decide when a log segment is "old" enough to be deleted? The answer is that it boils down to the timestamp of the last record in the segment when it is closed. However, there's a bit more nuance than that.

Kafka recognizes two different **timestamp types**:
* CreateTime (default): the producer-supplied timestamp is used by the broker; if the producer doesn't set a timestamp, then the producer's local time is substituted
* LogAppendTime: the broker overrides the producer timestamp with its own local time as it appends the record to the log

The timestamp type is configured at the broker level via `log.message.timestamp.type` and can be overridden at the per-topic level via `message.timestamp.type`.

## Issues

Let's examine some of the pitfalls that can arise from blindly trusting producer timestamps.

### Unreliable producer timestamps

Producer timestamps are unreliable. This is a bit of a generalization, but it's especially true if the producer's are external to your team or organization. Adversarial producers may deliberably set timestamps way into the past or far into the future. In more benign cases, a bug may have slipped into the code or the client's host machine may have an unsychronized clock. 

In any case, these scenarios pose a problem because old records are immediately subject to time-based retention (provided it ends up being the last record in the segment) and future records will linger around for far longer than they should.

Kafka partially solves this by allowing you to specify the maximum timestamp difference you're willing to tolerate before dropping a message for being too old or for supposedly occurring in the future.
* `log.message.timestamp.after.max.ms` (default: 1 hour): if the message's timestamp is newer than the broker's timestamp by this amount, the message is discarded for being from the future
* `log.message.timestamp.before.max.ms` (default: max long): if the message's timestamp is older than the broker's timestamp by this amount, the message is discarded for being too old

In either case, the two configs only apply to `CreateTime` timestamps.

### Poisoned time indices

Kafka also maintains a time index for each log segment. The time index maps a timestamp to a byte offset for its corresponding log segment. Older timestamps are ignored when adding to the time index, so out of order timestamps can cause the time index to become unreliable or poisoned. This makes it more challenging to confidently seek to a specific message in a topic when the timestamps do not monotonically increase as is the case with log-append time.

### Premature expiration

Likewise, if your data pipeline preserves producer timestamps from topic to topic, then messages may be expired earlier than you expect. For example, you may have a dead letter queue that holds failed messages for up to 30 days. You want to replay data from 30 days ago back to the source topic to be retried by the application. However, the source topic only has 4 days of retention so it will most likely drop the messages right away as closed segments may have the last record timestamp from the past.

## What is the solution?

`CreateTime` is typically fine if your producers are internal and you're reasonably sure they produce reliable timestamps. 

Otherwise, strongly consider using `LogAppendTime`. This does not mean you have to sacrifice any notion of event time. Simply store it as a field in the message itself or as a Kafka message header. Kafka retention will therefore depend on the log append time to ensure that messages stay in the topic for the expected amount of time, but your applications continue to process the data based on event time if that's more important.

If you prefer to keep the default setting but want to keep new records around for the expected time, you can also null out the input timestamp as you write out the new record. The producer will substitute with the local producer time. However, you'll have to evaluate which services should preserve timestamps and which services should override/unset timestamps. Or you can keep it simple by having new messages have new timestamps. Again, you'll need to store the event time somewhere else to ensure you don't lose it.

## Conclusion

We discussed how messages are grouped into topics, topics are divided into partitions, and partitions are composed of segments. We then examined when and why a segment is closed, and how closed segments can be expired to save space.

Finally, we considered different types of Kafka timestamps and how that can impact retention in unexpected ways. While `CreateTime` is the default, it doesn't always make sense given unreliable producer timestamps or certain edge cases like replaying a topic whilst preserving the timestamp. Use `LogAppendTime` in conjunction with an event time field in the message or Kafka message headers to get the best of both worlds.

## References

* [1] https://strimzi.io/blog/2021/12/17/kafka-segment-retention/
* [2] https://www.redpanda.com/guides/kafka-performance-kafka-logs
* [3] https://www.redpanda.com/guides/kafka-alternatives-kafka-retention
