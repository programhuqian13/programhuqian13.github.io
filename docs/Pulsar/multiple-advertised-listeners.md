# 多个播发侦听器(Multiple advertised listeners)
    当Pulsar集群部署在生产环境中时，它可能需要为代理公开多个广播地址。例如，当我们在Kubernetes中部署Pulsar集群，并希望不在同一Kubernetes集群中的其他客户端连接到Pulsar集群时，
    我们需要为外部客户端分配一个代理URL。但是同一Kubernetes集群中的客户端仍然可以通过Kubernetes的内部网络连接到Pulsar集群
#Advertised listeners
为了确保内部和外部网络中的客户端都可以连接到Pulsar集群，Pulsar在代理配置文件中引入了advertisedListeners和internalListenerName配置选项，
以确保代理支持公开多个已发布的侦听器，并支持内部和外部网络流量分离

- advertisedListeners用于指定多个已发布的侦听器。代理使用侦听器作为负载管理器和包所有者数据中的代理标识符。
  The advertisedListeners is formatted as <listener_name>:pulsar://<host>:<port>, <listener_name>:pulsar+ssl://<host>:<port>. You can set up the advertisedListeners like advertisedListeners=internal:pulsar://192.168.1.11:6660,internal:pulsar+ssl://192.168.1.11:6651
- internalListenerName用于指定代理使用的内部服务URL。我们可以通过选择一个advertisedlistener来指定internalListenerName。如果internalListenerName不存在，代理使用第一个发布的侦听器的侦听器名称作为internalListenerName

在设置advertisedlistener之后，客户机可以选择其中一个侦听器作为服务URL，以创建到代理的连接，只要网络是可访问的。但是，如果客户端在主题上创建生产者或消费者，
则客户端必须向代理发送查找请求以获取所有者代理，然后连接到所有者代理以发布消息或使用消息。因此，必须允许客户端获得与客户端使用的侦听器名称相同的服务URL。这有助于保持客户端简单和安全

# 使用
- 在代理配置文件中配置多个已发布的侦听器。
```text
advertisedListeners={listenerName}:pulsar://xxxx:6650,{listenerName}:pulsar+ssl://xxxx:6651
```
- 为客户端指定侦听器名称
```java
PulsarClient client = PulsarClient.builder()
    .serviceUrl("pulsar://xxxx:6650")
    .listenerName("external")
    .build();
```