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

`WSGIHandler` 实例化的时候，执行了 `load_middleware` 方法，该方法载入了 `django` 的中间件，并设置了 `_middleware_chain` 属性。（该属性有可能是 `_get_response` 或 某个中间件的实例(实例中有 `_get_response` 一个属性)。）

请求被转发到应用程序 `django`，也就是 `WSGIHandler` 下的 `__call__`，这里设置了路由解析器，并调用了 `_middleware_chain`，返回值是一个 `response` 对象。

接下来，我们看下 `_middleware_chain` 发生了什么？

## 渐入佳境

既然上文我们设定 `_middleware_chain` 是一个中间件，且是一个类的实例，那么问题来了？

实例加括号调用什么方法？ 答案是类的 `__call__` 方法。

`__call__` 方法被定义在了 `django.utils.deprecation` 下的 `MiddlewareMixin`。(继承它的子类并没有实现 `__call__` 方法。)

```python

# django.utils.deprecation.py

class MiddlewareMixin:
    def __init__(self, get_response=None):
        # 还记得上文提到的 `get_response` 吗？
        self.get_response = get_response
        super().__init__()

    def __call__(self, request):
        """ 上文我们提到的执行 `_middleware_chain`，其实就是执行到了这里。  

        """
        response = None
        # 开始处理中间件方法, 请求来了...
        if hasattr(self, 'process_request'):
            # 如果中间件方法返回了 response 对象，相当于截断了后续的操作。
            response = self.process_request(request)
        # 真正有趣的地方来了, 我们假设中间件方法没有拦截，执行了 `get_response` 方法。
        # 请求处理的逻辑就是在这里了 ... 我们继续向下看
        response = response or self.get_response(request)
        # 继续执行中间件方法
        if hasattr(self, 'process_response'):
            response = self.process_response(request, response)
        return response
```

上述主要调用了中间件的 `__call__` 方法，在该方法中，执行了中间件相关处理方法，继续向下剖析。

```python

# django.core.handlers.base.py BaseHandler 下的 `_get_response` 方法

def _get_response(self, request):
    """解析并调用视图，以及 视图、异常和模板响应 中间件。

    这个方法是 请求/响应 中间件中发生的所有事情。
    """
    response = None

    # 这个里面主要是根据 settings 配置的 url 节点，导入并获取一个解析器对象。
    if hasattr(request, 'urlconf'):
        urlconf = request.urlconf
        set_urlconf(urlconf)
        resolver = get_resolver(urlconf)
    else:
        resolver = get_resolver()

    # 根据访问的路由获取 django 路由解析器中的匹配的视图
    resolver_match = resolver.resolve(request.path_info)
    callback, callback_args, callback_kwargs = resolver_match
    request.resolver_match = resolver_match

    # 执行含有 `process_view` 方法的中间件
    for middleware_method in self._view_middleware:
        response = middleware_method(request, callback, callback_args, callback_kwargs)
        if response:
            break

    # 如果响应对象并没有拦截，执行视图函数
    if response is None:
        # 保证视图的原子性，这里的变量表示视图
        wrapped_callback = self.make_view_atomic(callback)
        try:
            # django 是视图写法有俩种，FBV 和 CBV
            # CBV 会调用 `as_view` 方法，内部实现了 view 函数，然后该函数分发到了类的 `dispatch` 方法，通过反射映射到具体的方法。
            # 其实本质上这里都是在调用视图函数。
            # 这一步就是真正在执行您的视图内的逻辑啦。
            response = wrapped_callback(request, *callback_args, **callback_kwargs)
        except Exception as e:
            # 如果异常，这里捕获掉，执行含有 `process_exception` 方法的中间件
            response = self.process_exception_by_middleware(e, request)

    if response is None:
        # 这里告知你，要必须返回一个 `response` 对象。
        ...

    # 如果上面的视图中间件返回有 response 对象且可被调用
    elif hasattr(response, 'render') and callable(response.render):
        # 执行含有 `process_template_response` 方法的中间件
        for middleware_method in self._template_response_middleware:
            response = middleware_method(request, response)

        try:
            response = response.render()
        except Exception as e:
            # 如果发生异常，这里捕获掉，执行含有 `process_exception` 方法的中间件
            response = self.process_exception_by_middleware(e, request)

    # 响应 response 对象。
    return response

```

到这里，通过上述剖析，如果请求进来了，我们知悉了请求是如何到我们的视图逻辑中的。

## 总结

当我们的服务跑起来的时候，设置应用处理器 `WSGIHandler` 实例化的时候加载了中间件的操作。

请求来了，请求被转发到了 `WSGIHandler` 下的 `__call__` 方法，这里初始化了请求来了的信号，及将请求信息封装到了 `WSGIRequest` 这个对象中。

接着向下执行 `get_response` 方法主要设置了路由的配置模块路径，到了 `_middleware_chain` 方法这里，它其实是一个 `_get_response` 对象 或是某个中间件的实例(这个实例中有个 `get_response` 属性指向 `_get_response` 这个方法)。这里我们假设它是中间件实例，调用的实例的 `__call__` 方法(存在于 `MiddlewareMixin` 下)。这个方法下执行了 `process_request` 和 `process_response` 这俩个中间件，在它们中间执行 `_get_response`，这个方法会解析路由并调用视图方法及一些中间件方法。

`_get_response` 方法下，获取了路由解析器实例，路由解析器根据请求信息匹配到视图，并执行视图方法，获取到响应结果（如果中间件设置了相关方法，会进行调用）。
