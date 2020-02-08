## 起步

正如我们所知，`django` 根据 `manage.py` 这个文件来提供很多命令来执行。

例如：`runserver`，`startapp`，`makemigrations`，`migrate` ...

那么这些命令是如何对应执行的（这里我们不具体叙述具体实现的功能项）。

## 开始

我们先看下 `manage.py` 这个文件（本质上执行这些命令就是执行的是 `manage.py` 这个文件）

```python

# manage.py

import os
import sys

if __name__ == "__main__":
    # 设置环境变量(主要是配置文件的)
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "proj.settings")
    try:
        from django.core.management import execute_from_command_line
    except ImportError:
        # 这里主要就是导入不成功，抛出 django 未安装的错误啥的
        ...

    # 执行这个函数，命令行参数作为参数传入
    execute_from_command_line(sys.argv)

# django/core/management

def execute_from_command_line(argv=None):
    """执行命令"""

    # 实例化
    utility = ManagementUtility(argv)
    # 执行命令
    utility.execute()

# django/core/management

class ManagementUtility:
    """封装了 django-admin 和 manage.py 的逻辑

    """
    def __init__(self, argv=None):
        self.argv = argv or sys.argv[:]
        self.prog_name = os.path.basename(self.argv[0])
        if self.prog_name == '__main__.py':
            self.prog_name = 'python -m django'
        self.settings_exception = None

    def execute(self):
        """给定命令行参数，找出命令，给这个命令创建一个解析器并运行它.

        """

        # 如果没有命令参数，则默认展示帮助
        try:
            subcommand = self.argv[1]
        except IndexError:
            subcommand = 'help'  

        # 预处理提取 --settings 和 --pythonpath 这俩项配置.
        # 这些参数会影响命令的可用性，因此它们必须要提前处理，当然了你不传也没任何影响.
        parser = CommandParser(usage='%(prog)s subcommand [options] [args]', add_help=False, allow_abbrev=False)
        parser.add_argument('--settings')
        parser.add_argument('--pythonpath')
        parser.add_argument('args', nargs='*')  # catch-all
        try:
            # 处理已知的参数
            options, args = parser.parse_known_args(self.argv[2:])
            # 处理默认参数
            handle_default_options(options)
        except CommandError:
            pass  # 此时忽略任何选项错误.

        try:
            # 检测 INSTALLED_APPS 是否访问正常
            settings.INSTALLED_APPS
        except ImproperlyConfigured as exc:
            self.settings_exception = exc
        except ImportError as exc:
            self.settings_exception = exc

        # 这一项配置何时为 true 的似乎不得而知, 忽略以下代码
        if settings.configured:
            ...


        self.autocomplete()

        # 如果子命令为 help
        if subcommand == 'help':
            if '--commands' in args:
                sys.stdout.write(self.main_help_text(commands_only=True) + '\n')
            elif not options.args:
                sys.stdout.write(self.main_help_text() + '\n')
            else:
                self.fetch_command(options.args[0]).print_help(self.prog_name, options.args[0])
        # 特殊情况，期望 django-admin 相关命令向后兼容
        elif subcommand == 'version' or self.argv[1:] == ['--version']:
            sys.stdout.write(django.get_version() + '\n')
        # 如果涵盖有 --help -h 等相关查看帮助信息的
        elif self.argv[1:] in (['--help'], ['-h']):
            sys.stdout.write(self.main_help_text() + '\n')
        # 如果是其它命令
        else:
            # 查找命令并运行
            self.fetch_command(subcommand).run_from_argv(self.argv)

```

以上主要通过执行 `manage.py` 文件并执行 `execute_from_command_line` 函数，把 `sys.argv` 上获得的参数作为参数传入该函数，`execute_from_command_line` 下，`ManagementUtility` 实例化，并调用 `execute` 函数，该功能内部针对一些特殊的情况进行了预处理，最后解析命令并执行。

相关逻辑继续向下看。

## 深究

上面除了针对特殊情况的处理之外，核心主要在 `self.fetch_command(subcommand).run_from_argv(self.argv)` 这里。

这是一个链式操作，接连调用了俩个方法。

我们先从 `fetch_command` 开始。

```python

def fetch_command(self, subcommand):
    # 获取所有相关的命令
    # 主要是通过文件查找获取每个 app 下的 management 文件夹下所有的文件
    # 是一个字典集合，key 为子命令名称，value 为 app 的名称
    commands = get_commands()
    try:
        # 通过字典映射到值
        app_name = commands[subcommand]
    except KeyError:
        # 如果发生异常，退出并进行提示 (并根据当前错误命令，提示与其相关的命令)
        # 这里用到名为 difflib 模块(标准库下的)
        ...

    # 如果命令已经加载，则直接使用它
    if isinstance(app_name, BaseCommand):
        klass = app_name
    else:
        # 一般会走这里，加载命令类
        # 传入俩个参数，app 名称 和 命令名称
        klass = load_command_class(app_name, subcommand)
    # 这里响应获取到一个 Command 实例，实则继承于 BaseCommand
    return klass

# 加载命令类
def load_command_class(app_name, name):
    # 从这里我们得知，如果我们需要自定义一些命令，需要在 app 下创建 management/commands 文件夹，命令名称为文件名
    module = import_module('%s.management.commands.%s' % (app_name, name))
    # 动态导入该模块后，调用模块下的 Command 类并实例化（这里有也说明这个类的名称只能叫作 Command）
    return module.Command()

```

> django/core/management/base 下的 BaseCommand

接下来，链式调用到了 `run_from_argv`，调用者的基类为 `BaseCommand`

我们找到 `BaseCommand` 下的 `run_from_argv` 函数，并把命令行参数传递给该函数。

```python
def run_from_argv(self, argv):
    # 以下为参数解析
    self._called_from_command_line = True
    parser = self.create_parser(argv[0], argv[1])

    options = parser.parse_args(argv[2:])
    cmd_options = vars(options)
    # Move positional args out of options to mimic legacy optparse
    args = cmd_options.pop('args', ())
    # 处理默认参数
    handle_default_options(options)
    try:
        # 执行
        self.execute(*args, **cmd_options)
    except Exception as e:
        # 执行出现异常打印相关信息
        ...
    finally:
        try:
            # 关闭数据库连接
            connections.close_all()
        except ImproperlyConfigured:
            ...

def execute(self, *args, **options):
    # 设置颜色等等相关
    if options['no_color']:
        self.style = no_style()
        self.stderr.style_func = None
    if options.get('stdout'):
        self.stdout = OutputWrapper(options['stdout'])
    if options.get('stderr'):
        self.stderr = OutputWrapper(options['stderr'], self.stderr.style_func)

    # 检查一些相关选项
    if self.requires_system_checks and not options.get('skip_checks'):
        self.check()
    if self.requires_migrations_checks:
        self.check_migrations()

    # 这里调用了 handle 方法
    # 也就是说自己定义的命令，需要实现 handle 方法
    # 这个方法内实现你自己的逻辑
    output = self.handle(*args, **options)

    if output:
        ...

    return output
```

## 总结

通过上述表述，我们知晓了 `runserver`, `startapp` 等命令是如何进行执行的。

首先从 `manage.py` 这个文件开始，执行 `execute_from_command_line` 函数（`sys.argv` 作为参数），然后调用 `ManagementUtility` 类进行实例化，该实例调用 `execute` 方法。

这里主要进行了一些参数的预处理，然后调用 `fetch_command(subcommand)` 方法，这个先找到所有可用的命令，根据 `subcommand` 参数获取到一个基类为 `BaseCommand` 的一个实例，然后这个实例调用 `run_from_argv` 方法（该方法位于 `BaseCommand` 中），然后调用 `execute` 方法，该方法下又调用 `handle` 方法来实现自己的逻辑。

以上就是 `django` 所实现的命令调用。

## 扩展

- `django` 自定义命令实现

通过上述源码解析，大概我们了解到如何实现自己定义的命令。

在 `app` 下创建 `management/commands` 文件夹（这个是必须的），文件名称你可以随便定义，例如 `custom_command.py`，那么调用的时候就是 `python manage.py custom_command`

然后在 `custom_command.py` 文件下定义一个 `Command` 类，并继承 `BaseCommand`（处于 `django/core/management/base` 下），然后类下实现一个 `handle` 方法（必须的）（实现你自己的逻辑）

- `difflib` 模块

在我们输入错误的命令的时候，比如执行 `python manage.py runserver` ，假设 `runserver` 打成了 `runservev`，这里会提示你是否是打出 `runserver` 这个单词。

这里 `django` 使用的 python 标准库下的 `difflib` 模块来达到这个目的。

该模块主要用于文本之间的对比等操作。例如我们举个例子。

```python
>>> get_close_matches("appel", ["ape", "apple", "peach", "puppy"])
 ['apple', 'ape']
 >>> import keyword as _keyword
 >>> get_close_matches("wheel", _keyword.kwlist)
 ['while']
 >>> get_close_matches("Apple", _keyword.kwlist)
 []
 >>> get_close_matches("accept", _keyword.kwlist)
 ['except']
```

这个只是这个模块下的其中的一个方法（`django` 也是使用的它）。

更多我们可以了解官方文档，详情 -> [请戳这里](https://docs.python.org/3.8/library/difflib.html)。
