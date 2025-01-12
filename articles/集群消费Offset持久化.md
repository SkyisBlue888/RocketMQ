# 前言

本篇总结集群消费场景下Offset如何存储、更新。

# Broker

首先来看服务端如何维护Offset。

#### Data Struct

Broker服务端使用文件存储每个消费组的Offset：

- 文件路径：${storePathRootDir}/store/config

- 文件名：ConsumerOffset.json

- 内容样例：
```json
  {
  	"dataVersion":{
  		"counter":1448,
  		"stateVersion":0,
  		"timestamp":1736661762584
  	},
  	"groupTopicMap":{
  		"pull_test_sub":["test"],
        "pull_test_asg":["test"],
		"test":["%RETRY%push_test","test"]
  	},
  	"lmqOffsetTable":{},
  	"offsetTable":{
  		"test@pull_test_asg":{0:37960,1:37961,2:37959,3:37959},
  		"test@push_test@broadcast":{0:100831,1:100832,2:100831,3:100830},
  		"%RETRY%push_test@test":{0:0},
  		"test@push_test":{0:165243,1:165245,2:165243,3:165242},
  		"test@pull_test_sub":{0:37958,1:37959,2:37959,3:37958}
  	}
  }
```

从上述样例可以看出：

- Offset存储在offsetTable字段中。
- Table下每行数据按{ConsumerGroup}@{Topic}命名。
- 每行数据包含Topic下每个队列的Offset。
- 以集群Push模式消费时还会另外存储RETRY Topic的Offset，而Pull模式不会。
- 广播消费会在每行命名最后加上@broadcast标记。

#### Memory Store

当Broker收到UPDATE_CONSUMER_OFFSET = 15请求码后，先将offset更新至内存map，处理逻辑：

- ConsumerManageProcessor.updateConsumerOffset：校验请求体以及Header。
  - ConsumerOffsetManager.commitOffset：调用Manager提交offset。
  - ConcurrentMap<Integer, Long>.put：将offset值存入offsetTable。

#### Disk Store

Broker启动时开启了定时任务每秒将offset持久化落盘：

- ConsumerOffsetManager.persist：
  - ConsumerOffsetManager.encode：将ConsumerOffsetManager所有字段编码成json格式字符串。
  - MixAll.string2File：json字符串写入磁盘配置文件。

# Client

再来看看客户端如何维护Offset。

#### OffsetStore

OffsetStore接口包括2个子类：

- LocalFileOffsetStore：基于本次文件存储，用于广播消费。
- RemoteBrokerOffsetStore：基于远端Broker存储，用于集群消费。

两个子类均使用offsetTable内存ConcurrentMap<MessageQueue, ContrllableOffset>维护offset，区别仅在与persist。

#### LocalFileOffsetStore

按不同消费组名称分组将offset持久化在本地offsets.json。

#### RemoteBrokerOffsetStore

发送UPDATE_CONSUMER_OFFSET请求码至Broker持久化。