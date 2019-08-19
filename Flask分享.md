# Flask

[TOC]

## Hello World

`demo.py`

```python
from flask import Flask
app = Flask(__name__)	# 实例化Flask应用


# 视图函数
@app.route('/')
def hello():
    return 'Hello World'


if __name__ == '__main__':
    app.run()	# 让应用运行在本地服务器上
```

这是一个最小的 Flask 应用，运行python demo.py然后访问[http://127.0.0.1:5000/](http://127.0.0.1:5000/)即可看到Hello World的响应。

## 蓝图

Flask 用 *蓝图（blueprints）* 的概念来在一个应用中或跨应用制作应用组件和支持通用的模式。蓝图很好地简化了大型应用工作的方式，并提供给 Flask 扩展在应用上注册操作的核心方法。

- 把一个应用分解为一个蓝图的集合。这对大型应用是理想的。一个项目可以实例化一个应用对象，初始化几个扩展，并注册一集合的蓝图。
- 以 URL 前缀和/或子域名，在应用上注册一个蓝图。 URL 前缀/子域名中的参数即成为这个蓝图下的所有视图函数的共同的视图参数（默认情况下）。
- 在一个应用中用不同的 URL 规则多次注册一个蓝图。
- 通过蓝图提供模板过滤器、静态文件、模板和其它功能。一个蓝图不一定要实现应用或者视图函数。
- 初始化一个 Flask 扩展时，在这些情况中注册一个蓝图。

总的来说，当我们的项目越来越臃肿的时候，需要通过蓝图来使我们的项目架构更加清晰。

```shell
.
├── app1
│   ├── __init__.py
│   ├── models.py
│   └── views.py
├── app2
│   ├── __init__.py
│   ├── models.py
│   └── views.py
├── app3
│   ├── __init__.py
│   ├── models.py
│   └── views.py
├── demo.py
├── main.py
└── requirements.txt

```

主要py文件：

```python
# /app01/views.py
from flask import Blueprint

app1 = Blueprint('app1', __name__)


@app1.route('/')
def show():
    return 'app1.hello'
```

```python
# /app02/views.py
from flask import Blueprint

app2 = Blueprint('app2', __name__)


@app2.route('/')
def show():
    return 'app2.hello'
```

```python
# /app03/views.py
from flask import Blueprint

app3 = Blueprint('app3', __name__)


@app3.route('/')
def show():
    return 'app3.hello'
```

```python
# /main.py
from flask import Flask

from app1.views import app1
from app2.views import app2
from app3.views import app3

app = Flask(__name__)
app.register_blueprint(app1, url_prefix='/app1')
app.register_blueprint(app2, url_prefix='/app2')
app.register_blueprint(app3, url_prefix='/app3')
application = app

if __name__ == '__main__':
    app.run()
```

## MTV

和Django一样，Flask同样可以用MTV模式来开发web服务。

MTV ( Model–Template–View )，一般是用户通过浏览器向我们的服务器发起一个请求(request)，这个请求会去访问视图函数，（如果不涉及到数据调用，那么这个时候视图函数返回一个模板也就是一个网页给用户），视图函数调用模型，模型去数据库查找数据，然后逐级返回，视图函数把返回的数据填充到模板中，最后返回网页内容给用户。

### Model

#### Peewee

[Peewee](http://docs.peewee-orm.com/en/latest/index.html)是一个简单小巧的Python ORM，它非常容易学习，并且使用起来很直观，和Django ORM的风格很相似。

安装

```shell
$ pip install pymysql 
$ pip install peewee
```

```python
# app1/models.py
import peewee
from settings import db_config

mysql_db = peewee.MySQLDatabase(db_config['database'],
                                user=db_config['user'],
                                password=db_config['password'],
                                host=db_config['host'],
                                port=db_config['port'])


class Token(peewee.Model):

    id = peewee.IntegerField(primary_key=True)
    name = peewee.CharField(max_length=16)
    status = peewee.IntegerField()
    token = peewee.CharField(max_length=32)
    desc = peewee.TextField(null=True)

    class Meta:
        database = mysql_db
        table_name = 'market_token'
```

```python
# app1/manager.py
from .models import Token


def get_token(id):
    return Token.get(Token.id == id)
```

```python
# app1/views.py
@app1.route('/token/<id>')
def db_query(id):
    q = dm.get_token(id)
    resp = {
        'name': q.name,
        'status': q.status,
        'token': q.token,
        'desc': q.desc
    }
    return jsonify(resp)
```

### Template

Flask配备了Jinja2模板引擎，它和Django的模板模块非常相似，主要用于渲染页面和数据，把我们想要展示的效果返回给用户。而目前我们大多数都是前后端分离，后端只负责提供restful api，因此在这里不详细描述。

### View

客户端将请求发送给web服务器，web服务器再将请求发送给flask程序实例，程序实例需要知道每个url请求要运行哪些代码，所以需要建立一个 url 到 python函数的映射，处理url和函数之间的关系的程序就是路由。

```python
from flask import Flask, request, jsonify, url_for
app = Flask(__name__)


@app.route('/')
def hello():
    return 'Hello World'


@app.route('/index')
@app.route('/index2', methods=['GET'])
def index():
    # 如果路由地址是/index, 访问 /index/ 会404
    # 如果路由地址是/index/, 访问 /index 会重定向到/index/
    return 'Index Page'


@app.route('/param/str/<val>')
def show_val(val):
    print(type(val))
    return 'Val: {0}'.format(val)


@app.route('/param/int/<int:int_val>')
def show_int(int_val):
    return 'int_val: {0}'.format(int_val)


@app.route('/param/float/<float:float_val>')
def show_float(float_val):
    print(type(float_val))
    return 'float_val: {0}'.format(float_val)


@app.route('/query_param')
def get_query_params():
    a = request.args.get('a')
    b = request.args.get('b')
    return 'query_param: a={0}, b={1}'.format(a, b)


# 反向解析: 通过视图处理函数的名称自动生成视图处理函数的访问路径
# Flask提供了`url_for()`辅助函数，`url_for()`最简单的用法是以视图函数名作为参数，返回对应URL。
@app.route('/url_for_test/<val>')
def url_for_test(val):
    return url_for('show_val', val=val)
```

#### g对象

1.在flask中，有一个专门用来存储用户信息的g对象，g的全称的为global。
2.g对象在一次请求中的所有的代码的地方，都是可以使用的，其生命周期为一次请求。

#### 一些装饰器

- app.before_request :在请求收到之前绑定一个函数做一些事情。 
- app.after_request: 每一个请求之后绑定一个函数，如果请求没有异常。 
- app.teardown_request: 每一个请求之后绑定一个函数，即使遇到了异常。
- app.errorhandler：发生异常时触发其装饰的函数。

这些装饰器都有点类似Django的中间件，可以应用在日志打印，权限认证等场景。

## 部署

```ini
[uwsgi]
http = 0.0.0.0:port
chdir = 项目目录
home = python环境目录
wsgi-file = 主函数入口文件，例如上例中的main.py
processes = %k(cpu核数)
# 开启线程操作模式。你必须指定每个工作进程的线程数。
threads = 8	
# 当服务器退出的时候自动删除unix socket文件和pid文件。
vacuum = true	
# 启动主进程。
master = true
# 使进程在后台运行，并将日志打到指定的日志文件或者udp服务器
daemonize = /data/release/py-choice-proxy/log/daemon.log
pidfile = pid文件路径
```

```shell
# 启动服务
$ uwsgi —ini uwsgi_config.ini
```

