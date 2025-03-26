+++
title = 'Flink - Metaspace OOM and Dynamic Classloading'
summary = 'Examining how Flink dynamically loads user code and how it can cause metaspace OOM errors'
tags = ["flink", "jvm", "classloader", "oom"]
date = 2025-03-25
showToc = true
draft = false
+++

## Introduction

Flink is a distributed framework that supports both batch and stream-oriented processing paradigms. A Flink cluster consists of two core Java processes [1]:
* JobManagers: manages resources, schedules tasks, and handles failures and recovery
* TaskManagers: workers that read, process, and write data

Improper configuration of a Flink application can lead to the dreaded OutOfMemory (OOM) error. In this article, we discuss how to avoid metaspace OOM errors when running a Flink cluster.

## Deployment Modes - Session vs. Application

Flink supports two cluster deployment modes [2]:
* session mode
* application mode

In session mode, a Flink cluster runs continuously. Flink clients submit one or more jobs to the session cluster. The session cluster requests resources from the resource manager such as Yarn or Kubernetes in order to complete the job. Multiple jobs can run concurrently on a single session cluster. Likewise, the session cluster's lifespan is independent of any job.

In contrast, an application cluster is tied to the lifecycle of its application. The application and its dependencies are bundled into the Flink cluster. When the application ends, so does the cluster.

## Metaspace OOM errors

### JVM Metaspace

Java divides its process memory into the heap space and the off-heap space. The heap is managed by the JVM garbage collector (GC). If there are no more references to an object, it is removed from the heap.

The off-heap is divided into several segments that each store different things. One segment is the metaspace. The **metaspace** stores, among other things, the class metadata which contain descriptions of the classes used by the application [3]. The metaspace is also managed by the GC even though it is not part of the heap. 

However, lingering references can cause classes and classloaders to not be fully unloaded from the metaspace. As a result, JVM applications may run into OOM errors due metaspace memory usage.

### Dynamic Class Loading in Flink

Flink has 3 distinct locations where it loads classes used by the app:
* Java classpath: this also includes Flink's `/lib` folder
* Flink plugins: aka the Flink `/plugins` folder
* Dynamic user code: part of the JARs of dynamically submitted jobs

Classes in the Java classpath, the `/libs` folder, and the `/plugins` folder are loaded just once at cluster startup. However, user code from dynamically submitted jobs are loaded and unloaded based on the lifecycle of the job. 

**One subtle point is that these dynamic user classes are also loaded when a job fails or restarts from a checkpoint.**

If the user classes do not correctly release objects that it has created over its lifetime, it can cause class leaks. This means that Flink's user code class loader will not fully unload the classes and this will cause a memory leak over time. 

You can verify the memory leak by monitoring the `flink_taskmanager_status_jvm_memory_metaspace_used` metric. For each failure or job restart, the metaspace memory usage will increase if classes are not fully unloaded.

### Fixing metaspace OOM errors

By default, the Flink task manager JVM metaspace is configured to 256 mb [4]. While we could increase this value to something larger, it merely delays the issue and does not resolve the root cause.

The fix is simple. If you're running a Flink cluster in application mode, put your code in the `/libs` folder [5]. The classes will only be loaded once by the JVM and not reloaded each time the job fails or restarts.

```Dockerfile
FROM flink:1.19.0-scala_2.12-java17

COPY target/my-app.jar $FLINK_HOME/lib/my-app.jar
```

If you're running jobs in session mode, double check that your classes are not holding onto resources or references such as JDBC connections [6]. 

## Conclusion

Flink dynamically loads user classes when jobs are submitted to a session cluster. Over time, this can cause metaspace OOM errors if the user classes are subsequently not properly unloaded.

If you're running a Flink application cluster, put your JARs and dependencies in the `/lib` folder so that it's only loaded once. If you're running a session cluster, make sure your classes are not holding onto resources or object references.

## References

* [1] https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/concepts/flink-architecture/#anatomy-of-a-flink-cluster
* [2] https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/overview/#deployment-modes
* [3] https://www.baeldung.com/java-permgen-metaspace
* [4] https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/config/#taskmanager-memory-jvm-metaspace-size
* [5] https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/ops/debugging/debugging_classloading/#avoiding-dynamic-classloading-for-user-code
* [6] https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/ops/debugging/debugging_classloading/#unloading-of-dynamically-loaded-classes-in-user-code
