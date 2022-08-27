# RocketMQ源码深入剖析

## 1 RocketMQ介绍

RocketMQ 是阿里巴巴集团基于高可用分布式集群技术，自主研发的云正式商用的专业消息中间件，既可为分布式应用系统提供异步解耦和削峰填谷的能力，同时也具备互联网应用所需的海量消息堆积、高吞吐、可靠重试等特性，是阿里巴巴双 11 使用的核心产品。

### 1.1 RocketMQ版本发展

如果想要了解RocketMQ的历史，则需了解阿里巴巴中间件团队中的历史。

2011年，Linkin(领英：全球知名的职场社交平台)推出Kafka消息引擎，阿里巴巴中间件团队在研究了Kafka的整体机制和架构设计之后，基于Kafka(Scala语言编写)的设计使用Java进行了完全重写并推出了MetaQ 1.0版本，主要是用于解决顺序消息和海量堆积的问题，由开源社区killme2008维护。课程重点不在此版本，具体见：[https://github.com/killme2008/Metamorphosis](https://github.com/killme2008/Metamorphosis)

2012年，阿里巴巴发现MetaQ原本基于Kafka的架构在阿里巴巴如此庞大的体系下很难进行水平扩展，于是对MetaQ进行了架构重组升级，开发出了MetaQ 2.0，同年阿里把Meta2.0从阿里内部开源出来，取名RocketMQ，为了命名上的规范以及版本上的延续，对外称为RocketMQ3.0。因为RocketMQ3只是RocketMQ的一个过渡版本，课程重点也不在此。

2016年11月28日，阿里巴巴宣布将开源分布式消息中间件RocketMQ捐赠给Apache，成为Apache 孵化项目。在孵化期间，RocketMQ完成编码规约、分支模型、持续交付、发布规约等方面的产品规范化，同时RocketMQ3也升级为RocketMQ4。现在RocketMQ主要维护的是4.x的版本，也是大家使用得最多的版本,所以本书重点将围绕此版本进行详细的讲解，项目地址：[https://github.com/apache/rocketmq/](https://github.com/apache/rocketmq/)

2015年，阿里基于RocketMQ开发了阿里云上的Aliware MQ，Aliware MQ(Message Queue)是RocketMQ的商业版本，是阿里云商用的专业消息中间件，是企业级互联网架构的核心产品，基于高可用分布式集群技术，搭建了包括发布订阅、消息轨迹、资源统计、定时（延时）、监控报警等一套完整的消息云服务。因为Aliware MQ是商业版本，课程也不对此产品进行讲述，产品地址：[https://www.aliyun.com/product/rocketmq](https://www.aliyun.com/product/rocketmq)

2021年，伴随众多企业全面上云以及云原生的兴起，RocketMQ也在github上发布5.0版本。目前来说还只是一个预览版，不过RocketMQ5的改动非常大，同时也明确了版本定位，RocketMQ 5.0定义为云原生的消息、事件、流的超融合平台。本课程也将会根据目前所发布的版本进行针对性的讲述。

#### 1.1.1 RocketMQ4.X版本更新概要

* 在 RocketMQ 4.3.0 版本之后，正式发布事务消息，通过类似于两阶段的方式去解决上下游数据不一致问题。
* 在 RocketMQ 4.4.0 版本中，RocketMQ 增加了消息轨迹的功能，使用户可以更好定位每一条消息的投放接收路径，帮助问题排查，另外还增加 ACL 权限控制，提高了 RocketMQ 的管控能力和安全性。
* 在 4.5.0 版本中，RocketMQ 推出了多副本，也就是 Raft 模式。在 Raft 模式下，一组 Broker 如果 Master 挂了，那么 Broker 中其他 Slave 就会重新选出主。因此 Broker 组内就拥有了自动故障转移的能力，也解决了像高可用顺序消息这样的问题，进一步提高了 RocketMQ 的可用性。
* 在 4.6.0 版本中，我们推出了轻量级 Pull Consumer，用户可以使用更加适合于流计算的 API，这一版本也开始支持全新的 Request-Reply 消息，使得 RocketMQ 具备了同步调用 RPC 的能力，RocketMQ 可以更好的打破网络隔离网络之间的调用，这个版本中 RocketMQ 也开始支持 IPV6，并且是首个支持 IPV6 的消息中间件。
* 在 4.7.0 版本中，RocketMQ 重构了主备同步复制流程，通过线程异步化，将同步复制和刷盘的过程 Pipeline 化，同步双写性能有将近数倍提升。
* 在 4.8.0 版本中，RocketMQ Raft 模式有了一个质的提升，包括通过异步化、批量复制等手段将性能提升了数倍，在稳定性上利用 OpenChaos 完成包括宕机、杀死进程，OOM、各种各样的网络分区和延迟的测试，修复了重要 Bug。在功能上，支持 Preferred Leader，从而 Broker 组内可以优先选主，也支持了批量消息等功能。
* 在 4.9.0 版本，主要是提升了可观测性，包括支持 OpenTracing，事务消息和 Pull Consumer 支持 Trace 等功能。

### 1.2 为什么要学RocketMQ源码

* 编写优雅、高效的代码经验

  RocketMQ作为阿里双十一交易核心链路产品，支撑千万级并发、万亿级数据洪峰。读源码可以积累编写高效、优雅代码的经验。
* 提升微观的架构设计能力，重点在思维和理念

  Apache RocketMQ作为Apache顶级项目，它的架构设计是值得大家借鉴的。
* 解决工作中、学习中的各种疑难杂症

  在使用RocketMQ过程中遇到消费卡死、卡顿等问题可以通过阅读源码的方式找到问题并给予解决。
* 在BATJ一线互联网公司面试中展现优秀的自己

  大厂面试中，尤其是阿里系的公司，你有RocketMQ源码体系化知识，必定是一个很大的加分项。

### 1.3 RocketMQ源码中的技术亮点

* 读写锁
* 原子操作类
* 文件存储设计
* 零拷贝：MMAP
* 线程池
* ConcurrentHashMap
* 写时复制容器
* 负载均衡策略
* 故障延迟机制
* 堆外内存

## 2 RocketMQ核心组件

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1507273036154273792/8cb8104fcb4547c7ba23ba6ad33ca74c.png)

**NameServer**

命名服务，更新和路由发现 broker服务。NameServer的作用是为消息生产者、消息消费者提供关于主题 Topic 的路由信息，NameServer除了要存储路由的基础信息，还要能够管理 Broker节点，包括路由注册、路由删除等功能。

**Producer/Consumer**

java版本的MQ客户端实现，包括生产者和消费者。

**Broker**

它能接收producer和consumer的请求，并调用store层服务对消息进行处理。HA服务的基本单元，支持同步双写，异步双写等模式。

**Store**

存储层实现，同时包括了索引服务，高可用HA服务实现。

**Netty Remoting Server/Netty Remoting Client**

基于netty的底层通信实现，所有服务间的交互都基于此模块。也区分服务端和客户端

## 3 RocketMQ源码下载及安装

### 3.1 下载地址

官方下载地址：[http://rocketmq.apache.org/dowloading/releases/]()

本课程使用的是4.8.0的版本

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/2d34a9ff7fc344aca68c09d08664bff1.png)

### 3.2 环境要求

* Linux64位系统
* JDK1.8(64位)
* Maven 3.2.x

### 3.3 使用IntelliJ IDEA导入安装源码

1）使用IDEA导入已经下载且已经解压后的代码

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/e5cff1a2b6c642138e73edac1ae32a82.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/65f9541910204f5d8ca04191e298109b.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/b1a15f4cf35242c893d89705f889c83f.png)![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/c0975117d5014fce92f711c029c65c91.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/06bf35d0d7ae4be0a26310f346f87b31.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/db09010dc3c44224ac09191452b2267f.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/cbf1333639ac45979edcfa49fd9afa9d.png)

2）下载且已经解压后的代码导入后执行Maven命令install：

```javascript
mvn install -Dmaven.test.skip=true
```

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/b0b9281d93904231b7606d807cf361e0.png)

使用Maven验证下没问题

### 3.4 配置与运行RocketMQ

#### 3.4.1 启动NameServer

RocketMQ启动必须先启动NameServer，启动类是namesrv/目录下的NamesrvStartup类，不过在运行这个类之前必须要配置环境变量ROCKETMQ_HOME,变量值为RocketMQ的运行主目录。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/9a67a0fb8e604d158d443c94d4c8f8f7.png)

RocketMQ的运行主目录一般使用新建的方式，同时在运行主目录中创建conf、logs、store三个文件夹，然后从源码目录中distribution目录下的中将broker.conf、logback_broker.xml、logback_namesrv.xml复制到conf目录中。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/c3c4902416b043f486816cf0919657b0.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/ae96e5c4c683446d8acebb1ecf7a2b32.png)

最后运行namesrv/目录下的NamesrvStartup类的main方法，NameServer启动成功！

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/5c546fa37d3b4b2c97e8f05e89e4c72f.png)

#### 3.4.2 启动Broker

在broker模块找到broker模块，同时找到启动类BrokerStartup.java

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/c463febb5961439ca9d878fa7fa25715.png)

源码启动前需要修改配置文件broker.conf （修改RocketMQ的消息存储路径）

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/79b7086fde444091a09cdf61be29f028.png)

配置环境变量，同时启动时需要加入参数（指定启动的配置文件）

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/51496b36d4f24d099f1af0e6be5d4ea9.png)

broker启动成功后的控制台如下：

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/c32d7757b2fe46fdb5dcdc050b62814a.png)

### 3.5 控制台安装及部署

#### 3.5.1 环境要求

运行前确保：已经有jdk1.8，Maven(打包需要安装Maven3.2.x)

#### 3.5.2 下载

老版本地址下载：[https://codeload.github.com/apache/rocketmq-externals/zip/master](https://codeload.github.com/apache/rocketmq-externals/zip/master)

新版本地址：[https://github.com/apache/rocketmq-dashboard](https://github.com/apache/rocketmq-dashboard)

解压后如图(以下使用的是老版本，新版本参考老版本即可)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/22140d31cf7b4659b952f37e3ea83fee.png)

#### 3.5.3 配置

后端管理界面是一个Java工程，独立部署，同时也需要根据不同的环境进行相关的配置。

**控制台端口及服务地址配置：**

下载完成之后，进入‘\rocketmq-console\src\main\resources’文件夹，打开‘application.properties’进行配置。

![](file:///C:\Users\ADMINI~1\AppData\Local\Temp\msohtmlclip1\01\clip_image002.jpg)![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/9764a90684c44a679687e19aea5bfd09.png)

进入‘\rocketmq-externals\rocketmq-console’文件夹，执行‘mvn clean package -Dmaven.test.skip=true’，编译生成运行jar包。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/5120c39440e54968bfb905237304ddc2.png)

编译成功之后，cmd命令进入‘target’文件夹，执行‘java
-jar rocketmq-console-ng-2.0.0.jar’，启动‘rocketmq-console-ng-2.0.0.jar’。

![](file:///C:\Users\ADMINI~1\AppData\Local\Temp\msohtmlclip1\01\clip_image006.jpg) ![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/600df6d4762b47abb0f440f080776270.png)

浏览器中输入‘127.0.0.1:8089’，成功后即可进行管理端查看。

![](file:///C:\Users\ADMINI~1\AppData\Local\Temp\msohtmlclip1\01\clip_image010.jpg)![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/352e872302264d87ab23936c9a4c36ad.png)

## 4 RocketMQ的核心三流程

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/a6b5a50999004bddb6b7480cb06bec32.png)

**整体模块如下：**
**1.rocketmq-namesrv：**命名服务，更新和路由发现 broker服务。
NameServer 要作用是为消息生产者、消息消费者提供关于主题 Topic 的路由信息，NameServer除了要存储路由的基础信息，还要能够管理 Broker节点，包括路由注册、路由删除等功能
**2.rocketmq-broker：**mq的核心。
它能接收producer和consumer的请求，并调用store层服务对消息进行处理。HA服务的基本单元，支持同步双写，异步双写等模式。
**3.rocketmq-store：**存储层实现，同时包括了索引服务，高可用HA服务实现。
**4.rocketmq-remoting：**基于netty的底层通信实现，所有服务间的交互都基于此模块。
**5.rocketmq-common：**一些模块间通用的功能类，比如一些配置文件、常量。
**6.rocketmq-client：**java版本的mq客户端实现
**7.rocketmq-filter：**消息过滤服务，相当于在broker和consumer中间加入了一个filter代理。
**8.rocketmq-srvutil：**解析命令行的工具类ServerUtil。
**9.rocketmq-tools：**mq集群管理工具，提供了消息查询等功能

RocketMQ的源码是非常的多，我们没有必要把RocketMQ所有的源码都读完，所以我们把核心、重点的源码进行解读，RocketMQ核心流程如下：

* 启动流程
  RocketMQ服务端由两部分组成NameServer和Broker，NameServer是服务的注册中心，Broker会把自己的地址注册到NameServer，生产者和消费者启动的时候会先从NameServer获取Broker的地址，再去从Broker发送和接受消息。
* 消息生产流程
  Producer将消息写入到RocketMQ集群中Broker中具体的Queue。
* 消息消费流程
  Comsumer从RocketMQ集群中拉取对应的消息并进行消费确认。

## 5 NameServer源码分析

### 5.1 NameServer整体流程

NameServer是整个RocketMQ的“大脑”，它是RocketMQ的服务注册中心,所以RocketMQ需要先启动NameServer再启动Rocket中的Broker。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/e6b134ae1fcc42508a236f2621aeeed1.png)

* NameServer启动
  启动监听，等待Broker、Producer、Comsumer连接。Broker在启动时向所有NameServer注册，生产者在发送消息之前先从NameServer获取Broker服务器地址列表，然后根据负载均衡算法从列表中选择一台服务器进行消息发送。消费者在订阅某个主题的消息之前从NamerServer获取Broker服务器地址列表（有可能是集群），但是消费者选择从Broker中订阅消息，订阅规则由 Broker 配置决定。
* 路由注册
  Broker启动后向所有NameServer发送路由及心跳信息。
* 路由剔除
  移除心跳超时的Broker相关路由信息。NameServer与每台Broker服务保持长连接，并间隔10S检查Broker是否存活，如果检测到Broker宕机，则从路由注册表中将其移除。这样就可以实现RocketMQ的高可用。

### 5.2 NameServer启动流程

从源码的启动可知，NameServer单独启动。
入口类：NamesrvController
核心方法：NamesrvController 类中main()->main0-> createNamesrvController()->start() -> initialize()

流程图如下：

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/6d98ca5a20d8405cbc6742d5ebc9cec8.png)

#### 5.2.1 加载KV配置

核心解读NamesrvController类中createNamesrvController()

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/f4e43321b0e443cb9baa239f1a72d6e0.png)

在源码中发现还有一个p的参数，直接在启动个参数中送入 -p 就可以打印这个NameServer的所有的参数信息（不过NameServer会自动终止），说明这个-p是一个测试参数。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/6ffdb882757d4f2aa63d679e6f40a62c.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/6d1b81c2f5f64143997b914053ca2778.png)

正常启动时，也可以在启动日志中一定可以找到所有的参数：

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/4813d9dc347247cd9199d6c1bc5b2a45.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/58d11bda9b434d388ce2cbb89467f749.png)

#### 5.2.2 构建NRS通讯接收路由、心跳信息

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/8c24bf438e6046c4ac5f4f89f2c274a1.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/040f33aeb7004fb59b794b78f300aac6.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/43e80ad97d8449ec8e295de08124f2ed.png)

#### 5.2.3 定时任务剔除超时Broker

核心控制器会启动定时任务： 每隔10s扫描一次Broker,移除不活跃的Broker。

Broker每隔30s向NameServer发送一个心跳包，心跳包包含BrokerId，Broker地址，Broker名称，Broker所属集群名称、Broker关联的FilterServer列表。但是如果Broker宕机，NameServer无法收到心跳包，此时NameServer如何来剔除这些失效的Broker呢？NameServer会每隔10s扫描brokerLiveTable状态表，如果BrokerLive的**lastUpdateTimestamp**的时间戳距当前时间超过120s，则认为Broker失效，移除该Broker，关闭与Broker连接，同时更新topicQueueTable、brokerAddrTable、brokerLiveTable、filterServerTable。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/384f29ed7cbc479bb20f5a4aeadda1d2.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/121d8bbcdd704b3fa625c5cb073a5f99.png)

路由剔除机制中，Borker每隔30S向NameServer发送一次心跳，而NameServer是每隔10S扫描确定有没有不可用的主机（120S没心跳），那么问题就来了！这种设计是存在问题的，就是NameServer中认为可用的Broker，实际上已经宕机了，那么，某一时间段，从NameServer中读到的路由中包含了不可用的主机，会导致消息的生产/消费异常，不过不用担心，在生产和消费端有故障规避策略及重试机制可以解决以上问题（原理后续源码解读）。这个设计符合RocketMQ的设计理念：整体设计追求简单与性能，同时这样设计NameServer是可以做到无状态化的，可以随意的部署多台，其代码也非常简单，非常轻量。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/8197b7cf417440158c2c745a165fcf71.png)

**RocketMQ有两个触发点来删除路由信息：**

* NameServer定期扫描brokerLiveTable检测上次心跳包与当前系统的时间差，如果时间超过120s，则需要移除broker。
* Broker在正常关闭的情况下，会执行unregisterBroker指令这两种方式路由删除的方法都是一样的，都是从相关路由表中删除与该broker相关的信息。

  在消费者启动之后，第一步都要从NameServer中获取Topic相关信息

### 5.3 NameServer设计亮点

#### 5.3.1 读写锁

RouteInfoManager类中有一个读写锁的设计

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/6207220edc0140449b46d30938d06746.png)

消息发送时客户端会从NameServer获取路由信息，同时Broker会定时更新NameServer的路由信息，所以路由表会有非常频繁的以下操作：

1、 生产者发送消息时需要频繁的获取。对表进行读。

RouteInfoManager类

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/60a5e1f8fc35488ca036b46e7cdbcd3d.png)

2、 Broker定时(30s)会更新一个路由表。对表进行写。

RouteInfoManager类

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/c88b8c6981724172977b0623f75a6699.png)

因为Broker每隔30s向NameServer发送一个心跳包，这个操作每次都会更新Broker的状态，但同时生产者发送消息时也需要Broker的状态，要进行频繁的读取操作。所以这个地方就有一个矛盾，Broker的状态会被经常性的更新，同时也会被更加频繁的读取。这里如何提高并发，尤其是生产者进行消息发送时的并发，所以这里使用了读写锁机制（针对读多写少的场景）。

synchronized和ReentrantLock基本都是排他锁，排他锁在同一时刻只允许一个线程进行访问，而读写锁在同一时刻可以允许多个读线程访问，但是在写线程访问时，所有的读线程和其他写线程均被阻塞。读写锁维护了一对锁，一个读锁和一个写锁，通过分离读锁和写锁，使得并发性相比一般的排他锁有了很大提升。

#### 5.3.2 存储基于内存

NameServer存储以下信息：

**topicQueueTable**：Topic消息队列路由信息，消息发送时根据路由表进行负载均衡

**brokerAddrTable**：Broker基础信息，包括brokerName、所属集群名称、主备Broker地址

**clusterAddrTable**：Broker集群信息，存储集群中所有Broker名称

**brokerLiveTable**：Broker状态信息，NameServer每次收到心跳包是会替换该信息

**filterServerTable**：Broker上的FilterServer列表，用于类模式消息过滤。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/5484ea8dc23f4e9f8ae148bea2033375.png)

NameServer的实现基于内存，NameServer并不会持久化路由信息，持久化的重任是交给Broker来完成。这样设计可以提高NameServer的处理能力。

#### 5.3.3 NameServer无状态化

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/fdd1e88003bb47a79038355b1c17b95c.png)

* NameServer集群中它们相互之间是不通讯
* 主从架构中，Broker都会向所有NameServer注册路由、心跳信息
* 生产者/消费者同一时间，与NameServer集群中其中一台建立长连接

**项目实战部署分析：**

假设一个RocketMQ集群部署在两个机房，每个机房都有一些NameServer、Broker和客户端节点，当两个机房的链路中断时，所有的NameServer都可以提供服务，客户端只能在本机房的NameServer中找到本机房的Broker。

RocetMQ集群中，NameSever之间是不需要互相通信的，所以网络分区对NameSever本身的可用性是没有影响的，如果NameSever检测到与Broker的连接中断了，NameServer会认为这个Broker不再能提供服务，NameServer会立即把这个Broker从路由信息中移除掉，避免客户端连接到一个不可用的Broker上去。
网络分区后，NameSever 收不到对端机房那些Broker的心跳，这时候，每个Namesever上都只有本机房的Broker信息。
