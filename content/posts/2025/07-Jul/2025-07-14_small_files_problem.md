+++
title = 'Small Files Problem'
summary = 'What is the small files problem and how to solve it.'
tags = ["big data", "optimization", "small files problem", "delta", "data lake"]
date = 2025-07-15
showToc = true
draft = false
+++

## Small Files Problem

The small files problem is, as its moniker suggests, one of IO overhead. 

In a typical streaming data lake architecture, a connector writes small batches of files to an object store such as S3 or GCS on some cadence. The file tends to be an analytics-friendly, columnar file format such as Parquet or ORC. The threshold could be based on:
* the size of the file
* the number of records/rows, 
* time
* or some combination of the triggers. 

To keep latency low, a connector might be configured to write out a new file every minute. With this configuration, the connector would generate 1440 files per day. Increase the threshold to once every 3 minutes, and we decrease the number of files down to 480 at the expense of a bit more latency. Keep in mind this is only for one table with one partition!

By building a data warehouse on top of vanilla Parquet using a technology such as Apache Hive, we can process the data using distributed engines such as Spark and familiar languages such as SQL. To query the data, we need to open quite a bit of files. This can slow down query performance as the processing engine needs to do a lot of IO. Even for newer table formats like Iceberg or Delta, many small files would generate more metadata to keep track of them all [1].

It's therefore in our best interest to compact the small files into bigger files to improve IO and to decrease the amount of metadata we need to track.

### Hive Partitions

As an aside, Hive tables can be physically partitioned using a key-value pair as part of the file path
> gs://my-bucket/my_prefix/date=2025-07-14/cohort=blue/(.\*).parquet

In the example above, the Hive table has 2 partitions: (1) date and (2) cohort. By including the partitions in our HiveQL query, we can cut down on the number of files we need to open rather than doing a full table scan.

## Solutions

Small files can be compacted manually or automated.

### Compaction

One option is to build a custom compaction solution using a distributed processing framework such as Spark. With Spark, you can read the Parquet files into a dataframe and then write out a smaller number of files using the `repartition` method [2]. 

The repartition method will hash the rows and sort them into the corresponding partitions. You can decide how many partitions you want. Databricks recommends targeting file sizes of ~1GB as those have been observed to perform optimally across most workloads. File sizes in the hundreds of megabytes are fine too, but you'll want to avoid file sizes smaller than 8MB to avoid IO overhead [3].

This job can be scheduled on an interval or run ad-hoc as needed. It just depends on how much data your producers generate. If a table grows slowly and the typical query focusses on a small number of partitions, you can schedule more infrequent compactions (e.g. weekly). Otherwise, you may consider more frequent compactions such as hourly or daily compactions. You'll also likely need to restructure your physical storage structure if you're using Hive.

The important thing is to ensure that the operation is atomic and isolated so that your customers don't see incomplete results. The repartition should either succeed and result in fewer files or fail. However, in neither case should customers see duplicated data or partial data. 
* If the compaction is performed in-place, consider keeping a manifest of old files so we can record what should be deleted later. However, it's harder to guarantee atomicity when we try to change the files in place.
* Alternatively, we can add an additional subdirectory to our partition locations to separate uncompacted data with compacted data. This allows us to perform the compaction in a safe, atomic step. If the compaction succeeds, we update the partition path location. If it fails, we don't update the path and try again later.
    * However, if you go down this path, do note that Hive does not handle non-compliant partition paths very well. Any path that contains an infix or suffix that does not follow the key=value convention is not compliant with Hive partition paths. Consequently, Hive cannot autodiscover these paths using commands such as `MSCK REPAIR TABLE` [4].
    * We can safely clean up the old, uncompacted data using a clean up job. Or we can use bucket lifecycle retention policies to clean up the data after X days.

### Delta Lake Optimize

If you're using a fancy open table format like Delta Lake, it supports compaction out of the box with the `OPTIMIZE` command [1]. This command automatically reduces the number of files by coalescing them to ~1GB files. This feature can save you the effort of having to author your own compaction job and scheduling it to run on a cadence. You can also easily configure the file sizes that are targetted by the command. 

By default, optimize works on all historical data. To save a bit of compute, you can configure it to compact incrementally on new data instead of wasting resources on older data which is likely already compacted.

The benefits of this command compared to a custom solution is that it's atomic and less error prone. It can be annoying to track and troubleshoot compaction failures when you have many tables and even more partitions.

## Conclusion

The small files problem happens when you stream data into a data lake. To minimize latency for consumers and readers, we try to write data as frequently as possible. However, each file tends to be small and many files need to be opened by a processing engine to find the desired data.

To avoid small files, we can increase the flush cadence as high as our customers are willing to tolerate. When that isn't enough, we should consider a custom compaction job (e.g. Spark) or use optimize command if we're using Delta Lake. Aim for files that are ~1GB after compaction.

## References

* [1] https://delta.io/blog/2023-01-25-delta-lake-small-file-compaction-optimize/
* [2] https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrame.repartition.html
* [3] https://docs.databricks.com/aws/en/optimizations/spark-ui-guide/slow-spark-stage-low-io#reading-a-lot-of-small-files
* [4] https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-repair-table
