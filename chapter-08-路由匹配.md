## 起步

接下来，我们可以了解下请求进来，是怎么进行路由匹配的。

在 `chapter-04-请求来了` 我们简单在那里提及过请求的匹配，但是并没有详细说明。

本篇章会了解到，我们编写的 `urls.py` 文件是如何加载至 django 的路由系统，以及是如何进行匹配的。

## 开始

在 `chapter-04-请求来了` 文章中，请求来了会映射到 `_get_response` 这个函数中去，其中有一段就是解析请求路径的，如下：

```python

if hasattr(request, 'urlconf'):
    urlconf = request.urlconf
    set_urlconf(urlconf)
    resolver = get_resolver(urlconf)
else:
    resolver = get_resolver()

resolver_match = resolver.resolve(request.path_info)
callback, callback_args, callback_kwargs = resolver_match
request.resolver_match = resolver_match

```

> *其它的暂不做分析，我们只针对路由匹配做解析*

接下来，我们先看第一步判断那里，这里面有个共同点，就是都会调用 `get_resolver` 这个方法，区别在于是否传递了 `urlconf` 这个参数。

我们可以看下 `get_resolver` 这个方法。

```python

# 这里涉及到一个 lru，关于 lru 后续我们会有一个小的讲解。
@functools.lru_cache(maxsize=None)
def get_resolver(urlconf=None):
    # 从上述来看，似乎设置与否 `urlconf` 参数，这里都会进行检测
    # 如果没有传递，就进行设置，值为 settings下的 ROOT_URLCONF
    # 这个参数指向了 django 的路由根配置，也就是 `project.urls`
    if urlconf is None:
        urlconf = settings.ROOT_URLCONF

    # URLResolver 类实例化并传递了俩个参数 并返回该实例
    # 暂且先不必关心构造方法内部做了什么。
    return URLResolver(RegexPattern(r'^/'), urlconf)

```

当调用 `get_resolver` 方法后，得到一个 `URLResolver` 类的实例，紧接着这个实例调用了 `resolve` 方法，并把当前请求路径( `e.g /api/banners/` )作为参数传递。

```python

class URLResolver:
    def __init__(self, pattern, urlconf_name, default_kwargs=None, app_name=None, namespace=None):
        # 指向的是 RegexPattern 对象
        # 关于该对象是一个类似于 re 模块，可以进行正则匹配
        # 正则表达式为 '^/' 也就是说必须以 `/` 开头
        self.pattern = pattern
        # 它可以是一个模块、或是具有 `urlpattern` 属性的对象
        self.urlconf_name = urlconf_name

        # 其它参数
        self.callback = None
        self.default_kwargs = default_kwargs or {}
        self.namespace = namespace
        self.app_name = app_name
        self._reverse_dict = {}
        self._namespace_dict = {}
        self._app_dict = {}
        # set of dotted paths to all functions and classes that are used in
        # urlpatterns
        self._callback_strs = set()
        self._populated = False
        self._local = threading.local()

    def resolve(self, path):
        # 路径有可能是 reverse_lazy 对象，所以 str 一下
        path = str(path)
        tried = []
        # 这一步就是校验，路径是否 `/` 开头，匹配不到返回 404
        match = self.pattern.match(path)
        if match:
            new_path, args, kwargs = match
            for pattern in self.url_patterns:
                try:
                    sub_match = pattern.resolve(new_path)
                except Resolver404 as e:
                    sub_tried = e.args[0].get('tried')
                    if sub_tried is not None:
                        tried.extend([pattern] + t for t in sub_tried)
                    else:
                        tried.append([pattern])
                else:
                    if sub_match:
                        # Merge captured arguments in match with submatch
                        sub_match_dict = {**kwargs, **self.default_kwargs}
                        # Update the sub_match_dict with the kwargs from the sub_match.
                        sub_match_dict.update(sub_match.kwargs)
                        # If there are *any* named groups, ignore all non-named groups.
                        # Otherwise, pass all non-named arguments as positional arguments.
                        sub_match_args = sub_match.args
                        if not sub_match_dict:
                            sub_match_args = args + sub_match.args
                        return ResolverMatch(
                            sub_match.func,
                            sub_match_args,
                            sub_match_dict,
                            sub_match.url_name,
                            [self.app_name] + sub_match.app_names,
                            [self.namespace] + sub_match.namespaces,
                        )
                    tried.append([pattern])
            raise Resolver404({'tried': tried, 'path': new_path})
        raise Resolver404({'path': path})

```
