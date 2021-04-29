---
title: Flask源码浅析
tags:
  - Python
  - Web
  - Flask
categories: Python
abbrlink: 584c5358
date: 2019-04-19 23:38:55
---

## 前言

学习一样东西，要先知其然，然后知其所以然。

这次，我们看看Flask Web框架的源码。我会以Flask 0.1的源码为例，把重点放在Flask如何处理请求上，看一看从一个请求到来到返回响应都经过了什么过程。

你可能会问，为什么以Flask 0.1为例啊，那都是好几年前的一坨老代码了？老，并不代表没有用。相反，Flask 0.1的源码设计极为精妙，包含了Flask的主干部分，整个项目只有一个文件，六百行左右，分析起来也简单，有利于我们了解整个Flask的脉络。你可以从[这里](https://github.com/pallets/flask/tree/0.1)来获取Flask 0.1的源码。

## Flask中定义的几个的类和函数

在Flask 0.1的源码中，一共定义了五个类：

1. `Request`和`Response`, 它们分别是Flask的请求和响应对象，分别继承自Werkzeug中的请求和响应类
2. `_RequestContext`，请求上下文类。它包含了所有请求的相关信息。包括程序实例app，url匹配器，请求对象，session对象，g对象以及用于记录闪现的消息的`flashes`
3. `_RequestGlobals`，使用该类创建g对象，这个对象内没有任何的属性，你可以给该类的实例（即g）绑定任何的全局属性。
4. `Flask`，它是整个Flask框架的中心类，它实现了WSGI程序用于处理请求和响应，并且，它是整个所有视图函数、模板配置、URL规则的中心注册处。

另外，Flask中还定义了一些函数：如`render_template`、 `url_for`、`flash`、`get_flashed_messages`等，相信大家都知道这些函数的作用，我就不在赘述。下面我们着重看看`Flask`类。

## Flask中本地上下文

在flask.py文件的最后，定义了几个全局对象：

```python
_request_ctx_stack = LocalStack()
current_app = LocalProxy(lambda: _request_ctx_stack.top.app)
request = LocalProxy(lambda: _request_ctx_stack.top.request)
session = LocalProxy(lambda: _request_ctx_stack.top.session)
g = LocalProxy(lambda: _request_ctx_stack.top.g)
```

其中，`_request_ctx_stack`是Flask的请求上下文堆栈，它的栈顶即是当前请求上下文对象的实例，当一个请求到来时，Flask会将一个请求上下文对象推入这个堆栈以便在程序中使用。`current_app`、`request`、`session`和`g`通过代理的方式从上下文堆栈中获取到所需要的值。如果你还不清楚`LocalStack`和`LocalProxy`，可以参见[什么时Werkzeug](/2019/04/09/什么是Werkzeug/#more)

## Flask类

下面，我们重点看一下Flask类是如何定义的。

### 从Flask类开头和`__init__`看起

在开始处，我们会看到Flask将`Request`和`Response`分别赋值给了`request_class`和`response_class`.

```python
request_class = Request
response_class = Response
```

Flask并没有在程序中直接使用`Request`和`Response`来生成请求和响应对象，而是通过调用`request_class`和`response_class`来生成，这就给我们自定义请求和响应类提供了方便。你可以通过继承Flask中的请求和响应类来构建自己的请求和响应类，并将它们赋值给`request_class`和`response_class`即可。

在下面：

```python
static_path = '/static'
secret_key = None
session_cookie_name = 'session'
```

我们可以看到在这里Flask定义了静态资源的目录，密钥以及cookie的名称，当然，这些你也可以手动进行修改。

下面，我们看一看`__init__`：

```python
def __init__(self, package_name):
    self.debug = False  # 是否开启调试

    self.package_name = package_name

    self.root_path = _get_package_path(self.package_name)  # 程序的根目录

    self.view_functions = {}  # 用于保存注册的视图函数

    self.error_handlers = {}  # 保存注册的错误处理函数

    self.before_request_funcs = []  # 保存请求开始时前调用的函数

    self.after_request_funcs = []  # 保存请求完成后调用的函数

    self.url_map = Map()  # 保存路由规则
```

可以看到，我们用字典来保存注册的视图函数和错误处理函数，以及用列表保存请求前后要掉用的函数。其中用一个`Map`对象`url_map`来保存我们对URL进行处理的路由规则，其中每个路由规则为一个`Rule`对象，我们会在下文看到。另外，`__init__`中还定义的用于保存模板函数的属性，以及Jinja2环境对象，这里不在一一列出。

### 用于注册视图函数的装饰器

众所周知，我们可以使用下面的方式来注册视图函数：

```python
@app.route('/')
def index():
    return 'Hello World'
```

我们看一下这个装饰器是如何实现的：

```python
def route(self, rule, **options):
    def decorator(f):
        self.add_url_rule(rule, f.__name__, **options)
        self.view_functions[f.__name__] = f
        return f
    return decorator
```

在`route`中，会对我们传入的视图函数进行包装，首先调用Flask中的`add_url_rule`方法，然后以函数名为键，将视图函数保存在`__init__`中定义的用于保存视图函数的`view_functions`字典中。

下面，我们看看在`add_url_rule(rule, f.__name__, **options)`内部发生了什么：

```python
def add_url_rule(self, rule, endpoint, **options):
    options['endpoint'] = endpoint
    options.setdefault('methods', ('GET',))  #  默认监听GET方法
    self.url_map.add(Rule(rule, **options))
```

在`add_url_rule`中，Flask首先将以`'endpoint'`为键，将端点值放入`options`中。如果`options`没有`'methods'`键，Flask会在这里给我们添加一个默认的GET方法，也就是说，当我们直接使用`@app.route('/')`，而不传入监听的方法时，Flask会默认监听GET方法。最后，Flask以当前的`rule`和`options`创建一个`Rule`对象放入到`url_map`中，为我们的程序新增了一条路由规则。

另外，除了`route`装饰器外，Flask中还有还提供了用于注册错误函数、请求前调用的函数、请求后调用的函数等的装饰器，这些装饰器和`route`装饰器基本相同，只是没有添加路由规则这个功能。例如请求处理前调用的函数的的装饰器:

```python
def before_request(self, f):
    self.before_request_funcs.append(f)
    return f
```

## Flask请求响应流程

Flask中定义了`wsgi_app(self, environ, start_response)`方法作为WSGI的程序，它并没有写死在`__call__`方法中，因此可以为其添加中间件。当请求到来时，WSGI服务器会调用此方法，并将请求的参数和用于发起响应的函数作为参数传递给它。

```python
def wsgi_app(self, environ, start_response):
    with self.request_context(environ):
        rv = self.preprocess_request()  # 预处理请求
        if rv is None:
            rv = self.dispatch_request()  # 请求分发
        response = self.make_response(rv)  # 生成响应
        response = self.process_response(response)  # 响应处理
        return response(environ, start_response)
```

Flask在`with`语句下执行相关操作，这会触发`_RequestContext`中的`__enter__`方法，从而推送请求上下文到堆栈中。在`with`中，Flask先通过`preprocess_request()`预处理请求，在`preprocess_request()`中调用所有在`beforce_request()`装饰器中注册的**请求前要调用的函数**。随后，Flask使用`dispatch_request()`来进行请求分发，获得视图函数的返回值或是错误处理器的返回值。然后Falsk将请求分发时获得的返回值传给`make_response()`方法来生成一个响应对象，接下来，Flask在`process_response()`方法中调用所有在`after_request()`装饰器中注册的**请求完成后要调用的函数**。最后，通过`response`来发起一个响应，这会自动调用`start_response`方法来发起响应并将响应的值返回给WSGI服务器。

### 预处理请求

```python
def preprocess_request(self):
        for func in self.before_request_funcs:
            rv = func()
            if rv is not None:
                return rv
```

上面的函数会在实际的请求分发之前调用，而且将会调用每一个使用`before_request()`装饰的函数。如果其中某一个函数返回一个值，这个值将会作为视图返回值处理并停止进一步的请求处理。

### 请求分发

```python
def dispatch_request(self):
    try:
        endpoint, values = self.match_request()
        return self.view_functions[endpoint](**values)
    except HTTPException, e:
        handler = self.error_handlers.get(e.code)
        if handler is None:
            return e
        return handler(e)
    except Exception, e:
        handler = self.error_handlers.get(500)
        if self.debug or handler is None:
            raise
        return handler(e)
```

在上面的方法中，Flask对URL进行匹配获取端点值和参数，然后调用相应的视图函数并将视图函数的返回值返回，或者返回相应的错误处理器的返回值。这里的返回值不一定是响应对象，比如我们可以在视图函数中返回一个字符串或者是使用`render_template()`渲染好的模板，所以，为了能够将返回值转换成合适的对象，我们需要`make_response()`方法来生成响应

### 生成响应

```python
def make_response(self, rv):
    """
    rv允许的类型如下所示：
    ======================= ===============================================
    response_class          这个对象将被直接返回
    str                     使用这个字符串作为主体创建一个请求对象
    unicode                 将这个字符串进行utf-8编码后作为主体创建一个请求对象
    tuple                   使用这个元组的内容作为参数创建一个请求对象
    a WSGI function         这个函数将作为WSGI程序调用并缓存为响应对象
    ======================= ===============================================
    :param rv: 视图函数返回值
    """
    if isinstance(rv, self.response_class):
        return rv
    if isinstance(rv, basestring):
        return self.response_class(rv)
    if isinstance(rv, tuple):
        return self.response_class(*rv)
    return self.response_class.force_type(rv, request.environ)
```

在上面的方法中，也印证了我们上面所说的请求分发中**视图函数的返回值不一定是请求对象**这一点。所以，我们在`make_response`方法中对请求分发中获取的返回值的类型进行判断，通过不同的方式来创建真正的响应对象并返回。

### 响应处理

```python
def process_response(self, response):
    session = _request_ctx_stack.top.session
    if session is not None:
        self.save_session(session, response)
    for handler in self.after_request_funcs:
        response = handler(response)
    return response
```

响应处理和预处理请求类似，都会循环调用所有注册的**请求后调用的函数**来对响应对象`response`进行处理，不过在此之前会先将session添加到响应对象中。


### 返回响应

我们Flask中的响应对象会继承自Werkzeug中的`Response`对象。`Response`的实例可以根据传入的参数，来发起一个特定的响应。你可以认为`Response`是你可以创建的另一个标准的WSGI应用，这个应用可以根据你传入的参数，来帮你做发起响应这件事。例如下面一个简易的WSGI程序：

```python
def application(environ, start_response):
    request = Request(environ)
    response = Response("Hello %s!" % request.args.get('name', 'World!'))
    return response(environ, start_response)
```

好了，到此，Flask的一次请求就处理完了。不难发现，在Flask中，对Werzeug这个工具库是很依赖的，从请求处理，路由匹配，到发起请求，都可见到Werkzeug的身影。

## 总结

请求响应类，请求上下文类，全局对象类，核心类`Flask`

Flask中，保存有视图函数、错误处理函数、路由规则，可以处理请求

请求处理流程：预处理请求、请求分发、生成响应、返回响应

---

参考：

- 《Flask Web开发实战》