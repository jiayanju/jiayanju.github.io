---

title: "RocketMQ消费者线程“大爆炸”" 

date: 2020-09-15T11:30:03+00:00 

categories: ["消息队列"]

tags: ["RocketMQ", "消息队列"] 

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

​	image: "https://raw.githubusercontent.com/jiayanju/imgrepo/main/bud.jpg"

​	alt: ""

​	caption: ""

​	relative: false

​	hidden: false

---



### 1. 问题描述

之前由于本地的consumer项目启动缓慢，查看了consumer的线程，发现consumer会创建很多的线程，多达300多个，每个RocketMQMessageListener

都会对应的开启一个PushConsumer并启动底层的相关的线程。例如下图中的PullMessageService线程的数量和RocketMQMessageListener的数量是对应的。

### ![](https://raw.githubusercontent.com/jiayanju/imgrepo/main/weifeng%E7%BA%BF%E7%A8%8B.png)

### 2. 问题分析

通过查看RocketMQ的spring boot代码和client的相关代码，每个PushConsumer公用相同的底层线程服务，不需要每个PushConsumer都启动一套。下图是简单总结的相关的组件图

![](https://raw.githubusercontent.com/jiayanju/imgrepo/main/RocketMQ%E6%B6%88%E8%B4%B9%E7%AB%AF%E7%BB%84%E4%BB%B6%E5%9B%BE.png)

有上图可知MQClientInstance对同一个clientId是个单例，MQClientInstance初始化出来的服务线程是所有Consumer实例共享的。因此项目里面的PullMessageService等线程服务不应该这么多。

### 3. 问题原因挖掘

1）首先，找出MQClientInstance创建的代码，

org.apache.rocketmq.client.impl.MQClientManager#getOrCreateMQClientInstance(org.apache.rocketmq.client.ClientConfig, org.apache.rocketmq.remoting.RPCHook)

![](https://raw.githubusercontent.com/jiayanju/imgrepo/main/getOrCreateMQClientInstance.png)



**可以看出同一个clientId对应一个MQClientInstance。**

2）下面分析一下clientId怎么获取的

org.apache.rocketmq.client.ClientConfig#buildMQClientId

![](https://raw.githubusercontent.com/jiayanju/imgrepo/main/buildMQClientId.png)

**可以看出主要靠instanceName区分，因为clientIP是一样的，unitName没有设置**

3）那么instanceName是如何设置的呢

经过查找，是在Consumer初始化的设置的

org.apache.rocketmq.spring.support.DefaultRocketMQListenerContainer#initRocketMQPushConsumer

![](https://raw.githubusercontent.com/jiayanju/imgrepo/main/setInstanceName.png)

**可以看出，instanceName是从RocketMQUtil.getInstanceName获取的。**

4）查看RocketMQUtil.getInstanceName的代码

![](https://raw.githubusercontent.com/jiayanju/imgrepo/main/getInstanceName.png)

**identify传的是consumerGroup，因此每个consumerGroup都会创建一个MQClientInstance，这是和RocketMQMessageListener注解吻合的。**

### 4. 问题怎么解决？

查看RocketMQ spring boot的github源码发现，master已经进行了修复，目前已经进入pre-release, 那么是怎么修改的呢

https://github.com/apache/rocketmq-spring/issues/289

https://github.com/apache/rocketmq-spring/blob/master/rocketmq-spring-boot/src/main/java/org/apache/rocketmq/spring/support/DefaultRocketMQListenerContainer.java#L565

![](https://raw.githubusercontent.com/jiayanju/imgrepo/main/newSetInstanceName.png)

由上面的代码可以看出，在使用ACL的情况下删除了设置instanceName的代码，改用nameServer来作为identity获取instanceName。



------



新的rocketmq-spring-boot-starter已经release，版本2.2.0, 已经解决了这个问题

![](https://raw.githubusercontent.com/jiayanju/imgrepo/main/rocketmq_consumer_threads.png)

由上面图片可知，线程数已经降到了一百多。PullMessageService和RebanlanceService的数量已经降下来了。

。。。额。。。但是为什么是3个而不是1个捏？1个就可以了啊

**原因：**

不管producer还是consumer都会初始化和使用MQClientInstance，MQClientInstance会初始化出PullMessageService和RebalanceService线程，其实对于producer，这俩个线程是没用的。对于MQClientInstance是区分不出来producer还是consumer。

这里其他2个是有默认producer和trace producer创建的。