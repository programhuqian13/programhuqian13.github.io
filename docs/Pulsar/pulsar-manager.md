# Pulsar Manager
    Pulsar Manager是一个基于web的GUI管理和监视工具，可帮助管理员和用户管理和监视租户、名称空间、主题、订阅、代理、集群等，并支持对多个环境进行动态配置
## 安装
### 快速安装
使用Pulsar 管理器最简单的方法是在Docker容器中运行：

```shell
docker pull apachepulsar/pulsar-manager:v0.3.0
docker run -it \
  -p 9527:9527 -p 7750:7750 \
  -e SPRING_CONFIGURATION_FILE=/pulsar-manager/pulsar-manager/application.properties \
  apachepulsar/pulsar-manager:v0.3.0
```

- Pulsar管理器分为前端和后端，前端业务端口为9527，后端业务端口为7750
- SPRING配置文件:SPRING的默认配置文件
- Pulsar管理器默认使用herddb数据库。HerdDB是一个用Java实现的SQL分布式数据库

## 配置数据库或JWT身份验证
### 配置数据库（可选）
如果我们有大量的数据，您可以使用自定义数据库。否则，可能会出现一些显示错误。例如，当主题超过10000时，将无法显示主题信息。

下面是PostgreSQL的一个示例:

- 使用该文件初始化数据库和表结构
- 下载并修改配置文件，然后添加PostgreSQL配置

```properties
spring.datasource.driver-class-name=org.postgresql.Driver
spring.datasource.url=jdbc:postgresql://127.0.0.1:5432/pulsar_manager
spring.datasource.username=postgres
spring.datasource.password=postgres
```

- 添加一个配置挂载，并从一个docker映像开始 

```shell
docker pull apachepulsar/pulsar-manager:v0.3.0
docker run -it \
    -p 9527:9527 -p 7750:7750 \
    -v /your-path/application.properties:/pulsar-manager/pulsar-manager/application.properties
    -e SPRING_CONFIGURATION_FILE=/pulsar-manager/pulsar-manager/application.properties \
    apachepulsar/pulsar-manager:v0.3.0
```

### 启用JWT身份验证(可选)

如果要启用JWT身份验证，请配置application.properties。属性文件:

```properties
backend.jwt.token=token

jwt.broker.token.mode=PRIVATE
jwt.broker.public.key=file:///path/broker-public.key
jwt.broker.private.key=file:///path/broker-private.key

or
jwt.broker.token.mode=SECRET
jwt.broker.secret.key=file:///path/broker-secret.key
```

- backend.jwt.token:超级用户的令牌。该参数需要在集群初始化时配置
- jwt.broker.token.mode:生成令牌的多种模式，包括PUBLIC、PRIVATE和SECRET
- jwt.broker.public.key:如果我们使用PUBLIC模式，请配置此选项
- jwt.broker.private.key:如果我们使用PRIVATE模式，请配置此选项
- jwt.broker.secret.key:如果您使用SECRET模式，请配置此选项

Docker命令添加profile和key文件mount文件挂载：

```shell
docker pull apachepulsar/pulsar-manager:v0.3.0
docker run -it \
  -p 9527:9527 -p 7750:7750 \
  -v /your-path/application.properties:/pulsar-manager/pulsar-manager/application.properties
  -v /your-path/private.key:/pulsar-manager/private.key
  -e SPRING_CONFIGURATION_FILE=/pulsar-manager/pulsar-manager/application.properties \
  apachepulsar/pulsar-manager:v0.3.0
```

## 设置管理员帐号和密码

```shell
CSRF_TOKEN=$(curl http://localhost:7750/pulsar-manager/csrf-token)
curl \
   -H 'X-XSRF-TOKEN: $CSRF_TOKEN' \
   -H 'Cookie: XSRF-TOKEN=$CSRF_TOKEN;' \
   -H "Content-Type: application/json" \
   -X PUT http://localhost:7750/pulsar-manager/users/superuser \
   -d '{"name": "admin", "password": "apachepulsar", "description": "test", "email": "username@test.org"}'
```

curl命令中的request参数:

```json
{"name": "admin", "password": "apachepulsar", "description": "test", "email": "username@test.org"}
```

- name:name是pulsar管理器的登录用户名，当前为admin
- password:password为pulsar管理器当前用户的密码，当前为apachepulsar。密码必须大于等于6位数字

## 配置环境
1. 登录到系统，请访问http://localhost:9527登录。当前默认帐户是admin/apachepulsar
2. 点击“新建环境”按钮添加环境
3. 输入“环境名称”,环境名称用于标识环境
4. 输入“服务URL”,服务URL是pulsar集群的管理服务URL

## 裸机安装
当使用二进制包进行直接部署时，我们可以遵循以下步骤：

- 下载并解压缩二进制包，该包可在pulsar下载页面上获得

```shell
wget https://dist.apache.org/repos/dist/release/pulsar/pulsar-manager/pulsar-manager-0.3.0/apache-pulsar-manager-0.3.0-bin.tar.gz
tar -zxvf apache-pulsar-manager-0.3.0-bin.tar.gz
```

- 提取后端服务二进制包，并将前端资源放在后端服务目录中

```shell
cd pulsar-manager
tar -xvf pulsar-manager.tar
cd pulsar-manager
cp -r ../dist ui
```

- 修改application.properties属性:

如果我们不想修改application.properties文件，则可以通过启动参数添加配置./bin/pulsar-manager --backend.jwt.token=token，将配置添加到启动参数中

- 启动

```shell
./pulsar-manager
```

## 自定义docker镜像安装

我们可以在Docker Hub目录中找到docker镜像，并从源代码构建镜像

```shell
git clone https://github.com/apache/pulsar-manager
cd pulsar-manager/front-end
npm install --save
npm run build:prod
cd ..
./gradlew build -x test
cd ..
docker build -f docker/Dockerfile --build-arg BUILD_DATE=`date -u +"%Y-%m-%dT%H:%M:%SZ"` --build-arg VCS_REF=`latest` --build-arg VERSION=`latest` -t apachepulsar/pulsar-manager .
```

