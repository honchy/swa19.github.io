---
layout: post
title: "开发遇到过的JAVA坑"
date: 2017-09-07 16:49:48 +0800
categories: 基础
tags: java
---

# 循环中的continue和break
前几天出的一个线上问题,符合条件的数据有3000条,但实际插入到数据库的只有两条.最后查看代码发现这批数据通过for循环插入,而在这个循环中,一个判断条件中增加了`break`导致循环中断.
所以对于循环中出现break和continue的,一定要重点检查下是否符合业务逻辑.


# RocketMqsubject和tag配置问题导致消息重复发送
在A系统中会发送给B系统一个RocketMq消息，B系统在接收到这个消息之后，发送消息给C系统  
根据日志，ProducerA发送了一条消息，但是到ConsumerB的之后，首先接收到和Producer发送消息一样的msgId的消息，然后不断刷日志，显示ConsumerB持续不断地接收到同样消息内容，但是msgId不同的消息。  
开始因为初次配置使用RocketMq，怀疑RocketMq的处理机制问题，但其他系统应用不存在这个问题；后来在梳理业务逻辑的时候，发现C系统接收的是来自A系统的消息，而这个是不合理的，需要对C系统和B系统接收消息的tag做一个区分，由此才意识到，原来B系统一直在接收消息，然后发送消息，发送的消息由于subject和tag配置的问题，B系统会接收到自己发送的消息并处理，由此，一个死循环导致了消息被不断地重复发送和接收和处理  
遇到问题而不得其解的时候，不要怀疑人生。有句话说：事出反常必有妖

# java引用
对变量做临时赋值后，后续在用的时候没有恢复成原有的值   
由于java中的变量传递的是变量的引用，所以在方法内对一个变量做的改变在这个方法调用结束之后会继续保留，这个错误很弱智，但也很容易被忽略

# static变量
对于一个类中定义的static变量，这个变量在多个类实例中共用。看一个类的定义:

~~~
public class ScheduleJobList {
    private static final Logger logger = LoggerFactory.getLogger(ScheduleJobList.class);
    private  static PriorityBlockingQueue<JobInfoWrapper> list;
}
~~~

使用时：

~~~
    public static void addRegisteredJob(String jobInfoStr) {
        Map<JobInfo, ScheduleJobList> jobMap = new ConcurrentHashMap<JobInfo, ScheduleJobList>();
        JobScheduleParser.ScheduleMsgBody msg = JSON.parseObject(jobInfoStr, JobScheduleParser.ScheduleMsgBody.class);
        ScheduleJobList scheduleJobList = new ScheduleJobList();
        scheduleJobList.addJob(JobInfoWrapper.newWrapperJob(msg.getNewJob()));//将当前需要执行的任务添加到任务列表中
        jobMap.remove(msg.getOldJob());
        jobMap.put(msg.getNewJob(), scheduleJobList);
    }
~~~

问题是，当我对一个JobInfo实例的ScheduleJobList做了dequeue操作，其他JobInfo实例的ScheduleJobList中的数据也会减少，一开始百思不得其解，直到看ScheduleJobList类的时候注意到static关键词，才恍然大悟。
static关键词修饰的静态变量只会保存在一个内存空间，当需要它在各个类中不共享的时候，坑就出现了。



