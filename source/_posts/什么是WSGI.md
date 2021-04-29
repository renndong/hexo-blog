---
title: 什么是WSGI
tags:
  - Python
  - Web
categories: Python
abbrlink: a2f3d6ca
date: 2019-04-06 19:05:50
---

WSGI全称为Python Web Server Gateway Interface，Python Web服务器网关接口，它是介于Web服务器和Web应用程序（或Web框架）之间的一种简单而通用的接口。

{% img /images/wsgi.jpg %}

我们知道，客户端和服务器端之间进行沟通遵循HTTP协议。但是我们用Python所编写的很多Web程序，并不会直接去处理HTTP请求，因为这太复杂了。所以WSGI诞生了，使从HTTP请求和Web程序之间，多了一种转换过程——从HTTP报文转换成WSGI的数据格式。这个时候，我们的Web程序就可以建立在WSGI之上，直接去处理WSGI解析给我们的请求，而我们就可以专注于Web程序本身的编写。

## 一个简单的WSGI程序

WSGI接口定义的非常简单。根据WSGI的规定，Web程序（即WSGI程序）必须是一个可调用的对象，这个可调用对象可以是函数、方法、类或是实现了`__call__`方法的类实例。这个可调用的对象接收两个参数：

- environ：包含了请求的所有信息的字典。
- start_response：需要在可调用对象中调用的函数，用来发起响应，参数是状态码，响应头部等。

另外，这个可调用对象的还要返回一个可迭代的对象。

我们看一个简单的WSGI程序

```python
def index(environ, start_response):
    status = '200 OK'
    response_header = [('Content-type', 'text/html')]
    start_response(status, response_header)
    yield b'<h1>Hello WSGi</h1>'
```

根据WSGI的定义，请求和响应的主体应为字节串，所以我们在这里返回的html格式字符串上加上了b前缀将其声明为`bytes`类型

## WSGI服务器

现在我们的Web程序（WSGI程序）编写好了，就需要一个WSGI服务器来运行它。Python提供了一个wsgiref库，我们可以在开发时进行使用。

完善上面的WSGI程序如下：

```python
from wsgiref.simple_server import make_server

def index(environ, start_response):
    status = '200 OK'
    response_header = [('Content-type', 'text/html')]
    start_response(status, response_header)
    yield b'<h1>Hello WSGi</h1>'

server = make_server('localhost', 5000, index)
server.serve_forever()
```

我们使用`make_server(host, port, application)`方法创建了一个本地服务器，分别传入主机地址、端口和可调用对象。然后使用`server_forever()`方法来运行它。当在shell中运行后，在浏览器中输入localhost:5000就可以看到我们编写的效果了。

WSGI服务器在启动后会监听本地端口，当收到请求时，他会将请求报文解析成一个environ字典，然后将其传给WSGI程序，同时传递`start_response`函数。当我们的WSGI程序将请求处理完后，会通过`start_response`方法来通知WSGI服务器来发起一个响应，并设置相应的响应头，然后返回响应的主体。然后WSGI服务器再将其解析成HTTP格式，返回给客户端。你也可以通过上面的图片来理解这个过程。

## WSGI中间件

WSGI允许使用中间件（Middleware）来包装Web程序，在程序在调用前添加额外的设置和属性。这个特性常用来解耦程序的功能。

我们也可以给我们的程序添加一个中间件

```python
from wsgiref.simple_server import make_server

def index(environ, start_response):
    status = '200 OK'
    response_header = [('Content-type', 'text/html')]
    start_response(status, response_header)
    yield b'<h1>Hello WSGi</h1>'

class Middleware(object):
    def __init__(self, web_app):
        self.web_app = web_app
    
    def __call__(self, environ, start_response):
        def before_start_response(status, header):
            header.append(('middleware', 'middleware'))
            return start_response(status, header)
        return self.web_app(environ, before_start_response)

new_index = Middleware(index)

server = make_server('localhost', 5000, new_index)
server.serve_forever()
```

这里我们使用实现了`__call__`方法的类实例来创建WSGI的可调用对象。并通过这个中间件来为我们的Web程序添加了一个响应头（尽管这没有意义）。真正的中间件远比我们这里实现的复杂、功能强大的多。而且往往不止一个中间件，而是一个中间件堆栈，通过层层包装，实现了非常多的功能。

## Web框架

现在有了WSGI，我们可以很容易实现一个Python Web程序，但是这还是不够方便，于是有了Web框架。

Python Web框架是在WSGI的上面又抽象出来一层，使之更易使用，编写的Python Web程序也更易维护。

我们以非常著名的Flask框架为例。重新实现一下上面的WSGI程序。

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def index():
    return '<h1>Hello WSGi</h1>'

app.run()
```

另外，Python还有很多流行的Web框架，例如Django，web.py、Tornado等，这里不在详细展开。

---


参考资料:

- https://zh.wikipedia.org/wiki/Web%E6%9C%8D%E5%8A%A1%E5%99%A8%E7%BD%91%E5%85%B3%E6%8E%A5%E5%8F%A3

- 《Flask Web开发实战》