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
        # 重点来了 -> 接下来我们看它背后发生了什么？
        response = self.get_response(request)

        response._handler_class = self.__class__

        status = '%d %s' % (response.status_code, response.reason_phrase)
        response_headers = list(response.items())
        for c in response.cookies.values():
            response_headers.append(('Set-Cookie', c.output(header='')))
        start_response(status, response_headers)
        if getattr(response, 'file_to_stream', None) is not None and environ.get('wsgi.file_wrapper'):
            response = environ['wsgi.file_wrapper'](response.file_to_stream)
        return response
```
