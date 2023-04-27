# Pulsar Functions CLI and YAML 配置

## Pulsar functions YAML配置
可以使用预定义的YAML文件来配置功能。下表概述了所需的字段和参数：

|字段名|类型|相关命令参数|描述|
|----|----|----|----|
|runtimeFlags|String|N/A|我们想传递给运行时的任何标志(仅适用于进程和Kubernetes运行时)|
|tenant|String|--tenant|函数的租户|
|namespace|String|--namespace|函数的命名空间|
|name|String|--name|函数的名称|
|className|String|--classname|函数的类名|
|functionType|String|--function-type|函数的类型|
|inputs|List<String>|-i, --inputs|函数的输入主题。可以将多个主题指定为逗号分隔的列表|
|customSerdeInputs|Map<String,String>|--custom-serde-inputs|从输入主题到SerDe类名的映射|
|topicsPattern|String|--topics-pattern|要从名称空间下的主题列表中使用的主题模式，注意:--input和--topic-pattern互斥。对于Java函数，需要在--custom-serde-inputs中为模式添加SerDe类名|
|customSchemaInputs|Map<String,String>|--custom-schema-inputs|从输入主题到模式属性的映射|
|customSchemaOutputs|Map<String,String>|--custom-schema-outputs|从输出主题到模式属性的映射|
|inputSpecs|Map<String,ConsumerConfig>|--input-specs|从输入到自定义配置的映射|
|output|String|-o, --output|函数的输出主题。如果不指定，则不输出|
|producerConfig|ProducerConfig|--producer-config|生产者的自定义配置|
|outputSchemaType|String|-st, --schema-type|用于消息输出的内置模式类型或自定义模式类名称|
|outputSerdeClassName|String|--output-serde-classname|用于消息输出的SerDe类|
|logTopic|String|--log-topic|生成函数日志的主题|
|processingGuarantees|String|--processing-guarantees|应用于函数的处理保证(交付语义)。可用值:至少一次，最多一次，有效一次，手动|
|retainOrdering|Boolean|--retain-ordering|函数是否按顺序消费和处理消息|
|retainKeyOrdering|String|--retain-key-ordering|函数是否按键顺序消费和处理消息|
|batchBuilder|String|--batch-builder|使用 producerConfig.batchBuilder 替换. 注意:batchBuilder将很快在代码中弃用|
|forwardSourceMessageProperty|Boolean|--forward-source-message-property|在处理过程中是否将输入消息的属性转发到输出主题。设置为false时，表示关闭转发功能|
|userConfig|Map<String,Object>|--user-config|用户定义的配置键/值|
|secrets|Map<String,Object>|--secrets|从secretName到对象的映射，这些对象封装了底层秘密提供程序获取secrets的方式|
|runtime|String|N/A|函数的运行时。取值:java、python、go|
|autoAck|Boolean|--auto-ack|框架是否自动确认消息，在未来的版本中将弃用此配置。如果指定传递语义，框架将自动确认消息。如果我们不希望框架自动ack消息，请将processinGassurances设置为MANUAL|
|maxMessageRetries|Int|--max-message-retries|在放弃之前重试处理消息的次数|
|deadLetterTopic|String|--dead-letter-topic|用于存储未成功处理的消息的主题|
|subName|String|--subs-name|如果需要，输入主题消费者使用的脉冲星源订阅的名称|
|parallelism|Int|--parallelism	|函数的并行性因子，即要运行的函数实例的数量|
|resources|Resources|N/A|N/A|
|fqfn|String|--fqfn|函数的完全限定函数名(FQFN)|
|windowConfig|WindowConfig|N/A|N/A|
|timeoutMs|Long|--timeout-ms|消息超时（以毫秒为单位）|
|jar|String|--jar|函数(用Java编写)的JAR文件的路径。它还支持worker可以下载包的URL路径，包括HTTP、HTTPS、file(文件协议，假设文件已经存在于worker主机上)和function(来自包管理服务的包URL)|
|py|String|--py|主python/python wheel 文件的路径（用python编写）。它还支持工人可以从HTTP，HTTPS，FILE（文件协议）和功能（Package URL中的文件协议）中下载软件包的URL路径（文件协议）|
|go|String|--go|函数的主Go可执行二进制文件的路径(用Go语言编写)。它还支持worker可以下载包的URL路径，包括HTTP、HTTPS、file(文件协议，假设文件已经存在于worker主机上)和function(来自包管理服务的包URL)|
|cleanupSubscription|Boolean|N/A|当函数被删除时，是否应该删除函数创建或使用的订阅|
|customRuntimeOptions|String|--custom-runtime-options|对选项进行编码以自定义运行时的字符串|
|maxPendingAsyncRequests|Int|--max-message-retries|每个实例挂起的最大异步请求数，以避免大量并发请求|
|exposePulsarAdminClientEnabled|Boolean|N/A|Pulsar管理客户端是否公开给函数上下文。缺省情况下，禁用该功能|
|subscriptionPosition|String|--subs-position|用于从指定位置消费消息的pulsar源订阅的位置。默认值为“Latest”|

## ConsumerConfig配置

|字段名|类型|相关命令参数|描述|
|----|----|----|----|
|schemaType|String|N/A|N/A|
|serdeClassName|String|N/A|N/A|
|isRegexPattern|Boolean|N/A|N/A|
|schemaProperties|Map<String,String>|N/A|N/A|
|consumerProperties|Map<String,String>|N/A|N/A|
|receiverQueueSize|Int|N/A|N/A|
|cryptoConfig|CryptoConfig|N/A||
|poolMessages|Boolean|N/A|N/A|

## ProducerConfig配置

|字段名|类型|相关命令参数|描述|
|----|----|----|----|
|maxPendingMessages|Int|N/A|用于保存等待从代理接收确认的消息的队列的最大大小|
|maxPendingMessagesAcrossPartitions|Int|N/A|跨所有分区的maxPendingMessages的数量|
|useThreadLocalProducers|Boolean|N/A|N/A|
|cryptoConfig|CryptoConfig|N/A||
|batchBuilder|String|--batch-builder|批量construction方法的类型。取值为DEFAULT和KEY_BASED。默认值为DEFAULT|

## Resources配置

|字段名|类型|相关命令参数|描述|
|----|----|----|----|
|cpu|double|--cpu|需要为每个函数实例分配的核心CPU(仅适用于Kubernetes运行时)|
|ram|Long|--ram|需要分配每个功能实例的字节中的RAM（仅适用于process/kubernetes运行时）|
|disk|Long|--disk|每个函数实例需要分配的磁盘字节数(仅适用于Kubernetes运行时)|

## WindowConfig配置

|字段名|类型|相关命令参数|描述|
|----|----|----|----|
|windowLengthCount|Int|--window-length-count|每个窗口的消息数|
|windowLengthDurationMs|Long|--window-length-duration-ms|每个窗口的持续时间(毫秒)|
|slidingIntervalCount|Int|--sliding-interval-count|窗口滑动后的消息数|
|slidingIntervalDurationMs|Long|--sliding-interval-duration-ms|窗口滑动的持续时间|
|lateDataTopic|String|N/A|N/A|
|maxLagMs|Long|N/A|N/A|
|watermarkEmitIntervalMs|Long|N/A|N/A|
|timestampExtractorClassName|String|N/A|N/A|
|actualWindowFunctionClassName|String|N/A|N/A|

## CryptoConfig配置

|字段名|类型|相关命令参数|描述|
|----|----|----|----|
|cryptoKeyReaderClassName|String|N/A||
|cryptoKeyReaderConfig|Map<String, Object>|N/A|N/A|
|encryptionKeys|String[]|N/A|N/A|
|producerCryptoFailureAction|ProducerCryptoFailureAction|N/A|N/A|
|consumerCryptoFailureAction|ConsumerCryptoFailureAction|N/A|N/A|

## 例子

```yaml
tenant: "public"
namespace: "default"
name: "config-file-function"
inputs:
  - "persistent://public/default/config-file-function-input-1"
  - "persistent://public/default/config-file-function-input-2"
output: "persistent://public/default/config-file-function-output"
jar: "function.jar"
parallelism: 1
resources:
  cpu: 8
  ram: 8589934592
autoAck: true
userConfig:
  foo: "bar"
```

# Window Function 上下文
    Java SDK提供了对可由Window 函数使用的Window 上下文对象的访问。这个上下文对象为脉冲星窗口函数提供了大量的信息和功能，如下所示

- Spec:Spec包含函数的基本信息
  - 与函数关联的所有输入主题和输出主题的名称
  - 与该功能关联的租户和命名空间
  - pulsar window function的名称、ID和版本
  - 调用window function的实例数
  - 运行window function的Pulsar函数实例ID
  - 输出模式的内置类型或自定义类名

获取input topics:

```java
public class GetInputTopicsWindowFunction implements WindowFunction<String, Void> {
    @Override
    public Void process(Collection<Record<String>> inputs, WindowContext context) throws Exception {
        Collection<String> inputTopics = context.getInputTopics();
        System.out.println(inputTopics);

        return null;
    }

}
```

获取 output topics：

```java
public class GetOutputTopicWindowFunction implements WindowFunction<String, Void> {
    @Override
    public Void process(Collection<Record<String>> inputs, WindowContext context) throws Exception {
        String outputTopic = context.getOutputTopic();
        System.out.println(outputTopic);

        return null;
    }
}
```

获取tenant:

```java
public class GetTenantWindowFunction implements WindowFunction<String, Void> {
    @Override
    public Void process(Collection<Record<String>> inputs, WindowContext context) throws Exception {
        String tenant = context.getTenant();
        System.out.println(tenant);

        return null;
    }

}
```

获取命名空间：

```java
public class GetNamespaceWindowFunction implements WindowFunction<String, Void> {
    @Override
    public Void process(Collection<Record<String>> inputs, WindowContext context) throws Exception {
        String ns = context.getNamespace();
        System.out.println(ns);

        return null;
    }

}
```

获取函数名：

```java
public class GetNameOfWindowFunction implements WindowFunction<String, Void> {
    @Override
    public Void process(Collection<Record<String>> inputs, WindowContext context) throws Exception {
        String functionName = context.getFunctionName();
        System.out.println(functionName);

        return null;
    }

}
```

获取函数id:

```java
public class GetFunctionIDWindowFunction implements WindowFunction<String, Void> {
    @Override
    public Void process(Collection<Record<String>> inputs, WindowContext context) throws Exception {
        String functionID = context.getFunctionId();
        System.out.println(functionID);

        return null;
    }

}
```

获取函数版本：

```java
public class GetVersionOfWindowFunction implements WindowFunction<String, Void> {
    @Override
    public Void process(Collection<Record<String>> inputs, WindowContext context) throws Exception {
        String functionVersion = context.getFunctionVersion();
        System.out.println(functionVersion);

        return null;
    }

}
```

获取实例id：

```java
public class GetInstanceIDWindowFunction implements WindowFunction<String, Void> {
    @Override
    public Void process(Collection<Record<String>> inputs, WindowContext context) throws Exception {
        int instanceId = context.getInstanceId();
        System.out.println(instanceId);

        return null;
    }

}
```

获取实例数：

```java
public class GetNumInstancesWindowFunction implements WindowFunction<String, Void> {
    @Override
    public Void process(Collection<Record<String>> inputs, WindowContext context) throws Exception {
        int numInstances = context.getNumInstances();
        System.out.println(numInstances);

        return null;
    }

}
```

获取output schema type:

```java
public class GetOutputSchemaTypeWindowFunction implements WindowFunction<String, Void> {

    @Override
    public Void process(Collection<Record<String>> inputs, WindowContext context) throws Exception {
        String schemaType = context.getOutputSchemaType();
        System.out.println(schemaType);

        return null;
    }
}
```

- Logger：window function使用的日志对象，可用于创建窗口函数日志消息

使用Java SDK的Pulsar window functions可以访问SLF4j Logger对象，该对象可用于在所选日志级别生成日志

```java
import java.util.Collection;
import org.apache.pulsar.functions.api.Record;
import org.apache.pulsar.functions.api.WindowContext;
import org.apache.pulsar.functions.api.WindowFunction;
import org.slf4j.Logger;

public class LoggingWindowFunction implements WindowFunction<String, Void> {
    @Override
    public Void process(Collection<Record<String>> inputs, WindowContext context) throws Exception {
        Logger log = context.getLogger();
        for (Record<String> record : inputs) {
            log.info(record + "-window-log");
        }
        return null;
    }

}
```

如果需要函数生成日志，请在创建或运行函数时指定日志主题:

```shell
bin/pulsar-admin functions create \
  --jar my-functions.jar \
  --classname my.package.LoggingFunction \
  --log-topic persistent://public/default/logging-function-logs \
  # Other function configs
```

- User config：访问任意用户配置值

当我们运行或更新使用SDK创建的Pulsar函数时，可以使用--user-config标志将任意键/值对传递给它们。键/值对必须指定为JSON

```shell
bin/pulsar-admin functions create \
  --name word-filter \
 --user-config '{"forbidden-word":"rosebud"}' \
  # Other function configs
```

我们可以使用以下api获取window function的用户定义信息

getUserConfigMap：

```java
/**
 * Get a map of all user-defined key/value configs for the function.
 *
 * @return The full map of user-defined config values
 */
Map<String, Object> getUserConfigMap();
```

getUserConfigValue:

```java
/**
 * Get any user-defined key/value.
 *
 * @param key The key
 * @return The Optional value specified by the user for that key.
 */
Optional<Object> getUserConfigValue(String key);
```

getUserConfigValueOrDefault:

```java
/**
 * Get any user-defined key/value or a default value if none is present.
 *
 * @param key
 * @param defaultValue
 * @return Either the user config value associated with a given key or a supplied default value
 */
Object getUserConfigValueOrDefault(String key, Object defaultValue);
```

Java SDK上下文对象使我们可以通过命令行访问向Pulsar窗口函数提供的键/值对（作为JSON）

**对于传递给Java窗口函数的所有密钥/值对，键和值都是字符串。要将该值设置为不同的类型，我们需要从字符串类型中对其进行验证**

```shell
bin/pulsar-admin functions create \
   --user-config '{"word-of-the-day":"verdure"}' \
  # Other function configs
```

每次调用该函数时，UserConfigFunction都会记录字符串“今日单词是绿色的”(这意味着每次消息到达时)。
只有当通过多种方式(如命令行工具或REST API)使用新的配置值更新函数时，word-of-the-day的用户配置才会更改

```java
import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;
import org.slf4j.Logger;

import java.util.Optional;

public class UserConfigWindowFunction implements WindowFunction<String, String> {
    @Override
    public String process(Collection<Record<String>> input, WindowContext context) throws Exception {
        Optional<Object> whatToWrite = context.getUserConfigValue("WhatToWrite");
        if (whatToWrite.get() != null) {
            return (String)whatToWrite.get();
        } else {
            return "Not a nice way";
        }
    }
}
```

如果没有提供值，则可以访问整个用户配置映射或设置默认值

```java
// Get the whole config map
Map<String, String> allConfigs = context.getUserConfigMap();

// Get value or resort to default
String wotd = context.getUserConfigValueOrDefault("word-of-the-day", "perspicacious");
```

- Routing：pulsar window function支持路由。pulsar window function根据发布接口向任意主题发送消息

可以使用context.publish()接口发布任意数量的结果

```java
public class PublishWindowFunction implements WindowFunction<String, Void> {
    @Override
    public Void process(Collection<Record<String>> input, WindowContext context) throws Exception {
        String publishTopic = (String) context.getUserConfigValueOrDefault("publish-topic", "publishtopic");
        String output = String.format("%s!", input);
        context.publish(publishTopic, output);

        return null;
    }

}
```

- Metrics：记录指标的接口

Pulsar window function可以将任意的度量发布到度量接口，并可以查询

注意：如果Pulsar window  functions使用Java的语言原生接口，则该函数无法向Pulsar发布度量和统计信息

我们可以在每个键的基础上使用上下文对象记录指标：

```java
import java.util.Collection;
import org.apache.pulsar.functions.api.Record;
import org.apache.pulsar.functions.api.WindowContext;
import org.apache.pulsar.functions.api.WindowFunction;


/**
 * Example function that wants to keep track of
 * the event time of each message sent.
 */
public class UserMetricWindowFunction implements WindowFunction<String, Void> {
    @Override
    public Void process(Collection<Record<String>> inputs, WindowContext context) throws Exception {

        for (Record<String> record : inputs) {
            if (record.getEventTime().isPresent()) {
                context.recordMetric("MessageEventTime", record.getEventTime().get().doubleValue());
            }
        }

        return null;
    }
}
```

- State storage：在状态存储器中存储和检索状态的接口 

pulsar window function使用Apache BookKeeper作为状态存储接口。Apache Pulsar安装(包括独立安装)包括BookKeeper bookies的部署

Apache Pulsar集成了Apache BookKeeper表服务来存储函数的状态。例如，WordCount函数可以通过Pulsar Functions状态api将其计数器状态存储到BookKeeper表服务中

状态是键值对，其中键是字符串，值是任意的二进制数据，将示例存储为64位大型二进制值。键的范围为单个pulsar function，并在该function的实例之间共享

目前，Pulsar window function公开Java API来访问、更新和管理状态。当我们使用Java SDK函数时，可以在上下文对象中使用这些api

|Java API|描述|
|----|----|
|incrCounter|增加按键引用的内置分布式计数器|
|getCounter|获取键的计数器值|
|putState|更新键的状态值|

我们可以使用以下api来访问、更新和管理Java窗口函数中的状态：

incrCounter：

```java
/**
 * Increment the built-in distributed counter referred by key
 * @param key The name of the key
 * @param amount The amount to be incremented
 */
void incrCounter(String key, long amount);
```

getCounter:

```java
/**
 * Retrieve the counter value for the key.
 *
 * @param key name of the key
 * @return the amount of the counter value for this key
 */
long getCounter(String key);
```

putState:

```java
/**
 * Update the state value for the key.
 *
 * @param key name of the key
 * @param value state value of the key
 */
void putState(String key, ByteBuffer value);
```






