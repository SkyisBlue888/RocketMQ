# 版本
基于开源5.3.1

# 模块
Server
- NameSrv
- Broker
- Proxy
- Container
- Controller

Client
- Producer
- Consumer
- Tracing

Metadata
- Topic
- Subscription
- Acl

Ops
- Metric
- Admin

Protocol
- grpc
- remoting

Basic
- Store
- Remoting
- TieredStore

# 知识点
4.x
- Remoting:
  - ~~remoting protocol~~
- Store:
  - ~~commitlog storage~~
  - ~~consume queue storage~~
  - ~~index storage~~
  - expired msg deletion
  - performance optimization
- Distribution:
  - master slave sync
  - dledger failover
- Advanced Msg:
  - tracing msg
  - scheduled msg
  - transaction msg
- Ops:
  - monitor metric
  - container

5.x
- Pop Consume:
  - pop consume protocol
  - pop consume server
  - pop consume 4.x client
  - pop consume 5.x client
- Grpc:
  - ~~grpc protocol~~
  - ~~grpc client~~
  - ~~grpc server~~
- Advanced Msg:
  - tracing msg
  - scheduled msg
  - transaction msg
  - fifo msg
- Store:
  - tiered storage
- Acl:
  - acl
  - auth
