# 前言
RocketMQ 5.0时代引入无状态Proxy，一方面承担了服务端协议适配、权限管理、消息管理等计算能力，使得Broker更专注存储能力从而实现存算分离，更好得适应云原生环境以实现资源弹性调度；另一方面系统将部分客户端计算能力下沉至Proxy，使得客户端变得更加轻量化。

# 启动流程
启动流程从ProxyStartup.main开始：
- initConfiguration
  - ConfigurationManager.initEnv：初始化proxy_home环境变量，用于找到ssl、broker相关配置文件。
  - ConfigurationManager.intConfig：读取并初始化json格式配置文件。
  - setConfigFromCommandLineArgument：优先读取并采用命令行参数，包括：brokerConfigPath、nameSrvAddr、proxyMode。
- initThreadPoolMonitor：初始化线程池监控定时任务，默认触发周期3s，检查项可包括：队列堆积是否达到最大值85%，队列等待时长是否超时。
- createServerExecutor：创建grpc server使用的线程池。
- createMessagingProcessor：创建请求处理器，分Cluster、Local两种模式；简单说Local模式许多内部组件会交由BrokerController代为管理。
- loadAccessValidators：加载ACL校验Service，支持多种自定义实现，只需implements AccessValidator接口。
- GrpcServerBuilder.newBuilder：初始化Grpc Netty Server。
  - addService(createServiceProcessor)：创建GrpcMessagingApplication，它维护了各种业务场景使用的线程池。
  - addService(ChannelzService.newInstance)：启动Channelz用于监控调试grpc server。
  - addService(ProtoReflectionService.newInstance)：为Protobuf服务提供反射服务（包括反射服务本身）。
    分别跟踪可变服务和不可变服务。如果任何一组服务包含具有相同服务、方法、类型或扩展名的声明的多个Protobuf文件，则抛出异常
  - configInterceptor(accessValidators)：配置指定的acl校验器。
- new RemotingProtocolServer(messagingProcessor, accessValidators)：启动remoting server。
- PROXY_START_AND_SHUTDOWN.start()：一个接一个的启动grpc server、remoting server。

# 关闭流程
- preShutdown：目前所见没有任何实现。
- shutdown：关闭所有线程池。
