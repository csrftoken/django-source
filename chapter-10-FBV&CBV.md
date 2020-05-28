## 起步

`django` 是支持视图进行 函数式 或 类 进行编写的。也就是我们所说的 `FBV` 和 `CBV`。

那么在请求信息匹配到相应的路由之后，这俩种模式是作了如何的处理的呢？

## 开始

在 `chapter-04-请求来了` 这个章节中，我们大概了解到路由匹配成功之后，会构造一个类似于一个函数的东西进行调用。

那么 `FBV` 模式我们没话说，直接调用函数传参数即可。

那么 `CBV` 模式是如何进行分发的。

在路由映射中，我们是这么配置路由和视图的：

```python

from django.urls import path

from . import views

urlpatterns = [
    # FBV (函数式我们没什么可说的，直接就到达了函数的内部)
    path("fbv/", views.fbv_api)
    # CBV
    path("cbv/", views.CbvApi.as_view()),
]
```

## 潜入

配置 `CBV` 视图的时候，类调用了 `as_view()` 方法，我们来看看它干了什么？

```python

# 写 CBV 视图的时候，默认视图类要继承来自于 django 提供的 View 类
# 它来自于：from django.views.generic import View

class View:

    ...

    @classonlymethod
    def as_view(cls, **initkwargs):
        """请求、响应的主要入口点"""

        for key in initkwargs:
            # 这里进行了一些参数初始化预检, 不必关心
            ...

        # 声明一个视图函数
        def view(request, *args, **kwargs):
            self = cls(**initkwargs)
            if hasattr(self, 'get') and not hasattr(self, 'head'):
                self.head = self.get
            self.request = request
            self.args = args
            self.kwargs = kwargs
            return self.dispatch(request, *args, **kwargs)

        # 赋值属性
        view.view_class = cls
        view.view_initkwargs = initkwargs

        # 以下这俩个的调用我们不必太过于关心。
        # 更新包装函数，使其看起来与包装函数相似
        update_wrapper(view, cls, updated=())
        # and possible attributes set by decorators
        # like csrf_exempt from dispatch
        update_wrapper(view, cls.dispatch, assigned=())

        # 这个方法返回了 view 视图函数
        return view
```

到这里已然了解到，其实 `CBV` 最终得到的结果也是个函数。

## 了然于心

当路由匹配成功之后，`CBV` 模式映射到视图调用的时候，会调用 `view` 函数。

```python

def view(request, *args, **kwargs):
    # 这里的 cls 表示你的那个视图类
    self = cls(**initkwargs)
    # 如果视图实现了 get 方法 和没有实现 head 方法，head 方法表示 get 方法。
    # 似乎没什么卵用。
    if hasattr(self, 'get') and not hasattr(self, 'head'):
        self.head = self.get

    # 将一些属性赋值到类实例上面
    self.request = request
    self.args = args
    self.kwargs = kwargs
    # 调用视图类的 dispatch 方法，接着向下看 dispatch 方法做了什么
    return self.dispatch(request, *args, **kwargs)

```

```python

def dispatch(self, request, *args, **kwargs):
    # 映射请求方式的方法，就是查看你有木有实现 http 的请求方法
    # 如果没有，会提醒 http 方法不被允许的错误提示
    if request.method.lower() in self.http_method_names:
        handler = getattr(self, request.method.lower(), self.http_method_not_allowed)
    else:
        handler = self.http_method_not_allowed
    # 这里就进入你的视图逻辑
    return handler(request, *args, **kwargs)

```

## 总结

`django` 提供了 `FBV`  & `CBV` 俩种模式进行视图的编写。主要突出在了 `CBV` 上面，进行配置的时候使用了 `View.as_view()` 方法，`as_view` 返回了 `view` 函数，这样请求进来调用了 `view` 方法，继而又调用了类的 `dispatch` 方法，通过反射映射到请求方法。

`CBV` 的模式优势在于，它可以给我们提供了很多可扩展的方法，比如在 `dispatch` 处进行一些定制，或者视图的属性封装。
