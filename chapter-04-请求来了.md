## 起步

上节我们了解到，`WSGI` 可以将请求信息转发给我们的应用。

这一点，原理以及已经请求的处理我们已经知道了，本节我们将专注于应用 `django` 是如何进行处理 `WSGI` 转发过来的请求的。

## 开始

```python
# django.core.handlers.wsgi.py

class WSGIHandler(base.BaseHandler):
    request_class = WSGIRequest

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.load_middleware()

    def __call__(self, environ, start_response):
        """请求会被转发到这里

        """
        # 似乎是加 `URL` 后缀用的, 暂时不够清晰
        set_script_prefix(get_script_name(environ))
        # 发送请求来了的信号, 关于信号的注册及处理我们在后面的篇章会进行介绍
        signals.request_started.send(sender=self.__class__, environ=environ)
        # 封装 `request` 请求对象 处理类是 -> WSGIRequest
        request = self.request_class(environ)

        # 划重点 -> 接下来我们看它背后发生了什么？ (子类没有就去父类里面去找)
        response = self.get_response(request)

        # 下面的待上述讲解结束, 客官继续向下看(代码已被省略)...
        ...
```

紧跟着，情节发展到了 `get_response` 这个方法这里。

```python
# django.core.handlers.base.py BaseHandler

class BaseHandler:
    _view_middleware = None
    _template_response_middleware = None
    _exception_middleware = None
    _middleware_chain = None

    def get_response(self, request):
        # 设置 url 解析器, 里面具体发生了我们暂且不关心。
        # 这里为后续的路由匹配做好了准备
        set_urlconf(settings.ROOT_URLCONF)

        # _middleware_chain 方法是什么 ...
        # 这里可能会有一些绕, 这个属性是在调用 `load_middleware` 这里赋值
        # 我们将视线转移到 `load_middleware` 处 ...
        # ---- 时间线 ----
        # 通过 `load_middleware` 方法我们知道了 `_middleware_chain` 是个什么东西。
        # 有可能是 `_get_response` 或 某个中间件实例(实例内部有个属性是 `_get_response` )
        # 假设它是个中间件实例, 请各位继续移步向下看。
        response = self._middleware_chain(request)

        # 这里删除了一些似乎不太重要的代码, 也就是说直接响应了 `response` 对象
        ...

        return response

    def load_middleware(self):
        """该方法在 runserver 的时候实例化应用的时候，在 __init__ 方法中调用了该方法

        """

        self._view_middleware = []
        self._template_response_middleware = []
        self._exception_middleware = []

        # `convert_exception_to_response` 写法上是一个装饰器。
        # 这个方法里面完成了对响应的一些异常的捕获
        # 比如 django 404、500 等页面提示以及请求异常信号的发送
        # 现在 `handler` 指向 `_get_response` 这个方法
        handler = convert_exception_to_response(self._get_response)

        # 获取 settings 中间件的配置
        for middleware_path in reversed(settings.MIDDLEWARE):
            # 动态导入模块
            middleware = import_string(middleware_path)
            try:
                # 注意：这里实例化的时候, 将 `handler` 传递为了属性
                mw_instance = middleware(handler)  # 实例化
            except MiddlewareNotUsed as exc:
                # 一些不太重要的代码被我删除了..
                ...

            # 塞入中间件实现的方法
            if hasattr(mw_instance, 'process_view'):
                self._view_middleware.insert(0, mw_instance.process_view)
            if hasattr(mw_instance, 'process_template_response'):
                self._template_response_middleware.append(mw_instance.process_template_response)
            if hasattr(mw_instance, 'process_exception'):
                self._exception_middleware.append(mw_instance.process_exception)

            # 这里的 handler 被重新赋值，变为了中间件实例(似乎是一直不断的被重新赋值)
            handler = convert_exception_to_response(mw_instance)

        # 这个属性可能是 `__get_response` 方法 或 是一个中间件实例
        self._middleware_chain = handler
```

上述我们了解到，通过实现了 `wsgi` 框架跑起来一个服务的时候，将 `django.core.handlers.wsgi.py` 下的 `WSGIHandler` 的实例 设置为了应用程序。

`WSGIHandler` 实例化的时候，执行了 `load_middleware` 方法，该方法载入了 `django` 的中间件，并设置了 `_middleware_chain` 属性。（该属性有可能是 `_get_response` 或 某个中间件的实例(实例中有 `_get_response` 一个属性)。）

请求被转发到应用程序 `django`，也就是 `WSGIHandler` 下的 `__call__`，这里设置了路由解析器，并调用了 `_middleware_chain`，返回值是一个 `response` 对象。

接下来，我们看下 `_middleware_chain` 发生了什么？

## 渐入佳境
