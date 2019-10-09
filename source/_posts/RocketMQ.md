---
title: RocketMQ
date: 2019-07-12 17:39:38
tags:
---
# RocketMQ消息发送方式
RocketMQ支持3种消息发送方式：同步（Sync）、异步（Async）、单向（Oneway）
- 同步：发送者向MQ 执行发送消息API 时，同步等待， 直到消息服务器返回发送结果。
- 异步：发送者向MQ 执行发送消息API 时，指定消息发送成功后的回调函数，然后调用消息发送API 后，立即返回，消息发送者线程不阻塞，直到运行结束。待消息发送成功或失败的时候，回调任务在一个新的线程中执行。
- 单向：消息发送者向MQ 执行发送消息AP I 时，直接返回，不等待消息服务器的结果，也不注册回调函数，简单地说，就是只管发，不在乎消息是否成功存储在消息服务器上。

# RocketMQ 消息发送考虑的问题
- 消息队列如何负载？
- 消息发送如何实现高可用？
- 批量消息发送如何实现一致性？

# RocketMQ消息结构
RocketMQ 消息封装类是 org.apache.rocketmq.common.message.Message，其类设计如下
![Message设计](https://i.loli.net/2019/07/12/5d28562b7b9b866974.png)
Message类的全属性构造函数
```Java
public Message(String topic, String tags, String keys, int flag, byte[] body, boolean waitStoreMsgOK) {
    this.topic = topic;  // topic：消息所在的topic通道，主要属性
    this.flag = flag;
    this.body = body; // body：消息的真实内容，主要属性

    if (tags != null && tags.length() > 0)
        this.setTags(tags);  // tags：消息标签，用于消息过滤

    if (keys != null && keys.length() > 0)
        this.setKeys(keys); // keys：Message索引键，多个则用空格隔开，RocketMQ可以根据这些Key快速检索到消息

    this.setWaitStoreMsgOK(waitStoreMsgOK);  // waitStoreMsgOK：消息发送时是否等消息存储完成后再返回
}
```
Message 的基础属性主要包括消息所属主题topic ， 消息Flag(RocketMQ 不做处理）、扩展属性（properties）、消息体（body）、事务ID（transactionId，用于分布式事务）。

其中，RocketMQ Message的一些扩展属性properties还包含：
delayTimeLevel：消息延迟级别，用于定时消息或消息重试
buyerId： 买家ID（这个字段一看就带有很浓重的电商气息）

透过这些属性的set方法可以知道，这些扩展属性存储在Message的Map类型的properties变量中。

# 生产者的启动流程
消息生产者的代码都在client 模块中，相对于RocketMQ 来说，它就是客户端，也是消息的提供者，我们在应用系统中初始化生产者的一个实例即可使用它来发消息。
# DefaultMQProducer 默认的消息发送者
消息生产者的启动流程，我们可以从org.apache.rocketmq.client.producer.DefaultMQProducer.start()入口开始看起
```Java
//启动的简易入口
public void start() throws MQClientException {
    this.start(true);
}
 //程序的真实启动入口
 public void start(final boolean startFactory) throws MQClientException {
    switch (this.serviceState) {
        case CREATE_JUST:
            this.serviceState = ServiceState.START_FAILED; // 设置默认的状态是失败
            // 验证配置，主要是验证group配置，不能为默认group
            this.checkConfig();
            // 将group的的名称设置为当前线程的后缀id
            if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                this.defaultMQProducer.changeInstanceNameToPID();
            }
            // 获得发送客户端工厂，该工程是复用设计，内部是client的配置
            this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQProducer, rpcHook);
            // 注册当前的消息发送者，确保每个group都是唯一的，否则报错
            boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
            if (!registerOK) {
                this.serviceState = ServiceState.CREATE_JUST;
                throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
                    + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                    null);
            }
            // topic的发送消息管理，发生自动创建topic的请求
            this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());
            //如果是静态工程启动，需要手动的启动
            if (startFactory) {
                mQClientFactory.start();
            }

            log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),
                this.defaultMQProducer.isSendMessageWithVIPChannel());
            this.serviceState = ServiceState.RUNNING;
            break;
        case RUNNING:
        case START_FAILED:
        case SHUTDOWN_ALREADY:
            throw new MQClientException("The producer service state not OK, maybe started once, "
                + this.serviceState
                + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                null);
        default:
            break;
    }
    // 启动成功后，发送心跳
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
}
```
接下来，重点看一下MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQProducer, rpcHook)的操作
```Java
// 基于本地缓存的客户端管理
public MQClientInstance getAndCreateMQClientInstance(final ClientConfig clientConfig, RPCHook rpcHook) {
    String clientId = clientConfig.buildMQClientId(); //id为当前服务器的id
    MQClientInstance instance = this.factoryTable.get(clientId); //是否有可复用的通信客户端，资源占用比较大，可以复用
    if (null == instance) {
        instance =
            new MQClientInstance(clientConfig.cloneClientConfig(),
                this.factoryIndexGenerator.getAndIncrement(), clientId, rpcHook); //初始化客户端请求实例
        MQClientInstance prev = this.factoryTable.putIfAbsent(clientId, instance); //设置请求对象，如果存在则返回原先的值，同时不覆盖，用原来的请求
        if (prev != null) {
            instance = prev;
            log.warn("Returned Previous MQClientInstance for clientId:[{}]", clientId);
        } else {
            log.info("Created new MQClientInstance for clientId:[{}]", clientId);
        }
    }

    return instance;
}
```
继续跟进去看初始化MQClientInstance的构造，最终的操作都会围绕该类进行操作和整合
```Java
//初始化客户端请求实例
public MQClientInstance(ClientConfig clientConfig, int instanceIndex, String clientId, RPCHook rpcHook) {
    this.clientConfig = clientConfig; //mq的核心配置信息
    this.instanceIndex = instanceIndex; //当前进程内的唯一标识，升序数值
    this.nettyClientConfig = new NettyClientConfig(); //netty通信的客户端配置
    this.nettyClientConfig.setClientCallbackExecutorThreads(clientConfig.getClientCallbackExecutorThreads());
    this.nettyClientConfig.setUseTLS(clientConfig.isUseTLS());
    this.clientRemotingProcessor = new ClientRemotingProcessor(this); //解析客户端请求，封装的事件处理
    this.mQClientAPIImpl = new MQClientAPIImpl(this.nettyClientConfig, this.clientRemotingProcessor, rpcHook, clientConfig); //客户端实例的实际实现，网络通信的核心，只是初始化了通信框架，具体的链接后面根据不同的地址再进行链接操作
    //设置核心的nameserv地址
    if (this.clientConfig.getNamesrvAddr() != null) {
        this.mQClientAPIImpl.updateNameServerAddressList(this.clientConfig.getNamesrvAddr());
        log.info("user specified name server address: {}", this.clientConfig.getNamesrvAddr());
    }

    this.clientId = clientId;

    this.mQAdminImpl = new MQAdminImpl(this); //mq管理

    this.pullMessageService = new PullMessageService(this); //拉取消息的实现

    this.rebalanceService = new RebalanceService(this);  //负载均衡的实现，可能有相关的机器增加删除，需要定期的进行重负载操作

    this.defaultMQProducer = new DefaultMQProducer(MixAll.CLIENT_INNER_PRODUCER_GROUP);
    this.defaultMQProducer.resetClientConfig(clientConfig);

    this.consumerStatsManager = new ConsumerStatsManager(this.scheduledExecutorService); //消费端的状态管理

    log.info("Created a new client Instance, InstanceIndex:{}, ClientID:{}, ClientConfig:{}, ClientVersion:{}, SerializerType:{}",
        this.instanceIndex,
        this.clientId,
        this.clientConfig,
        MQVersion.getVersionDesc(MQVersion.CURRENT_VERSION), RemotingCommand.getSerializeTypeConfigInThisServer());
}
```
继续跟进去网络通信的构造方法 org.apache.rocketmq.client.impl.MQClientAPIImpl.MQClientAPIImpl(NettyClientConfig, ClientRemotingProcessor, RPCHook, ClientConfig)
```Java
// 客户端网络通信
public MQClientAPIImpl(final NettyClientConfig nettyClientConfig,
    final ClientRemotingProcessor clientRemotingProcessor,
    RPCHook rpcHook, final ClientConfig clientConfig) {
    this.clientConfig = clientConfig; //核心的配置
    topAddressing = new TopAddressing(MixAll.getWSAddr(), clientConfig.getUnitName()); //该功能主要是判断如果namesrv为空，则从约定的服务上去拉取
    this.remotingClient = new NettyRemotingClient(nettyClientConfig, null); //通信客户端的核心实现，底层基于netty的链接
    this.clientRemotingProcessor = clientRemotingProcessor; //请求事件封装处理
    //注册rpc调用的钩子方法，并将事件处理绑定到上层传递过来的事件处理封装类上
    this.remotingClient.registerRPCHook(rpcHook);
    this.remotingClient.registerProcessor(RequestCode.CHECK_TRANSACTION_STATE, this.clientRemotingProcessor, null);

    this.remotingClient.registerProcessor(RequestCode.NOTIFY_CONSUMER_IDS_CHANGED, this.clientRemotingProcessor, null);

    this.remotingClient.registerProcessor(RequestCode.RESET_CONSUMER_CLIENT_OFFSET, this.clientRemotingProcessor, null);

    this.remotingClient.registerProcessor(RequestCode.GET_CONSUMER_STATUS_FROM_CLIENT, this.clientRemotingProcessor, null);

    this.remotingClient.registerProcessor(RequestCode.GET_CONSUMER_RUNNING_INFO, this.clientRemotingProcessor, null);

    this.remotingClient.registerProcessor(RequestCode.CONSUME_MESSAGE_DIRECTLY, this.clientRemotingProcessor, null);
}
```
至此，初始化配置的操作已经完成。接下来，就是继续调用mqClientFactory.start()方法，
```java
public void start() throws MQClientException {

    synchronized (this) {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;
                // If not specified,looking address from name server
                if (null == this.clientConfig.getNamesrvAddr()) {
                    this.mQClientAPIImpl.fetchNameServerAddr();
                }
                // Start request-response channel
                this.mQClientAPIImpl.start(); //启动netty的客户端配置
                // Start various schedule tasks
                this.startScheduledTask();  //启动定时任务，定时进行更新、验证、发送心跳等操作
                // Start pull service
                this.pullMessageService.start();  //拉取消息消费
                // Start rebalance service
                this.rebalanceService.start();  // 设置消费端重新负载，请求的初始化操作也在此方法内执行
                // Start push service
                this.defaultMQProducer.getDefaultMQProducerImpl().start(false);  // 推送消息消费，这里入参为false，是因为前面已经初始化过了，只需要初始化其它操作
                log.info("the client factory [{}] start OK", this.clientId);
                this.serviceState = ServiceState.RUNNING;
                break;
            case RUNNING:
                break;
            case SHUTDOWN_ALREADY:
                break;
            case START_FAILED:
                throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
            default:
                break;
        }
    }
}
```
以上，就是producer（消息发送端）的启动操作过程。接下来，就是重头——消息发送过程。

# RocketMQ消息发送
Broker在启动后会周期性地向NameSrv注册自身及Topic路由信息，而生产者Producer同样会周期性地从NameSrv上拉取最新更新至本地的Topic路由信息。当Producer要开始发送某一Topic的消息时，便会从本地的路由表中找到Topic对应的路由，选择Topic下合适的Broker来发送消息。RocketMQ中，Topic底下包含若干个队列（Queue），也就是说，Topic对Queue是一对多的关系。每个Queue都记录了自己所属的Broker，对于同一个Topic而言，它的多个Queue可能指向同一个Broker。
Producer在发送消息的时候，会根据消息的Topic，选出对应的路由信息，再挑选出具体某个Queue，将消息发送至Queue对应的Broker。

![消息发送](https://i.loli.net/2019/07/12/5d2858b29cc1d81086.png)

假设TopicX上有4个Queue（queue1，queue2，queue3，queue4），那么Producer发送TopicX的消息时，会将消息平均发送到每个Queue，从而发送到每个Queue对应的Broker，至于Broker这边，仅Master节点才能接收Producer发来的消息并写入到本地存储，如果有Slave，则会再从Master同步至Slave。
接下来是发送消息的源码分析环节。

# 消息发送的流程解析
消息发送的主要步骤包括：验证消息、查找路由、消息发送（包含异常处理机制）。
直接来看发送消息的默认实现，org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.sendDefaultImpl(Message, CommunicationMode, SendCallback, long)
```Java
org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.sendDefaultImpl(Message, CommunicationMode, SendCallback, long)
```

由于这个方法逻辑比较多，接下来我们分拆成几个部分来分析。

# 验证消息
第一步，消息发送之前，首先确保生产者处于运行状态，这里调了 this.makeSureStateOK()，然后便是验证消息 Validators.checkMessage(msg, this.defaultMQProducer)，点进去org.apache.rocketmq.client.Validators.checkMessage(Message, DefaultMQProducer)会看到是验证消息是否符合相应的规范，包括具体的规范要求包括：Topic名称，消息体不能为空，消息长度不能等于0且默认不能超过允许发送消息的最大长度4M（maxMessageSize = 1024 * 1024 * 4）。
```Java
public static void checkMessage(Message msg, DefaultMQProducer defaultMQProducer)
    throws MQClientException {
    if (null == msg) {
        throw new MQClientException(ResponseCode.MESSAGE_ILLEGAL, "the message is null");
    }
    // topic
    Validators.checkTopic(msg.getTopic());

    // body
    if (null == msg.getBody()) {
        throw new MQClientException(ResponseCode.MESSAGE_ILLEGAL, "the message body is null");
    }

    if (0 == msg.getBody().length) {
        throw new MQClientException(ResponseCode.MESSAGE_ILLEGAL, "the message body length is zero");
    }

    if (msg.getBody().length > defaultMQProducer.getMaxMessageSize()) {
        throw new MQClientException(ResponseCode.MESSAGE_ILLEGAL,
            "the message body size over max value, MAX: " + defaultMQProducer.getMaxMessageSize());
    }
}
```
# 查找路由
第二步，查找Topic对应的路由信息（留意方法体的代码注释）。
```Java
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
    // 查本地缓存的表
    TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
    if (null == topicPublishInfo || !topicPublishInfo.ok()) { // 本地缓存中没有，则向NameSrv发起请求，并更新本地路由缓存
        this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
    }
    // 如果从NameSrv上查找到了，此处便直接返回找到的路由信息topicPublishInfo
    if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
        return topicPublishInfo;
    } else { // 没有查找到，再次查询topic路由
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
        return topicPublishInfo;
    }
}
```
这里有点吊诡的是，为什么在前面从NameSrv查不到路由信息，第二次就再来查一次，难道再试一次就能查到吗？带着疑问，跟进去方法体里边一探究竟
```java
public boolean updateTopicRouteInfoFromNameServer(final String topic, boolean isDefault,
    DefaultMQProducer defaultMQProducer) {
    try {
        if (this.lockNamesrv.tryLock(LOCK_TIMEOUT_MILLIS, TimeUnit.MILLISECONDS)) {
            try {
                TopicRouteData topicRouteData;
                if (isDefault && defaultMQProducer != null) {
                    topicRouteData = this.mQClientAPIImpl.getDefaultTopicRouteInfoFromNameServer(defaultMQProducer.getCreateTopicKey(),
                        1000 * 3);
                    if (topicRouteData != null) {
                        for (QueueData data : topicRouteData.getQueueDatas()) {
                            int queueNums = Math.min(defaultMQProducer.getDefaultTopicQueueNums(), data.getReadQueueNums());
                            data.setReadQueueNums(queueNums);
                            data.setWriteQueueNums(queueNums);
                        }
                    }
                } else {
                    topicRouteData = this.mQClientAPIImpl.getTopicRouteInfoFromNameServer(topic, 1000 * 3);
                }
   ...
}
```
这次调用 updateTopicRouteInfoFromNameServer(...) ，传入的 isDefault 参数为 true，也就是说，会走 if 分支，这里是调 this.mQClientAPIImpl.getDefaultTopicRouteInfoFromNameServer(defaultMQProducer.getCreateTopicKey(), 1000 * 3)从NameSrv查询Topic路由，不过这回不是查询消息所属的Topic路由信息，而是查询RocketMQ设置的一个默认Topic的路由，进去 defaultMQProducer.getCreateTopicKey() 看到 这个默认的 Topic 是 TBW102 （AUTO_CREATE_TOPIC_KEY_TOPIC = "TBW102"），这个Topic就是用来创建其他Topic所用的。如果某Broker配置了 autoCreateTopicEnable，允许自动创建Topic，那么在该Broker启动后，便会向自己的路由表中插入 TBW102 这个Topic，并注册到NameSrv，表明处理该Topic类型的消息。 如果默认Topic下查询到路由信息，则替换路由信息中读写队列个数为消息生产者默认的队列个数（defaultTopicQueueNums ）。如果isDefault 为false ，则使用参数topic 去查询；如果未查询到路由信息，则返回false ，表示路由信息未变化。
```java
...
// 如果路由信息找到，与本地缓存中的路由信息进行对比，判断路由信息是否发生了改变， 如果未发生变化，则直接返回chaged=false
if (topicRouteData != null) {
    TopicRouteData old = this.topicRouteTable.get(topic);
    boolean changed = topicRouteDataIsChange(old, topicRouteData);
    if (!changed) {
        changed = this.isNeedUpdateTopicRouteInfo(topic);
    } else {
        log.info("the topic[{}] route info changed, old[{}] ,new[{}]", topic, old, topicRouteData);
    }
  ...
}
...
```
然后，更新MQClientInstance Broker地址缓存（路由信息转化为PublishInfo）以及更新该MQClientInstance所管辖的所有消息发送关于该topic的路由信息（路由信息转化为MessageQueue列表，此具体实现在topicRouteData2TopicSubscribeInfo(...)方法，再根据MessageQueue列表进行更新）。
```Java
// Update Pub info
{
    TopicPublishInfo publishInfo = topicRouteData2TopicPublishInfo(topic, topicRouteData);
    publishInfo.setHaveTopicRouterInfo(true);
    Iterator<Entry<String, MQProducerInner>> it = this.producerTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, MQProducerInner> entry = it.next();
        MQProducerInner impl = entry.getValue();
        if (impl != null) {
            impl.updateTopicPublishInfo(topic, publishInfo);
        }
    }
}

// Update sub info
{
    Set<MessageQueue> subscribeInfo = topicRouteData2TopicSubscribeInfo(topic, topicRouteData);
    Iterator<Entry<String, MQConsumerInner>> it = this.consumerTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, MQConsumerInner> entry = it.next();
        MQConsumerInner impl = entry.getValue();
        if (impl != null) {
            impl.updateTopicSubscribeInfo(topic, subscribeInfo);
        }
    }
}
```
```Java
/*
* 循环遍历路由信息的QueueData 信息，如果队列没有写权限，则继续遍历下一个QueueData ；根据brokerName 找到brokerData 信息，
* 找不到或没有找到Master节点（仅Master节点Broker才提供写消息服务），则遍历下一个QueueData ；
* 根据写队列个数，根据topic＋序号创建MessageQueue ，填充topicPublishlnfo 的List<QuueMessage＞ 。完成消息发送的路由查找。
*/
public static Set<MessageQueue> topicRouteData2TopicSubscribeInfo(final String topic, final TopicRouteData route) {
    Set<MessageQueue> mqList = new HashSet<MessageQueue>();
    List<QueueData> qds = route.getQueueDatas();
    for (QueueData qd : qds) {
        if (PermName.isReadable(qd.getPerm())) {
            for (int i = 0; i < qd.getReadQueueNums(); i++) {
                MessageQueue mq = new MessageQueue(topic, qd.getBrokerName(), i);
                mqList.add(mq);
            }
        }
    }

    return mqList;
}
```
所以，当消息所属的Topic，假设叫Topic X吧，它本身没有在任何Broker上配置的时候，生产者就会查询默认Topic TBW102的路由信息，暂时作为Topic X的的路由，并插入到本地路由表中。当TopicX利用该路由发送到 Broker后，Broker发现自己并没有该Topic信息后，便会创建好该Topic，并更新到NameSrv中，表明后续接收TopicX的消息。

整理一下获取Topic路由的步骤：
先从本地缓存的路由表中查询；
没有找到的话，便向NameSrv发起请求，更新本地路由表，再次查询；
如果仍然没有查询到，表明Topic没有事先配置，则用Topic TBW102向NameSrv发起查询，返回TBW102的路由信息，暂时作为Topic的路由。
查找路由的过程解析到此，接下来是选择消息队列的过程。
发送消息
我们此处所谓发送消息，其实是发送到Queue里的，RocketMQ里边的Queue是个抽象的概念，并不是我们所理解的数据结构里的队列Queue，上文已经提到，每个Topic的路由信息（topicRouteData）中可能包含若干Queue，而topicRouteData是由元数据管理中心NameSrv返回的。也就是说，Producer是从NameSrv拉取的路由信息为TopicRouteData，我们不妨先来看下它的属性：
![TopicRouteData](https://i.loli.net/2019/07/12/5d285a8104cb484785.png)
queueDatas 中包含了Topic对应的所有Queue信息，QueueData的结构如下：
![QueueData](https://i.loli.net/2019/07/12/5d285ad2c100e12930.png)
```Java
// 重试次数内进行重试           
for (; times < timesTotal; times++) {
    String lastBrokerName = null == mq ? null : mq.getBrokerName();
    MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName); // 选择某个Queue 用来发送消息
    if (mqSelected != null) {
        mq = mqSelected;
        brokersSent[times] = mq.getBrokerName();
        try {
            beginTimestampPrev = System.currentTimeMillis();
            if (times > 0) {
                //Reset topic with namespace during resend.
                msg.setTopic(this.defaultMQProducer.withNamespace(msg.getTopic()));
            }
            long costTime = beginTimestampPrev - beginTimestampFirst;
            if (timeout < costTime) { // 在超时时间内进行重试
                callTimeout = true;
                break;
            }
            // 进行消息发送的核心实现
            sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
            endTimestamp = System.currentTimeMillis();
            this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
            switch (communicationMode) { // 三种不同的发送方式，相应的处理：除了同步需要重试另一个Broker以确保返回结果，其它直接返回null
                case ASYNC:
                    return null;
                case ONEWAY:
                    return null;
                case SYNC:
                    if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                        if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                            continue;
                        }
                    }

                    return sendResult;
                default:
                    break;
            }
        }
        // 以下是各种catch异常，此处省略这部分的代码
        ...
```
由上面代码可知，选择Queue的具体逻辑在topicPublishInfo.selectOneMessageQueue（lastBrokerName）中。这里在调用时传入了lastBrokerName，目前我们还不知道是为了什么，所以带着疑惑进入方法内部看看吧。
```Java
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    if (this.sendLatencyFaultEnable) {
        try {
            int index = tpInfo.getSendWhichQueue().getAndIncrement();
            for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
                int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
                if (pos < 0)
                    pos = 0;
                MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                    if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
                        return mq;
                }
            }

            final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
            int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
            if (writeQueueNums > 0) {
                final MessageQueue mq = tpInfo.selectOneMessageQueue();
                if (notBestBroker != null) {
                    mq.setBrokerName(notBestBroker);
                    mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);
                }
                return mq;
            } else {
                latencyFaultTolerance.remove(notBestBroker);
            }
        } catch (Exception e) {
            log.error("Error occurred when selecting message queue", e);
        }

        return tpInfo.selectOneMessageQueue();
    }

    return tpInfo.selectOneMessageQueue(lastBrokerName);
}
```
我们来分析一下这段逻辑：
当lastBrokerName不为空时，将计数器进行自增，再遍历TopicPulishInfo中的MessageQueue列表，按照计数器数值对MessageQueue总个数进行取模，再根据取模结果，取出MessageQueue列表中的某个Queue，并判断Queue所属Broker的Name是否和lastBrokerName一致，一致则继续遍历。
当lastBrokerName为空时，同样将计数器进行自增，按照计数器数值对MessageQueue总个数进行取模，再根据取模结果，取出MessageQueue列表中的某个Queue，直接返回。
概括一下，这段逻辑的主要部分就是利用计数器，来进行Queue的负载均衡。
至于lastBrokerName的作用，就是为了做负载均衡。
当某条消息第一次发送时，lastBrokerName 为空，此时就是直接取模进行负载均衡操作。但是如果消息发送失败，就会触发重试机制，发送失败有可能是因为Broker出现来某些故障，或者某些网络连通性问题，所以当消息第N次重试时，就要避开第N-1次时消息发往的Broker，也就是lastBrokerName。
好了，我们已经了解了选择Queue 的来源及消息发送时Queue的负载均衡以及重试机制。下面让我们来看看消息的核心发送过程。

# 发送消息的核心实现
好了，消息发送的核心，就在于最后一步，网络传输了，我们跟踪到 org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.sendKernelImpl(Message, MessageQueue, CommunicationMode, SendCallback, TopicPublishInfo, long) 方法里边
```Java
String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
if (null == brokerAddr) {
    tryToFindTopicPublishInfo(mq.getTopic());
    brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
}
```
拿到Broker地址后，要将消息内容及其他信息封装进请求头：
```java
SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
requestHeader.setTopic(msg.getTopic());
requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
requestHeader.setQueueId(mq.getQueueId());
requestHeader.setSysFlag(sysFlag);
requestHeader.setBornTimestamp(System.currentTimeMillis());
requestHeader.setFlag(msg.getFlag());
requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
requestHeader.setReconsumeTimes(0);
requestHeader.setUnitMode(this.isUnitMode());
requestHeader.setBatch(msg instanceof MessageBatch);
```
请求头部封装好之后，接下来重点来看 org.apache.rocketmq.client.impl.MQClientAPIImpl.sendMessage(String, String, Message, SendMessageRequestHeader, long, CommunicationMode, SendCallback, TopicPublishInfo, MQClientInstance, int, SendMessageContext, DefaultMQProducerImpl)，这方法内部便是创建网络请求，调用封装的Netty接口进行网络传输了。

首先创建请求：
```java
RemotingCommand request = null;
if (sendSmartMsg || msg instanceof MessageBatch) {
    SendMessageRequestHeaderV2 requestHeaderV2 = SendMessageRequestHeaderV2.createSendMessageRequestHeaderV2(requestHeader);
    request = RemotingCommand.createRequestCommand(msg instanceof MessageBatch ? RequestCode.SEND_BATCH_MESSAGE : RequestCode.SEND_MESSAGE_V2, requestHeaderV2);
} else {
    request = RemotingCommand.createRequestCommand(RequestCode.SEND_MESSAGE, requestHeader);
}

request.setBody(msg.getBody());
```
这里按照是否发送 smartMsg ，创建了不同请求命令号的请求，接下来，按照发送方式（单向、同步、异步），调用不同的发送函数：
```java
switch (communicationMode) {
    case ONEWAY:
        this.remotingClient.invokeOneway(addr, request, timeoutMillis);
        return null;
    case ASYNC:
        final AtomicInteger times = new AtomicInteger();
        long costTimeAsync = System.currentTimeMillis() - beginStartTime;
        if (timeoutMillis < costTimeAsync) {
            throw new RemotingTooMuchRequestException("sendMessage call timeout");
        }
        this.sendMessageAsync(addr, brokerName, msg, timeoutMillis - costTimeAsync, request, sendCallback, topicPublishInfo, instance,
            retryTimesWhenSendFailed, times, context, producer);
        return null;
    case SYNC:
        long costTimeSync = System.currentTimeMillis() - beginStartTime;
        if (timeoutMillis < costTimeSync) {
            throw new RemotingTooMuchRequestException("sendMessage call timeout");
        }
        return this.sendMessageSync(addr, brokerName, msg, timeoutMillis - costTimeSync, request);
    default:
        assert false;
        break;
}
```
至此，消息的发送——从Producer把消息传输到Broker的过程分析就已经结束了。
