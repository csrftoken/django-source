## 起步

上节知悉了 `django` 是如何做到自动重启的之后，那么接下来了解下它是如何运行起一个 web 服务的。

## 开始

> 很熟悉的命令

通过下面的命令，我们便可将 django 跑起来。那么它发生了什么呢？

`python3 manage.py runserver`  

## 简单的 web 服务

根据 django 运行的服务，我们也利用 wsgiref 这个模块实现一个简单的 web 服务。

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

运行起来 `server_demo.py` 这个文件，然后通过浏览器访问 `127.0.0.1:1234`，会发现页面响应了一堆内容。

## WSGI

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
        # ThreadingMixIn.daemon_threads indicates how threads will behave on an
        # abrupt shutdown; like quitting the server by the user or restarting
        # by the auto-reloader. True means the server will not wait for thread
        # termination before it quits. This will make auto-reloader faster
        # and will prevent the need to kill the server manually if a thread
        # isn't terminating correctly.
        httpd.daemon_threads = True
    # 这里的 wsgi_handler 来自于
    # from django.core.handlers.wsgi import WSGIHandler
    httpd.set_app(wsgi_handler)
    httpd.serve_forever()

```
