## 初试 - 文件变化后 `server` 自动重启

> 在此之前，不妨先了解下 `django` 是如何做到自动重启的

### 开始

`django` 使用 `runserver` 命令的时候，会启动俩个进程。

`runserver` 主要调用了 `django/utils/autoreload.py` 下 `main` 方法。  
*至于为何到这里的，我们这里不作详细的赘述，后面篇章会进行说明。*

主线程通过 `os.stat` 方法获取文件最后的修改时间进行比较，继而重新启动 `django` 服务（也就是子进程）。

大概每秒监控一次。

```python
# django/utils/autoreload.py 的 reloader_thread 方法

def reloader_thread():
    ...
    # 监听文件变化
    # -- Start
    # 这里主要使用了 `pyinotify` 模块，因为目前可能暂时导入不成功，使用 else 块代码
    # USE_INOTIFY 该值为 False
    if USE_INOTIFY:
        fn = inotify_code_changed
    else:
        fn = code_changed
    # -- End
    while RUN_RELOADER:
        change = fn()
        if change == FILE_MODIFIED:
            sys.exit(3)  # force reload
        elif change == I18N_MODIFIED:
            reset_translations()
        time.sleep(1)
```

`code_changed` 根据每个文件的最好修改时间是否发生变更，则返回 `True` 达到重启的目的。

### 父子进程&多线程

关于重启的代码在 `python_reloader` 函数内

```python

# django/utils/autoreload.py

def restart_with_reloader():
    import django.__main__
    while True:
        args = [sys.executable] + ['-W%s' % o for o in sys.warnoptions]
        if sys.argv[0] == django.__main__.__file__:
            # The server was started with `python -m django runserver`.
            args += ['-m', 'django']
            args += sys.argv[1:]
        else:
            args += sys.argv
        new_environ = {**os.environ, 'RUN_MAIN': 'true'}
        exit_code = subprocess.call(args, env=new_environ)
        if exit_code != 3:
            return exit_code


def python_reloader(main_func, args, kwargs):
    # 一开始环境配置是没有该变量的，所有走的是 else 语句块
    if os.environ.get("RUN_MAIN") == "true":
        # 开启一个新的线程启动服务
        _thread.start_new_thread(main_func, args, kwargs)
        try:
            # 程序接着向下走，监控文件变化
            # 文件变化，退出该进程，退出码反馈到了 subprocess.call 接收处...
            reloader_thread()
        except KeyboardInterrupt:
            pass
    else:
        try:
            # 而在 restart_with_reloader 这个函数设置了 RUN_MAIN 变量
            exit_code = restart_with_reloader()
            if exit_code < 0:
                os.kill(os.getpid(), -exit_code)
            else:
                sys.exit(exit_code)
        except KeyboardInterrupt:
            pass
```

程序启动，因为没有 `RUN_MAIN` 变量，所以走的 else 语句块。

颇为有趣的是，`restart_with_reloader` 函数中使用 `subprocess.call` 方法执行了启动程序的命令（ e.g：python3 manage.py runserver ），此刻 `RUN_MAIN` 的值为 `True` ，接着执行 `_thread.start_new_thread(main_func, args, kwargs)` 开启新线程，意味着启动了 `django` 服务。

如果子进程不退出，则停留在 `call` 方法这里（进行请求处理），如果子进程退出，退出码不是3，while 则被终结。反之就继续循环，重新创建子进程。

### 检测文件修改

具体检测文件发生改变的函数实现。

```python

# django/utils/autoreload.py

def code_changed():
    global _mtimes, _win
    # 获取所有文件
    for filename in gen_filenames():
        # 通过 os 模块查看每个文件的状态
        stat = os.stat(filename)
        # 获取最后修改时间
        mtime = stat.st_mtime
        if _win:
            mtime -= stat.st_ctime
        if filename not in _mtimes:
            _mtimes[filename] = mtime
            continue
        # 比较是否修改
        if mtime != _mtimes[filename]:
            _mtimes = {}
            try:
                del _error_files[_error_files.index(filename)]
            except ValueError:
                pass
            return I18N_MODIFIED if filename.endswith('.mo') else FILE_MODIFIED
    return False
```

### 总结

以上就是 `django` 检测文件修改而达到重启服务的实现流程。

结合 `subprocess.call` 和 环境变量 创建俩个进程。主进程负责监控子进程和重启子进程。
子进程下通过开启一个新线程（也就是 `django` 服务）。主线程监控文件变化，如果变化则通过 `sys.exit(3)` 来退出子进程，父进程获取到退出码不是3则继续循环创建子进程，反之则退出整个程序。

好，到这里。我们勇敢的迈出了第一步，我们继续下一个环节！！！ ヾ(◍°∇°◍)ﾉﾞ
