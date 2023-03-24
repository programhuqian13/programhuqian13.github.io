# 设置Functions Workers
我们有两种方式设置Functions Workers：
- 使用brokers 运行function workers
  - 在进程或线程模式下运行函数时，不需要资源隔离
  - 将function workers配置为在Kubernetes上运行函数(Kubernetes解决资源隔离问题)
- 隔离使用function workers：当我们想隔离函数和broker
## 使用brokers 运行function workers
下图演示了与broker一起运行的函数工作者的部署：
![img_52.png](img_52.png)
图中的服务url表示Pulsar client和Pulsar admin用来连接到Pulsar集群的Pulsar服务url