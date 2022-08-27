## RocketMQ源码深入剖析

## 7 Producer源码分析

### 7.1 消息发送整体流程

下面是一个生产者发送消息的demo（同步发送）

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650003366015/aa5e19b1c0094c9eae42d065b78b0138.png)

**主要做了几件事：**

* 初始化一个生产者（DefaultMQProducer）对象
* 设置 NameServer 的地址
* 启动生产者
* 发送消息

### 7.2 消息发送者启动流程

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650003366015/b4c50b6bd584436195e83050c295f3ad.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650003366015/18e518317fc442d88e1027e1037728cf.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650003366015/a2353d0fd19a45ec9448e6f74f204798.png)

**DefaultMQProducerImpl**类start()

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650003366015/8b812bb8f89a4cd2ba3db3a18f9cdc63.png)

#### 7.2.1 检查

**DefaultMQProducerImpl**类

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650003366015/6305ee76e1544555b6cbb14e211d3b2c.png)

#### 7.2.2 获得MQ客户端实例

整个JVM中只存在一个MQClientManager实例，维护一个MQClientInstance缓存表

DefaultMQProducerImpl类start()

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650003366015/9a5dc6efd8ab46e8ba39af042f477d18.png)

一个clientId只会创建一个MQClientInstance

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650003366015/03b0b3e84fe5408ba4fc9875841fa305.png)

clientId生成规则：IP@instanceName@unitName

ClientConfig类

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650003366015/54632e05bf3945929570c172be4c3ef9.png)

RocketMQ中消息发送者、消息消费者都属于”客户端“

每一个客户端就是一个MQClientInstance，每一个ClientConfig对应一个实例。

故不同的生产者、消费端，如果引用同一个客户端配置(ClientConfig)，则它们共享一个MQClientInstance实例。所以我们在定义的的时候要注意这种问题（生产者和消费者如果分组名相同容易导致这个问题）

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650003366015/f8903ffaea2641678a0757aee21047c3.png)

#### 7.2.3 启动实例

MQClientInstance类start()

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650003366015/584bc657b2a84579883a1b45e9ec3b82.png)

#### 7.2.4 定时任务

MQClientInstance类startScheduledTask()

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650003366015/8552e3d4798248a5b4349ae816086504.png)

### 7.3 Producer消息发送流程

我们从一个生产者案例的代码进入代码可知：DefaultMQProducerImpl中的sendDefaultImpl()是生产者消息发送的核心方法

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650003366015/30aeb13b47104c7e9dab2168be43053b.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650003366015/61ab3256ecbb4ca594b594c4da4b1a12.png)

从核心方法可知消息发送就是4个步骤：验证消息、查找路由、选择队列、消息发送。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/e4e64c245cbc4cc892120faad1ad3544.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/f1efd4fc85f64375848cb59fc6648230.png)

### 7.4 消息发送队列选择

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650003366015/81b9286248de4315be724d36837298c9.png)

#### **7.4.1 默认选择队列策略**

采用了最简单的轮询算法，这种算法有个很好的特性就是，保证每一个Queue队列的消息投递数量尽可能均匀。这种算法只要消息投递过程中没有发生重试的话，基本上可以保证每一个Queue队列的消息投递数量尽可能均匀。当然如果投递中发生问题，比如第一次投递就失败，那么很大的可能性是集群状态下的一台Broker挂了，所以在重试发送中进行规避。这样设置也是比较合理的。

#### 7.4.2 **故障延迟机制策略**

采用此策略后，每次向Broker成功或者异常的发送，RocketMQ都会计算出该Borker的可用时间（发送结束时间-发送开始时间，失败的按照30S计算），并且保存，方便下次发送时做筛选。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/7e754d3c39e248279d9b974115a85f14.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/c570b9b51fa74ada85e65f35687751a8.png)

除了记录Broker的发送消息时长之外，还要计算一个Broker的不可用时长。这里采用一个经验值：

如果消息时长在550ms之内，不可用时长为0。

达到550ms，不可用时长为30S

达到1000ms，不可用时长为60S

达到2000ms，不可用时长为120S

达到3000ms，不可用时长为180S

达到15000ms，不可用时长为600S

以最大的计算。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/97684fb3343c4a7787e291b38d9fe6dd.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/63c3d5926f6d404f83d3ba1036658e45.png)

有了以上的Broker规避信息后发送消息就非常简单了。

在开启故障延迟机制策略步骤如下：

1、根据消息队列表时做轮训

2、选好一个队列

3、判断该队列所在Broker是否可用

4、如果是可用则返回该队列，队列选择逻辑结束

5、如果不可用，则接着步骤2继续

6、如果都不可用，则随机选一个

**代码如下：**

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/bc24f6d85543424d8c0a97c5e16c2a0c.png)

#### 7.4.3 两种**策略的选择**

从这种策略上可以很明显看到，默认队列选择是轮训策略，而故障延迟选择队列则是优先考虑消息的发送时长短的队列。那么如何选择呢？

首先RocketMQ默认的发送失败有重试策略，默认是2，也就是如果向不同的Broker发送三次都失败了那么这条消息的发送就失败了，作为RocketMQ肯定是尽力要确保消息发送成功。所以给出以下建议。

如果是网络比较好的环境，推荐默认策略，毕竟网络问题导致的发送失败几率比较小。

如果是网络不太好的环境，推荐故障延迟机制，消息队列选择时，会在一段时间内过滤掉RocketMQ认为不可用的broker，以此来避免不断向宕机的broker发送消息，从而实现消息发送高可用。

当然以上成立的条件是一个Topic创建在2个Broker以上的的基础上。

#### 7.4.4 技术亮点:ThreadLocal

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/e1f44264ea664895923c3bfca34115fb.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/2f8d7ce137754388af2db0d431aec043.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/63a7a45afe1c4649a1d579c32d064645.png)

### 7.5 客户端建立连接的时机

Producer、Consumer连接的建立时机，有何关系？

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/d0857f7c1b434622bdcd8f74da001d06.png)

源码分析一波：

DefaultMQProducerImpl类中sendKernelImpl方法

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/c6197cb8ed5140958a03287d9f533e0e.png)![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/647d793d9a844d1eaa16b128dcd8b0bf.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/3e9456819859447095221d9ff3f41e5e.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/3c1db7c4bc2b4faa8dc46f9c128e74fa.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/746c15a913b24bdcb4febcec94ff013c.png)

根据源码分析：客户端(MQClientInstance)中连接的建立时机为按需创建，也就是在需要与对端进行数据交互时才建立的。建立的是长连接。
