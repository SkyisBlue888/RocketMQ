# 前言

本篇总结 4.x Push消费者集群消费者从Broker拉取并消费一条Remoting协议消息的过程。

# Client

###  相关类继承体系

![PushConsumer类关系.drawio](C:\Users\c00567567\Desktop\PushConsumer类关系.drawio.png)

###  消费者启动

DefaultPushConsumer.start：

- TraceDispatcher.start：启动消息轨迹服务，如果开启消息轨迹。
- DefaultPushConsumerImpl.start：
  - DefaultLitePullConsumerImpl.checkConfig：校验配置是否合法。
  - DefaultLitePullConsumerImpl.copySubscription：从DefaultPushConsumer复制subscribe信息。
  - 初始化mQClientFactory、rebalanceImpl、pullAPIWrapper、offsetStore、consumeMessageService。
  - OffsetStore.load：加载Offset。
  - ConsumeMessageService.start：
    - ConsumeMessageConcurrentlyService.start：启动线程池周期性清理所有MQ中过期的消息。
    - ConsumeMessageOrderlyService.start：启动线程池周期性锁住所有MQ，确保顺序消费。
  - MQClientInstance.registerConsumer：向MQClientInstance单例实例注册当前消费者。
  - MQClientInstance.start：启动MQClientInstance单例。
    - MQClientAPIImpl.start：
      - RemotingClient.start：启动RemotingClient。
    - MQClientInstance.startScheduledTask：启动调整线程池、持久化消费offset、发送心跳、清理离线Broker、刷新路由信息、刷新namesrv地址等定时任务。
    - PullMessageService.start：启动消息拉取服务，用于Push消费场景。
    - RebalanceService.start：启动重平衡服务，用于Push消费场景。
    - DefaultMQProducerImpl.start：启动CLIENT_INNER_PRODUCER_GROUP生产者组的内部默认生产者，用于消息回发等场景。
  - DefaultPushConsumerImpl.updateTopicSubscribeInfoWhenSubscriptionChanged：更新Topic订阅关系路由信息。
  - MQClientInstance.rebalanceImmediately：立即重平衡一次。

###  Rebalance

 Push消费首先由RebalanceService定时触发：

- RebalanceService.run：
  - MQClientInstance.doRebalance：
    - DefaultPushConsumerImpl.tryRebalance：
    - RebalancePushImpl.doRebalance：
    - RebalanceImpl.rebalanceByTopic：
      - AllocateMessageQueueStrategy.allocate：按消费组统一队列分配策略，确定当前消费者所分配的队列。
      - RebalanceImpl.updateProcessQueueTableInRebalance：
        - RebalanceImpl.removeUnnecessaryMessageQueue：移除不再分配给此消费者的队列。
        - RebalancePushImpl.createProcessQueue：如分配了新队列，创建ProcessQueue。
        - RebalancePushImpl.computePullFromWhere：计算下次拉取Offset。
        - RebalancePushImpl.dispatchPullRequest：分发处理pullRequest。
          - PullMessageService.executePullRequestImmediately：将pullRequest加入messageRequestQueue，待PullMessageService线程无限循环拉取处理。

###  Execute Pull request

RebalanceService将PullRequest放入messageRequestQueue后，PullMessageService线程将开始无限循环从messageRequestQueue中取出并处理PullRequest：

- PullMessageService.run：
  - PullMessageService.pullMessage：
  - DefaultPushConsumerImpl.pullMessage：
    - RebalancePushImpl.computePullFromWhereWithException：根据offset策略计算待拉取offset。
    - PullAPIWrapper.pullKernelImpl：处理客户端版本兼容，构造RequestHeader。
    - MQClientAPIImpl.pullMessage：根据communicationMode，选择oneway、sync、async请求方式。
      - MQClientAPIImpl.pullMessageASync：Push模式默认采用Async方式。
        - RemotingClient.invokeASync：发送RequestCode至Broker。
        - MQClientAPIImpl.processPullResponse：处理响应结果，构造PullResult。

###  PullCallback OnSuccess

请求返回并成功构建PullResult后，将由事先注册的PullCallback处理响应结果：

- FOUND：
  - PullRequest.setNextOffset：更新下次拉取的Offset。
  - ConsumerStatsManager.incPullRT：更新Consumer RT时延统计。
  - 如果拉取的消息为空：
    - DefaultPushConsumerImpl.executePullRequestImmediately：立即再次拉取消息。
  - 如果拉取的消息不为空：
    - ProcessQueue.putMessage：填充ProcessQueue。
    - ConsumeMessageConcurrentlyService.submitConsumeRequest：提交ConsumeRequest。
- NO_NEW_MSG、NO_MATCHED_MSG：
  - PullRequest.setNextOffset：更新下次拉取的Offset。
  - DefaultPushConsumerImpl.executePullRequestImmediately：立即再次拉取消息。
- OFFSET_ILLEGAL：
  - ProcessQueue.setDropped：冻结ProcessQueue。
  - DefaultPushConsumerImpl.executeTask：提交异步Task，重新更新并持久化目标队列的Offset，之后立即触发重平衡重新加入ProcessQueue。

###  ConsumeMessageConcurrentlyService submitConsumeRequest

按照DefaultMQPushConsumer.consumeMessageBatchMaxSize将所有消息分批提交至consumeExecutor线程池。

consumeMessageBatchMaxSize默认为1，因此只存在消费成功、失败两种情况，不存在部分成功、部分失败的情况。

如果consumeMessageBatchMaxSize > 1，那么消费方需要在Context中自行设置ackIndex来反馈从哪条消息开始失败。

###  ConsumeRequest run

ConsumeRequest实现了Runnable，每个Request在线程池中并发执行run完成消费：

- DefaultPushConsumerImpl.executeHookBefore：执行前置钩子函数。
- MessageListenerConcurrently.consumerMessage：执行业务侧编写的监听器代码，完成消息消费。
- DefaultPushConsumerImpl.executeHookAfter：执行后置钩子函数。
- ConsumeMessageCurrentlyService.processConsumeResult：根据业务消费返回的status、context，处理消费结果。

###  ConsumeMessageCurrentlyService processConsumeResult

- 根据消费结果更新ConsumerState：
  - CONSUME_SUCCESS：消费成功
    - 根据业务侧消费后在ConsumeConcurrentlyContext中设置的ackIndex，计算这批消息中消费成功、消费失败的消息数量。
    - 根据消费成功的消息数量，调用ConsumerStatsManager.incConsumeOKTPS。
    - 根据消费失败的消息数量，调用ConsumerStatsManager.incConsumeFailedTPS。
  - RECONSUME_LATER：消费失败
    - ackIndex直接设为-1，方便后续将这批消息全部重投。
    - 调用ConsumerStatsManager.incConsumeFailedTPS。
- ConsumeMessageConcurrentlyService.sendMessageBack：
  - CLUSTERING：集群消费，遍历这批消息，下标ackIndex之后是消费失败的消息，重投至RETRY Topic待重新消费。
  - BROADCASTING：广播消费，消费失败的消息仅打印日志不做任何其他处理。

- ProcessQueue.removeMessage：移除这批所有消息，失败的消息会通过RETRY机制重新消费。
- OffsetStore.updateOffset：更新Offset，包括处理失败的消息，失败的消息会通过RETRY机制重新消费。
