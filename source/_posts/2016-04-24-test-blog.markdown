---
layout: post
title: "Java 8 并行"
date: 2016-04-24 00:50:29 -0700
comments: true
categories: 
---

最近在读OReilly的java 8 Lambdas这本书，边读边做笔记吧。这篇是对chapter 6的data parallelism的总结。

在java8中提供了stream的并行，减少了之前对线程池的部分需求。

首先，需要搞清楚parallelism和concurrency的区别。parallelism是并行，指的是多核同时执行任务。concurrency是并发，只有一个核，多个任务被分到不同的time slice上。 parallelism实际上包括很多种，比如task parallelism，data parallelism。Stream中实现的是data parallelism。也就是你有一个很大的数据，然后每个线程处理其中一部分。

但是使用并行是否能够提高效率是看情况的，盲目地使用反而没有串行速度快。以下五个因素影响了并行的速度：

1. 数据大小。在分解问题和汇合结果的时候都是有overhead的。
2. 源数据的结构。一般是把源数据平均的分配给各个可用核的，如果数据结构不好均分就会很慢。可以把常见的数据结构分成以下几类：
    * Good: ArrayList, array, IntStream.range constructor。
    * Okay: HashSet, TreeSet
    * Bad: LinkedList, Streams.iterate, BufferedReader.lines
3. Packing。基本数据类型比boxed的快。
4. 可用核的数量。注意这里是可用核，如果你的程序运行的时候有其他程序在跑，自然可以用到的核就会少。
5. 每个元素的操作花费。如果元素的操作花销越大，那么overhead所带来时间的比重就越小。这时候使用并行效果就会好。
6. stream的具体操作。stateless比stateful的操作快，因为stateful需要有overhead并且要维护状态。stateless操作有map, filter, flatmap。stateful的操作有sorted，distinct和limit。

除了stream上有并行，array中也提供了一些并行的方法：
* parallelPrefix. 
* parallelSetAll. 可以用来初始化。
* parallelSort.


