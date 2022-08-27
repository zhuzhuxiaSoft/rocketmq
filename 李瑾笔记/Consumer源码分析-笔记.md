# RocketMQ源码深入剖析

## 8 Consumer源码分析

### 8.1 消息发送时数据在ConsumeQueue的落地

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/f502482db5bf43859d76b9a6d76aa66c.png)

连续发送5条消息，消息是不定长，首先所有信息先放入 Commitlog中，每一条消息放入Commitlog的时候都需要上锁，确保顺序的写入。

当Commitlog写成功了之后。数据通过ReputMessageService类定时同步到ConsunmeQueue中，写入Consume Queue的内容是定长的，固定是20个Bytes（offset 8个、size 4个、Hashcode of Tag 8个）。

**这种设计非常的巧妙：**

查找消息的时候，可以直按根据队列的消息序号，计算出索引的全局位置（比如序号2，就知道偏移量是20），然后直接读取这条索引，再根据索引中记录的消息的全局位置，找到消息。这两次查找是差不多的：第一次在通过序号在consumer Queue中获取数据的时间复杂度是O(1)，第二次查找commitlog文件的时间复杂度也是O(1)，所以消费时查找数据的时间复杂度也是O(1)。

#### 8.1.1 ReputMessageService.doReput源码分析

DefaultMessageStore. start()

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/162d11e97e3342dfbc5673bd155cb508.png)

maxPhysicalPosInLogicQueue 就是commitlog的文件名（这个文件记录的最小偏移量）

ReputMessageService.run()

ReputMessageService线程每执行一次任务推送休息1毫秒就继续尝试推送消息

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/fd69a19a0d5f447fb04b2b3a885dfd64.png)

1）返回reputFromOffset偏移量开始的全部有效数据(commitlog文件)。然后循环读取每一条消息。

2）从result返回的ByteBuffer中循环读取消息，一次读取一条，创建DispatchRequest对象。如果消息长度大于0，则调用doDispatch方法。最终将分别调用CommitLogDispatcherBuildConsumeQueue(构建消息消费队列)CommitLogDispatcherBuildIndex(构建索引文件)。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/a1dcd2235e634b02bc71788685352ce9.png)

3）构建消息消费队列

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/f52bec8a4ac84d6dbba32425c3a8ffc9.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/567bab3f991a49d3870c401fc390ab1f.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/f85359252019422dbe56d7d904368de7.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/00402bba25ee4c44ad8b1ba3e7fb025c.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/c7c4a8023e624a7088a609143be6c431.png)

### 8.2 消费者启动流程

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/e6b7a79e10254f8c9b73345e732e57a2.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/4718e072bb064c79b9a252444e06941a.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/9835114ff6574dd1a76f6c1782fbba6b.png)

DefaultMQPushConsumerImpl类是核心类

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/1d8e0653688b42988ec8d15559cef15d.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/a7cf27734aa74f18964a589af4ca039d.png)

### 8.3 消费者模式

#### 8.3.1 集群消费

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/5e4661cbbac848a89b527d9d83733347.png)

消费者的一种消费模式。一个Consumer
Group中的各个Consumer实例分摊去消费消息，即一条消息只会投递到一个Consumer Group下面的一个实例。

实际上，每个Consumer是平均分摊Message
Queue的去做拉取消费。例如某个Topic有3条Q，其中一个Consumer Group 有 3 个实例（可能是 3 个进程，或者 3 台机器），那么每个实例只消费其中的1条Q。

而由Producer发送消息的时候是轮询所有的Q,所以消息会平均散落在不同的Q上，可以认为Q上的消息是平均的。那么实例也就平均地消费消息了。

这种模式下，**消费进度** **(Consumer Offset)**  **的存储会持久化到Broker** 。

**代码演示**

![](file:///C:\Users\ADMINI~1\AppData\Local\Temp\msohtmlclip1\01\clip_image004.jpg)![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/7b1d77d9e88b4f5cbe78a8d5b9bf43f8.png)

![](file:///C:\Users\ADMINI~1\AppData\Local\Temp\msohtmlclip1\01\clip_image006.jpg)![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/c7432b1849ba444e93b4eb1dce8b352d.png)

#### 8.3.2 广播消费

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/d9dfc3660ed04c98b750fb17e13f40e6.png)

消费者的一种消费模式。消息将对一个Consumer
Group下的各个Consumer实例都投递一遍。即即使这些 Consumer 属于同一个Consumer Group，消息也会被Consumer Group 中的每个Consumer都消费一次。

实际上，是一个消费组下的每个消费者实例都获取到了topic下面的每个Message Queue去拉取消费。所以消息会投递到每个消费者实例。

这种模式下，**消费进度** **(Consumer Offset)**  **会存储持久化到实例本地** 。

**代码演示**

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/1865848c2bc745baa970046949c4c476.png)

### 8.4 Consumer负载均衡

#### 8.4.1 集群模式

在集群消费模式下，每条消息只需要投递到订阅这个topic的Consumer Group下的一个实例即可。RocketMQ采用主动拉取的方式拉取并消费消息，在拉取的时候需要明确指定拉取哪一条message queue。

而每当实例的数量有变更，都会触发一次所有实例的负载均衡，这时候会按照queue的数量和实例的数量平均分配queue给每个实例。

默认的分配算法是AllocateMessageQueueAveragely

还有另外一种平均的算法是AllocateMessageQueueAveragelyByCircle，也是平均分摊每一条queue，只是以环状轮流分queue的形式

如下图：

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/64d1e94006be4a34a805508e8a6bb5d6.png)

需要注意的是，集群模式下，queue都是只允许分配只一个实例，这是由于如果多个实例同时消费一个queue的消息，由于拉取哪些消息是consumer主动控制的，那样会导致同一个消息在不同的实例下被消费多次，所以算法上都是一个queue只分给一个consumer实例，一个consumer实例可以允许同时分到不同的queue。

通过增加consumer实例去分摊queue的消费，可以起到水平扩展的消费能力的作用。而有实例下线的时候，会重新触发负载均衡，这时候原来分配到的queue将分配到其他实例上继续消费。

但是如果consumer实例的数量比message queue的总数量还多的话，多出来的consumer实例将无法分到queue，也就无法消费到消息，也就无法起到分摊负载的作用了。所以需要控制让queue的总数量大于等于consumer的数量。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/e50c3372d15c4b6baa41f45828275c05.png)

#### 8.4.2 广播模式

由于广播模式下要求一条消息需要投递到一个消费组下面所有的消费者实例，所以也就没有消息被分摊消费的说法。

在实现上，其中一个不同就是在consumer分配queue的时候，所有consumer都分到所有的queue。

### 8.5 并发消费流程

一般我们在消费时使用回调函数的方式，使用得最多的是并发消费，消费者客户端代码如下：

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/76ba301346f341289f24a37bb64cf444.png)

参考RocketMQ核心流程

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/d29bd24240e9423690a89e5c875cc3ab.png)

在RocketMQ的消费时，整体流程如下：

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/76f69623f2b24597b1822715dddb0881.png)

#### 8.5.1 获取topic配置信息

在消费者启动之后，第一步都要从NameServer中获取Topic相关信息。

这一步设计到组件之间的交互，RocketMQ使用功能号来设计的。GET_ROUTEINFO_BY_TOPIC在idea上使用ctrl+H 查找功能。很快就定位这段代码：

MQClientInstance类中

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/1020a8b1a2ea49e09eceeaf14a623597.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/b975fa02e6004262bb80a06e438c3698.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/8dc092ddee2446579a0e805ae40d8f89.png)

最终在MQClientAPIImpl类中完成调用

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1648432544069/69f0de261f9a4ae0ab150dbeee4788f5.png)

具体这里是30S定时执行一次。

#### 8.5.2 获取Group的ConsumerList

在消费消息前，需要获取当前分组已经消费的相关信息：ConsumerList

MQClientInstance类的start()

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/c8cc1167f8c242b6b0f8c2b18a855d4e.png)

这里就是每间隔20S就执行一个doRebalance方法

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/d70c89a6b5b24a6a90c3745b7f98d562.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/ed8d08863a2f4ba880745260010b2561.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/d4424b0e926448af9d0fbe82a44bd7d0.png)

进入RebalanceImpl类

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/a91846cf2ae44c63a05b82770dedb9e6.png)

再进入具体的类,如果是广播消费模式，则不需要从服务器获取消费进度（广播消费模式把进度在本地&#x3c;消费端&#x3e;进行存储）

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/97cd2a30871945b2a92f587bfaf4e16e.png)

而广播消费模式，则需要从服务器获取消费进度相关信息，具体如下：

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/f859becddd744f3b81469b7f6a45d869.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/af05ad37b1a247ab8835ebe29457a9f8.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/442dd0232c6e462da0a659389517a344.png)

#### 8.5.3 获取Queue的消费Offset

在分配完消费者对应的Queue之后，如果是集群模式的话，需要获取这个消费者对应Queue的消费Offset,便于后续拉取未消费完的消息。

RebalanceImpl类中rebalanceByTopic方法

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/ad1e86cb21504219b0c96b1b4b4cf08e.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/835649b00df8457f91063d7160c09db7.png)

进入RebalancePushImpl类

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/f1447d7cba194d578e15e05c06532aaa.png)![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/e73610cfeff9472b90b75910c71c2879.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/497d09a6ef73438eaf535754a37032ce.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/19c8c2a4e64e46f2a1a827e5761e6a38.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/9f21ba41a3cf44e48e0377685ae29ef1.png)

#### 8.5.4 拉取Queue的消息

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/88e5142f887b4cd7aed3a1093745118e.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/1d2da34c7bd348b1969802fee434529f.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/d82c7c52e7e54a57b4beb26f2a80de16.png)

最终进入DefaultMQPushConsumerImpl类的pullMessage方法

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/53fcb3419ffd45b6b9bdb6374cad0ff8.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/5be0eccef6654e1ab5c655f2aa754bfb.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/8a8313f2437c4a508d509b7d9122a65e.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/34d6e4de72e94463b908a09d6ce09fdf.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/917d246e70bb4933a240c7cd2385d3c9.png)

#### 8.5.5 更新Queue的消费Offset

这里要注意，因为RocketMQ的推模式是基于拉模式实现的，因为拉消息是一批批拉，所以不能做到拉一批提交一次偏移量，所以这里使用定时任务进行偏移量的更新。

MQClientInstance类中的start方法

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/c33e917beea64afd8c3782514902fdf4.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/6697a12b683e48cdb54118cb0a39d74f.png)![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/499f32f9e41d4008b21526f3de956049.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/11143385bab541199425758f13def658.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/5215868a1abe46e8817ee54578477ef1.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/206da1ba71164bed8ee259b8ab2d261b.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/af5f625948d947ecbadab41e409fc255.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/63ed3d2561514aecb3dfa304550cc586.png)

#### 8.5.6 注销Consumer

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/cea331b474064687a7e31e152bde0f14.png)

### 8.6 顺序消费流程

顺序消费代码：![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/e9ed63e4787545dfa23711dab91d4245.png)顺序消费的流程和并发消费流程整体差不多，唯一的多的就是使用锁机制来确保一个队列同时只能被一个消费者消费，从而确保消费的顺序性

ConsumeMessageOrderlyService类

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/f113d65403c9424683684875efacae84.png)

这里有一个定时任务，是每个20秒运行一次（周期性的去续锁，锁的有效期是60S）

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/81cecfa993ec49a4989972ea750a6037.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/71e47a85856a411db30ef1b986daf751.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/9cd3bd2e62ce48b38259976a4cd10487.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/842c11a33e7a4e09997884db969bf2ea.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/4f89afe96edb49019fd4c033183c1dc7.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/2d8c6ccc44a0413abb97047028a52022.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/519d1f6ef88c419a8911f6058b81df6a.png)

### 8.7 消费卡死

之前我讲到了消费的流程中，尤其是针对顺序消息，我们感觉上会有卡死的现象，由于顺序消息中需要到Broker中加锁，如果消费者某一个挂了，那么在Broker层是维护了60s的时间才能释放锁，所以在这段时间只能，消费者是消费不了的，在等待锁。

另外如果还有Broker层面也挂了，如果是主从机构，获取锁都是走的Master节点，如果Master节点挂了，走Slave消费，但是slave节点上没有锁，所以顺序消息如果发生了这样的情况，也是会有卡死的现象。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/51ea45dd280741debbb2f52ce3f48b68.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/86762069b1a3451fa7a457f1e7c1642b.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/d6b904756c084ddc84fc2716e7622454.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/1a9c8c9380c8418690a12f5339dd5aad.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/01b09d51b54b4dad85d3f8a737a4c531.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/7f3d051a2ea1479082b989fb9bcf283d.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/f6001eec78394f8aaebe9134c2b3869f.png)

### 8.8 启动之后较长时间才消费

在并发消费的时候，当我们启动了非常多的消费者，维护了非常多的topic的时候、或者queue比较多的时候，你可以看到消费的流程的交互是比较多的（5~6步），要启动多线程，也要做相当多的事情，所以你会感觉要启动较长的时间才能消费。

还有顺序消费的时候，如果是之前的消费者挂了，这个锁要60秒才会释放，也会导致下一个消费者启动的时候需要等60s才能消费。



### 8.8 消费端整体流程预览

这个流程比较复杂，建议有兴趣的同学可以根据这张图研究下

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1650087469088/75e544f71f5e486692e7cac203879046.png)
