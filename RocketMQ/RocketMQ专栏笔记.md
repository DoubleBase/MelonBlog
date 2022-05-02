# 基础概念

1. RocketMQ是什么，有哪些功能？
2. RocketMQ有哪些组件，集群架构是怎么样的



# 系统架构

- NameServer：broker的注册中心，管理broker集群元数据
- broker集群：接收消息，存储消息，转发消息；基于主从架构实现数据多副本和高可用
- 生产者：消息发送方
- 消费者：消息接收方

# NameServer

- NameServer集群化部署，保证高可用，管理broker集群元数据
- 每个broker启动都会向**所有**的NameServer进行注册
- 多个NameServer之间无通信，挂了一台，其他可用继续提供服务
- 生产者，消费者主动从NameServer拉取broker元数据更新

## broker挂了，NameServer如何感知

> 心跳机制

- Broker每隔30s会给所有的NameServer发送心跳，告诉NameServer自己还活着，NameServer收到一个Broker心跳，就会更新一下它最近一次心跳的时间
- NameServer会每隔10s运行一个任务，检查一下各个Broker最近一次的心跳时间，如果某个Broker超过120s都没发送心跳了，就认为这个broker挂了



## 【核心】

NameServer集群化部署

Broker会注册到所有的NameServer上去

30s心跳机制和120s故障检测机制

生产者消费者客户端容错机制

# Broker

## 主从架构

- 主从数据同步方式：slave broker通过pull模式，不断的从master broker中拉取数据
  - 如果是master向slave推，那么master必须维护所有broker的节点，以及所有broker当前的同步进度
    - 什么时候推又成了一个问题，写一条消息推一次，还是批量推一次，同步推送还是异步推送？
  - slave向master拉取，只需在每个slave broker中维护它是master是谁，当前的同步进度是多少就可以了
- 消费者从哪个broker拉取数据？有可能从master，有可能从slave
  - 在master返回数据给消费者时，会根据自己的负载情况，以及slave的同步情况，向消费者建议下一次拉取数据的时候从master拉取还是slave拉取
  - 如果master负载太大，slave数据与master同步的差不多，会建议从slave中去拉取
  - 如果slave数据落后master太多，没法从slave中获取最新的消息，则仍然会去master中拉取
- broker挂了怎么办？
  - 如果是slave挂了，影响不大，消息的写入和读取仍旧可以通过master，整体影响不大，只是master会承担更多的读请求
  - 如果master挂了
    - 在4.5版本之前，slave无法自动切换为master，需要手动修改配置，重启，导致一段时间不可用
    - 4.5之后，通过Dledger实现自动切换（基于Raft协议），一旦master宕机，可以在多个slave中选举出一个新的master，对外提供服务

## 【重点】数据存储

> 决定了生产者消息写入的吞吐量，决定了消息不能丢失，决定了消费者获取消息的吞吐量

### CommitLog

Broker收到消息，将消息按顺序写入CommitLog文件，追加在尾部。

CommitLog是多个磁盘文件，每个文件大小最多为1G，如果一个文件写满了，就会创建一个新的CommitLog文件

### MessageQueue

在Broker中，对于Topic下每个MessageQueue都会有一系列的**ConsumeQueue**文件。

在磁盘上，会有以下格式的一系列文件：

```bash
$HOME/store/consumequeue/{topic}/{queueId}/{filename}
# 例如：/store/consumequeue/order-topic/1/00000000000000000000
```

在ConsumeQueue文件中存储的是一条消息对应在CommitLog文件中的**offset偏移量**

在ConsumeQueue中存储的每条消息不只是CommitLog中的offset偏移量，还包含了消息的长度，tag hashcode，一条数据是20个字节，每个ConsumeQueue文件保存30w条数据，大概每个文件是5.72MB

### 近似内存的写入性能

基于OS操作系统的**PageCache**和**顺序写入**机制，来提升CommitLog的写入性能

**磁盘文件顺序写 + OS PageCache写入 + OS异步刷盘策略**



## 刷盘策略

- 同步刷盘

  生产者把消息发送给Broker，Broker必须强制将消息刷入底层的物理磁盘文件中，才会返回ack给producer

  优点：保证数据不丢失

  缺点：消息写入性能下降，写入吞吐量下降

- 异步刷盘

  生产者把消息发送给Broker，Broker将消息写入PageCache中，就直接返回ACK给producer

  优点：消息写入吞吐量非常高

  缺点：有数据丢失风险



## Dledger自动选举

- Dledger它自己有一个CommitLog机制，给它数据，它会写入CommitLog磁盘文件。
- 使用Dledger来管理CommitLog，基于Dledger替换各个Broker上的CommitLog管理组件
- 基于Raft协议来进行Leader Broker选举

### 选举流程

- 每个broker投票给自己，然后发送自己的投票给别的broker
- 每个broker进入一个随机时间的休眠，例如，broker01休眠3秒，broker02休眠5秒，broker03休眠6秒，此时broker01先苏醒，直接会继续尝试给自己投票，并且发送自己的选票给别；接着broker02苏醒过来，发现broker01已经发送来了一个选票投给broker01自己，broker02此时还没投票，会尊重别人选择，就直接把票投给broker01了，同时把自己的选票发送给别人；
- 当有超过一半的投票投给某个broker，那么就会选举他当Leader
- 依靠随机休眠的机制，经过几轮投票后，一般都是可以快速选举出来一个Leader

### 数据同步

分为两个阶段，一个uncommited阶段，一个commited阶段

- Leader Broker上的Dledger收到一条数据后，会标记为uncommited状态，然后通过自己的DledgerServer组件把这个uncommited数据发送给Follow Broker的DledgerServer
- Follow Broker的DledgerServer收到uncommited消息之后，必须返回一个ack给Leader Broker的DledgerServer。
- 如果Leader Broker收到超过半数的Follow Broker返回的ack之后，就会将消息标记为commited状态
- 然后Leader Broker上的DledgerServer就会发送commited消息给Follower Broker的DledgerServer，让他们也把消息标记为commited状态。

这就是基于Raft协议实现的两阶段完成的数据同步机制。

## 【核心】

- 主从同步原理
- 故障手动切换缺点
- Dledger自动选举

# 生产者

- 怎么将消息写入MessageQueue

  每个Broker中有多个Topic

  每个Topic中由多个MessageQueue组成

  一个Topic可以配多个MessageQueue，同一个topic在多个broker上有分片，通过这种分片机制使得数据有多副本

  一个Topic有多个MessageQueue，通过均匀写入的策略将数据写入各个Broker上的MessageQueue

- Broker故障，自动容错机制

  如果有Broker发生异常，在Producer中可以配置`sendLatencyFaultEnable`开关，开启自动容错机制。如果某次访问某个Broker发现网络延迟有500ms，然后还无法访问，那么就会自动回避这个Broker一段时间。比如接下来3000ms内，就不会访问这个Broker

  





1. 如何拉取broker元数据
2. 怎么发送消息
3. 消息发送异常怎么处理
4. 怎么处理粘包拆包
5. 怎么接受数据，响应处理
6. 同步发送，异步发送

## 消息发送模式

- 同步发送：可靠性最高，吞吐量低，同步获取返回结果
  - 使用场景：对可靠性要求靠，保证消息不能丢失；消息量小
- 异步发送：性能最佳，消息存在丢失风险，通过回调函数，异步获取返回结果
  - 使用场景：消息量大，对吞吐量要求高，允许消息丢失
- 单向发送：性能佳，没有返回结果
  - 使用场景：不关注发送结果，适合发送日志消息

# 消费者

## 消费者组

```java
// 设置消费者组
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("simple-consume-group");
```

- 作用：不同的消费者组，都会拉取到消息。

### 集群模式

同一消费者组，只有一台机器会获取到消息

### 广播模式

```java
// 设置广播模式
consumer.setMessageModel(MessageModel.BROADCASTING);
```

同一消费者组，每台机器都会获取到消息

## MessageQueue与消费者关系

一个Topic的多个MessageQueue会均匀分摊给消费者组内的多个机器去消费

一个MessageQueue只能被一个机器处理，但是一个机器可以处理多个MessageQueue的消息

## 消费模式

两者模式的本质都是消费者机器主动去broker机器拉取一批消息

### pull模式



### push模式

底层也是基于消费者主动拉取的模式来实现，只不过它的名字是push，broker会尽可能实时的把新消息交给消费者机器处理，他的消息时效性会更好

- 实现思路：当消费者发送请求到Broker去拉取消息，如果有新的消息可以消费那么就会立即返回一批消息到消费者机器去处理，处理完之后接着立刻发送请求到broker去拉取下一批消息
- 请求挂起和长轮询：当请求发送到broker，结果broker没有消息要发送，就会将请求线程挂起15秒；期间会有后台线程每隔一会就去检查一下是否有新的消息给你，另外如果在这个挂起过程中，如果有新的消息到达了会主动唤醒挂起的线程，然后把消息返回给你

## broker如何读取消息

根据消费的MessageQueue以及开始消费的位置，去找对应ConsumeQueue读取里面对应的消息在CommitLog中的物理offset偏移量，然后到CommitLog中根据offset偏移量读取数据


## 如何响应消息

注册回调函数

```java
consumer.registerMessageListener(new MessageListenerConcurrently() {
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
        System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), list);
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
```

消息处理成功，broker会记录消费成功的位置



## 消费者宕机或扩容

会进入rebalance环节，重新给各个消费机器分配他们要处理的MessageQueue





# 网络通信框架

