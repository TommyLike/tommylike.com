---
layout: post
title:  "Python常见特性入门(一)"
draft: false
subtitle:   "Python常见特性——装饰器和描述器"
date:   2016-08-14 21:59:44 +0800
categories: ["Python"]
author: "TommyLike"
resources:
  - name: "featured-image"
    src: "featured-image.jpeg"
tags:
    - Python

---

# Python常见特性入门

## 装饰器
有过AOP编程经历的人对以下的场景肯定都不陌生:我们暴露了API接口给客户端调用，基于运维的考虑，我们需要自动拦截API接口的耗时参数等信息，通常的方式就是在基类中包装一层或者使用拦截器，大致的代码如下:   

```csharp
public string WrapExecute(string fake_param)
{
  ExecuteBefore(fake_param);
  var result = Execute();
  ExecuteAfter(result);
  return result
}
```

Python的装饰器天生就可以承担类似的工作,下面是一个简单的装饰器:   

```python
def execute_before():
    print('this is before')


def execute_after():
    print('this is after')


def decorator(func):

    def wrapper():
        execute_before()
        func()
        execute_after()

    return wrapper


def execute():
    print('this is executing')

execute = decorator(execute)
```

跟我们在其他语言中的用法保持一致，当然我们的python在此之上，还提供了更便捷的方式:**@**,python中以@开头并紧跟装饰器的名字可理解成给对应的对应的对象使用装饰器，所以，上面的调用方式，可以优化成:  

```python
@decorator
def execute():
    pass
```

通常情况下，被修饰的函数都是带有参数的，为了能在装饰器中获取参数信息，python中引入了可变参数的概念，`*args`表明所有未被捕获的参数,`**kwargs`表明所有未被捕获的keyword参数，我们把我们的wrapper函数调整为如下，则能捕获所有类型的参数:

```python
de decorator(func)

  def wrapper(*args, **kwargs)
  execute_before()
  func(args, kwargs)
  execute_after()
#other code
```

还有一种情况，就是我们的装饰器本身是可以支持配置参数的，在这种情况下，我们往往还需要在外面再增加一层，在现在的例子基础上，比如我们需要支持配置日志级别,那代码可调整如下:  

```python
def outter_wrapper(log_level='info')
  def decorator(func):

      def wrapper():
          execute_before(log_level)
          func()
          execute_after(log_level)
      return wrapper

  return decorator

@outter_wrapper(log_level='error')
def execute():
  pass
```  

## 描述器
描述符是一个具有绑定行为的对象属性，其属性的访问被描述符协议方法覆写。这些方法是__get__()、 __set__()和__delete__()，一个对象中只要包含了这三个方法（至少一个），就称它为描述符。在python的多个特性:属性(特性)/方法/静态方法/类方法中都有应用。  

1. 如果一个描述器定义了*__get__*和*__set__*，他就是一个资料描述器(data descriptor),可以用来定义属性。  
2. 如果一个描述器只定义了*__get__*，他就是一个非资料描述器，通常用来定义方法。  

以下是一个最基本的描述器的用法(属性):  

```python
class PropertyDescriptor(object):

    def __init__(self, value=None):
        self.value = value

    def __get__(self, obj, objtype):
        print('get value')
        return self.value

    def __set__(self, obj, value):
        print('set value')
        self.value = value


class TestClass(object):
    attr_x = Property(12)      

```

这样每次我们在调用对象*attr_x*属性的时候，相应的*__set__*和*__get__*函数就会被调用。    
实际上，在python中，描述器在被调用的时候根据类型不同，内部具体调用的方法也不一样。  
1. 如果调用方为object instance那相当于调用:*type(instance).__dict__['x'].__get__(instance, type(instance))*    
2. 如果调用方为type那相当于调用:*type(instance).__dict__['x'].__get__(None, type(instance))*      

以上面的代码为例:

```python
instance = TestClass()
instance.attr_x == type(instance).__dict__['attr_x'].__get__(instance, type(instance))
TestClass.attr_x == TestClass.__dict__['attr_x'].__get__(None, TestClass)
```

### 特性
Python内建的property能够帮我们快速构建属性，假设我们现在有个Person对象，需要一个adult(成年)属性根据age(年龄)自动计算，那通过property，代码则大致如下:  

```python
class Person:

    def __init__(self,age = 1):
        self.age = age

    def _is_adult(self):
        return self.age >= 18

    adult = property(_is_adult)

tommy = Person(24)
print(tommy.adult) #True
```

python的property支持定义属性的读取/修改/删除/文档操作，用法如下:    

```python
def get_property_x(self):
  return self.x
#other code
document_string = 'this is property x'
property_x = property(get_property_x,set_property_x,delete_property_x,document_string)
```

### 函数
Python中的一切资源都是面向对象的，这里也包括函数,我们看一个简单的例子:  

```python
class TestMethod(object):

    def hello(self):
        print('hello world')

print(TestMethod.__dict__['hello'])  #<function TestMethod.hello at 0x01178BB8>
```

实际上从类中调用*TestMethod.hello*，返回的是一个unbound对象，而从实例中调用*instance.hello*返回的是一个bound对象，在[这里](http://damnever.github.io/2015/05/07/adding-a-method-to-an-existing-object/)可以看到更多关于bound/unbound的资料  
python中定义类方法和静态方法都是使用描述器和装饰器组合来实现的，具体实现代码参考:[link](https://harveyqing.gitbooks.io/python-read-and-write/content/python_advance/python_descriptor.html)。     

#### 类方法  

```python
class ClassMethod(object):
    "Emulate PyClassMethod_Type() in Objects/funcobject.c"

    def __init__(self, f):
        self.f = f

    def __get__(self, obj, klass=None):
        if klass is None:
            klass = type(obj)
        def newfunc(*args):
            return self.f(klass, *args)
        return newfunc
```

#### 静态方法  

```python
class StaticMethod(object):
    "Emulate PyStaticMethod_Type() in Objects/funcobject.c"

    def __init__(self, f):
        self.f = f

    def __get__(self, obj, objtype=None):
        return self.f  
```
