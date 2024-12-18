# All Request Code

| 类别              | 请求码                             |
| :---------------- | ---------------------------------- |
| Properties Config | UPDATE_NAMESRV_CONFIG              |
| Properties Config | GET_NAMESRV_CONFIG                 |
| KV Config         | PUT_KV_CONFIG                      |
| KV Config         | GET_KV_CONFIG                      |
| KV Config         | DELETE_KV_CONFIG                   |
| Broker Register   | REGISTER_BROKER                    |
| Broker Register   | UNREGISTER_BROKER                  |
| Broker Register   | BROKER_HEARTBEAT                   |
| Broker Route Info | GET_BROKER_MEMBER_GROUP            |
| Broker Route Info | GET_BROKER_CLUSTER_INFO            |
| Broker Write Perm | WIPE_WRITE_PERM_OF_BROKER          |
| Broker Write Perm | ADD_WRITE_PERM_OF_BROKER           |
| Topic Management  | GET_ALL_TOPIC_LIST_FROM_NAMESERVER |
| Topic Management  | GET_TOPICS_BY_CLUSTER              |
| Topic Management  | DELETE_TOPIC_IN_NAMESRV            |
| Topic Management  | REGISTER_TOPIC_IN_NAMESRV          |
| Topic Management  | GET_SYSTEM_TOPIC_LIST_FROM_NS      |
| Topic Management  | GET_UNIT_TOPIC_LIST                |
| Topic Management  | GET_HAS_UNIT_SUB_TOPIC_LIST        |
| Topic Management  | GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST |

# Properties Config
UPDATE_NAMESRV_CONFIG：更新namesrv.conf；更新项必须是系统所支持的；更新项会同步至磁盘文件。
GET_NAMESRV_CONFIG：查询namesrv.conf配置，包含NameSrvConfig、NettyServerConfig。

# KV Config
PUT_KV_CONFIG：更新${user.home}/namesrv/kvConfig.json配置，支持区分namespace。
GET_KV_CONFIG：查询${user.home}/namesrv/kvConfig.json配置，支持区分namespace。
DELETE_KV_CONFIG：删除${user.home}/namesrv/kvConfig.json配置，支持区分namespace。

#  Broker Register
REGISTER_BROKER：见[RegisterBroker](https://github.com/SkyisBlue888/RocketMQ/blob/main/articles/NameSrv%E8%B7%AF%E7%94%B1%E7%AE%A1%E7%90%86%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86.md#registerbroker)。

BROKER_HEARTBEAT：见[BrokerHeartBeat](https://github.com/SkyisBlue888/RocketMQ/blob/main/articles/NameSrv%E8%B7%AF%E7%94%B1%E7%AE%A1%E7%90%86%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86.md#brokerheartbeat)。

UNREGISTER_BROKER：清理各字段与UnRegister Broker相关信息。

# Broker Route Info

GET_BROKER_MEMBER_GROUP：仅用于Broker同步其所属集群分片成员信息，默认周期1秒。

GET_BROKER_CLUSTER_INFO：获取NameSrv已知悉的所有集群信息。

# Broker Write Perm

WIPE_WRITE_PERM_OF_BROKER：擦除指定Broker所有Topic所有Queue写入权限。

ADD_WRITE_PERM_OF_BROKER：增加指定Broker所有Topic所有Queue写入权限。

# Topic Management

GET_UNIT_TOPIC_LIST：参考[ISSUE 1712](https://github.com/apache/rocketmq/issues/1712)。

GET_HAS_UNIT_SUB_TOPIC_LIST：同上。

GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST：同上。
