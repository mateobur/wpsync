---
ID: 10
post_title: satanesbuenos
author: user
post_excerpt: ""
layout: post
permalink: >
  https://wordpress.fourtytwo.space/2018/10/03/satanesbuenos/
published: true
post_date: 2018-10-03 14:30:34
---
## Abstract

This guide will cover how to monitor the Java Runtime Environment (JRE). You will learn how to assess the performance of your Java application profiling its memory usage, the garbage collector metrics, Java daemon and user threads and other fundamental JRE metrics. We will finish with a real Java JRE troubleshooting example using Sysdig and Docker.

When working on a big Java application, it's likely that something will eventually fail, misbehave, or maybe you have an OutOfMemoryException that catches you off guard. And if you're also deploying your Java application as a containerized microservice, monitoring Java in Docker and Kubernetes may present some new and unexpected challenges.

## About the guide:

This is the first chapter of the "Ultimate guide to monitoring Java inside Docker and Kubernetes". We will cover how to monitor **Java JRE** heap memory usage, threads, memory leaks, and, also, **Java Management Extensions (JMX) metrics** using JConsole, VisualVM, JMXTerm, JMXTrans, and Prometheus in the context of container microservices, **Docker** images, and **Kubernetes** pods.

We also packed it full of practical examples so you can see how efficient and effective Docker and Kubernetes monitoring and troubleshooting are when using Sysdig, Csysdig, Sysdig Inspect, and Sysdig Monitor.

Stay tuned for future chapters of our "Ultimate guide to monitoring Java inside Docker and Kubernetes", which will include:

[<img src="https://478h5m1yrfsa3bbe262u7muv-wpengine.netdna-ssl.com/wp-content/uploads/2018/09/MonitorJavaHeapSpace.png" alt="Monitor Java Heap Space" width="922" height="326" class="aligncenter size-full wp-image-10823" />][1] 

*   Monitoring Java Management Extensions (JMX), custom JMX metrics for performance profiling and tuning
*   Java monitoring in Docker and Kubernetes: troubleshooting java applications and performance issues
*   Java troubleshooting guide: Fixing a memory leak in Kubernetes

Monitoring #Java inside #Docker and #Kubernetes: main memory, garbage collector and threads, basic concepts and metrics to watch. 

## Monitoring Java application performance: Java Runtime Environment metrics and profiling

There are several metric sources that Java places at your disposal. While you can develop your own custom Java Management Extensions, General JMX metrics supply tools for managing and monitoring applications, system objects, devices (such as printers), and service-oriented networks—all of which we'll cover in depth in future chapters of this guide.

The Java Runtime Environment (JRE) that we'll be discussing today contains a lot of general information about how your application is doing including, CPU usage, number of threads running, number of classes loaded, garbage collector information, main memory usage, and more.

### Monitoring Java memory usage

One of the most important Java resources to monitor, especially if you're using Docker or Kubernetes microservices which are meant to be lightweight and easy to destroy and recreate, is the main memory consumption.

The RAM can be split into two different parts: **stack** and **heap**. To monitor Java main memory usage, you need to understand the differences between these memory data zones and the specific behavior of each.

#### Stack

The [stack][2] is the part of the RAM that contains information about function calls. It's where local variables get stored and destroyed and operates quickly thanks to its strictly last-in, first-out design.

With RAM, it is a simple matter of a move operation and a decrement/increment operation on the stack register. The cost of destroying a variable is free because their memory is just dropped.

We can configure the stack size per thread (each thread has its own stack) with the following parameter:

<table>
  <tr>
    <td>
      -Xss
    </td>
    
    <td>
      set java thread stack size
    </td>
  </tr>
</table>

The stack itself is split into different regions: **permanent generation** and **code cache**.

<div style="width: 595px;height: 485px;margin: 0 auto">
  <div style="margin-bottom:5px">
    <strong> <a href="//www.slideshare.net/Sysdig/wtf-my-container-just-spawned-a-shell" title="WTF my container just spawned a shell!" target="_blank">WTF my container just spawned a shell!</a> </strong> from <strong><a target="_blank" href="//www.slideshare.net/Sysdig">Sysdig </a></strong>
  </div>
</div>

##### Permanent generation

The permanent generation region of the stack contains metadata required by the JVM to describe classes and methods used in the application. When a class is loaded, the permanent generation is populated with it and its methods. This region is also included in a full garbage collection because when a Classloader dies all its classes must be unloaded.

##### Code cache

Code cache is a region of the stack where all the native code is stored and compiled by the [JIT compiler][3].

#### Heap

The [heap][4] is the rest of the RAM reserved for object allocation.

Normally, when you allocate an object instance in the Heap, that memory must be manually freed when you are not going to use the object anymore. Not doing so can lead to a common problem in languages like C or C++: a [memory leak][5].

A memory leak happens when all the pointers to an allocated object in the heap are lost and the instance can no longer be referred to nor released. This part of the memory cannot be used again until the program is closed. If this happens often in your code, you may reach a moment where no more RAM is available for allocation.

In Java, the memory from objects not being referenced by any pointer is released automatically by the [garbage collector][6] (GC). However this has its limits as well—references and resources remaining open.

If you allocate memory for an object and keep a reference to it even though you don't use it anymore, that's another memory leak scenario you want to avoid.



    
    File: redis_prometheus_exporter.yaml
    ------------------------------------
    
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: redis
    spec:
      replicas: 1
      template:
        metadata:
          annotations:
            prometheus.io/scrape: "true"
            prometheus.io/port: "9121"
          labels:
            app: redis
        spec:
          containers:
          - name: redis
            image: redis:4
            resources:
              requests:
                cpu: 100m
                memory: 100Mi
            ports:
            - containerPort: 6379
          - name: redis-exporter
            image: oliver006/redis_exporter:latest
            resources:
              requests:
                cpu: 100m
                memory: 100Mi
            ports:
            - containerPort: 9121
    ---
    kind: Service
    apiVersion: v1
    metadata:
      name: redis
    spec:
      selector:
        app: redis
      ports:
      - name: redis
        protocol: TCP
        port: 6379
        targetPort: 6379
      - name: prom
        protocol: TCP
        port: 9121
        targetPort: 9121
    
    



Also, when you open a resource like a stream and don't close it, if it goes out of scope it will remain open and continue to consume heap. A good solution to this is to use the "[try with resource][7]" blocks that were released with Java 7.

In Java, when you are not able to reserve more memory the JVM throws an exception called [java.lang.OutOfMemoryError][8] and terminates the program.

The maximum heap memory allowed per application can be configured when launching the application with the following parameters:

<table>
  <tr>
    <td>
      -Xms
    </td>
    
    <td>
      set initial Java heap size
    </td>
  </tr>
  
  <tr>
    <td>
      -Xmx
    </td>
    
    <td>
      set maximum Java heap size
    </td>
  </tr>
</table>

The heap is also split into several regions including **eden space**, **survivor space**, and **tenured generation**.

##### Eden space

This region of the heap is where all new objects are allocated. When an object survives a garbage collection, it's moved to the survivor space, which is considered part of the Young generation.

##### Survivor space

When an object survives a garbage collection it's moved to the survivor space, which is considered part of the young generation.

When the young generation fills up, a *minor garbage collection* is performed. This collection is optimized with the assumption of a high object mortality rate. It always performs a stop-the-world event, but is a really fast way to clean a young generation that's full of dead objects.

#### Tenured generation

Also called the old generation, the tenured generation contains all the long-surviving objects.

When it fills up, a *major garbage collection* is fired. This is another stop-the-world event but is much more slow because it involves all live objects.

### Monitoring the Java garbage collector

The garbage collector is a key component to monitor Java inside Docker and Kubernetes. Correct and predictable GC behaviour will ensure that your pod doesn't hit the configured limits and get eventually killed and replaced.

### Garbage collector

he GC aims to free heap memory that's not reachable anymore to make it available for enabling new object allocation.

It can be called using System.gc(); but this doesn't guarantee it's execution. When the GC is performing, all states from all threads must be saved. If it executes while an object is being allocated, you may break the [Java Virtual Machine specification][9].

The garbage collector consumes computer resources in deciding which memory must be freed, which can lead to overhead and decreased performance.

Most modern garbage collectors try not to make a [stop-the-world][10] collection, but this JVM element must be kept in mind—especially if you are creating a real-time application.

There are multiple implementations of garbage collection in Java, which we'll explore below.

#### Serial garbage collector

The serial garbage collector is intended to be used in single-thread environments. It is designed for use with simple command-line programs and is not suitable for server environments. A serial GC uses a single thread for collection and performs a stop-the-world event while running.

**Parameter: *-XX:+UseSerialGC***

#### Parallel garbage collector

Also called a throughput collector, the parallel garbage collector was the default in the JVM from its introduction until Java 9.

Unlike a serial GC, a parallel GC uses multiple threads to run. It also performs a stop-the-world to stop the whole application until the garbage collection is complete.

**Parameter:_ -XX:+UseParallelGC_**

#### CMS garbage collector

The [concurrent mark sweep (CMS) garbage collector][11] implements a mark-and-sweep algorithm in two steps:

1.  **Mark**: perform a tree search to find objects that are being used and marking them as "in use"
2.  **Sweep**: those that are not marked as "in use" are considered to be unreachable so their memory is freed and "in use" marks are swept away

With a CMS GC, the whole system must be stopped in order to prevent the tree from being modified and all the memory must be scanned twice.

    Parameters: -XX:+UseConcMarkSweepGC  and -XX:ParallelCMSThreads=_<num>
    

#### G1 garbage collector

The garbage-first collector (G1) was introduced in JVM 6 and supported from JVM 7. It was planned to replace the CMS GC and **has been **the default GC since Java 9.

With G1, garbage collection is focused in memory regions with the least amount of live data while global heap collection is performed concurrently to avoid long interruptions.

There are two major differences between the CMS and the G1 GCs:

1.  G1 is a [compacting collector][12], which simplifies parts of the collector and eliminates fragmentation issues.
2.  Garbage collection pauses are more predictable and can be configured by the user with G1.

**Parameter:** *-XX:+UseG1GC*

### Monitoring Java threads

#### Threads

Creating threads in your application for completing concurrent tasks such as fetching or writing data into a database or a file can improve the performance of your application when you have I/O. However, you can start to run into issues when you have a lot of threads doing concurrent work.

Creating and destroying a thread, as well as saving and restoring the state of the thread, creates significant overhead because of the way finite hardware resources are shared. This produces a general slowdown in the application.

The solution is to limit the number of threads you have running.

Ideally, your computing threads will be separated from I/O threads since those are blocking ones. Try to design your computing threads so they behave as runnable threads most of the time and never block on external events.

There are two types of threads in Java: **daemon threads** and **user threads**.

#### Daemon thread

*Daemon threads* are service providers to the user threads.

They are created by the JVM. Their life depend on user threads, they are low priority, and they're used to perform garbage collection and other housekeeping tasks. The JVM will not wait for daemon threads to finish their execution.

#### User thread

User threads are created by the user or the application. hey are high priority and the JVM will wait until they have finished their tasks.

# Monitoring Java with Sysdig: A use case

Imagine you have a container running a Java application and you need to know what threads are running but you don't have any JMX being exported by the JRE and the container is isolated. With [Sysdig][13], you can monitor what's going on within the container even with zero previous configuration.

If you don't have any Java containers running at the moment, you can use the following example:

    docker run -d tembleking/java-thread-names
    

Once this container is running in the background, you can launch a terminal inside a new Sysdig container:

    docker run -i -t --name sysdig --privileged -v /var/run/docker.sock:/host/var/run/docker.sock -v /dev:/host/dev -v /proc:/host/proc:ro -v /boot:/host/boot:ro -v /lib/modules:/host/lib/modules:ro -v /usr:/host/usr:ro sysdig/sysdig
    

Once you get the terminal, you can use the Sysdig cli:

    sysdig -c topprocs_cpu container.image = tembleking/java-thread-names
    

All the running threads with their names will now be displayed on the screen:

## Conclusion

The Java Runtime Environment is the most fundamental source of information for monitoring Java—especially inside Docker and Kubernetes where our applications need to stay lightweight so they can be quickly destroyed, recreated, or transferred to a different node.

Sysdig technology, both in its [open source][14] and [commercial][15] versions can easily monitor the JRE. It doesn't take any instrumentation on the Java application side—just an inspection of the kernel metadata and events. While this is already quite interesting for collecting a static set of metrics, [autodiscovery of dynamically appearing JRE and JMX metrics][16] is even more interesting (as in less work and human errors). Learn more about what system-level observability can do for you and find some cool troubleshooting examples [here][17].

In the next chapter of our "Ultimate guide to monitoring Java inside Docker and Kubernetes", we will shift from the foundational details of the Java Virtual Machine to the application layer, making use of Java Management Extensions and Managed Beans.

 [1]: https://478h5m1yrfsa3bbe262u7muv-wpengine.netdna-ssl.com/wp-content/uploads/2018/09/MonitorJavaHeapSpace.png
 [2]: https://en.wikipedia.org/wiki/Stack-based_memory_allocation
 [3]: https://en.wikipedia.org/wiki/Just-in-time_compilation
 [4]: https://en.wikipedia.org/wiki/Memory_management#HEAP
 [5]: https://en.wikipedia.org/wiki/Memory_leak
 [6]: https://en.wikipedia.org/wiki/Garbage_collection_(computer_science)
 [7]: https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html
 [8]: https://docs.oracle.com/javase/8/docs/api/index.html?java/lang/OutOfMemoryError.html
 [9]: https://docs.oracle.com/javase/specs/jvms/se8/jvms8.pdf
 [10]: https://en.wikipedia.org/wiki/Tracing_garbage_collection#Stop-the-world_vs._incremental_vs._concurrent
 [11]: https://en.wikipedia.org/wiki/Concurrent_mark_sweep_collector
 [12]: https://en.wikipedia.org/wiki/Mark-compact_algorithm
 [13]: https://sysdig.com/opensource/sysdig/
 [14]: https://sysdig.com/opensource/
 [15]: https://sysdig.com/
 [16]: https://sysdigdocs.atlassian.net/wiki/spaces/Monitor/pages/204734661/Integrate+JMX+Metrics+from+Java+Virtual+Machines
 [17]: https://github.com/draios/sysdig/wiki/Sysdig-Examples