## 起步

上节我们了解到，`django` 是如何进行处理请求的。

那么 `django` 是如何进行 `app` 模块导入的。

我们所知道的。

运行 `django` 应用分为 生产 和 本地开发 模式。一个通过 `wsgi.py` 来达到生产环境的运行，另一个通过 `manage.py runserver` 来达到本地开发的运行。

但是我们发现，这俩处入口均有 `django.setup()` 这一步。

*这俩点均有一些不同之处，但是整体功能相似。*

## 开始

我们从 `setup` 这个功能函数说起。

```python

def setup(set_prefix=True):
    """配置 logging 及应用注册。

    """
    # 这里的 apps 是一个单例模式，指向 django/apps/registry 下的 Apps 这个类
    from django.apps import apps
    from django.conf import settings
    from django.urls import set_script_prefix
    from django.utils.log import configure_logging

    # 从 settings 文件下读取 LOGGING_CONFIG 配置日志文件
    configure_logging(settings.LOGGING_CONFIG, settings.LOGGING)
    # 这里只针对开发模式下有效，生产环境忽略
    if set_prefix:
        set_script_prefix(
            '/' if settings.FORCE_SCRIPT_NAME is None else settings.FORCE_SCRIPT_NAME
        )
    # 重点其实在这里，从 settings 文件获取 INSTALLED_APPS 配置
    apps.populate(settings.INSTALLED_APPS)

```

接下来，我们移步到 `populate` 下的函数。

```python

def populate(self, installed_apps=None):
    """加载应用配置和模型

    导入每个应用程序模块，然后导入每个模型模块。
    它是线程安全的、幂等的，但不是可重入的。

    """
    if self.ready:
        return

    # 在初始化 WSGI 回调之前，在服务器上创建的线程可能会并行调用 `populate`
    # 此处设置了线程锁
    with self._lock:
        if self.ready:
            return

        # RLock 防止其他线程进入这个部分。下面的比较和设置操作是原子性的。
        if self.loading:
            # Prevent reentrant calls to avoid running AppConfig.ready()
            # methods twice.
            raise RuntimeError("populate() isn't reentrant")
        self.loading = True

        # 阶段 1: 初始化 app 配置和导入 app 模型
        for entry in installed_apps:
            if isinstance(entry, AppConfig):
                app_config = entry
            else:
                # 这里通过一个工厂方法创造出一个应用实例
                app_config = AppConfig.create(entry)
            if app_config.label in self.app_configs:
                raise ImproperlyConfigured(
                    "Application labels aren't unique, "
                    "duplicates: %s" % app_config.label)

            # 载入到 app_configs 配置
            # key 为 应用标签，值为 app 实例
            self.app_configs[app_config.label] = app_config
            app_config.apps = self

        # 检测重复的 app 名称，确保唯一性
        counts = Counter(
            app_config.name for app_config in self.app_configs.values())
        duplicates = [
            name for name, count in counts.most_common() if count > 1]
        if duplicates:
            raise ImproperlyConfigured(
                "Application names aren't unique, "
                "duplicates: %s" % ", ".join(duplicates))

        # 设置应用加载完成标识位
        self.apps_ready = True

        # 阶段 2: 导入模型模块
        for app_config in self.app_configs.values():
            app_config.import_models()

        # 清空缓存
        self.clear_cache()

        # 设置模型加载完成标识位
        self.models_ready = True

        # 阶段 3: 运行 app 下的 ready 方法（钩子方法）
        for app_config in self.get_app_configs():
            app_config.ready()

        # 初始化完成
        self.ready = True

```

## 总结

到此，我们在 `run` 一个 django 应用的时候，会调用 `django` 的 `setup` 方法，主要是载入所有的应用和模型。
