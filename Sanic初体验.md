---
title: Sanic初体验
tags: 
  - sanic
  - web
categories:
  - sanic
---
<!-- more -->
### 基本使用

```python
# -*- encoding: utf-8 -*-
# @Time       : 2018/8/5 下午3:39
# @Author     : Hulk Wu
# @File       : index.py
# @Description:

from sanic import Sanic
from sanic.response import text
from sanic.response import redirect


app = Sanic()


@app.route('/async')
async def async_hello(request):
    return text('Hello, async sanic')


@app.route('/sync')
def sync_hello(request):
    return text("Hello, sync sanic")


@app.route('/json')
async def json_hello(request):
    from sanic.response import json
    return json({"res": "hello, json"})


# 参数

@app.route('/tag/<tag>')
def tag_handler(request, tag):
    return text('Tag: {}'.format(tag))


@app.route('/tag/number/<number:int>')
# number和int之间不能有空格, route内参数不允许有多余的空格
def tag_number_handler(request, number):
    return text('Tag, number: {}'.format(number))


@app.route('/tag/rex/<rex:\w+>')
def tag_rex_handler(request, rex):
    return text('Tag, rex: {}'.format(rex))


@app.route('/tag/wrong/<wrong:a>')
def tag_wrong_handler(request, wrong):
    return text('Tag, wrong: {}'.format(wrong))


@app.route('/tag/limit/<limit:[A-z0-9]{0,4}>')
def tag_limit_handler(request, limit):
    return text('Tag, limit: {}'.format(limit))


@app.route('/url_for')
def url_for_handler(request):
    res = app.url_for('json_hello')
    return redirect(res)


# 中间件
@app.route('/middleware')
def middleware_handler(request):
    print("this is middleware test function")
    return text("middleware")


@app.middleware('response')
def modify_response(request, response):
    print("modify response")


@app.middleware('request')
def modify_request1(request):
    print("modify request 1")


@app.middleware('request')
def modify_request2(request):
    # 在modify_request1 后调用
    print("modify request 2")


@app.middleware('test')
def modify_test_request(request):
    # 不会被调用
    print("test middleware")


# 监听器

@app.listener('before_server_start')
def bss(app, loop):
    print('before_server_start')


@app.listener('after_server_start')
def ass(app, loop):
    print('after_server_start')


@app.listener('before_server_stop')
def bs_stop(app, loop):
    print('before_server_stop')


@app.listener('after_server_stop')
def af_stop(app, loop):
    print('after_server_stop')


async def task():
    import asyncio
    await asyncio.sleep(5)
    print('sleep 5s')


if __name__ == "__main__":
    app.add_task(task()) 
    app.run(host="0.0.0.0", port=9000)
```

### 基于类的视图


```python
# -*- encoding: utf-8 -*-
# @Time       : 2018/8/5 下午8:05
# @Author     : Hulk Wu
# @File       : views.py
# @Description:

from sanic import Sanic
from sanic.views import HTTPMethodView
from sanic.response import text


app = Sanic()


# @app.route('/views') 不能这样用
class SimpleView(HTTPMethodView):

    def get(self, request):
        return text('This is get method')


if __name__ == "__main__":
    app.add_route(SimpleView.as_view(), '/views')
    app.run(host="0.0.0.0", port=9000)
```


### 蓝图

##### my blueprint.py


```python
# -*- encoding: utf-8 -*-
# @Time       : 2018/8/5 下午7:07
# @Author     : Hulk Wu
# @File       : myblueprint.py
# @Description:

from sanic import Sanic
from bp1 import bp1
from bp2 import bp2

app = Sanic(__name__)

app.blueprint(bp1)
app.blueprint(bp2)

app.run(host="0.0.0.0", port=9000)
```


##### bp1.py


```python
# -*- encoding: utf-8 -*-
# @Time       : 2018/8/5 下午8:53
# @Author     : Hulk Wu
# @File       : bp1.py
# @Description:


from sanic import Blueprint
from sanic.response import text


bp1 = Blueprint("first_bp")


@bp1.route('/test1')
def test(request):
    return text('this is bp1 test')
```


##### bp2.py


```python
# -*- encoding: utf-8 -*-
# @Time       : 2018/8/5 下午8:53
# @Author     : Hulk Wu
# @File       : bp2.py
# @Description:


from sanic import Blueprint
from sanic.response import text


bp2 = Blueprint("second_bp")


@bp2.route('/test2')
def test(request):
    return text('this is bp2 test')
```


