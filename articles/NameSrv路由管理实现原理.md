# 前言
本篇关注NameSrv如何实现路由信息的存储、查询、注册与更新。

# 路由信息
首先对路由信息做一个总体的描述，我觉得路由信息的内容从逻辑上可以粗略分为2个层面：
- 部署层面，包括：集群信息、分片信息、节点信息、心跳信息
- 业务层面，包括：主题信息、队列信息

# 关键元数据
通过举例来理解一些关键元数据的含义，以下图的部署场景为例：
- 包含2个独立的RocketMQ Cluster
- 每个Cluster包含3组Broker
- 每组Broker包含1主1从共2个Broker节点

![2个独立的RocketMQ Cluster](../RocketMQClusters.drawio.png)

关键元数据的含义：
- brokerAddr：Broker节点的ip地址，在网络层面区分不同Broker节点的连接地址。
- clusterName：RocketMQ集群的名称，在集群维度区分不同的RocketMQ集群。
- brokerName：Broker分组的名称，在集群维度区分不同的Broker分组。
- brokerRole：Broker节点的角色，在分组维度区分每个Broker节点的角色。
- brokerId：Broker节点的ID，在分组维度唯一标识每个Broker节点。

# 数据结构
NameSrv RouteInfoManager中总共使用了以下字段存储路由信息：
- Map<String/* brokerName */, BrokerData> brokerAddrTable
- Map<BrokerAddrInfo/* brokerAddr */, BrokerLiveInfo> brokerLiveTable
- Map<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable
- Map<String/* topic */, Map<String/* brokerName */, QueueData>> topicQueueTable
- Map<BrokerAddrInfo/* brokerAddr \*/, List\<String\>/\* Filter Server */> filterServerTable
- Map<String/* topic */, Map<String/*brokerName*/, TopicQueueMappingInfo>> topicQueueMappingInfoTable
下面开始分析每个字段以及相关的数据结构。

#### brokerAddrTable
保存每个Broker分组地址信息的键值对，键为BrokerName，值为BrokerData，以上图中的2个集群为例，json格式如下：
```json
{
    "broker-0": {
        "cluster": "cluster-0",
        "brokerName": "broker-0",
        "brokerAddrs": {
            "0": "192.168.41.84:10101",
            "1": "192.168.41.87:10101"
        }
    },
    "broker-1": {
        "cluster": "cluster-0",
        "brokerName": "broker-1",
        "brokerAddrs": {
            "0": "192.168.41.85:10101",
            "1": "192.168.41.88:10101"
        }
    },
    "broker-2": {
        "cluster": "cluster-0",
        "brokerName": "broker-2",
        "brokerAddrs": {
            "0": "192.168.41.86:10101",
            "1": "192.168.41.89:10101"
        }
    },
    "broker-3": {
        "cluster": "cluster-1",
        "brokerName": "broker-3",
        "brokerAddrs": {
            "0": "192.168.41.74:10101",
            "1": "192.168.41.77:10101"
        }
    },
    "broker-4": {
        "cluster": "cluster-1",
        "brokerName": "broker-4",
        "brokerAddrs": {
            "0": "192.168.41.75:10101",
            "1": "192.168.41.78:10101"
        }
    },
    "broker-5": {
        "cluster": "cluster-1",
        "brokerName": "broker-5",
        "brokerAddrs": {
            "0": "192.168.41.76:10101",
            "1": "192.168.41.79:10101"
        }
    }
}
```

#### brokerLiveTable
保存每个Broker节点的心跳存活信息的键值对，以上图中Cluster-0/Broker-0/master为例，键为BrokerAddrInfo，json格式如下：
```json
{
    "clusterName": "cluster-0",
    "brokerAddr": "192.168.41.84"
}



值为BrokerLiveInfo，json格式如下：

{
    "channel": null,
    "dataVersion": {
        "counter": 1,
        "stateVersion": 0,
        "timestamp": 1710387063790
    },
    "haServerAddr": "192.168.41.84:10102",
    "heartbeatTimeoutMillis": 120000,
    "lastUpdateTimestamp": 1710387065225
}
```

#### clusterAddrTable
保存每个集群下的Broker分组信息，，键为以上图中2个集群为例，json格式如下：
```json
{
    "cluster-0": [
        "broker-0",
        "broker-1",
        "broker-2"
    ],
    "cluster-1": [
        "broker-3",
        "broker-4",
        "broker-5"
    ]
}
```

#### topicQueueTable
保存每个Topic在每个Broker分组上的队列的总览信息，键为Topic名称，值为BrokerName与QueueData间的映射Map，以上图中的topic-1为例，json格式如下：
```json
{
    "topic-1": {
        "broker-0": {
            "brokerName": "broker-0",
            "readQueueNums": 1,
            "writeQueueNums": 1,
            "perm": 6,  /*2禁写，4禁读，6读写放开*/
            "topicSysFlag": 0 /*主题系统标签，默认为0*/
        }
    }
}
```

#### RegisterBroker
每个Broker节点默认每30s向NameSrv注册一次路由信息，NameSrv相关代码见RouteInfoManager.registerBroker，大致的流程如下：
- 加全局写锁
- 更新clusterAddrTable
- 更新brokerAddrTable
  - 新Broker加入brokerAddrTable
  - 如果此次注册观察到主备倒换，移除旧master Addr
- 如果请求注册的Broker节点之前已经注册过，并且此次注册的数据版本低于已注册的数据版本，则拒绝此次注册
- 如果deleteTopicWithBrokerRegistration=true，清理已经被删除的Topic路由信息
- 更新topicQueueTable
- 更新topicQueueMappingInfoTable
- 更新brokerLiveTable
- 更新filterServerTable
- 如果请求注册的Broker是备节点，响应返回体中填充主节点地址信息
- 如果此次注册观察到主备倒换，namesrv将主动通知broker倒换事件
- 释放全局写锁
