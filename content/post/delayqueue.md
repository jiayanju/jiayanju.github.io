---
title: "延迟队列的实现原理和使用" 

date: 2021-03-20T12:00:00+08:00 

categories: ["Redis"]

tags: ["Redis"] 

author: "Me" 

showToc: true

TocOpen: false 

draft: false 

hidemeta: false 

comments: false 

description: "" 

disableHLJS: true  

disableShare: false 

cover:
  image: "https://raw.githubusercontent.com/jiayanju/imgrepo/main/delayQueue.jpg"
  alt: ""
  caption: ""
  relative: false
  hidden: false





---

目前项目中使用的延迟消息发送是基于Redis的键空间通知，监听Redis Key的失效过期事件来实现的。这种方式可以满足延迟发送的功能需求，但是有如下的问题。

1. 时效性问题。Redis过期Key的扫描处理机制是分批进行处理，每次扫描一部分key判断是否过期。因此在redis的key比较多（数量上万）时会有延迟。
2. 键空间通知依赖于pub/sub模式，消息通知不能持久化。如果服务宕机或者客户端断开连接，在这段期间失效key的通知就会丢失。

考虑到系统中已经使用延迟任务的功能已经后续的新功能也会有这方面的需求，因此在开发超时回复功能时，实现了一个基于redis的延迟队列功能。

### 1. 延迟队列架构

首先看一下架构图：

![](https://raw.githubusercontent.com/jiayanju/imgrepo/main/delayqueue.png)

主要有下面几个部分构成

1. Bucket - 基于Redis的有序集合(Sorted Set)，score为到期的时间戳，值为jobId。使用SortedSet的rangeByScore找出到期的job，bucket可以有多个，以达到均衡的处理大量的job，防止单个bucket的过大造成的阻塞。
2. Job Pool - 基于Redis的Hash结构，存放延迟任务job的元信息，包括任务体，topic等信息。
3. Ready Queue - 基于Redis的List结构。把已经到期要执行的jobId放入队列里面，供消费者去处理。
4. Bucket Handler Thread - Java线程不断的轮询bucket，获取到期的jobId并放入相应的Ready Queue里面。

整个架构还是挺简单明了的。

### 2. 延迟队列的实现代码简单解释

#### DelayJob类

```java
public class DelayJob {

  /**
   * Job的唯一标识。用来检索和删除指定的Job信息
   */
  private Long id;

  /**
   * 延迟任务的业务标识类型，不同的业务可以使用不同的topic进行区分
   */
  private String topic;

  /**
   * Job需要延迟的时间。单位：秒
   */
  private Long delay;

  /**
   * Job的内容，供消费者做具体的业务处理，以json格式存储。
   */
  private String body;

}
```



#### DelayQueue类

延迟队列的接口类，主要使用这个类来操作延迟队列。封装了一下基本的操作，例如push，pop，remove等函数

```java
/**
 * 向延迟队列添加延迟任务
 *
 * @param delayJob
 * @return
 */
public Long push(DelayJob delayJob) {
  Long runTime = System.currentTimeMillis() + (delayJob.getDelay() * 1000);   
  //生成唯一ID
  Long jobId = snowFlakeIdGenerator.nextId();
  delayJob.setId(jobId);
  delayJobPool.addDelayJob(delayJob);
  delayBucket.addToBucket(getBucketName(jobId), jobId, runTime);

  return jobId;
}
```

这个方法主要做了一下工作：

计算到期时间

生成唯一的jobId，这里使用了snowflake算法

把DelayJob放入Job Pool

把延迟任务的jobId放入bucket

```java
/**
 * 获取延迟队列中已经准备好的任务
 *
 * @param topic
 * @return
 */
public DelayJob pop(String topic) {
  DelayJob result = null;
  Long jobId = readyJobQueue.poll(topic);
  if (jobId != null) {
    result = delayJobPool.getDelayJob(jobId);
  }

  return result;
}
```

pop方法，主要是从ReadyQueue中获取要处理的延迟任务jobId，然后从Job Pool中返回延迟任务的信息。

#### DelayBucket类

```java
public void addToBucket(String bucketName, Long jobId, Long timestamp) {
  redisTemplateLong.opsForZSet().add(getDelayBucketKey(bucketName), jobId, timestamp.doubleValue());
}
```

添加延迟任务的JobId到bucket中，以到期的timestamp为score，jobId为值

```java
public List<Long> getExpiredJobFromBucket(String bucketName) {
  List<Long> jobIdList = new ArrayList<>();
  long now = System.currentTimeMillis();
  Set<Long> jobIds = redisTemplateLong
      .opsForZSet().rangeByScore(getDelayBucketKey(bucketName), 0, now);
  if (jobIds != null && jobIds.size() > 0) {
    jobIdList = jobIds.stream().collect(Collectors.toList());
  }
  return jobIdList;
}
```

使用rangeByScore获取到期的任务Id并返回。

#### DelayQueueHandler

轮询bucket，并将到期的job放入ready queue里面

```java
while (true) {
  RedisDistributeLock lock = new RedisDistributeLock(redisTemplate, DELAY_BUCKET_PROCESS_LOCK_KEY + bucketName);
  String lockId = null;
  try {
    lockId = lock.lock();
    List<Long> jobIds = delayBucket.getExpiredJobFromBucket(bucketName);
    if (jobIds != null && jobIds.size() > 0) {
      List<DelayJob> delayJobs = delayJobPool.getDelayJobs(jobIds);
      if (delayJobs != null && delayJobs.size() > 0) {
        Map<String, List<Long>> topicJobIdMap = delayJobs.stream().collect(
            Collectors.groupingBy(DelayJob::getTopic, Collectors.mapping(DelayJob::getId, Collectors.toList())));
        topicJobIdMap.forEach((topic, jobIdList) -> {
          readyJobQueue.pushAll(topic, jobIdList);
        });
      }
      delayBucket.batchDeleteFromBucket(bucketName, jobIds);
    }
  } catch (Exception e) {
    log.error(e.getMessage(), e);
  } finally {
    try {
      lock.unlock(lockId);
    } catch (Exception e) {
      log.error(e.getMessage(), e);
    }
  }

  sleep();
}
```

delayBucket.getExpiredJobFromBucket(bucketName)  - 返回到期的延迟任务列表

delayJobPool.getDelayJobs(jobIds) - 从Job Pool返回延迟任务信息

接下来把到期的延迟任务放入起设置的topic对应的ReadyQueue里面。

最后从bucket里面删除任务Id

### 3. 延迟队列使用

#### 添加延迟任务

```java
DelayJob delayJob = new DelayJob();
delayJob.setTopic(ChatRecordConstants.DELAY_QUEUE_REPLAY_TIMEOUT_TOPIC);
delayJob.setDelay(timeoutSeconds);
delayJob.setBody(jobBody);
Long jobId = delayQueue.push(delayJob);
```

生成DelayJob，需要设置topic，延迟时间和消息体。然后调用delayQueue.push(delayJob)放入延迟队列里面。返回这个延迟任务的唯一标识jobId。如果后续业务逻辑需要取消这个延迟任务，需要通过jobId来讲延迟任务从队列里面删除。在这种情况下需要记录存储jobId。

topic可以根据业务逻辑生成不同的topic，来达到多队列多线程的消费消息。

#### 消费延迟任务

```java
@Component
@Slf4j
@AllArgsConstructor
public class DelayJobMessageConsumer implements Runnable, ApplicationListener<ApplicationReadyEvent> {

  private static final String DELAY_QUEUE_CONSUMER_TOPIC_LOCK = "delay:queue:consumer:topic:";

  private final Executor consumerExecutor;

  private final DelayQueue delayQueue;

  private final RedisTemplate<String, String> redisTemplate;

  @Override
  public void onApplicationEvent(ApplicationReadyEvent event) {
    consumerExecutor.execute(this::run);
  }

  @Override
  public void run() {
    log.info("DelayJobMessageConsumer thread started");
    while (true) {
      RedisDistributeLock lock = new RedisDistributeLock(redisTemplate,
          DELAY_QUEUE_CONSUMER_TOPIC_LOCK + ChatRecordConstants.DELAY_QUEUE_REPLAY_TIMEOUT_TOPIC);
      String lockId = null;
      DelayJob delayJob = null;
      try {
        lockId = lock.lock();

        //从ReadyQueue获取延迟任务
        delayJob = delayQueue.pop(ChatRecordConstants.DELAY_QUEUE_REPLAY_TIMEOUT_TOPIC, 5, TimeUnit.SECONDS);

        log.debug("DelayJobMessageConsumer get job from queue returned: {}",
            ChatRecordConstants.DELAY_QUEUE_REPLAY_TIMEOUT_TOPIC);
      } catch (Exception e) {
        log.error("DelayJobMessageConsumer get job from queue {} exception: {}",
            ChatRecordConstants.DELAY_QUEUE_REPLAY_TIMEOUT_TOPIC, e.getMessage(), e);
      } finally {
        try {
          lock.unlock(lockId);
        } catch (Exception e) {
          log.error("DelayJobMessageConsumer redis unlock error happened. {}", e.getMessage(), e);
        }
      }

      //process delayJob
      if (delayJob != null) {
        try {
          //对任务进行处理

          // 将任务从Job Pool删除
          delayQueue.finish(delayJob.getId());
        } catch (Exception e) {
          log.error("DelayJobMessageConsumer process delay job error. {}", e.getMessage(), e);
        }

      }
    }
  }
}
```

消费者线程轮询从ReadyQueue里面获取要执行的延迟任务，并进行处理。

里面使用了分布式锁是考虑到将来可能会集群部署消费者服务时，保证只有一个线程进行消费处理。

### 4. 总结

目前实现了延迟队列的基本功能，后续使用过程可能还需要进行一些功能调整，比如对消费失败的消息放入死信队列等。