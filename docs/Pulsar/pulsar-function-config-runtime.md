# Pulsar 配置Function runtime
## 配置thread  runtime
我们可以在conf/functions_worker.yml文件中使用线程运行时的默认配置

如果需要自定义更多参数，如线程组名称，请参考以下示例：
```yaml
functionRuntimeFactoryClassName: org.apache.pulsar.functions.runtime.thread.ThreadRuntimeFactory
functionRuntimeFactoryConfigs:
  threadGroupName: "Your Function Container Group"
```
要设置线程运行时的客户机内存限制，可以配置pulsarClientMemoryLimit:
```yaml
functionRuntimeFactoryConfigs:
#  pulsarClientMemoryLimit
# # the max memory in bytes the pulsar client can use
#   absoluteValue:
# # the max memory the pulsar client can use as a percentage of max direct memory set for JVM
#   percentOfMaxDirectMemory:
```
如果同时设置了absoluteValue和percentOfMaxDirectMemory，则使用较小的值.

## 配置process runtime
我们可以在配置文件conf/functions_worker.yml文件中使用流程运行时的默认配置

如果需要自定义更多参数，请参考以下示例：
```yaml
functionRuntimeFactoryClassName: org.apache.pulsar.functions.runtime.process.ProcessRuntimeFactory
functionRuntimeFactoryConfigs:
  # the directory for storing the function logs
  logDirectory:
  # change the jar location only when you put the java instance jar in a different location
  javaInstanceJarLocation:
  # change the python instance location only when you put the python instance jar in a different location
  pythonInstanceLocation:
  # change the extra dependencies location:
  extraFunctionDependenciesDir:
```
## 配置Kubernetes runtime
当函数工作者生成并应用Kubernetes清单时，Kubernetes运行时工作。由函数工作者生成的清单包括：
1. 默认情况下，StatefulSet清单有一个带有多个副本的pod。这个数由函数的并行度决定。pod在pod启动时下载函数有效负载(通过函数工作者REST API)。如果配置了函数运行时，pod的容器映像是可配置的。
2. a Service(用于与pod通信)
3. 用于验证凭证的Secret(当适用时)。Kubernetes运行时支持秘钥。我们可以创建一个Kubernetes秘钥，并将其作为pod中的环境变量公开

## 配置基本设置
为了快速配置Kubernetes运行时，可以在conf/functions_worker.yml文件中使用KubernetesRuntimeFactoryConfig的默认设置

如果你已经使用Helm chart在Kubernetes上建立了一个Pulsar集群，这意味着函数工作者也已经在Kubernetes上建立了，你可以使用与函数工作者运行的pod关联的serviceAccount。
否则，我们可以通过将functionRuntimeFactoryConfigs设置为k8Uri来配置函数工作者与Kubernetes集群通信

### 集成 Kubernetes 密钥
Kubernetes中的Secret是一个保存一些机密数据(如密码、令牌或密钥)的对象。当我们在部署函数的Kubernetes名称空间中创建一个Secret时，函数可以安全地引用和分发它。
要启用该特性，请将配置文件conf/functions-worker.yml的secretsProviderConfiguratorClassName设置为org.apache.pulsar.functions.secretsproviderconfigurator.KubernetesSecretsProviderConfigurator

例如，我们将一个函数部署到pulsar-func Kubernetes名称空间，并且有一个名为database-creds的密钥名称和一个字段名password，我们希望将其作为一个名为DATABASE_PASSWORD的环境变量挂载到pod中。下面的配置允许函数引用密钥并将该值作为pod中的环境变量挂载
```yaml
tenant: "mytenant"
namespace: "mynamespace"
name: "myfunction"
inputs: [ "persistent://mytenant/mynamespace/myfuncinput" ]
className: "com.company.pulsar.myfunction"

secrets:
  # the secret will be mounted from the `password` field in the `database-creds` secret as an env var called `DATABASE_PASSWORD`
  DATABASE_PASSWORD:
    path: "database-creds"
    key: "password"
```

## 启用token认证
当我们使用令牌身份验证、TLS加密或自定义身份验证来保护与Pulsar集群的通信时，Pulsar将我们的证书颁发机构(CA)传递给客户端，以便客户端可以使用我们的签名证书对集群进行身份验证

要为Pulsar集群启用身份验证，需要通过实现org.apache.pulsar.functions.auth.KubernetesFunctionAuthProvider接口，为运行函数的pod指定一种机制来对代理进行身份验证

对于令牌身份验证，Pulsar包含了上述接口的实现，用于分发CA。函数工作者捕获部署(或更新)函数的令牌，将其保存为密钥，并将其装入pod

在配置文件conf/function-worker.yml中配置functionAuthProviderClassName：
```yaml
functionAuthProviderClassName: org.apache.pulsar.functions.auth.KubernetesSecretsTokenAuthProvider
```
对于TLS或自定义身份验证，我们可以实现org.apache.pulsar.functions.auth.KubernetesFunctionAuthProvider接口或使用替代机制

**如果用于部署函数的令牌有到期日期，则可能需要在到期后重新部署函数**

## 自定义 Kubernetes 运行时
自定义Kubernetes运行时允许我们自定义由运行时创建的Kubernetes资源，包括如何生成清单，如何将经过身份验证的数据传递到pods，以及如何集成密钥secrets

需要在配置文件conf/functions-worker.yml配置runtimeCustomizerClassName，使用全限定类名

函数API提供了一个名为customRuntimeOptions的标志，该标志被传递给org.apache.pulsar.functions.runtime.KubernetesManifestCustomizer 接口。

去实例化KubernetesManifestCustomizer，可以在配置文件conf/functions-worker.yml中设置runtimeCustomizerConfig 

**runtimeCustomizerConfig在所有函数中都是相同的。如果同时提供runtimeCustomizerConfig和customRuntimeOptions，则需要决定如何在KubernetesManifestCustomizer接口的实现中管理这两个配置**

Pulsar包含一个用runtimeCustomizerConfig初始化的内置实现。它允许我们将JSON文档作为customRuntimeOptions传递，并添加某些属性。
要使用这个内置实现，将runtimeCustomizerClassName设置为org.apache.pulsar.functions.runtime.kubernetes.BasicKubernetesManifestCustomizer

如果同时提供了runtimeCustomizerConfig和customRuntimeOptions并且存在冲突，BasicKubernetesManifestCustomizer使用customRuntimeOptions覆盖runtimeCustomizerConfig

customRuntimeOptions配置实例：
```json
{
  "jobName": "jobname", // the k8s pod name to run this function instance
  "jobNamespace": "namespace", // the k8s namespace to run this function in
  "extractLabels": {           // extra labels to attach to the statefulSet, service, and pods
    "extraLabel": "value"
  },
  "extraAnnotations": {        // extra annotations to attach to the statefulSet, service, and pods
    "extraAnnotation": "value"
  },
  "nodeSelectorLabels": {      // node selector labels to add on to the pod spec
    "customLabel": "value"
  },
  "tolerations": [             // tolerations to add to the pod spec
    {
      "key": "custom-key",
      "value": "value",
      "effect": "NoSchedule"
    }
  ],
  "resourceRequirements": {  // values for cpu and memory should be defined as described here: https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container
    "requests": {
      "cpu": 1,
      "memory": "4G"
    },
    "limits": {
      "cpu": 2,
      "memory": "8G"
    }
  }
}
```

## 如何定义pulsar资源名称时运行pulsar在Kubernetes
如果我们在Kubernetes上运行Pulsar Functions或连接器，则需要遵循Kubernetes命名约定来定义Pulsar资源的名称，无论我们使用哪个管理界面

Kubernetes需要一个RFC 1123中定义的可以用作DNS子域名的名称。Pulsar支持比Kubernetes命名惯例更多的合法字符
如果我们使用Kubernetes不支持的特殊字符创建Pulsar资源名称(例如，在Pulsar名称空间名称中包含冒号)，Kubernetes运行时将Pulsar对象名称转换为符合RFC 1123格式的Kubernetes资源标签。
因此，我们可以使用Kubernetes运行时运行函数或连接器。Pulsar对象名称转换为Kubernetes资源标签的规则如下：
1. 截短至63个字符
2. 将以下字符替换为破折号(-)：
   1. 非字母数字字符
   2. 下划线
   3. .
3. 将开头和结尾的非字母数字字符替换为0

## 自定义 Java 运行时选项
**此设置仅适用于进程运行时和Kubernetes运行时**

为函数工作者启动的每个进程向JVM命令行传递附加参数,我们可以配置additionalJavaRuntimeArguments 参数在conf/functions_worker.yml配置文件中
1. 添加 JVM 参数, 如： -XX:+ExitOnOutOfMemoryError
2. 传递自定义系统属性，如：-Dlog4j2.formatMsgNoLookups
```yaml
additionalJavaRuntimeArguments: ['-XX:+ExitOnOutOfMemoryError','-Dfoo=bar']
```

