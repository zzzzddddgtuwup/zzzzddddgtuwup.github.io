---
layout: post
title: "disruptor"
date: 2016-06-11 01:46:44 -0700
comments: true
categories: 
---

本文是视频笔记及思考，具体视频请看[bittiger 聊聊Disruptor](https://www.bittiger.io/videos/QHkP5QobvhNWGZv9f/2EkqDyjCYc5x8WPfW).

## 问题
在金融行业, "How to handle the high volume incoming data in minimum latency?" 

latency是由什么决定的呢？End to end latency = computational time + latency of moving data. 

## 1 方法一 blocking queue
我们可以使用blocking queue来解决这个问题，producer写入queue的头部，consumer从queue的尾部读取数据。

{% img /images/disruptor/blockqueue.png %}

一切似乎看上去很美好，实际上producer和consumer的速度是不同的，经常queue要么全满要么全空。Blocking queue的缺点主要是以下两点：

  1. 竞争存在. 
    * 多个producer竞争写头部指针。
    * consumers虽然是读操作，但是需要用到queue的poll操作，也会是写竞争。
    * 正如之前所说，queue经常是全空状态，头指针和尾指针是同一个，于是造成更多竞争。

    这时候需要一个锁来进行synchronization，但是引入锁会造成性能问题。
  2. False sharing. 

###1.1 什么是false sharing?
{% img /images/disruptor/falsesharing.png %}
如图所示，有两个变量a,b分别是13，14。进行了如下操作：

  1. Core1把a修改为21，Core2的cache line就无效了,Core2只能重新更新cache
  2. Core2把b修改为23，又把Core1的cache1弄无效了,Core1只能更新cache

这样来来回回会不断地从下一层的cache或者内存去更新数据，造成性能问题。放到blocking queue的应用环境中，头指针和尾指针一般是定义在一起的，很有可能在同一个cache line。同时又有很多的生产者和消费者在不同的core上运行，一个访问头指针，一个访问尾指针，就会造成false sharing。

https://en.wikipedia.org/wiki/False_sharing

http://ifeve.com/falsesharing/

###1.2 什么是cache line？
缓存系统中是以缓存行（cache line）为单位存储的。缓存行是2的整数幂个连续字节，一般为32-256个字节。最常见的缓存行大小是64个字节。有人做过[实验](http://cenalulu.github.io/linux/all-about-cpu-cache/)，从命令行接收一个参数作为数组的大小创建一个数量为N的int数组。并依次循环的从这个数组中进行数组内容访问，循环10亿次。最终输出数组总大小和对应总执行时间。然后发现在数组大小超过64Bytes的时候总执行时间有明显增加。这是因为当数组小于64的时候，数组极有可能落在一条Cache Line内，而一个元素的访问就会使得整条Cache Line被填充，因而值得后面的若干个元素受益于缓存带来的加速。而当数组大于64Bytes时，必然至少需要两条Cache Line，继而在循环访问时会出现两次Cache Line的填充，由于缓存填充的时间远高于数据访问的响应时间，因此多一次缓存填充对于总执行的影响会被放大

###1.3 为什么用锁会造成性能问题？
{% img /images/disruptor/lockperf.png %}
如图所示，一开始Thread1在左边的core上运行，然后碰到了一个锁，它只能交出CPU然后去sleep。这时候Thread2占了左边的Core，发现cache全都没用就清空了。过一会Thread1的锁释放了，于是Thread1可以继续运行，但是左边的Core被占了，只能用右边的Core。此时右边的Core cache是空的，需要重新读cache。这里对cache的反复操作造成了性能的下降。

##2 方法2 Disruptor！！！！
###2.1 设计原则
- Single writer principle
- No locks
- Avoid false sharing

### 2.2 Disruptor定义
一句话，circular array with a sequence number.

### 2.3 如何解决之前的问题？
false sharing: 使用padding技术，在sequence number前面加7个long占位，这样一个sequence number可以占一个cache line.
Single writer解决写竞争: 
  * 如果写到了队列尾部，就overwrite之前的数据，继续增加sequence number。
  * 使用two phase commit
  * 整个queue上的对象都是提前分配好的，这样不会有GC问题（用java实现的）。
环形数组直接解决了多个Consumer写竞争。
去除了锁的设计。

### 2.4 多种情况下的Disruptor
####2.4.1 只有producer
{% img /images/disruptor/single_producer.png %}

####2.4.2 一个producer和一个consumer
{% img /images/disruptor/single_producer_consumer.png %}

####2.4.2 一个producer和多个consumer
{% img /images/disruptor/single_producer_multiconsumer.png %}

### 2.5 如何去除lock? disruptor如何实现synchronization？
1. consumer是在等待producer，每次读下一个之前要确定这个sequence number是否准备好。如果producer很慢，consumer会一直跟在producer后面。等待的策略比较多，一种就是busy spin。
2. Producer也是在检查sequence number，防止写了consumer没有读的数据。这里原理还不懂。。。。。
3. Memory barrier

####2.5.1 Memory barrier
CPU会对指令重排，这个会造成问题。Disruptor要求最后再update cursor里面的sequence number。如果CPU把写entry和写sequence number的顺序打乱，就会造成数据还没有准备好，sequence number已经改变，consumer就去读数据的情况。

Memory barrier可以阻止重排，一般java使用的是volatile字段，每次遇到这样的变量的赋值，就立马写出去。但是会降低效率。

### 2.6 新的疑问
#### 2.6.1 遇到很慢的consumer怎么办？ 
1.Batching effect！2.把buffer变大。。3.说明producer太快或者consumer太慢，需要重新考虑设计，比如分流producer，多个Disruptor。
{% img /images/disruptor/batch_effect.png %}
#### 2.6.2 我有多个producer怎么办？
{% img /images/disruptor/multi_producer.png %}
hints:CAS是神马？Compare and swap，是一种原子级的数据更新方法。先比较是不是原来的数据，是的话就更新。不是的话再考虑其他。这里看上去compare和swap是两步，实际上是机器里面会把bus锁住，所以其它线程也无法访问，因此不会有race condition。看上起CAS是没有锁，其实只是锁的级别非常低，而且没有context的切换，所以性能好。
#### 2.6.3 什么时候用disruptor？
1.single writer， multiple consumer 2.延迟要求很高，CPU不是很在乎，毕竟要busy wait

## Reference
1. 源码 https://lmax-exchange.github.io/disruptor
2. 论文 http://disruptor.googlecode.com/files/Disruptor-1.0.pdf
3. 博客 http://bad-concurrency.blogspot.com   https://mechanitis.blogspot.com
4. Latency Numbers Every Programmer Should Know http://www.eecs.berkeley.edu/~rcs/research/interactive_latency.html
