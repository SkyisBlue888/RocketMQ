# 前言

Broker是RocketMQ核心组件，负责接收并存储消息。

# 启动流程

启动流程从main开始：

- main
  - createBrokerController：
    - buildBrokerController：
      - buildCommandlineOptions：解析命令行参数，获取配置文件路径。
      - 解析配置文件：
        - 校验namesrv地址。
        - 设置brokerId。
        - 设置日志路径系统变量。
      - 创建Controller。
      - 同步Controller配置。
    - controller.initialize：
      - initializeMetadata：加载topic、queue、offset、group、filter、order info等元数据。
      - initializeMessageStore：初始化MessageStore。
      - recoverAndInitService：
        - registerMessageStoreHook：注册消息存储钩子。
        - messageStore.load：初始化消息存储服务。
        - timerMessageStore.load：启动任意时间定时消息服务。
        - scheduleMessageService.load：启动定时消息服务。
        - brokerAttachedPlugin.load：启动broker插件，目前还没有任何实现类。
        - new BrokerMetricsManager：启动监控服务。
        - initializeRemotingServer：初始化remotingServer。
        - initializeResources：初始化所有线程池。
        - registerProcessor：注册各请求码对应Processor。
        - initializeScheduledTasks：初始化所有Broker相关的定时任务。
        - initialTransaction：初始化事务消息相关服务。
        - initialAcl：注册ACL相关RPC钩子。
        - initialRpcHooks：加载所有自定义RPC钩子。
  - start：
    - controller.start：
      - startBasicService：启动存储、网络等基础服务。
      - changeSpecialServiceStatus：启动定时消息、事务消息、pop消费等特定服务。
      - registerBrokerAll：向所有namesrv注册broker。
      - scheduleSendHeartbeat：启动定时向namesrv上报心跳。

# 关闭流程
- shutdownBasicService：关闭存储、网络等各种基础服务，将消息、元数据落盘持久化。
- scheduledFuture.cancel：停止各种线程池定时任务。
- brokerOuterAPI.shutdown：关闭remotingClient、brokerOuterExecutor线程池。

# 部署形态
在代码中可以看到各种部署形态相关的条件判断，在此按时间线统一梳理Broker部署形态的演进：

### 主备集群
集群分master、slave两种角色，master与slave之间支持异步、同步复制消息，不支持failover：
- 数据可靠性：异步复制丢失少量消息，同步复制一条不丢但性能低10%。
- 生产可用性：队列分布在多master，只要还有master可用生产就不中断。
- 消费可用性：master故障后，slave继续提供消费服务。

![多主多从部署形态](https://camo.githubusercontent.com/35ff50a089ea9e56682722001509550e0a165afa658052553698c841a09743ee/68747470733a2f2f73342e617831782e636f6d2f323032322f30312f32362f374c756f7a642e706e67)

### Dledger集群

4.5.0引入dledger，集群对外分master、slave两种角色，内部选举分leader、follower、candidate、unknown 4种角色，要求分片内节点数为单数且 >= 3。

- 数据可靠性：Dledger通过raft commitlog同步副本。
- 生产、消费可用性：master故障后，failover从节点升主继续提供生产、消费服务。

![Dledger部署形态](https://camo.githubusercontent.com/79f573e987efff4eae45a9f3d21727f9125fd9cd062c85b93d2e03a3d260f550/68747470733a2f2f73342e617831782e636f6d2f323032322f30312f32362f374c4b736b382e706e67)

### Broker Container

5.0引入Container特性，支持单进程内混部多个Broker提高资源利用率，并且能动态增减Container内的Broker，相关[RIP](https://github.com/apache/rocketmq/wiki/RIP-31-Support-RocketMQ-BrokerContainer)。

BrokerContainer单进程部署视图：

![BrokerContainer单进程视图](https://camo.githubusercontent.com/700669e86ad6b638a1eab30024e80505bcfe0ebbe016f04c079e5654eed99d64/68747470733a2f2f73342e617831782e636f6d2f323032322f30312f32362f374c4d5a48502e706e67)

BrokerContainer 2主2备对端部署视图：

![BrokerContainer 2主2备对端部署](https://camo.githubusercontent.com/58314a991f867f0a88a17c6db0e0ca2034e15184bb5a353f464d66156fb468b2/68747470733a2f2f73342e617831782e636f6d2f323032322f30312f32362f374c516935542e706e67)

BrokerContainer 3主3备对端部署视图：

![BrokerContainer 3主3备对端部署](https://camo.githubusercontent.com/fc0365b6e81c12aeb4f6b19ccc342563057eee4aafbba99d0d7d7045e299a247/68747470733a2f2f73342e617831782e636f6d2f323032322f30312f32362f374c514d61362e706e67)

BrokerController继承体系演化：

非Container部署时使用BrokerController，Container部署时dledger模式下任意节点使用InnerBrokerControlle；普通主备模式的主节点使用InnerBrokerControlle，备节点使用InnerSalveBrokerController。

![BrokerController继承体系](https://camo.githubusercontent.com/d6ef64c941af591af49b75528f0b7ce7c02d9aa88c8ea46bb73fec3ecaac66ab/68747470733a2f2f73342e617831782e636f6d2f323032322f30312f32362f374c513347442e706e67)

### Dledger Controller

5.0将dledger抽离为独立可选的一致性选主组件：dledger controller，可嵌入namesrv部署也可以独立部署，角色类似redis sentinel。

![Dledger Controller核心设计](https://camo.githubusercontent.com/0f892ca6926f1f6eebb0807412e8b504948432f076b93e782438db244ee01956/68747470733a2f2f73312e617831782e636f6d2f323032322f30372f30312f6a51703265312e706e67)

### Slave Acting Master Mode

主备部署模式下，主节点下线后，目标副本组的部分操作将受限，包括：

- 客户端无法续锁导致无法顺序消费。
- 客户端无法主动结束半状态的事务消息。
- 运维工具无法查询earliestMsgStoreTime 、offset。
- 副本组无法提供定时消息、事务消息、pop消费能力。
- 主节点重新上线后将立即向从节点同步元数据，容易发生offset回退导致重复消费。

5.0引入Slave Acting Master Mode，主备部署模式下：

- 主节点下线后，brokerId最小的备节点将临时接管消费能力、特殊消息能力以及一些只有主节点才能完成的任务。
- 主节点恢复上线后，首先进入pre-online状态从接管的备节点反向同步元数据后再正式上线，避免元数据回退。

![Slave Acting Master](https://camo.githubusercontent.com/200847573b7dd275f91236daaf2bf092aa065c728bde7cf79965a20dd3fefc26/68747470733a2f2f73342e617831782e636f6d2f323032322f30322f30352f486e576744782e706e67)
