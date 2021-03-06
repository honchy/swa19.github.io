---
layout: post
title:  "JAVA线程池"
date:   2017-09-06 17:01:38 +0800
categories: 基础
tags: java
---

* TOC
{:toc}

# 什么是线程池
线程池可以认为是一个存放当前可用线程的list，当需要新建一个线程的时候，不会新建线程，而是先检查线程池中是否有可用线程，如果有的话，从线程池中取出线程执行当前的任务。否则等待。那么为什么需要一个线程池来管理线程呢？在服务器端的服务中，有些线程的创建和消耗是比较耗费资源的，如处理web请求，数据库连接，文件IO操作等等，如果每个请求的到来都为它创建一个线程，那么在并发量比较大的情况下，对系统的压力可想而知。  
为了降低这种线程创建和销毁的资源消耗，采用了线程池的方案，通过这种方式实现了线程的复用，降低服务器资源消耗。

# 线程池对线程的管理  
线程池实现了对池内线程的管理。当请求尚未到来时，线程池就会创建一批线程放入到空闲线程的列表中。这些线程尚未分配任务，所以只占用一部分内存，不会消耗CPU资源。当有任务到来时，会通过线程池获取一个空闲线程，然后通过这个空闲线程来执行当前任务。

# 线程池的相关概念
* 核心线程数
核心线程数是线程池初始化之后，预先创建的线程个数　　
* 最大线程数　　
最大线程数要大于或者等于核心线程数。当核心线程不够用的时候，会根据最大线程数继续创建新的线程。要注意的是，这里的线程只有当核心线程数不够用的时候才会创建新线程。那么如果当前线程数达到了最大线程数呢？这个时候如果继续添加任务，这些任务根据处理的不同而不同，一般而言会被丢弃掉，或者把任务放在一个任务队列中，等待空闲线程。　　
* 线程空闲时间　　
当线程池中的线程数大于核心线程数，且这些线程的空闲时间达到了设置的线程空闲时间，这些多出的线程将会被销毁掉。所以这里的最大线程可以理解为一种应急机制。不会

# 线程池的设计
典型的线程池包含以下几个部分：  
1. 线程池管理器，用来启动、停用、管理线程池  
2. 工作线程：线程池中的可用线程  
3. 请求接口:创建请求对象，以供工作线程调度任务的执行　　
4. 请求队列:用于存放和提取请求
5. 结果队列:用于存储请求执行后返回的结果
 

以java的ThreadPoolExecutor类为例，
`private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));`  
用来控制线程池的运行状态，包括RUNNING,SHUTDOWN,STOP,TIDYING,TERMINATED这五种，那么每次在对线程池做操作，如添加任务，终止线程的时候，需要通过ctl来检查当前线程池的运行状态，所以这个变量实现了线程池的管理功能。　　

`private final HashSet<Worker> workers = new HashSet<Worker>();`  
workers是当前线程池中所有的工作线程，需要注意的是，在访问这个变量的时候需要先获取到锁，即`private final ReentrantLock mainLock = new ReentrantLock();`  





