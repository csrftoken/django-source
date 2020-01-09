## 起步

上节知悉了 `django` 是如何做到自动重启的之后，那么接下来了解下它是如何运行起一个 web 服务的。

## 开始

> 很熟悉的命令

通过下面的命令，我们便可将 django 跑起来。那么它发生了什么呢？

`python3 manage.py runserver`  

## 简单的 web 服务

在此之前，根据 django 运行的服务，我们也利用 wsgiref 这个模块实现一个简单的 web 服务。

```python

# server_demo.py

from wsgiref.simple_server import WSGIServer, WSGIRequestHandler

def demo_app(environ, start_response):
    from io import StringIO
    stdout = StringIO()
    print("Hello world!", file=stdout)
    print(file=stdout)
    h = sorted(environ.items())
    for k, v in h:
        print(k, '=', repr(v), file=stdout)
    start_response("200 OK", [('Content-Type', 'text/plain; charset=utf-8')])
    return [stdout.getvalue().encode("utf-8")]


s = WSGIServer(("127.0.0.1", 1234), WSGIRequestHandler)
# 设置应用，注意了这里不一定非是个函数，可是一切可调用的对象
# 例如 函数、方法、类或实现了 __call__ 方法的实例
s.set_app(demo_app)
# 开始监听请求，这里应该是用的 poll 或者 selector 的网络模型。
s.serve_forever()

```

运行起来 `server_demo.py` 这个文件，然后通过浏览器访问 `127.0.0.1:1234`，会发现页面响应了一堆内容(不必关心展示的内容具体是什么)。

## django 的服务

接下来观察源码。

> django/core/servers/basehttp.py 下的 run 方法

```python

def run(addr, port, wsgi_handler, ipv6=False, threading=False, server_cls=WSGIServer):
    server_address = (addr, port)
    if threading:
        httpd_cls = type('WSGIServer', (socketserver.ThreadingMixIn, server_cls), {})
    else:
        httpd_cls = server_cls
    httpd = httpd_cls(server_address, WSGIRequestHandler, ipv6=ipv6)
    if threading:
        httpd.daemon_threads = True
    # 这里的 wsgi_handler 来自于
    # from django.core.handlers.wsgi import WSGIHandler
    httpd.set_app(wsgi_handler)
    httpd.serve_forever()

```

## WSGIHandler

这是一个 wsgi 应用，它被定义在了 `proj/wsgi.py` 下的 `application` 属性。(`application` 对应刚刚上面代码中出现的 `wsgi_handler`)

`application` 这个属性对应的是个 `get_wsgi_application` 函数，该函数调用的过程中，针对 `django.core.handlers.wsgi.WSGIHandler` 这个类进行了实例化，在 `__init__` 方法中，加载了 `django` 中间件。

```python

# django.core.handlers.wsgi.py
class WSGIHandler(base.BaseHandler):
    request_class = WSGIRequest

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # 载入中间件
        self.load_middleware()
```

## 总结

至此，我们了解到了 django 的服务是从 `django/core/servers/basehttp.py` 下的 `run` 方法启动了一个 `wsgi` 服务。这个可调用的对象来自于 `django.core.handlers.wsgi.WSGIHandler`，这个类在实例化(`__init__`)的过程中，加载了中间件。

我们也更应该了解下 `wsgi`。
