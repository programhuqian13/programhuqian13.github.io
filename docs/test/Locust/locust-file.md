# Locust file讲解
Locust file是locust 测试的脚本文件，下面是一个实例：

```python
import time
from locust import HttpUser, task, between

class QuickstartUser(HttpUser):
    wait_time = between(1, 5)

    @task
    def hello_world(self):
        self.client.get("/hello")
        self.client.get("/world")

    @task(3)
    def view_items(self):
        for item_id in range(10):
            self.client.get(f"/item?id={item_id}", name="/item")
            time.sleep(1)

    def on_start(self):
        self.client.post("/login", json={"username":"foo", "password":"bar"})
```

```text
import time
from locust import HttpUser, task, between
```

locust 文件只是一个普通的Python模块，它可以从其他文件或包中导入代码

```text
class QuickstartUser(HttpUser):
```

这里我们为将要模拟的用户定义一个类。它继承自HttpUser，它为每个用户提供了一个客户端属性，
这是HttpSession的一个实例，可用于向我们想要加载测试的目标系统发出HTTP请求.
当测试开始时，locust将为它模拟的每个用户创建该类的一个实例，并且每个用户都将在自己的green gevent thread中开始运行

**要使文件成为有效的locustfile，它必须包含至少一个继承自User的类**

```text
wait_time = between(1, 5)
```

我们的类定义了一个等待时间，它将使模拟用户在每个任务执行后等待1到5秒

```text
@task
    def hello_world(self):
```

用 **@task** 装饰的方法是locust文件的核心。对于每个正在运行的用户，Locust创建一个greenlet(微线程)，它将调用这些方法

```text
@task
def hello_world(self):
    self.client.get("/hello")
    self.client.get("/world")

@task(3)
def view_items(self):
    for item_id in range(10):
        self.client.get(f"/item?id={item_id}", name="/item")
        time.sleep(1)
```

我们通过用 **@task** 修饰两个方法来声明两个任务，其中一个被赋予了更高的权重(3)。
当我们的QuickstartUser运行时，它将选择一个声明的任务 — 在本例中是hello_world或view_items — 并执行它。
任务是随机挑选的，但我们可以给它们不同的权重。
上述配置将使Locust选择view_items的可能性是hello_world的三倍。当任务完成执行后，User将在其等待时间(在本例中为1到5秒)内休眠。
等待一段时间后，它会选择一个新的任务，并不断重复

**注意，只有用@task装饰的方法才会被选中，因此我们可以按照自己喜欢的方式定义自己的内部助手方法**

```text
self.client.get("/hello")
```

self.client属性使得可以进行HTTP调用，这些调用将由Locust记录。有关如何发出其他类型请求的信息，验证响应 等

**HttpUser不是一个真正的浏览器，因此不会解析HTML响应来加载资源或呈现页面。但它会跟踪cookie**

```text
@task(3)
def view_items(self):
    for item_id in range(10):
        self.client.get(f"/item?id={item_id}", name="/item")
        time.sleep(1)
```

view_items 任务是加载10个不同的urls使用可变查询参数，为了避免在Locust的统计数据中获得10个单独的条目 — 因为统计数据是根据URL分组的
我们使用name参数将所有这些请求分组到 **/item** 一个条目下

```text
def on_start(self):
    self.client.post("/login", json={"username":"foo", "password":"bar"})
```

另外，我们声明了一个on_start方法。当每个模拟用户启动时，将调用具有此名称的方法

## 自动生成一个locustfile
    我们可以使用har2locust根据浏览器记录(HAR-file)生成locustfile
对于不习惯编写自己的locustfile的初学者来说，它特别有用，但对于更高级的用例来说，它也是高度可定制的

**Har2locust仍处于测试阶段。它可能并不总是生成正确的locustfile，并且它的接口可能在不同版本之间发生变化**

## User Class
User Class代表系统的一种user/scenario。进行测试运行时，我们会指定要模拟的并发用户的数量，而Locust将创建每个用户实例。
我们可以将我们喜欢的任何属性添加到这些类/实例中，但有一些对locust有特殊意义：

- wait_time属性:User的wait_time方法很容易在每个任务执行之后引入延迟。如果没有指定等待时间，则在下一个任务完成后立即执行

  - constant:在固定的时间内
  - between:对于最小值和最大值之间的随机时间
  - constant_throughput:用于确保任务每秒运行(最多)X次的自适应时间
  - constant_pacing:确保任务(最多)每X秒运行一次的自适应时间(它是恒定吞吐量的数学倒数)
  
```text
from locust import User, task, between

class MyUser(User):
    @task
    def my_task(self):
        print("executing my_task")

    wait_time = between(0.5, 10)
```
使每个用户在每个任务执行之间等待0.5到10秒

例如，如果希望Locust在峰值负载下每秒运行500次任务迭代，则可以使用wait_time=constant_throughput(0.1)和5000个用户计数
等待时间只能约束吞吐量，而不能启动新用户以达到目标。因此，在我们的示例中，如果任务迭代的时间超过10秒，吞吐量将小于500。
在执行任务后将应用等待时间，因此，如果我们的rate/ramp较高，则最终可能会超过目标。
等待时间适用于任务，而不是请求。例如，如果我们指定wait_time = content_throughput（2），并且在任务中执行两个请求，
则我们的rate/RPS为每个用户4。

也可以直接在类上声明自己的等待时间方法。例如，下面的User类将休眠一秒钟，然后是两秒钟，然后是三秒钟，等等

```text
class MyUser(User):
    last_wait_time = 0

    def wait_time(self):
        self.last_wait_time += 1
        return self.last_wait_time
```
    
- weight 和 fixed_count 属性

如果文件中存在多个用户类，并且在命令行上没有指定用户类，则Locust将生成相同数量的每个用户类。
我们还可以通过将用户类作为命令行参数传递，来指定要从同一个locustfile中使用哪些用户类

```text
locust -f locust_file.py WebUser MobileUser
```

如果希望模拟某种类型的更多用户，可以在这些类上设置权重属性。例如，网络用户的可能性是手机用户的三倍

```text
class WebUser(User):
    weight = 3
    ...

class MobileUser(User):
    weight = 1
    ...
```

还可以设置fixed_count属性。在这种情况下，weight属性将被忽略，并将生成确切的用户计数。首先生成这些用户。
在下面的示例中，将只生成一个AdminUser实例，以便独立于总用户数更准确地控制请求计数

```text
class AdminUser(User):
    wait_time = constant(600)
    fixed_count = 1

    @task
    def restart_app(self):
        ...

class WebUser(User):
    ...
```

- host属性

host属性是将要加载的主机的URL前缀。通常，这是在Locust的Web UI或命令行中指定的，当启动locust时，使用 **--host** 选项

如果一个人在用户类中声明host属性，则将在命令行或Web请求中未指定主机的情况下使用它。

- task属性

User类可以使用 **@task** 装饰器将任务声明为方法，但也可以使用tasks属性指定任务

- environment属性

对用户运行的environment的引用。使用它与环境或它所包含的runner进行交互。例如，停止执行任务方法的运行程序

```text
self.environment.runner.quit()
```

如果在独立的locust实例上运行，这将停止整个运行。如果在工作节点上运行，它将停止该特定节点

- on_start和on_stop方法

User（和TaskSet）可以声明on_start方法和/或on_stop方法。
对于用户，开始运行时将调用其on_start方法，而在停止运行时，将调用其on_stop方法。
对于任务集，当模拟用户开始执行该任务集时，将调用on_start方法，当模拟用户停止执行该任务集（当调用Interrupt（）或杀死用户）时，将调用ON_STOP。
这个和java 测试框架中的before和after的注解一样

## Task

当启动负载测试时，将为每个模拟用户创建一个User类的实例，并且它们将在自己的 green thread中开始运行。
当这些用户运行时，他们选择要执行的任务，休息一会儿，然后选择一个新任务，以此类推

这些任务是普通的python可调用程序，如果我们在对一个拍卖网站进行负载测试，它们可以做一些事情，比如加载起始页、搜索某些产品、进行竞标等等

### @task 注解
为User添加任务的最简单方法是使用task注解

```text
from locust import User, task, constant

class MyUser(User):
    wait_time = constant(1)

    @task
    def my_task(self):
        print("User instance (%r) executing my_task" % self)
```

@task接受一个可选的weight参数，该参数可用于指定任务的执行比率。在下面的例子中，task2被选中的可能性是task1的两倍

```text
from locust import User, task, between

class MyUser(User):
    wait_time = between(5, 15)

    @task(3)
    def task1(self):
        pass

    @task(6)
    def task2(self):
        pass
```

### task 属性
    定义User任务的另一种方法是设置tasks属性

tasks属性要么是一个任务列表，要么是一个<Task: int>字典，其中Task要么是一个python可调用对象，要么是一个TaskSet类。
如果任务是一个普通的python函数，它们会接收一个参数，即正在执行任务的User实例

下面是一个将User任务声明为普通python函数的示例

```text
from locust import User, constant

def my_task(user):
    pass

class MyUser(User):
    tasks = [my_task]
    wait_time = constant(1)
```

如果将tasks属性指定为列表，则每次要执行任务时，将从tasks属性中随机选择该任务。
然而，如果任务是一个字典 — 可调用对象作为键，整型作为值 - 要执行的任务将随机选择，但以整型作为比率

```text
{my_task: 3, another_task: 1}
```
my_task执行的可能性是其他任务的3倍

在内部，上面的字典实际上会扩展为如下所示的列表(tasks属性也会更新):
```text
[my_task, my_task, my_task, another_task]
```
然后使用Python的random.choice()从列表中选择任务

## @tag注解
通过使用@tag注解标记任务，我们可以使用--tags和--exclude-tags参数对测试期间执行的任务进行挑选。考虑下面的例子：

```text
from locust import User, constant, task, tag

class MyUser(User):
    wait_time = constant(1)

    @tag('tag1')
    @task
    def task1(self):
        pass

    @tag('tag1', 'tag2')
    @task
    def task2(self):
        pass

    @tag('tag3')
    @task
    def task3(self):
        pass

    @task
    def task4(self):
        pass
```
如果我们以--tags tag1开始这个测试，那么在测试期间只有task1和task2会被执行。如果以--tags tag2 tag3开始，则只执行task2和task3

--exclude-tags将以完全相反的方式运行。因此，如果我们使用--exclude-tags tag3开始测试，那么只有task1、task2和task4会被执行。
排除总是胜过包含，所以如果一个任务有一个包含的标签和一个排除的标签，它将不会被执行

## 事件
    如果我们希望将一些设置代码作为测试的一部分运行，通常将其放在locustfile的模块级别就足够了，
    但有时我们需要在运行的特定时间执行某些操作。为了满足这种需求，Locust提供了事件钩子
### test_start和test_stop
如果需要在负载测试的开始或停止时运行一些代码，则应该使用test_start和test_stop事件。我们可以在locustfile的模块级别为这些事件设置侦听器：

```text
from locust import events

@events.test_start.add_listener
def on_test_start(environment, **kwargs):
    print("A new test is starting")

@events.test_stop.add_listener
def on_test_stop(environment, **kwargs):
    print("A new test is ending")
```

### init
    init事件在每个locust进程开始时触发。这在分布式模式下特别有用，因为每个工作进程(而不是每个用户)都需要有机会进行一些初始化

例如，假设我们有一些全局状态，从该进程生成的所有用户都需要：

```text
from locust import events
from locust.runners import MasterRunner

@events.init.add_listener
def on_locust_init(environment, **kwargs):
    if isinstance(environment.runner, MasterRunner):
        print("I'm on master node")
    else:
        print("I'm on a worker or standalone node")
```

## HttpUser class
    HttpUser是最常用的User。它添加了一个用于发出HTTP请求的客户端属性

```text
from locust import HttpUser, task, between

class MyUser(HttpUser):
    wait_time = between(5, 15)

    @task(4)
    def index(self):
        self.client.get("/")

    @task(1)
    def about(self):
        self.client.get("/about/")
```

### client 属性/HttpSession
    client是httpsession的实例。Httpsession是request.Session的subclass/wrapper。
    HttpSession添加的内容主要是将请求结果报告为locust（成功/失败，响应时间，响应长度，名称）
它包含所有HTTP方法的方法:get, post, put...
就像request.Session一样，它保留请求之间的cookie，因此它可以很容易地用于登录网站

发出POST请求，查看响应并隐式地重用我们为第二个请求获得的任何会话cookie

```text
response = self.client.post("/login", {"username":"testuser", "password":"secret"})
print("Response status code:", response.status_code)
print("Response text:", response.text)
response = self.client.get("/my-profile")
```

HttpSession捕获任何request.Session抛出的RequestException(由连接错误、超时或类似原因引起)，而是返回一个虚拟的Response对象，状态码设置为0，内容设置为None

### 验证responses
    如果HTTP响应代码为OK(<400)，则认为请求成功，但是对响应进行一些额外的验证通常是有用的
可以通过使用catch_response参数、with-statement和调用response.failure()将请求标记为失败

```text
with self.client.get("/", catch_response=True) as response:
    if response.text != "Success":
        response.failure("Got wrong response")
    elif response.elapsed.total_seconds() > 0.5:
        response.failure("Request took too long")
```

我们还可以将请求标记为成功，即使响应代码是错误的：

```text
with self.client.get("/does_not_exist/", catch_response=True) as response:
    if response.status_code == 404:
        response.success()
```

我们甚至可以通过抛出一个异常，然后在with-block之外捕获它，从而完全避免记录请求。
或者我们可以抛出一个locust异常，就像下面的例子一样，让locust捕获它

```text
from locust.exception import RescheduleTask
...
with self.client.get("/does_not_exist/", catch_response=True) as response:
    if response.status_code == 404:
        raise RescheduleTask()
```

### REST/JSON APIs
FastHttpUser提供了一个现成的rest方法，但是我们也可以自己做：
```text
from json import JSONDecodeError
...
with self.client.post("/", json={"foo": 42, "bar": None}, catch_response=True) as response:
    try:
        if response.json()["greeting"] != "hello":
            response.failure("Did not get expected value in greeting")
    except JSONDecodeError:
        response.failure("Response could not be decoded as JSON")
    except KeyError:
        response.failure("Response did not contain expected key 'greeting'")
```

### 分组请求
    对于网站来说，其url包含某种动态参数的页面是很常见的。
    通常，在用户统计信息中将这些url分组在一起是有意义的。这可以通过向HttpSession的不同请求方法传递一个name参数来实现
```text
# Statistics for these requests will be grouped under: /blog/?id=[id]
for i in range(10):
    self.client.get("/blog?id=%i" % i, name="/blog?id=[id]")
```

在某些情况下，不可能将参数传递到请求函数中，
例如，与libraries/SDK交互时，将请求会话包含。通过设置client.request_name属性，可以提供另一种分组请求的方式

```text
# Statistics for these requests will be grouped under: /blog/?id=[id]
self.client.request_name="/blog?id=[id]"
for i in range(10):
    self.client.get("/blog?id=%i" % i)
self.client.request_name=None
```

如果希望用最少的样板文件链接多个组，可以使用client.rename_request()上下文管理器
```text
@task
def multiple_groupings_example(self):
    # Statistics for these requests will be grouped under: /blog/?id=[id]
    with self.client.rename_request("/blog?id=[id]"):
        for i in range(10):
            self.client.get("/blog?id=%i" % i)

    # Statistics for these requests will be grouped under: /article/?id=[id]
    with self.client.rename_request("/article?id=[id]"):
        for i in range(10):
            self.client.get("/article?id=%i" % i)
```

使用catch_response和直接访问request_meta，我们甚至可以根据响应中的某些内容重命名请求

```text
with self.client.get("/", catch_response=True) as resp:
    resp.request_meta["name"] = resp.json()["name"]
```

## Http Proxy 设置
为了提高性能，我们将请求配置为不通过设置请求来查找环境中的HTTP代理设置。
Session  trust_env 属性为False。如果我们不想这样做，可以手动将locust_instance.client.trust_env设置为True

## 连接池
由于每个HttpUser都创建新的HttpSession，因此每个用户实例都有自己的连接池。这类似于真实用户与web服务器的交互方式
但是，如果希望在所有用户之间共享连接，则可以使用单个池管理器。为此，将pool_manager类属性设置为urllib3.PoolManager的实例

```text
from locust import HttpUser
from urllib3 import PoolManager

class MyUser(HttpUser):
    # All users will be limited to 10 concurrent connections at most.
    pool_manager = PoolManager(maxsize=10, block=True)
```

## 如何构建测试代码
重要的是要记住，locustfile.py只是一个由Locust导入的普通Python模块。从这个模块中，你可以自由地导入其他python代码，就像你在任何python程序中通常做的那样.
当前工作目录会自动添加到python的sys.path中,因此，可以使用Python导入语句导入任何位于工作目录中的file/module/packages

对于小型测试，将所有测试代码保存在单个locustfile.py中应该可以很好地工作，但是对于较大的测试套件，我们可能希望将代码拆分到多个文件和目录中

当然，如何构建测试源代码完全取决于您，但建议遵循Python最佳实践。下面是一个假想的locust项目的示例文件结构
- Project root
    - common/
      - __init__.py
      - auth.py
      - config.py
    - locustfile.py
    - requirements.txt（外部Python依赖项通常保存在requirements.txt中）

具有多个locustfile的项目也可以将它们保存在单独的子目录中：
- Project root
    - common/
        - __init__.py
        - auth.py
        - config.py
    - my_locustfiles/
      - api.py
      - website.py
    - requirements.txt（外部Python依赖项通常保存在requirements.txt中）

使用上述任何项目结构，我们的locustfile都可以使用

```text
import common.auth
```
















