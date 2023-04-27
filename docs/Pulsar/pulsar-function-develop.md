# 如何开发Pulsar 函数
## 使用API
    我们可以使用Java,Python和Go的API进行开发Pulsar function
### 使用原生API
    语言原生接口提供了一种简单而干净的编写Java/Python函数，通过向所有传入字符串添加感叹号并将输出字符串发布到主题。它没有外部依赖。

```java
package org.tony.pulsar.function.develop;

import java.util.function.Function;

/**
 * 通过原生api开发function
 * @author Tony
 */
public class JavaNativeFunction implements Function<String,String> {

    @Override
    public String apply(String s) {
        return String.format("%s!",s);
    }
}

```

### 使用Pulsar SDK开发
    SDK指定了一个包含上下文对象作为参数的功能接口
```java
package org.tony.pulsar.function.develop;

import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;

/**
 * 通过pulsar sdk开发pulsar function
 * @author Tony
 */
public class JavaPulsarSdkDevelopFunction implements Function<String,String> {
    @Override
    public String process(String input, Context context) throws Exception {
        return String.format("%s!",input);
    }
}
```

函数的返回类型可以包装在Record泛型中，这使您可以更好地控制输出消息，例如主题、模式、属性等。使用Context::newOutputRecordBuilder方法来构建这个Record输出
```java
package org.tony.pulsar.function.develop;

import org.apache.pulsar.client.api.Schema;
import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;
import org.apache.pulsar.functions.api.Record;

import java.util.HashMap;
import java.util.Map;

/**
 * 包装返回类型
 * @author Tony
 */
public class JavaPulsarSdkRecordFunction implements Function<String, Record<String>> {
    @Override
    public Record<String> process(String input, Context context) throws Exception {
        String output = String.format("%s!",input);
        Map<String,String> properties = new HashMap<>(context.getCurrentRecord().getProperties());
        context.getCurrentRecord().getTopicName().ifPresent(topic-> properties.put("input_topic",topic));
        return context.newOutputRecordBuilder(Schema.STRING).value(output).properties(properties).build();
    }
}
```
### 使用Pulsar 扩展SDK开发（JAVA）
    这个扩展的Pulsar Functions SDK提供了两个额外的接口来初始化和释放外部资源
- 通过使用initialize接口，可以初始化外部资源，这些资源只需要在函数实例启动时进行一次初始化
- 通过使用close接口，可以在函数实例关闭时关闭所引用的外部资源

下面的示例使用了Pulsar Functions SDK for Java的扩展接口，在函数实例启动时初始化RedisClient，并在函数实例关闭时释放它：
```java
import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;
import io.lettuce.core.RedisClient;

public class InitializableFunction implements Function<String, String> {
    private RedisClient redisClient;

    private void initRedisClient(Map<String, Object> connectInfo) {
        redisClient = RedisClient.create(connectInfo.get("redisURI"));
    }

    /**
     * init的时候的操作逻辑
     * @param context
     */
    @Override
    public void initialize(Context context) {
        Map<String, Object> connectInfo = context.getUserConfigMap();
        redisClient = initRedisClient(connectInfo);
    }

    @Override
    public String process(String input, Context context) {
        String value = client.get(key);
        return String.format("%s-%s", input, value);
    }

    /**
     * 关闭的时候的操作逻辑
     */
    @Override
    public void close() {
        redisClient.close();
    }
}
```

## 传递用户定义的配置
    我们在运行和升级函数的时候，可以通过--user-config传递相关的配置，格式为json格式，key/value的形式
```text
bin/pulsar-admin functions create \
  # Other function configs
  --user-config '{"word-of-the-day":"verdure"}'
```

我们在java中可以这样使用：
```java
import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;
import org.slf4j.Logger;

import java.util.Optional;

public class UserConfigFunction implements Function<String, Void> {
    @Override
    public void apply(String input, Context context) {
        Logger LOG = context.getLogger();
        Optional<String> wotd = context.getUserConfigValue("word-of-the-day");
        if (wotd.isPresent()) {
            LOG.info("The word of the day is {}", wotd);
        } else {
            LOG.warn("No word of the day provided");
        }
        return null;
    }
}
```
## 使用指标监控函数
    要确保正在运行的函数在任何时候都是健康的，可以配置函数将任意指标发布到可查询的指标接口
 注意：使用Java或Python的语言原生接口无法向Pulsar发布指标和统计信息

可以使用内置指标和自定义指标来监视函数：

- 使用内置的函数度量。Pulsar Functions公开了可收集和用于监视Java、Python和Go函数运行状况的指标。
- 设置您的自定义指标。除了内置的度量之外，Pulsar还允许我们为Java和Python函数定制度量。函数工作者自动向Prometheus收集用户定义的指标，你可以在Grafana中检查它们
  
下面是一个示例，说明了如何通过在每个键的基础上使用Context对象来定制Java、Python和Go函数的指标。

例如，可以为process-count键设置一个指标，并在函数每次处理消息时为eleven-count键设置另一个指标：
```java
import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;

public class MetricRecorderFunction implements Function<Integer, Void> {
    @Override
    public void apply(Integer input, Context context) {
        // Records the metric 1 every time a message arrives
        context.recordMetric("hit-count", 1);

        // Records the metric only if the arriving number equals 11
        if (input == 11) {
            context.recordMetric("elevens-count", 1);
        }

        return null;
    }
}
```
## 启用函数安全
    如果我们需要启用安全设置，我们需要在function workers中启用安全设置
配置function workers

要使用上下文中的secret api，我们需要为函数工作者设置以下两个参数：
- secretsProviderConfiguratorClassName
- secretsProviderConfiguratorConfig

Pulsar Functions提供了两种类型的SecretsProviderConfigurator实现，它们都可以直接用作secretsProviderConfiguratorClassName的值
- org.apache.pulsar.functions.secretsproviderconfigurator.DefaultSecretsProviderConfigurator:这是一个secrets提供程序的基本版本，它在ClearTextSecretsProvider中连接到函数实例。
- org.apache.pulsar.functions.secretsproviderconfigurator.KubernetesSecretsProviderConfigurator:它使用Kubernetes内置的secrets，并将它们绑定为函数容器中的环境变量(通过EnvironmentBasedSecretsProvider)，以确保secrets在运行时对函数可用

function workers使用org.apache.pulsar.functions.secretsproviderconfigurator.SecretsProviderConfigurator接口在启动函数实例时选择SecretsProvider类名及其相关配置

函数实例使用org.apache.pulsar.functions.secretsprovider.SecretsProvider接口来获取Secrets。SecretsProvider使用的实现由SecretsProviderConfigurator决定

一旦设置了SecretsProviderConfigurator，我们就可以使用上下文对象获取secrets，如下所示：

```java
import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;
import org.slf4j.Logger;

public class GetSecretValueFunction implements Function<String, Void> {

    @Override
    public Void process(String input, Context context) throws Exception {
        Logger LOG = context.getLogger();
        String secretValue = context.getSecret(input);

        if (!secretValue.isEmpty()) {
            LOG.info("The secret {} has value {}", intput, secretValue);
        } else {
            LOG.warn("No secret with key {}", input);
        }

        return null;
    }
}
```

## 配置状态存储
Pulsar函数使用Apache BookKeeper作为状态存储接口。Pulsar集成了BookKeeper表服务来存储函数的状态。例如，WordCount函数可以通过state api将其计数器的状态存储到BookKeeper表服务中。

状态是键值对，其中一个键是一个字符串和它的值是任意二进制数据，计数器被存储为64位的二进制。键的作用域为单个函数，并在该函数的实例之间共享

### 调用状态 API
    Pulsar函数公开了用于改变和访问状态的api
- 增量计数器:可以使用incrCounter将给定键的计数器增加给定的量。如果密钥不存在，则创建一个新key

```java
void incrCounter(String key, long amount);
```
要异步增加计数器，可以使用incrCounterAsync
```java
CompletableFuture<Void> incrCounterAsync(String key, long amount);
```

- 检索计数器:可以使用getCounter来检索被incrCounter改变的给定键的计数器

```java
long getCounter(String key);
```
要异步检索由incrCounterAsync改变的计数器
```java
CompletableFuture<Long> getCounterAsync(String key);
```

- 更新状态:除了计数器 API,Pulsar也提供了修改状态的api

```java
void putState(String key, ByteBuffer value);
```
要异步更新给定键的状态，可以使用putStateAsync
```java
CompletableFuture<Void> putStateAsync(String key, ByteBuffer value);
```

- 检索状态:可以使用getState来检索给定键的状态

```java
ByteBuffer getState(String key);
```
要异步检索给定键的状态，可以使用getStateAsync
```java
CompletableFuture<ByteBuffer> getStateAsync(String key);
```
- 删除状态：计数器和二进制值共享相同的键空间，因此此API将删除其中任何一种类型

```java
void deleteState(String key);
```
### 通过 CLI 查询状态
    除了使用State api将函数的状态存储在Pulsar的状态存储中并从存储中检索它之外，我们还可以使用CLI命令查询函数的状态

```text
bin/pulsar-admin functions querystate \
    --tenant <tenant> \
    --namespace <namespace> \
    --name <function-name> \
    --state-storage-url <bookkeeper-service-url> \
    --key <state-key> \
    [---watch]
```

WordCountFunction的例子演示了如何在脉冲星函数中存储状态：
1. 函数使用正则表达式\\.将接收到的字符串拆分为多个单词
2. 对于每个单词，该函数通过incrCounter(key, amount)将counter加1

```java
import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;

import java.util.Arrays;

public class WordCountFunction implements Function<String, Void> {
    @Override
    public Void process(String input, Context context) throws Exception {
        Arrays.asList(input.split("\\.")).forEach(word -> context.incrCounter(word, 1));
        return null;
    }
}
```

## 调用pulsar admin api
    使用Java SDK的Pulsar函数可以访问Pulsar管理客户端，这允许Pulsar管理客户端管理对Pulsar集群的API调用

下面是如何使用从函数上下文中公开的Pulsar管理客户端的示例：

```java
import org.apache.pulsar.client.admin.PulsarAdmin;
import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;

/**
 * In this particular example, for every input message,
 * the function resets the cursor of the current function's subscription to a
 * specified timestamp.
 */
public class CursorManagementFunction implements Function<String, String> {

    @Override
    public String process(String input, Context context) throws Exception {
        PulsarAdmin adminClient = context.getPulsarAdmin();
        if (adminClient != null) {
            String topic = context.getCurrentRecord().getTopicName().isPresent() ?
                    context.getCurrentRecord().getTopicName().get() : null;
            String subName = context.getTenant() + "/" + context.getNamespace() + "/" + context.getFunctionName();
            if (topic != null) {
                // 1578188166 below is a random-pick timestamp
                adminClient.topics().resetCursor(topic, subName, 1578188166);
                return "reset cursor successfully";
            }
        }
        return null;
    }
}
```
使我们的函数能够访问Pulsar管理客户端，需要在conf/functions_worker.yml设置exposeAdminClientEnabled=true
要测试它是否启用，可以使用命令pulsar-admin functions localrun，并带标志--web-service-url，如下所示:
```text
bin/pulsar-admin functions localrun \
 --jar my-functions.jar \
 --classname my.package.CursorManagementFunction \
 --web-service-url http://pulsar-web-service:8080 \
 # Other function configs
```

## 使用schema注册
    Pulsar有一个内置的模式注册表，并与流行的模式类型捆绑在一起，如Avro、JSON和Protobuf。Pulsar Functions可以利用来自输入主题的现有模式信息并派生输入类型。模式注册表也适用于输出主题

## 使用SerDe进行序列化和反序列化
    当向Pulsar主题发布数据或从Pulsar主题消费数据时，Pulsar函数使用SerDe(序列化和反序列化)。
    默认情况下，SerDe的工作方式取决于我们为特定函数使用的语言(Java或Python)。
    但是，在这两种语言中，我们都可以为更复杂的、特定于应用程序的类型编写自定义SerDe逻辑

以下基本Java类型是内置的，默认情况下支持Java函数:string、double、integer、float、long、short和byte。要自定义Java类型，需要实现以下接口：

```java
public interface SerDe<T> {
    T deserialize(byte[] input);
    byte[] serialize(T input);
}
```

SerDe以以下方式处理Java函数:
- 如果输入和输出主题具有模式，则Pulsar Functions将为SerDe使用该模式
- 如果输入或输出主题不存在，则Pulsar Functions采用以下规则确定SerDe:
  - 如果指定了模式类型，则Pulsar Functions使用指定的模式类型
  - 如果指定了SerDe，则Pulsar Functions使用指定的SerDe，并且输入和输出主题的模式类型是字节
  - 如果既没有指定Schema类型也没有指定SerDe，则Pulsar Functions使用内置的SerDe。对于非基本模式类型，内置的SerDe以JSON格式序列化和反序列化对象

例如，假设我们正在编写一个处理tweet对象的函数。我们可以参考下面的Java中的Tweet类示例：
```java
public class Tweet {
    private String username;
    private String tweetContent;

    public Tweet(String username, String tweetContent) {
        this.username = username;
        this.tweetContent = tweetContent;
    }

    // Standard setters and getters
}
```

要在函数之间直接传递Tweet对象，需要提供一个自定义SerDe类。在下面的例子中，Tweet对象基本上是字符串，用户名和Tweet内容由|分隔:

```java
package com.example.serde;

import org.apache.pulsar.functions.api.SerDe;

import java.util.regex.Pattern;

public class TweetSerde implements SerDe<Tweet> {
    
    public Tweet deserialize(byte[] input) {
        String s = new String(input);
        String[] fields = s.split(Pattern.quote("|"));
        return new Tweet(fields[0], fields[1]);
    }

    public byte[] serialize(Tweet input) {
        return "%s|%s".format(input.getUsername(), input.getTweetContent()).getBytes();
    }
}
```

要将定制的SerDe应用于特定函数，我们需要：
- 将Tweet和TweetSerde类打包到一个JAR中
- 在部署函数时，指定JAR和SerDe类名的路径

下面是使用create命令通过应用自定义SerDe来部署函数的示例：
```text
bin/pulsar-admin functions create \
  --jar /path/to/your.jar \
  --output-serde-classname com.example.serde.TweetSerde \
  # Other function attributes
```
**注意：自定义SerDe类必须打包到函数jar中。**