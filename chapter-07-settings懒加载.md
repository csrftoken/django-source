## 起步

在之前几篇的描述中，在 `django` 的配置文件 `settings` 也能看到一些配置的引用。例如 `INSTALLED_APPS`、`MIDDLEWARE`、`ROOT_URLCONF` 等配置。

那么这个配置模块是如何实现的呢？

## 开始

一般我们引用 django 配置推荐这样引入

```python
from django.conf import settings
```

而不是

```python
from proj import settings
```

如果使用第二种形式的话，`settings` 中引用一些其它模块的话，那么会有可能造成循环引用。使用第一种形式的话，django 使用的是**懒加载**机制（用到的时候才加载）。

详情我们可以通过源码去查看。

## 了解

```python

settings = LazySettings()

class LazySettings(LazyObject):
    def _setup(self, name=None):
        # 从环境变量中提取 settings 模块路径
        settings_module = os.environ.get(ENVIRONMENT_VARIABLE)
        self._wrapped = Settings(settings_module)

    def __getattr__(self, name):
        """返回设置的值并缓存在 __dict__ 中"""
        if self._wrapped is empty:
            self._setup(name)
        val = getattr(self._wrapped, name)
        self.__dict__[name] = val
        return val

    def __setattr__(self, name, value):
        if name == '_wrapped':
            self.__dict__.clear()
        else:
            self.__dict__.pop(name, None)
        super().__setattr__(name, value)

    ... # 省略其它函数
```

所谓懒加载，就是需要的时候才去加载。django 通过代理类 `LazyObject` 实现这个机制。加载函数是 `_setup`，当获取属性时才去加载，并缓存至实例的 `__dict__` 中。

`LazySettings` 继承了 `LazyObject`，重写了 `__setattr__` 和 `__getattr__`，假设调用 `settings.DEBUG` 属性时，会调用 `__getattr__` 方法实现。

自此，我们可以观察到，所有的属性都是从 `_wrapped` （也就是 `Settings(settings_module)` 实例）这个私有属性中获取到的。

## 配置加载

上述我们了解到从环境变量中提取 `settings` 模块的路径，继而 `_wrapped` 属性指向 `Settings` 这个类的实例。

```python
class Settings:
    def __init__(self, settings_module):
        # 读取默认配置
        for setting in dir(global_settings):
            if setting.isupper():
                setattr(self, setting, getattr(global_settings, setting))

        # 配置模块
        self.SETTINGS_MODULE = settings_module

        # 动态导入
        mod = importlib.import_module(self.SETTINGS_MODULE)

        tuple_settings = (
            "INSTALLED_APPS",
            "TEMPLATE_DIRS",
            "LOCALE_PATHS",
        )
        self._explicit_settings = set()
        # 读取配置模块下的属性（可能会覆盖一些默认配置）
        for setting in dir(mod):
            if setting.isupper():
                setting_value = getattr(mod, setting)
                setattr(self, setting, setting_value)
                self._explicit_settings.add(setting)
```

## 总结

综上，我们在读取 `settings` 某个配置时，会触发 `__getattr__` 方法，如果 `_wrapped` 为空，则调用 `_setup` 方法，这个方法内部获取配置文件模块，`_wrapped` 属性指向 `Settings` 类的实例，这个类在实例化的时候，构造函数先读取 `global_settings` 来设置一些默认属性，接着通过动态导入模块的形式 `importlib.import_module` 加载配置模块的属性，继而读取的属性从 `_wrapped` 中获取。
