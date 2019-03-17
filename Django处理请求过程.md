---
title: Django处理请求过程
tags: 
  - Django
  - web
categories:
  - Django
---
## 一.wsgi,uWSGI,uwsgi概念
### wsgi
wsgi是一种实现python解析的通用接口标准/协议。wsgi将web组件分成了三类：web服务器，web中间件，web应用
### uWSGI
uWSGI是一种python web server，负责响应python的web请求。
### uwsgi
uwsgi是一种uWSGI服务器的协议。

## 二.浏览器通过url发起请求
Django的请求到响应的流程，简单的来说就是利用wsgi，当用户发来一个request进行response，响应前发送request_started信号，经过中间件的process_request，响应完成后会调用中间件的process_response。

#### Projct/wsgi.py

```python

import os

from django.core.wsgi import get_wsgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "Project.settings")

application = get_wsgi_application()
```

#### django/core/wsgi.py
```python
import django
from django.core.handlers.wsgi import WSGIHandler


def get_wsgi_application():
    """
    The public interface to Django's WSGI support. Should return a WSGI
    callable.

    Allows us to avoid making django.core.handlers.WSGIHandler public API, in
    case the internal WSGI implementation changes or moves in the future.
    """
    django.setup(set_prefix=False)
    return WSGIHandler()
```

#### django/core/handlers/wsgi.py
```python
class WSGIHandler(base.BaseHandler):
    request_class = WSGIRequest

    def __init__(self, *args, **kwargs):
        super(WSGIHandler, self).__init__(*args, **kwargs)
        self.load_middleware()

    def __call__(self, environ, start_response):
        set_script_prefix(get_script_name(environ))
        signals.request_started.send(sender=self.__class__, environ=environ)
        try:
            request = self.request_class(environ)
        except UnicodeDecodeError:
            logger.warning(
                'Bad Request (UnicodeDecodeError)',
                exc_info=sys.exc_info(),
                extra={
                    'status_code': 400,
                }
            )
            response = http.HttpResponseBadRequest()
        else:
            response = self.get_response(request)

        response._handler_class = self.__class__

        status = '%d %s' % (response.status_code, response.reason_phrase)
        response_headers = [(str(k), str(v)) for k, v in response.items()]
        for c in response.cookies.values():
            response_headers.append((str('Set-Cookie'), str(c.output(header=''))))
        start_response(force_str(status), response_headers)
        if getattr(response, 'file_to_stream', None) is not None and environ.get('wsgi.file_wrapper'):
            response = environ['wsgi.file_wrapper'](response.file_to_stream)
        return response
```

#### django/core/handlers/base.py
```python
class BaseHandler(object):

    def __init__(self):
        self._request_middleware = None
        self._view_middleware = None
        self._template_response_middleware = None
        self._response_middleware = None
        self._exception_middleware = None
        self._middleware_chain = None

    def load_middleware(self):
        """
        Populate middleware lists from settings.MIDDLEWARE (or the deprecated
        MIDDLEWARE_CLASSES).

        Must be called after the environment is fixed (see __call__ in subclasses).
        """
        self._request_middleware = []
        self._view_middleware = []
        self._template_response_middleware = []
        self._response_middleware = []
        self._exception_middleware = []
        ...
        ...
        ...
```

当Web应用启动后，会生成一个WSGIHandler的实例，其继承自BaseHandler，然后会加载相应的middleware。在settings里可以通过WSGI_APPLICATION来设置。一个middleware类可以包括请求响应过程的四个阶段：request，view，response和exception。对应的方法：process_request，process_view， process_response和process_exception。

## 三.Django中间件
Django中间件定义在project/settings.py的MIDDLEWARE中，用来处理HTTP的request和response，和插件的作用有点相似，当用户请求到达的时候，会按照MIDDLEWARE的顺序依次经过中间件，执行process_request，然后到达views中，经过views处理后，又按照MIDDLEWARE的逆序，即原路返回，依次穿过中间件执行process_response，最后返回，如下图所示，其中会包含process_view和process_exception（process_request和process_view是顺序访问，process_exception和process_response是逆序访问，当中间某个中间件不符合条件返回的时候，会把请求发给相对应的中间件，然后依次向上返回，不会继续向下执行process_request）

#### 参考
[区分wsgi、uWSGI、uwsgi、php-fpm、CGI、FastCGI的概念](https://www.jianshu.com/p/2b6900358646)

[Django进阶之中间件](https://www.cnblogs.com/zhaof/p/6281541.html)

