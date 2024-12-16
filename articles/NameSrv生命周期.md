# 1 前言
NameServer是一种轻量级、无状态的注册中心，提供注册、查询Broker路由信息的能力。本篇关注NameSrv组件的启动流程、关闭流程。

# 2 启动流程
启动流程从NameSrvStartUpmain函数开始，调用链如下：


main
- main0
  - parseCommandlineAndConfigFile：解析来自命令行和配置文件的配置项
  - createAndStartNamesrvController
    - createNamesrvController
    - start
      - controller.initialize
	    - loadConfig
		- initiateNetworkComponents
		  - newNettyRemotingServer
          - newNettyRemotingClient
		- initiateThreadExecutor
		  - initdefaultExecutor
		  - initclientRequestExecutor
		- registerProcessor
		  - registerProcessor：注册处理GET_ROUTEINFO_BY_TOPIC请求的Processor，线程池使用clientRequestExecutor
		  - registerDefaultProcessor：注册默认处理请求的Processor，线程池使用defaultExecuto
		- startScheduleService
		  - scanNotActiveBroker
		  - printAllPeriodically
		  - printWaterMark
		- initiateSslContext
		  - remotingServer.loadSslContext
		- initiateRpcHooks
		  - registerRPCHookZoneRouteRPCHook
		- controller.start
		  - remotingServer.start：remotingServer用于启动端口供外部节点访问namesrv
		  - remotingClient.start：remotingClient用于从namesrv访问外部节点
		  - fileWatchService.start：监听SSL相关文件，用于动态加载SSL认证信息
		  - routeInfoManager.start：nameSrv提供的路由信息管理核心服务
- createAndStartControllerManager
  - controllerManagerMain：在开启Controller模式下才会创建controllermanager，一般默认不会开启，跳过不分析。

# 3关闭流程
在启动过程中会注册进程退出时的钩子函数：NameSrvController.shutdown，内容非常少，总体来说就是：
- 关闭线程池
- 关闭fileWatchService
- 关闭routeInfoManager
- 关闭remotingClient、remotingServer
