# Pulsar client(客户端)
    Pulsar公开了一个带有Java、Go、Python、c++和c#语言绑定的客户端API。客户端API优化并封装Pulsar的客户端-代理通信协议，并公开一个简单直观的API供应用程序使用。
    实际上，目前官方的Pulsar客户端库支持透明重连接和连接故障转移到代理，消息排队，直到被代理确认，以及一些启发式操作，如连接重试和回退。
# 客户端设置阶段
    当应用创建producer或者consumer时，Pulsar客户端库需要启动一个设置阶段，包括两个步骤
    1. 客户端试图通过向代理发送HTTP查找请求来确定主题的所有者。请求可以到达一个活动代理，通过查看(缓存的)zookeeper元数据，
    知道谁在服务主题，或者在没有人服务主题的情况下，尝试将它分配给负载最少的代理
    2. 一旦客户端库拥有代理地址，它就创建一个TCP连接(或重用池中现有的连接)并对其进行身份验证，在此连接中，客户端和代理交换来自自定义协议的二进制命令。
    此时，客户端发送一个创建producer或者consumer的命令到broker中。在验证了授权策略之后，它将符合要求。
每当TCP连接中断时，客户端立即重新启动这个设置阶段，并不断尝试以指数级回退重新建立生产者或消费者，直到操作成功
# Reader interface
    在Pulsar中，“standard”使用者界面包括使用使用者侦听主题、处理传入消息，并最终在处理这些消息时确认这些消息。每当创建一个新订阅时，它最初被定位在主题的末尾(默认情况下)，
    与该订阅关联的消费者从随后创建的第一条消息开始阅读。每当使用者使用已存在的订阅连接到主题时，它就开始从该订阅中解压缩的最早的消息中读取。总之，使用使用者接口，订阅游标由Pulsar自动管理，以响应消息确认。
Pulsar的阅读器接口允许应用程序手动管理游标。当使用阅读器连接到主题(而不是使用者)时，需要指定阅读器连接到主题时从哪条消息开始阅读。当连接到一个主题时，阅读器接口允许从如下几点读取：

- 主题中最早可用的消息
- 主题中最新可用的消息
- 还有一些最早和最晚之间的信息。如果选择此选项，则需要显式地提供消息ID。应用程序将负责提前“知道”这个消息ID，可能是从持久数据存储或缓存中获取它

对于使用Pulsar为流处理系统提供有效的一次处理语义等用例，阅读器接口很有帮助。对于这个用例，流处理系统必须能够将主题“倒回”到特定的消息，并从那里开始读取。阅读器接口为Pulsar客户端提供了在主题中“手动定位”自己所需的低级抽象

在内部，阅读器接口作为使用者实现，使用具有随机分配名称的独占的、非持久的主题订阅。
    
    不像subscription/consumer，reader本质上是非持久性的，并且不会阻止主题中的数据被删除，因此`强烈建议`配置数据保留。如果在足够的时间内没有为主题配置数据保留，
    则可能会删除reader尚未阅读的消息。这会导致reader基本上跳过消息。为主题配置数据保留可以保证reader在一定的时间内阅读消息
    还请注意，一个阅读器可能有一个“backlog”，但该指标仅用于用户了解阅读器的落后程度。该指标不用于任何积压配额计算.

![img_26.png](img_26.png)

```java
import org.apache.pulsar.client.api.Message;
import org.apache.pulsar.client.api.MessageId;
import org.apache.pulsar.client.api.Reader;
// Create a reader on a topic and for a specific message (and onward)
Reader<byte[]> reader = pulsarClient.newReader()
    .topic("reader-api-test")
    .startMessageId(MessageId.earliest)
    .create();

while (true) {
    Message message = reader.readNext();

    // Process the message
}
```

```java
// create a reader that reads from the latest available message
Reader<byte[]> reader = pulsarClient.newReader()
    .topic(topic)
    .startMessageId(MessageId.latest)
    .create();
```

```java
// create a reader that reads from some message between the earliest and the latest
byte[] msgIdBytes = // Some byte array
MessageId id = MessageId.fromByteArray(msgIdBytes);
Reader<byte[]> reader = pulsarClient.newReader()
    .topic(topic)
    .startMessageId(id)
    .create();
```
