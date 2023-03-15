## 1.预备环境准备

Nacos 依赖 [Java](https://docs.oracle.com/cd/E19182-01/820-7851/inst_cli_jdk_javahome_t/) 环境来运行。如果您是从代码开始构建并运行Nacos，还需要为此配置 MAVEN环境，请确保是在以下版本环境中安装使用:

1. 64 bit OS，支持 Linux/Unix/Mac/Windows，推荐选用 Linux/Unix/Mac。
2. 64 bit JDK 1.8+；
3. Maven 3.2.x+；

## 2.下载源码或者安装包

你可以通过源码和发行包两种方式来获取 Nacos。

### 从 Github 上下载源码方式

```bash
git clone https://github.com/alibaba/nacos.git
cd nacos/
mvn -Prelease-nacos -Dmaven.test.skip=true clean install -U  
ls -al distribution/target/

// change the $version to your actual path
cd distribution/target/nacos-server-$version/nacos/bin
```

### 下载编译后压缩包方式

您可以从 [最新稳定版本](https://github.com/alibaba/nacos/releases) 下载 `nacos-server-$version.zip` 包。

```bash
  unzip nacos-server-$version.zip 或者 tar -xvf nacos-server-$version.tar.gz
  cd nacos/bin
```

## 3.启动服务器

- 注：Nacos的运行需要以至少2C4g60g*3的机器配置下运行。

  ### Linux/Unix/Mac

  启动命令(standalone代表着单机模式运行，非集群模式):

  ```
  sh startup.sh -m standalone
  ```

  如果您使用的是ubuntu系统，或者运行脚本报错提示[[符号找不到，可尝试如下运行：

  ```
  bash startup.sh -m standalone
  ```

  ### Windows

  启动命令(standalone代表着单机模式运行，非集群模式):

  ```
  startup.cmd -m standalone
  ```



# 4. 遇到的问题

- 下载压缩包很慢，因为是在github上面，下载经常下载不下来，因此使用源码部署

- 通过源码编译的时候，我使用的JDK 11进行编译，之前使用的maven 3.6.3的版本，会出现如下问题：

  编译的项目的时候出现 **Unsupport target jdk 11/8 **问题，我这是了JDK的版本，项目编译的版本也设置了11依旧会出现这个问题，最终换了maven的版本为3.8.2的版本最终可以编译成功。

