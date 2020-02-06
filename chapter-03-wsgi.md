
## 起步

上节我们知道了 `django` 是如何运行起一个服务的。

但是这里面涉及到一个 `wsgi` 的知识点。

## WSGI 历史

## WSGI 协议

> 全称叫做 `Web Server Gateway Interface`

首先这个如标题所说，它是一种协议，一种规范。它不是服务器、框架、模块、API或软件。

它描述了 `web server（web服务器）` 和 `web application（web应用）` 通信的规范。这种规范在 [PEP333](https://www.python.org/dev/peps/pep-0333/) 提出。(主要为了支持3.x并提供一个通用、高级的接口。)

既然是协议、规范，那么一定是为了解决一些问题的。

如果没有该规范，一个服务器调用应用是用这种方式，另一个服务器调度是那种方式，如此的话，编写的应用部署只能局限于某些服务器，达不到通用的效果。

例如我们常常接触的 web 框架有 `django`、`flask`、`bottle` 等等，它们均支持了 `wsgi` 协议。

*关于支持了 WSGI 协议的 web server，具体可参见 [Servers which support WSGI](https://wsgi.readthedocs.io/en/latest/servers.html)。*

## Why you need WSGI

WSGI 加快了Python web应用程序的开发，因为只需了解关于WSGI的基本信息。如果使用 `django`、`cherrypy`，则不需要关心特定框架如何利用WSGI标准。但是，了解如何实现 WSGI 将有非常大的好处。


## WSGI 接口

### 概念

WSGI接口有两个方面：服务器(网关)方面，以及应用程序(框架)方面。

服务器端调用应用程序端提供的可调用对象。如何提供该对象的细节取决于服务器或网关。假设某些服务器或网关需要应用程序的部署人员编写一个简短的脚本来创建服务器或网关的实例，并为其提供应用程序对象。其他服务器和网关可以使用配置文件或其他机制来指定应该从何处导入应用程序对象或以其他方式获取应用程序对象。

### 要求

WSGI 对于 `application` 对象有如下三点要求：

- 必须是一个可调用的的对象。
- 接收俩个必选参数 `environ`、`start_response`。
- 返回值必须是可迭代对象，用来表示 `http body`。

### 做什么

- `web server` 负责从客户端接收请求，将 `request` 转发给 `application`，继而将 `application` 返回的 `response` 返回给客户端。
- `application` 接收由 `web server` 转发的 `request`，并将处理结果返回给 `server`。

像 `django`、`falsk`、`bottle` 等框架都有自己实现的简单 `WSGI Server`，但是这种一般用于开发环境下调试，生产环境下建议使用其它的 `wsgi server`。

## 参考文献

- [WSGI: The Server-Application Interface for Python](https://www.toptal.com/python/pythons-wsgi-server-application-interface)
- [An Introduction to Python WSGI Servers](https://www.appdynamics.com/blog/engineering/an-introduction-to-python-wsgi-servers-part-1/)
- [PEP333](https://www.python.org/dev/peps/pep-0333/)
- [做python Web开发你要理解：WSGI & uwsgi](https://www.jianshu.com/p/679dee0a4193)
- [理解Python WSGI](letiantian.me/2015-09-10-understand-python-wsgi/)
- [花了两个星期，我终于把 WSGI 整明白了](https://juejin.im/post/5cff300a6fb9a07ef06f8a43#heading-8)
