# Oslo Message 入门


# Oslo Message 入门

## 什么是AMQP?
AMQP(Advanced Message Queuing Protocol)是一个异步消息传递所使用的开方的应用层协议规范，主要包括了消息的导向，队列，路由，可靠性和安全性。通过定义消息在网络上传输的字节流格式，不同的具体AMQP实现之间可以进行互操作。  

> The Advanced Message Queuing Protocol (AMQP) is an open standard application layer
> protocol for message-oriented middleware. The defining features of AMQP are
> message orientation, queuing, routing (including point-to-point and publish-and-subscribe), reliability and security

![AMQP structure](/images/amqp_structure.gif)   

他大体的结构如上所示,包括几个重要的元素:   

1. Publisher，即我们的消息发送方。
2. Broker/Server，服务中间件(转发消息，确定映射规则，存储消息等)。
3. Subscriber，消息的接收和订阅方。
4. Exchange，负责将不同的消息送达到不同Subscriber的Queue上，同一个Exchange可以有多个Queue。
5. Queue，接收方的消息队列，用来保存来不及处理的信息。 
6. Routing Key，消息携带的路由信息，决定了消息可以送到哪些接收方。
7. Binding Key，Queue的路由信息，决定了Queue可以接收哪些信息。
8. Exchange Type,交换类型，决定了消息转发的具体匹配模式，有三种模式:
    1. Direct:单一的匹配模式，类似于通过id直接指定接收方。
    2. Topic:正则的匹配模式，符合正则匹配的Queue都能收到该消息。
    3. Fanout:广播模式，所有的都能收到。



AMQP只是一个通用的协议，在此协议上有不同实现的服务中间件，比较常见的有以下几种: 

1. RabbitMQ([主页](https://www.rabbitmq.com/))  
2. Qpid([主页](https://qpid.apache.org/))  
3. ZeroMQ([主页](http://zeromq.org/))  
4. Kombu([主页](https://pypi.python.org/pypi/kombu))

我们这里使用**RabbitMQ**作为我们的中间件实现。 

## RabbitMQ 

### 环境准备  
[官网](https://www.rabbitmq.com)上面可以很方面找到下载的操作指导，这里以我们的环境为例，下载完成后，需要把压缩包解压到我们的安装目录,运行前可以在配置文件中进行简单的配置调整:

```
$RABBITMQ_HOME/etc/rabbitmq/rabbitmq-env.conf
```

执行脚本*sbin/rabbitmq-server*，服务即可拉起:  

```shell  
#以后台进程的方式拉起
rabbitmq-server -detached
#暂停服务
rabbitmqctl stop
#查看服务状态
rabbitmqctl status
```

服务拉起后，就可以配置和使用message了。

## OSLO.Message

### 基本概念  

开始前，我们也需要了解message里面几个重要的概念，他们跟AMQP里面的概念是一一对应的:

#### 1.Transport  

传输层，可以通过URI来获得不同的transport实现句柄，我们可以把rabbitmq/qpid这些理解成不同的传输层(transports)，URI的具体格式如下:   

```
transport://user:password@host1:port1[,host2:port2][,hostN:portN]/virtual_host
```

比如我们现在是使用rabbitmq，那URI就大致如下:  

```
rabbit://admin:password@127.0.0.1:8888/
```

#### 2.Target
封装了描述最终目的地的所有信息，有以下几个字段: 

1. exchange,指明能处理的topic范围，不指定则默认使用配置文件中的control_exchange配置  。
2. topic,服务端和消息都会使用，用来表明发送和可以接受的主题(组)，例如:*topic.subtopic*。
3. namespace,服务可以在一个topic上，提供多种方法集合， 这些方法集合通过namespace来分开管理，可以理解成topic的一个子集。
4. server,客户端可以通过该字段指定具体的某一台服务器，而不是符合这个topic的任意一台。
5. fanout,指明时，会发送到这个topic下面所有的服务端。

#### 3.Server

Server，为各个Client提供RPC接口,它是消息的最终处理者，单个Server上面可以绑定多个EndPoints。

#### 4.RPC Client
远程调用的客户端，调用是需要指定具体的方法和参数，现在支持两种: 

1. cast:异步调用，调用后马上返回
2. call:同步调用，调用后需要等待结果返回

#### 5.Notifier Listener
Notifier Listener与Server一致，不同的地方在于下面挂载的Endpoints暴露的接口名需要与消息不同的优先级保持一致，例如:  

```python

class NotificationEndpoint(object):

    def warn(self,ctxt,publisher_id,event_type,payload,metadata):
        print('in class warning')

    # other methods
```

### 测试样例

#### Client与Server


**Server代码**(假设rabbitmq端口是:5672),我们创建了两个Endpoints,他们都绑定到了topic**test**下面，有着不同的namespace，我们在客户端可以通过不同的namespace指定具体的endpoint:

```python
import sys
import logging

from oslo_config import cfg
import  oslo_messaging as messaging


class TestEndpoint(object):

    target = messaging.Target(namespace='namespace1',
                              version='1.0')

    def test(self,ctx,arg):
        print('this is in text endpoint')
        return arg


class AnotherTestEndpoint(object):

    target = messaging.Target(namespace='namespace2',
                              version='1.0')

    def test(self,ctx,arg):
        print('this is in another text endpoint')
        return arg


transport = messaging.get_transport(cfg.CONF,url='rabbit://127.0.0.1:5672/')
target = messaging.Target(topic='test',server='server1')
endpoints = [TestEndpoint(),AnotherTestEndpoint()]

server = messaging.get_rpc_server(transport,target,endpoints,executor='blocking')
server.start()
server.wait()
```


client代码:

```python
from oslo_config import cfg
import oslo_messaging as messaging


transport = messaging.get_transport(cfg.CONF,url='rabbit://127.0.0.1:5672/')
target = messaging.Target(topic='test',server='server1',namespace='namespace2',fanout=True)
client = messaging.RPCClient(transport,target)
ret = client.cast(ctxt={},method='test',arg = 'this is text')
```

我们使用cast调用服务端的接口，所以实际执行的时候，程序会等待**AnotherTestEndpoint.test**接口执行完毕，并获取到最终的返回值。

#### Notification Listener和Notifier

Notification Listener代码: 

```python
from oslo_config import cfg
import oslo_messaging as messaging


class NotificationEndpoint(object):

    def warn(self,ctxt,publisher_id,event_type,payload,metadata):
        print('in class warning')
return messaging.NotificationResult.HANDLED


class ErrorEndpoint(object):
    def error(self, ctxt, publisher_id, event_type, payload, metadata):
        print('in class error')
return messaging.NotificationResult.HANDLED


transport = messaging.get_transport(cfg.CONF,url='rabbit://127.0.0.1:5672/')
targets = [
messaging.Target(topic='notification_1'),
messaging.Target(topic='notification_2')
]
endpoints = [NotificationEndpoint(),ErrorEndpoint()]

server = messaging.get_notification_listener(transport,targets,endpoints)
server.start()
server.wait()
```


Notifier代码:

```python
from oslo_config import cfg
import oslo_messaging as messaging


transport = messaging.get_transport(cfg.CONF,url='rabbit://127.0.0.1:5672/')
notifier = messaging.Notifier(transport,driver='messaging',topic='notification_3')
notifier.error(ctxt={},event_type='this is type',payload={'hello': 'world'})
notifier.warn(ctxt={},event_type='this is type',payload={'hello': 'world'})
```

Notification listenser在实现的时候直接对应消息级别，比如 warning, error 等，在这个样例中，我们的ErrorEndpoint和NotificationEndpoint会依次被调用，需要注意这里不会等待执行完成。











