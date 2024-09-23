+++
title = 'Lambda vs. Kappa Architecture'
summary = 'A brief description and comparison of lambda and kappa architectures'
tags = ["batch processing", "stream processing", "lambda architecture", "kappa architecture"]
date = 2024-09-22
showToc = true
draft = false
+++

## Introduction

In this article, I discuss and compare two big data architectures: Lambda & Kappa. 

## A brief history

In halcyon days of yore, data was collected in batches and processed using algorithms such as MapReduce (MR). Apache Hadoop was a popular **batch processing** framework that uses MR to crunch a lot of data on commodity hardware. Latencies on the order of hours or days was acceptable since the typical customer at the time expected daily, weekly, or even monthly reports. 

Such a delay no longer cuts the mustard in today's world where users want fresh data as soon as possible. If you're ordering a taxi to get to the airport, you want to know what the expected travel time is right now so you don't miss your flight. If you're a trader, you want data as fast as possible to beat your competition. It's no surprise that high-frequency trading firms situate their offices right next to a data center. This type of system is called real-time **user-facing analytics**.

**Stream processing** is like batch processing if each batch contained a single event and there were a never-ending stream of events. Latency is minimized since the event is processed as soon as it arrives rather than having to wait for some arbitrary time or size threshold.

## Big data architectures

### Lambda Architecture

The first generation of stream processing technologies were less fault-tolerant and therefore traded low latency for inaccurate results [1]. For example, a streaming framework might provide an approximate result by computing a running average over the last 100 measurements of device temperatures. A batch process may elect to wait for a day's worth of data before computing a more accurate result. If you retained historical data, it could compute an even more accurate result over a longer time horizon.

Lambda architecture was proposed to get the best of both worlds. Dual pipelines are deployed that process the same data source(s). One pipeline provides low-latency, approximate results using stream processing. The other pipeline processes historical batches to produce accurate results, albeit with significant delay. Whether the user gets approximate or accurate results depends on whether they're querying recent or historical data.

The batch processing pipeline is also referred to as the **cold path**. Likewise, the stream processing layer is referred to as the **hot path** or speed layer.

![lambda_architecture](/images/2024-09-22_lambda.png)

Looking at the diagram, I suppose it somewhat resembles a lambda if you rotate it clockwise. ¯\_(ツ)_/¯

As one might surmise, there's several major drawbacks to this architecture:
1. **Complexity**: is your team prepared and capable of supporting two different pipelines
2. **Code duplication**: business logic needs to be synchronized across pipelines otherwise results may differ in undesirable ways beyond accuracy
3. **Cost**: after all, you're processing the same data twice and that's not even getting into the operational/toil side of things

Newer generations of stream processing frameworks like Flink don't have the same drawbacks as the first generation technologies. Flink is fault-tolerant and provides exactly-once state semantics. In fact, Flink can run in both batch and stream processing modes. Hence, you can get fresh and accurate results with Flink.

With technologies such as Flink, there's no reason to run expensive, complex lambda architectures.

### Kappa Architecture

Kappa is a more modern architectural pattern enabled by technologies such as Apache Kafka and Apache Flink. The way it works is that data is streamed into an immutable, append-only event log (e.g. Kafka). The event log provides ordering, data partitioning, and persistence. 

Instead of running dual pipelines, you simply run a single stream processing framework that computes both accurate and low-latency results (e.g. Flink).

![kappa_architecture](/images/2024-09-22_kappa.png)

To enable data re-processing (in case you discover a bug or want to change your business logic), you can replay from the event log. However, it can be expensive to store a lot of historical data in a system like Kafka which keeps fresh data in memory and spills old data to disk. Hence, data is also archived to a cheap bulk storage system like S3 or GCS which Flink can easily replay.

## Conclusion

It's interesting to see this exact evolution play out in my team. Roughly 5 years ago, we ran dual pipelines that provided hot data for the past couple hours and cold data for anything older than that. Today, we run a Kappa architecture that's powered by Kafka and Flink. Of course, there's certainly nuances in how we deal with late-arriving data or handle state at scale, but that'll be a discussion for another day.

At any rate, I sure don't miss having to develop and maintain two vastly different systems. One pipeline already keeps me plenty busy.

## References

[1] Kleppmann, Martin. Chapter 12, Lambda Architecture. _Designing Data-Intensive Applications_, 2018 (4th edition), pages 497-498.

[2] Tejada, Zoiner (2024, September 22nd). Big data architectures. Microsoft Learn. **https://learn.microsoft.com/en-us/azure/architecture/databases/guide/big-data-architectures#kappa-architecture**
