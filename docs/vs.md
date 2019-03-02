# VS

RocketMQ 是站在巨人的肩膀上（Kafka），又对其进行了优化让其更满足互联网公司的特点。它是纯 Java 开发，具有高吞吐量、高可用性、适合大规模分布式系统应用的特点。RocketMQ目前在阿里集团被广泛应用于交易、充值、流计算、消息推送、日志流式处理、binglog 分发等场景。

2011年初，Linkin开源了 Kafka 这个优秀的消息中间件，淘宝中间件团队在对Kafka做过充分Review之后，**Kafka无限消息堆积，高效的持久化速度**吸引了我们，但是同时发现 **Kafka主要定位于日志传输**，对于使用在淘宝交易、订单、充值等场景下还有诸多特性不满足，为此我们重新用 Java 语言编写了RocketMQ，定位于非日志的可靠消息传输（日志场景也OK），目前RocketMQ在阿里集团被广泛应用在订单，交易，充值，流计算，消息推送，日志流式处理，binglog 分发等场景。

## 数据可靠性

- RocketMQ支持异步实时刷盘，同步刷盘，同步复制，异步复制
- Kafka 使用异步刷盘方式，异步复制/同步复制

> 总结：RocketMQ的同步刷盘在单机可靠性上比Kafka更高，不会因为操作系统Crash，导致数据丢失。Kafka同步Replication理论上性能低于RocketMQ的同步Replication，原因是Kafka的数据以分区为单位组织，意味着一个Kafka实例上会有几百个数据分区，RocketMQ一个实例上只有一个数据分区，RocketMQ可以充分利用IO组Commit机制，批量传输数据，配置同步Replication与异步Replication相比，性能损耗约20%~30%，Kafka没有亲自测试过，但是个人认为理论上会低于RocketMQ。

## 性能对比

- Kafka 单机写入TPS约在百万条/秒，消息大小10个字节
- RocketMQ单机写入TPS单实例约7万条/秒，单机部署3个Broker，可以跑到最高12万条/秒，消息大小10个字节

> 总结：**Kafka的TPS跑到单机百万，主要是由于Producer端将多个小消息合并，批量发向Broker**。 RocketMQ为什么没有这么做？
>
> - Java语言，缓存过多消息，GC是个很严重的问题
>
> - Producer 调用发送消息接口，消息未发送到 Broker，向业务返回成功，此时Producer宕机，会导致消息丢失，业务出错
>
> - Producer通常为分布式系统，且每台机器都是多线程发送，我们认为线上的系统单个Producer每秒产生的数据量有限，不可能上万。
>
> - 缓存的功能完全可以由上层业务完成。



## 单机支持的队列数

- Kafka单机超过64个队列/分区，Load会发生明显的飙高现象，队列越多，load越高，发送消息响应时间变长。Kafka分区数无法过多的问题
- RocketMQ单机支持最高5万个队列，负载不会发生明显变化

## 队列多有什么好处？

1. 单机可以创建更多 Topic，因为每个主题都是由一批队列组成
2. 消费者的集群规模和队列数成正比，队列越多，消费类集群可以越大

## 消息投递实时性

- Kafka使用短轮询方式，实时性取决于轮询间隔时间，0.8以后版本支持长轮询。
- RocketMQ使用长轮询，同Push方式实时性一致，消息的投递延时通常在几个毫秒。

## 消费失败重试

- Kafka 消费失败不支持重试。
- RocketMQ 消费失败支持定时重试，每次重试间隔时间顺延

> 总结：例如充值类应用，当前时刻调用运营商网关，充值失败，可能是对方压
>
> 力过多，稍后再调用就会成功，如支付宝到银行扣款也是类似需求。
>
> 这里的重试需要可靠的重试，即失败重试的消息不因为Consumer宕机导致丢失。

## 严格的消息顺序

- Kafka 支持消息顺序，但是一台代理宕机后，就会产生消息乱序
- RocketMQ 支持严格的消息顺序，在顺序消息场景下，一台Broker宕机后，发送消息会失败，但是不会乱序

> MySQL的二进制日志分发需要严格的消息顺序

## 定时消息

- Kafka 不支持定时消息
- RocketMQ 支持两类定时消息
- 开源版本 RocketMQ 仅支持定时级别，定时级用户可定制
- 阿里云MQ指定的毫秒级别的延时时间

## 分布式事务消息

- Kafka 不支持分布式事务消息
- 阿里云MQ支持分布式事务消息，未来开源版本的RocketMQ也有计划支持分布式事务消息

## 消息查询

- Kafka 不支持消息查询
- RocketMQ 支持根据消息标识查询消息，也支持根据消息内容查询消息（发送消息时指定一个消息密钥，任意字符串，例如指定为订单编号）

> 总结：消息查询对于定位消息丢失问题非常有帮助，例如某个订单处理失败，是消息没收到还是收到处理出错了。

## 消息回溯

- Kafka 理论上可以按照偏移来回溯消息
- RocketMQ 支持按照时间来回溯消息，精度毫秒，例如从一天之前的某时某分某秒开始重新消费消息

> 总结：典型业务场景如consumer做订单分析，但是由于程序逻辑或者依赖的系统发生故障等原因，导致今天消费的消息全部无效，需要重新从昨天零点开始消费，那么以时间为起点的消息重放功能对于业务非常有帮助。

## 消费并行度

- Kafka的消费并行度依赖Topic配置的分区数，如分区数为10，那么最多10台机器来并行消费（每台机器只能开启一个线程），或者一台机器消费（10个线程并行消费）。即消费并行度和分区数一致。
- RocketMQ 消费并行度分两种情况
- 顺序消费方式并行度同 Kafka 完全一致
- 乱序方式并行度取决于Consumer的线程数，如Topic配置10个队列，10台机器消费，每台机器100个线程，那么并行度为1000。

## 消息轨迹

- Kafka 不支持消息轨迹
- 阿里云MQ支持消息轨迹

## 开发语言友好性

- Kafka 采用scala编写
- RocketMQ采用的Java语言编写

## 券商端消息过滤

- Kafka 不支持代理端的消息过滤
- RocketMQ支持两种代理端消息过滤方式
- 根据消息变量来过滤，相当于子主题概念
- 向服务器上传一段Java代码，可以对消息做任意形式的过滤，甚至可以做Message身体的过滤拆分。



## 消息堆积能力

理论上Kafka要比RocketMQ的堆积能力更强，不过RocketMQ单机也可以支持亿级的消息堆积能力，我们认为这个堆积能力已经完全可以满足业务需求。



## 开源社区活跃度

- 卡夫卡社区更新较慢
- RocketMQ的GitHub的社区有250个个人，公司用户登记了联系方式，QQ群超过1000人。 



## 商业支持

- Kafka 原开发团队成立新公司，目前暂没有相关产品看到

- RocketMQ在阿里云已经商业化，目前以云服务形式供大家商用，并向用户承诺99.99%的可靠性，同时彻底解决了用户自己搭建MQ产品的运维复杂性问题



## 成熟度

- Kafka 在日志领域比较成熟
- RocketMQ在阿里集团内部有大量的应用在使用，每天都产生海量的消息，并且顺利支持了多次天猫双十一海量消息考验，是数据削峰填谷的利器。



## Read More

- 原文来自
  - [阿里RocketMQ](https://mp.weixin.qq.com/s/KfBruI-tOz-eJuM2fgqyew)