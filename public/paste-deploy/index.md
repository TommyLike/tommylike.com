# Paste Deploy 入门


# Paste Deploy 入门

## 什么是Paste Deploy
Paste Depoly是一个用于发现和配置WSGI application和server的系统,以下是官方对PasteDeploy的定义:   

> Paste Deployment is a system for finding and configuring WSGI applications and servers.
> For WSGI application consumers it provides a single, simple function (loadapp) for loading a 
> WSGI application from a configuration file or a Python Egg. For WSGI application providers it 
> only asks for a single, simple entry point to your application, so that application users don’t 
> need to be exposed to the implementation details of your application.


最主要的就是能快速构建复杂且独立的WSGI程序，还能屏蔽具体的实现。

## 安装Paste Deploy
PasteDeploy已经从Paste库中剥离出来，可以单独安装:  

```shell
sudo pip install PasteDeploy
```

## Paste Deploy 服务加载
使用PasteDeploy加载服务非常简单，大致的代码如下:  

```python
import os
from paste.deploy import loadapp
from wsgiref.simple_server import make_server


config_path = os.path.abspath('config.ini')  
appname='test_app'
wsgi_app = loadapp('config:%s' % 'config.ini', appname,relative_to=os.getcwd())
server = make_server('localhost', 8080, wsgi_app)
server.serve_forever()
```

使用loadapp指定配置文加载app最终放入server即可，需要注意的是:   

1. appname即你需要执行的application名，需要与配置文件中的名称保持一致( *[composite:xxxx]* )
2. load_app需要指定配置文件的绝对路径，如果是相对路径，需要通过 *relative_to* 来指定相对的目录
3. 配置文件需要类似 *config:absolute_path* 格式

## Config 配置解析
配置文件使用**.INI(initialize)**的文件格式，在ini文件中一般使用键值对的方式进行配置:

```ini
name=value
```

配置文件中可以使用**;**或者**#**作为注释符(必须在行首)，配置文件一般如下所示:

```ini
#this is a comment
[composite:main]
use = egg:Paste#urlmap
/ = home
/blog = blog
/wiki = wiki
/cms = config:cms.ini

;this is another comment
[app:home]
use = egg:Paste#static
document_root = %(here)s/htdocs

[filter-app:blog]
use = egg:Authentication#auth
next = blogapp
roles = admin
htpasswd = /home/me/users.htpasswd

[app:blogapp]
use = egg:BlogApp
database = sqlite:/home/me/blog.db

[app:wiki]
use = call:mywiki.main:application
database = sqlite:/home/me/wiki.db
```

文件中使用单行的 **[]** 将配置字段划分成多个区域(section)。Paste Deploy只关注有前缀的区域，格式一般如下所示:

```ini
[composite:main]
```

:前面表示该区域的类型，后面表示区域的名称，类型一般有 *app,filter,pipeline,composition* 等，下面以常见的类型做简单的介绍:  

### composite
composite可以理解成是一个application的组合器或者url的分发器，负责把不同的url分发到不同的application中：

```ini
[composite:main]
#即使用Paste中的urlmap模块进行url映射分发
use = egg:Paste#urlmap 
#指向本配置文件中的home app 
/ = home                
#指向本配置文件中的blog app
/blog = blog   
#指向本配置文件中的wiki app         
/wiki = wiki            
#指向同级目录的另一个配置文件，作为二级app挂在到/cms下
/cms = config:cms.ini   
```

### app
即application，app可以支持多种不同的应用，通过以下几种举例:  

通过.ini文件指向另一个配置文件里的应用  

```ini
[app:configapp]
use = config:another_config_file.ini#config_name
```    
指向Egg封装的应用  

```ini
[app:eggapp]
use = egg:AnotherApp
```  
指向模块中的应用  

```ini
[app:moduleapp]
use = call:another_module:app
```  
指向配置文件中的其他app  

```ini
[app:sectioneapp]
use=antoher_section_app
```  
指向paste/factory配置的app 

```ini  
[app:factoryapp]
paste.app_factory = module_a.class_b.factory
```   

指向静态资源路径  

```ini
[app:staticsfolder]
use = egg:Paste#static   
document_root = %(here)s/htdocs 这里使用%()s用于动态载入实际目录  
```  

### filter
即过滤器，快速构建AOP模块，通常用来做校验，日志等，我们通过以下几种类型举例: 
1.使用filter-with对指定app进行过滤

```ini
[filter-app:blog]
use = egg:Authentication#baseauth
next = blogapp
```
以上配置中通过use和next，指明使用baseauthfilter对blogapp进行过滤 

2.直接使用filter-with指定过滤器

```ini
[app:main]
use = egg:MyEgg
filter-with = printdebug
[filter:printdebug]
use = egg:Paste#printdebug
```
跟第一种方式比起来，filter-app能够快速被多个application复用。 

3.使用pipeline快速建立多级过滤

```ini
[pipeline:main]
pipeline = filter1 egg:FilterEgg#filter2 filter3 app
```
需要注意的事，这里通过**pipeline=**指定具体的过滤，可以指定多个，配置的先后顺序就是依次通过的顺序，但是必须要以一个application作为末尾节点。


### Config Variables
配置文件中支持配置全局或者局部变量，这些字段会在构建app的过程中作为参数传入,实例如下:   

```ini
[DEFAULT]
key_one = value
key_another = value
[app:another]
key_section = value
```
在局部的配置中支持对变量进行覆盖重定义,使用关键字set  

```ini
[app:another]
set key_one = new_value
```

### Factories
Factories即我们的App/Filter生成器，需要遵守协议和规范定义，根据用途的不同，可以分为以下几种:

#### 1.app_factory 
生成标准的WSGI application，规范如下  

```python
def app_factory(global_config, **local_conf):
    return wsgi_app

```
传入的参数:global_config和local_conf，分别对应我们的不同配置变量,以我们上面的配置文件举例，那实际传入的值如下:

```ini
#一般还会有其他字段,如__file__
global_config = {key_one:value,key_another:value,...} 
local_conf = {key_section:value}
```

举一个实际的例子，加入我们现在要定义一个application，计算传入的param1和param2的和，并返回最终的和:  

```python
from webob import Request


class Calculator:

    def __init__(self):
        pass

    def __call__(self, environ, start_response):
        req = Request(environ)        
        param1 = int(req.GET.get("param1", None))
        param2 = int(req.GET.get("param2", None))                 
        start_response("200 OK",[("Content-type", "text/plain")])
        return ["RESULT={0}".format(param1+param2).encode('utf-8')]

    @classmethod
    def factory(cls, global_conf, **local_conf):        
        return Calculator()

```

```ini
#config files
[app:calculator]
paste.app_factory = tommylike:Calculator.factory
```

#### 2.filter_factory  
跟我们的app_factory差不多，区别在于他接受的是一个WSGI app，同样假设我们要定义一个filter-app,用来做日志拦截记录:

```python
class LogFilter:

    def __init__(self, app):

        self.app = app

    def __call__(self, environ, start_response):
        print("Log Filter")
        return self.app(environ,start_response)

    @classmethod
    def factory(cls, global_conf, **kwargs):
        return LogFilter

```

```ini
#config files
[filter:apilog]
paste.filter_factory = tommylike:LogFilter.factory
```

#### 3.filter_app_factory 
跟filter_factory相比，传入的参数多了一个app,可以借用上面的例子比较下

```pythhon
#used for  filter_factory
@classmethod
  def factory(cls, global_conf, **kwargs):      
      return LogFilter1

#used for filter_app_factory
@classmethod
  def factory(cls,app, global_conf, **kwargs):      
      return LogFilter2(app)      
```

#### 4. filter_server 
接受一个app，并返回封装这个app的server，举例代码如下

```python
def server_factory(global_conf, host, port):
    port = int(port)
    def serve(app):
        s = Server(app, host=host, port=port)
        s.serve_forever()
    return serve
```

#### 5. server_factory 
与filter_server差不多，区别在于server会马上拉起来，并且app是作为第一个参数传入,根filter_factory与filter_app_factory的区别一致。 

### Oslo.middleware
在Openstack中middleware提供了内置的filter以便我们方便取用，目前主要有两种filter，healthcheck和cors middleware  

### 安装
oslo.middleware目前是独立的组件，可以直接安装

```shell
sudo pip install oslo.middleware
```

### healthcheck middleware
healthcheck middleware主要用于服务的健康状态检查，根据后端是否正常，返回200(服务正常)/503(服务不可用)配置文件如下:  

```ini
[filter:healthcheck]
paste.filter_factory = oslo_middleware:Healthcheck.factory
path = /healthcheck
backends = disable_by_file
disable_by_file_path = /var/run/nova/healthcheck_disable

[pipeline:public_api]
pipeline = healthcheck sizelimit [...] public_service
```

其中 paste.filter_factory指明使用oslo_middleware中的healthcheck，path用来指明检测的具体url：*{resource}/healthcheck*,backends用来指定后端的判定方式，目前有两种：  

使用文件是否存在来检测，配置如下:

```ini
[filter:healthcheck]
#other config
#指明使用文件来校验
backends = disable_by_file  
#指明具体的文件，文件如果存在，healthcheck任务后端返回异常
disable_by_file_path = /var/run/nova/healthcheck_disable 
```

使用文件以及端口检测，配置如下:

```ini
[filter:healthcheck]
#other configs
#指明文件以及端口校验
backends = disable_by_files_ports   
#指明具体的端口和文件
disable_by_file_paths = 5000:/var/run/keystone/healthcheck_disable,  
/35357:/var/run/keystone/admin_healthcheck_disable  
```
**disable_by_file_paths**可以指明多个端口和对应的文件，根据访问的端口进行校验，比如当前端口是5000，则根据文件*/var/run/keystone/healthcheck_disable*的存在与否决定服务是否健康，可以理解为**disable_by_file**的增强版本 

### cors middleware
cors middlreware用来控制跨域访问控制，也可以直接在配置文件中配置:

```ini
[filter:cors]
paste.filter_factory = oslo_middleware.cors:filter_factory
allowed_origin=https://website.example.com:443,https://website2.example.com:443    #设置允许跨域访问的站点和方式等。
max_age=3600
allow_methods=GET,POST,PUT,DELETE
allow_headers=X-Custom-Header
expose_headers=X-Custom-Header
```



