# 如何部署Pulsar 函数
部署函数有两种模式：
1. 集群模式：我们可以向Pulsar集群提交一个函数，该集群将负责运行该函数
2. 单机模式：我们可以确定函数在本地机器上运行的位置
## 前期准备
1. 在自己的机器上本地运行一个独立集群
2. 在Kubernetes、Amazon Web Services、bare metal等上运行Pulsar集群
## CLI 的默认参数
我们可以在pulse-admin CLI下使用功能相关的命令来部署功能。pulsar提供了多种命令，例如：
- create：集群模式下部署function
- trigger： 用于trigger函数

CLI下需要配置的参数及默认值如下表所示：

|参数|默认值|
|----|----|
|函数名称|N/A，可以为函数名指定任何值|
|Tenant|N/A，可以为函数名指定任何值|
|Namespace|N/A，可以为函数名指定任何值|
|Output topic|{input topic}-{function name}-output 如果函数的输入主题名是incoming且函数名是exclamation，则输出主题名是incoming-exclamation-output。|
|Processing guarantees|ATLEAST_ONCE|
|Pulsar service URL|pulsar://localhost:6650|

以create命令为例。下面的函数具有函数名(MyFunction)、租户(public)、命名空间(default)、订阅类型(SHARED)、处理保证(至少一次)的默认值,和Pulsar服务URL:pulsar://localhost:6650
```text
bin/pulsar-admin functions create \
  --jar my-pulsar-functions.jar \
  --classname org.example.MyFunction \
  --inputs my-function-input-topic1,my-function-input-topic2
```
## 在localrun模式下部署函数
    在localrun模式下部署函数时，它将运行在我们在笔记本电脑上输入命令的机器上，或者运行在AWS EC2实例中
可以使用localrun命令运行函数的单个实例。要运行多个实例，可以多次使用localrun命令

以localrun命令的使用示例如下：
```text
bin/pulsar-admin functions localrun \
  --py myfunc.py \
  --classname myfunc.SomeFunction \
  --inputs persistent://public/default/input-1 \
  --output persistent://public/default/output-1
```
**在localrun模式下，Java函数使用线程运行时;Python和Go函数使用进程运行时。**

默认情况下，该函数通过本地代理服务URL连接到运行在同一台机器上的Pulsar集群。如果希望将其连接到非本地Pulsar集群，可以使用--brokerServiceUrl标志指定不同的代理服务URL
```text
bin/pulsar-admin functions localrun \
  --broker-service-url pulsar://my-cluster-host:6650 \
  # Other function parameters
```
## 在集群模式下部署函数
    在集群模式下部署函数会将函数上传到函数worker，这意味着函数是由该worker调度的
在集群模式下部署功能，使用create命令：
```text
bin/pulsar-admin functions create \
  --py myfunc.py \
  --classname myfunc.SomeFunction \
  --inputs persistent://public/default/input-1 \
  --output persistent://public/default/output-1
```
当需要更新运行在集群模式下的函数时，使用update命令:
```text
bin/pulsar-admin functions update \
  --py myfunc.py \
  --classname myfunc.SomeFunction \
  --inputs persistent://public/default/new-input-topic \
  --output persistent://public/default/new-output-topic
```
### 将资源分配给函数实例
    在集群模式下运行函数时，可以指定可以分配给每个函数实例的资源
下表概述了可以分配给函数实例的资源:

|资源|规定|支持运行时|
|----|----|----|
|CPU|cpu核数|Kubernetes|
|RAM|内存|Kubernetes|
|Disk space|磁盘空间|Kubernetes|

例如，下面的命令为一个函数分配8个内核、8GB RAM和10GB磁盘空间：
```text
bin/pulsar-admin functions create \
  --jar target/my-functions.jar \
  --classname org.example.functions.MyFunction \
  --cpu 8 \
  --ram 8589934592 \
  --disk 10737418240
```
### 启用并行处理
    在集群模式下，我们可以指定并行度(要运行的实例数量)以启用一个函数的并行处理
实例1：

在部署函数时指定create命令的--parallelism标志
```text
bin/pulsar-admin functions create \
  --parallelism 3 \
  # Other function info
```
**对于已存在的函数，可以使用update命令调整并行度。**

实例2：

在通过YAML部署函数配置时指定并行度参数：
```yaml
# function-config.yaml
parallelism: 3
inputs:
  - persistent://public/default/input-1
output: persistent://public/default/output-1
# other parameters
```
对于已存在的函数，可以使用update命令调整并行度:
```text
bin/pulsar-admin functions update \
  --function-config-file function-config.yaml
```
### 启用端到端加密
    要执行端到端加密，我们可以在pulse-admin CLI中使用应用程序配置的公钥和私钥对指定--producer-config和--input-specs
    只有拥有有效密钥的使用者才能解密加密的消息
加密/解密的相关配置 CryptoConfig包含了两个ProducerConfig 和inputSpecs，关于CryptoConfig的具体可配置字段如下：
```java
public class CryptoConfig {
    private String cryptoKeyReaderClassName;
    private Map<String, Object> cryptoKeyReaderConfig;

    private String[] encryptionKeys;
    private ProducerCryptoFailureAction producerCryptoFailureAction;

    private ConsumerCryptoFailureAction consumerCryptoFailureAction;
}
```
- producerCryptoFailureAction:定义生产者在加密数据失败时采取的操作。可用的选项有FAIL或SEND
- consumerCryptoFailureAction:定义使用者在未能解密接收到的数据时所采取的操作。可用的选项有FAIL、DISCARD或CONSUME

### 启用包管理服务
    包管理服务支持版本管理和简化升级和回滚用于函数、接收器和源的进程。在不同的名称空间中使用相同的函数、接收器和源时，可以将它们上传到公共包管理系统
启用包管理服务后，可以将功能包上传到服务并获取包URL。因此，您可以通过设置--jar、--py或--go到包URL来创建函数

缺省情况下，关闭包管理服务。在集群中启用它，需要在conf/broker.conf配置如下：
```properties
enablePackagesManagement=true
packagesManagementStorageProvider=org.apache.pulsar.packages.management.storage.bookkeeper.BookKeeperPackagesStorageProvider
packagesReplicas=1
packagesManagementLedgerRootPath=/ledgers
```
**要确保生产部署(具有多个代理的集群)中的高可用性，请将packagesReplicas设置为与broker数量相等。仅在单节点集群部署时，默认值为“1”**

### 使用内置函数
    与内置连接器类似，被打包为NAR的Java函数代码放在函数工作者的函数目录中，在启动时加载，并可以在创建函数时引用
例如，如果你有一个内置函数的名称exclamation 在Pulsar-io.yaml，你可以创建一个函数实例:
```text
bin/pulsar-admin functions create \
  --function-type exclamation \
  --inputs persistent://public/default/input-1 \
  --output persistent://public/default/output-1
```

### 触发一个函数
    触发函数意味着通过CLI向其中一个输入主题生成消息来调用函数。可以在任何时候使用trigger命令触发某个函数
使用pulsar-admin CLI，我们可以向函数发送消息，而无需使用pulsar-client工具或特定于语言的客户端库

要了解如何触发函数，可以从一个Python函数开始，该函数根据输入返回一个简单的字符串，如下所示:
```text
# myfunc.py
def process(input):
    return "This function has been triggered with a value of {0}".format(input)
```
- 在集群模式下执行此函数
```text
bin/pulsar-admin functions create \
  --tenant public \
  --namespace default \
  --name myfunc \
  --py myfunc.py \
  --classname myfunc \
  --inputs persistent://public/default/in \
  --output persistent://public/default/out
```
- 使用pulsar-client consume命令指定使用者在输出主题上侦听来自myfunc函数的消息
```text
bin/pulsar-client consume persistent://public/default/out \
  --subscription-name my-subscription \
  --num-messages 0 # Listen indefinitely
```
- 触发函数
```text
bin/pulsar-admin functions trigger \
  --tenant public \
  --namespace default \
  --name myfunc \
  --trigger-value "hello world"
```
**在trigger命令中，topic信息是不需要的。只需要指定函数的基本信息，如租户、命名空间、函数名等**
