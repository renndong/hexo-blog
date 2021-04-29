---
title: 什么是Werkzeug
tags:
  - Python
  - Web
categories: Python
abbrlink: 59fca22f
date: 2019-04-09 23:08:37
---

上一节介绍了[什么是WSGI](/2019/04/06/什么是WSGI/#more)，这一节我们看看Werkzeug

按照官方的说法，Werkzeug(源自德语，工具的意思)是一个WSGI工具库，它开始于一个适用于WSGI的多样化的工具集，后来发展成了现在非常流行的WSGI工具库。Werkzeug可以在程序中单独使用，也作为许多Python Web框架的底层库，例如现在非常流行的Flask Web框架。

## Werkzeug的基本功能

正如官方的说法，Werkzeug提供了非常丰富的功能，但是其功能总的可分为两个方面：开发测试方面的功能和其用于Web程序中的工具函数及工具类

### 开发测试方面

一、Werkzeug提供了一个简易的开发用服务器
二、Werkzeug提供了一些测试工具，如`Client`类、`EnvironBuilder`类。
三、Werkzeug提供了Debug的工具，提供了可用于Debug的中间件。当程序出错时，并不会返回500错误，而是显示程序出错的地方以及出错的原因，这就为程序的开发提供了方便。

### 工具方面

Werkzeug主要提供了如下几种工具

#### 一、请求和相应对象。

提供了`Request`和`Response`。`Request`可以包装WSGI服务器传入的`environ`参数，并对其进行进一步的解析，以使我们更容易的使用请求中的参数。`Response`可以根据传入的参数，来发起一个特定的响应。你可以认为`Response`是你可以创建的另一个标准的WSGI应用，这个应用可以根据你传入的参数，来帮你做发起响应这件事。

```python
from werkzeug.wrappers import Request, Response

def application(environ, start_response):
    request = Request(environ)
    response = Response("Hello %s!" % request.args.get('name', 'World!'))
    return response(environ, start_response)
```

#### 二、路由解析。

Werkzeug提供了强大的路由解析功能。比如Flask框架中经常用到的`Rule`、`Map`类等。

如下面一个程序。

```python
from werkzeug.routing import Map, Rule, NotFound, RequestRedirect

url_map = Map([
    Rule('/', endpoint='blog/index'),
    Rule('/<int:year>/<int:month>/<int:day>/', endpoint='blog/archive'),
    Rule('/about', endpoint='blog/about_me'),
    Rule('/feeds/<feed_name>.rss', endpoint='blog/show_feed')
])

def application(environ, start_response):
    urls = url_map.bind_to_environ(environ)
    try:
        endpoint, args = urls.match()
    except HTTPException, e:
        return e(environ, start_response)
    start_response('200 OK', [('Content-Type', 'text/plain')])
    return ['Rule points to %r with arguments %r' % (endpoint, args)]
```

我们创建了一个`Map`类实例`url_map`来保存一系列的URL规则。并且给它传递了一个`Rule`对象的列表。其中，每个`Rule`对象都包含两个参数：一个字符串和`endpoint`。字符串代表了URL匹配的规则（也叫路由规则），`endpoint`（也叫端点）代表了该路由规则对应的视图函数。即当对一个URL匹配成功后，便可获取到它对应的视图函数。不同的规则可以对应相同的`endpoint`，但是必须有不同的参数用于URL的构建，不能产生歧义，类似于函数的重载。

在`application`函数中，我们使用`Map`的`bind_to_environ`方法将`url_map`与`environ`绑定，这会返回给我们一个新的`MapAdapter`对象，这个对象可用于URL的匹配。随后，我们调用`MapAdapter`对象中的`match()`方法，获取**当前请求**的URL匹配到的`endpoint`和其参数信息，最后，我们用获取到的`endpoint`和参数信息发起一个响应。

用于匹配URL的路由规则字符串是由基本的URL加上占位符组成的。

例如`Rule('/pages/<path:page>')`，尖括号中，冒号后面为变量名，前面为变量的类型。`path`类型表示只匹配路径。这里的`path`也可以是`int`，表示匹配一个整型以及`float`等。

当不包含尖括号中的变量不写明类型时，如`Rule('/pages/<page>')`，这里的`page`可以匹配任何字符串，但是只能就受一个路径段，因此不能含有`/`。

更详细的规则还请参见[文档](https://werkzeug.palletsprojects.com/en/0.15.x/routing/)

#### 三、本地上下文

在许多Web程序中，本地上下文是个非常重要的概念。而实现本地上下文需要用到不同线程间数据的隔离。`werkzeug.local`中定义了`Local`、`LocalStack`和`LocalProxy`等类用于实现全局数据的隔离。

在Python中，我们可以使用`thread locals`来保证多线程状态下数据的隔离，但是这在Web程序中，却并不是很好使。

- 一是因为有些Web应用是使用协程实现的，无法保证数据的隔离。
- 二是即使使用的是线程，WSGI也不能保证每次请求使用的线程都是一个全新的线程，可能是一个**之前请求**的线程，而里面的数据也是原线程剩下的。

所以，Werkzeug给我们提供了`Local`这个更好用的解决工具。

下面是一个如何使用`werkzeug.local`例子：

```python
from werkzeug.local import Local, LocalManager

local = Local()
local_manager = LocalManager([local])

def application(environ, start_response):
    local.request = request = Request(environ)
    ...

application = local_manager.make_middleware(application)
```

可以看到，我们把一个`Request`对象赋值给了全局对象`local.request`，这样，我们就可以在全局范围内使用`local.request`，而且获取到的仅仅是当前请求的数据。因为`Local`对象不会在请求结束后自动清除本地上下文，所以这里我们需要使用`LocalManager`来管理。我们需要将管理的`Local`对象以列表的方式传给`LocalManager`，并在最后使用`LocalManager`的`make_middleware`方法为WSGI程序添加中间件，来使请求结束后自动清除本次请求的数据。

那么`Local`是如何实现的呢？其实很简单，在`Local`中，重写了`__getattr__`和`__setattr__`方法，使得在获取数据和存储数据之前，先获取到线程id（或协程id），以线程id（或协程id）为键，数据为值，存储在一个字典中。这样我们在操作数据的时候，操作的只会是当前线程（或协程）的数据，从而实现了数据隔离。感兴趣的同学可以查看一下文末`Local`的源码。

`LocalStack`对`Local`进行了封装，使其可以以栈的方式使用。如下：

```python
>>> ls = LocalStack()
>>> ls.push(42)
>>> ls.top
42
>>> ls.push(23)
>>> ls.top
23
>>> ls.pop()
23
>>> ls.top
42
```

`LocalProxy`类用于实现werkzeug本地代理，将所有的操作转发给代理对象。如果你熟悉C++的话，你会发现这和C++的引用很像，但比引用更强大。使用方法如下：

```python
from werkzeug.local import Local
l = Local()

# 以下是两个代理
request = l('request') # Local中实现了__call__方法，用于返回一个代理，具体可以查看文末Local的源码
user = l('user')


from werkzeug.local import LocalStack
_response_local = LocalStack()

# 这也是个代理
response = _response_local() # 同理，LocalStack返回的也是代理
```

除了以上创建代理的方式外，还可以手动创建一个代理

```python
from werkzeug.local import Local, LocalProxy
local = Local()
request = LocalProxy(local, 'request')
```

如果你想拥有一个根据指定函数来返回不同的对象代理，也是支持的。

```python
session = LocalProxy(lambda: get_current_request().session)
```

但我们为什么要使用代理呢。这里简单说一下，我们知道，一个变量被赋值后如果不重新赋值，它的值是不会改变的，那么这在程序的某些地方就会变得很不方便。但是如果使用代理的话，那么我们在使用这个变量的时候就能动态的获取到它所代理的对象的最新的值。

#### 四、其他

除了上面三个方面外，Werkzeug还提供了很多工具，例如WSGI中间件、HTTP异常类、数据结构等。这里就不在一一详述，感兴趣的同学可以参考[文档](https://werkzeug.palletsprojects.com/en/0.15.x/)。

---

`Local`对象部分源码：

```python
try:
    from greenlet import getcurrent as get_ident
except ImportError:
    try:
        from thread import get_ident
    except ImportError:
        from _thread import get_ident

class Local(object):
    __slots__ = ('__storage__', '__ident_func__')

    def __init__(self):
        object.__setattr__(self, '__storage__', {})
        object.__setattr__(self, '__ident_func__', get_ident)

    def __iter__(self):
        return iter(self.__storage__.items())

    def __call__(self, proxy):
        """Create a proxy for a name."""
        return LocalProxy(self, proxy)

    def __release_local__(self):
        self.__storage__.pop(self.__ident_func__(), None)  # 清除数据

    def __getattr__(self, name):
        try:
            return self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)

    def __setattr__(self, name, value):
        ident = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            storage[ident] = {name: value}

    def __delattr__(self, name):
        try:
            del self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)
```

---

参考：
- [Werkzeug(Flask)之Local、LocalStack和LocalProxy](https://www.jianshu.com/p/3f38b777a621)
- [Werkzeug0.15.x文档](https://werkzeug.palletsprojects.com/en/0.15.x/)

本篇参考Werkzeug文档写成，如有错误或与文档不符的地方，还请以文档为准，也欢迎您反馈给我。
