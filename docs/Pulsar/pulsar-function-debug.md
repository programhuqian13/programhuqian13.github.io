# 调试Pulsar Functions

## 使用捕获的 stderr 进行调试
    要调试函数启动失败的原因，可以查看logs/functions/<tenant>/<namespace>/<function>/<function>-<instance>.log文件

## 使用单元测试进行调试
    与任何具有输入和输出的函数一样，我们可以以与测试任何其他函数类似的方式测试Pulsar function
    Pulsar使用TestNG进行测试

例如，如果您通过Java的语言本地接口编写了以下函数：

```java
import java.util.function.Function;

public class JavaNativeExclamationFunction implements Function<String, String> {
   @Override
   public String apply(String input) {
       return String.format("%s!", input);
   }
}
```

我们可以编写一个简单的单元测试来测试该函数：
```java
@Test
public void testJavaNativeExclamationFunction() {
   JavaNativeExclamationFunction exclamation = new JavaNativeExclamationFunction();
   String output = exclamation.apply("foo");
   Assert.assertEquals(output, "foo!");
}
```

下面的示例是通过Java SDK编写的:
```java
import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;

public class ExclamationFunction implements Function<String, String> {
   @Override
   public String process(String input, Context context) {
       return String.format("%s!", input);
   }
}
```

我们可以编写一个单元测试来测试这个函数并模拟Context参数，如下所示:

```java
@Test
public void testExclamationFunction() {
   ExclamationFunction exclamation = new ExclamationFunction();
   String output = exclamation.process("foo", mock(Context.class));
   Assert.assertEquals(output, "foo!");
}
```

## localrun模式进行调试
    在localrun模式下，函数消耗并生成实际数据到Pulsar集群，并反映该函数在Pulsar集群中的运行方式。这提供了一种测试函数的方法，并允许我们在本地机器上作为线程启动函数实例，以便于调试
    使用localrun模式进行调试仅适用于Pulsar 2.4.0或更高版本中的Java函数

在使用localrun模式之前，我们需要添加以下依赖项：
```XML
<dependency>
    <groupId>org.apache.pulsar</groupId>
    <artifactId>pulsar-functions-local-runner-original</artifactId>
    <version>${pulsar.version}</version>
</dependency>

<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>3.21.9</version>
</dependency>
```

例如，我们可以以以下方式运行函数：

```java
FunctionConfig functionConfig = new FunctionConfig();
functionConfig.setName(functionName);
functionConfig.setInputs(Collections.singleton(sourceTopic));
functionConfig.setClassName(ExclamationFunction.class.getName());
functionConfig.setRuntime(FunctionConfig.Runtime.JAVA);
functionConfig.setOutput(sinkTopic);

LocalRunner localRunner = LocalRunner.builder().functionConfig(functionConfig).build();
localRunner.start(true);
```

我们可以使用IDE调试函数。设置断点并手动步进函数以使用实际数据进行调试：
下面的代码示例展示了如何在localrun模式下运行函数
```java
public class ExclamationFunction implements Function<String, String> {

    @Override
    public String process(String s, Context context) throws Exception {
        return s + "!";
    }

    public static void main(String[] args) throws Exception {
        FunctionConfig functionConfig = new FunctionConfig();
        functionConfig.setName("exclamation");
        functionConfig.setInputs(Collections.singleton("input"));
        functionConfig.setClassName(ExclamationFunction.class.getName());
        functionConfig.setRuntime(FunctionConfig.Runtime.JAVA);
        functionConfig.setOutput("output");

        LocalRunner localRunner = LocalRunner.builder().functionConfig(functionConfig).build();
        localRunner.start(false);
    }
}
```

## 使用日志主题进行调试
    使用Pulsar Functions时，可以将函数中预定义的日志生成到指定的日志主题，并配置消费者消费该日志主题的消息

例如，下面的函数根据输入的字符串是否包含单词danger记录warning级别或info级别的日志:
```java
import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;
import org.slf4j.Logger;

public class LoggingFunction implements Function<String, Void> {
    @Override
    public void apply(String input, Context context) {
        Logger LOG = context.getLogger();
        String messageId = new String(context.getMessageId());

        if (input.contains("danger")) {
            LOG.warn("A warning was received in message {}", messageId);
        } else {
            LOG.info("Message {} received\nContent: {}", messageId, input);
        }

        return null;
    }
}
```
如示例中所示，我们可以通过context.getLogger()获取记录器，并将记录器分配给slf4j的LOG变量，这样我们就可以使用LOG变量在函数中定义所需的日志

同时，我们需要指定可以生成日志的主题。示例如下：
```text
bin/pulsar-admin functions create \
  --log-topic persistent://public/default/logging-function-logs \
  # Other function configs
```
发布到日志主题的消息包含以下几个属性:
- loglevel:日志消息的级别
- Fqn:推送此日志消息的完全限定函数名
- instance:推送日志的功能实例ID

## 使用Functions CLI进行调试
    使用Pulsar函数CLI，我们可以使用以下子命令调试Pulsar函数
    get
    status
    list
    trigger

- get：要获取关于函数的信息，可以像下面这样指定--fqfn
    
```text
 ./bin/pulsar-admin functions get public/default/ExclamationFunctio6
```

或者，我们可以指定--name、--namespace和--tenant，如下所示：

```text
./bin/pulsar-admin functions get \
    --tenant public \
    --namespace default \
    --name ExclamationFunctio6
```

如下所示，get命令显示了关于ExclamationFunctio6函数的输入、输出、运行时和其他信息。

```json
{
  "tenant": "public",
  "namespace": "default",
  "name": "ExclamationFunctio6",
  "className": "org.example.test.ExclamationFunction",
  "inputSpecs": {
    "persistent://public/default/my-topic-1": {
      "isRegexPattern": false
    }
  },
  "output": "persistent://public/default/test-1",
  "processingGuarantees": "ATLEAST_ONCE",
  "retainOrdering": false,
  "userConfig": {},
  "runtime": "JAVA",
  "autoAck": true,
  "parallelism": 1
}
```

- list:列出在特定租户和命名空间下运行的所有Pulsar 函数

```text
bin/pulsar-admin functions list \
    --tenant public \
    --namespace default
```

- status:查询函数的状态

```text
./bin/pulsar-admin functions status \
    --tenant public \
    --namespace default \
    --name ExclamationFunctio6
```

- stats:获取函数的当前状态

```text
bin/pulsar-admin functions stats \
    --tenant public \
    --namespace default \
    --name ExclamationFunctio6
```

- trigger:用提供的值触发指定函数

```text
./bin/pulsar-admin functions trigger \
    --tenant public \
    --namespace default \
    --name ExclamationFunctio6 \
    --topic persistent://public/default/my-topic-1 \
    --trigger-value "hello pulsar functions"
```

# 打包Java Functions
    打包Java函数有两种方法，即uber JAR和NAR
    如果我们计划打包和分发我们的函数以供其他人使用，那么我们有义务对自己的代码进行适当的许可和版权保护。
    请记住将许可证和版权添加到代码使用的所有库和我们的发行版中。
    如果使用NAR方法，NAR插件会自动在生成的NAR包中创建一个DEPENDENCIES文件，其中包括函数的所有库的适当许可和版权
    
## 以jar包的方式打包
### 要将Java函数打包为JAR，请完成以下步骤：
- 用pom文件创建一个新的maven项目。在下面的代码示例中，mainClass的值是我们的包名
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>java-function</groupId>
    <artifactId>java-function</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.apache.pulsar</groupId>
            <artifactId>pulsar-functions-api</artifactId>
            <version>2.11.1</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <configuration>
                    <appendAssemblyId>false</appendAssemblyId>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                    <archive>
                        <manifest>
                            <mainClass>org.example.test.ExclamationFunction</mainClass>
                        </manifest>
                    </archive>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>assembly</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <configuration>
                    <release>17</release>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

- 打包java Function

```text
 mvn package
```
Java函数打包完成后，会自动创建target 目录。打开target目录，检查是否有类似于java-function-1.0-SNAPSHOT.jar的JAR包

- 将打包的jar文件复制到Pulsar映像中
```shell
 docker exec -it [CONTAINER ID] /bin/bash
 docker cp <path of java-function-1.0-SNAPSHOT.jar>  CONTAINER ID:/pulsar
```

- 使用以下命令运行Java函数

```shell
./bin/pulsar-admin functions localrun \
    --classname org.example.test.ExclamationFunction \
    --jar java-function-1.0-SNAPSHOT.jar \
    --inputs persistent://public/default/my-topic-1 \
    --output persistent://public/default/test-1 \
    --tenant public \
    --namespace default \
    --name JavaFunction
```

### 要将Java函数打包为NAR，请完成以下步骤：
- 用pom文件创建一个新的maven项目
```XML
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>java-function</groupId>
    <artifactId>java-function</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.apache.pulsar</groupId>
            <artifactId>pulsar-functions-api</artifactId>
            <version>2.11.1</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.nifi</groupId>
                <artifactId>nifi-nar-maven-plugin</artifactId>
                <version>1.2.0</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <configuration>
                    <release>17</release>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

我们也可以创建pulsar-io.yaml文件在resources/META-INF/services，在下面的代码示例中，functionClass的值是函数类名。该名称是函数作为内置函数部署时使用的名称

```yaml
name: java-function
description: my java function
functionClass: org.example.test.ExclamationFunction
```

- 打包java Function

```text
 mvn package
```
Java函数打包完成后，会自动创建target 目录。打开target目录，检查是否有类似于java-function-1.0-SNAPSHOT.nar的NAR包

- 将打包的nar文件复制到Pulsar映像中

```shell
 docker cp <path of java-function-1.0-SNAPSHOT.nar>  CONTAINER ID:/pulsar
```

- 使用以下命令运行Java函数

```shell
./bin/pulsar-admin functions localrun \
    --jar java-function-1.0-SNAPSHOT.nar \
    --inputs persistent://public/default/my-topic-1 \
    --output persistent://public/default/test-1 \
    --tenant public \
    --namespace default \
    --name JavaFunction
```
