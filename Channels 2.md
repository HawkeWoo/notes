---
title: Channels 2
tags: 
  - Django
  - Channels
  - WebSocket
categories:
  - Django
---
# Channels 2

[TOC]

`Channels 2.1.3`和以前用的还是有区别啊，特意翻了翻官方文档，做做笔记。BTW，文档写得很容易看懂，尽量去看官方的文档吧。

`Websocket`是一个持久化的协议 。和HTTP协议不一样的地方是：

`HTTP` 协议是一种无状态的、无连接的、单向的应用层协议，它采用了请求/响应模型，通信请求只能由客户端发起，服务端对请求做出应答处理。 

而`WebSocket` 连接允许客户端和服务器之间进行全双工通信，以便任一方都可以通过建立的连接将数据推送到另一端。`WebSocket` 只需要建立一次连接，就可以一直保持连接状态。这相比于轮询方式的不停建立连接显然效率要大大提高。 

`Django` 可以通过`Channels`实现`websocket`的相关功能。

[Channels 官方文档](https://channels.readthedocs.io/en/latest/index.html)

`requirements.txt`

```python
django==1.11
channels==2.1.3
channels-redis==2.3.0
```

## 一. 创建项目与应用

```shell
$ python3 django-admin.py startproject channels_example

$ cd channels_example

$ python3 manage.py startapp chat
```

## 二. 相关设置

在`channels_example/channels_example`目录下新建`routing.py`

`websocket`路由本身是ASGI应用，建议用`ProtocolTypeRouter` 作为项目的根应用，并在里面嵌套特定协议的路由。

相关链接：

[Routing](https://channels.readthedocs.io/en/latest/topics/routing.html)

`routing.py`

```python
from channels.auth import AuthMiddlewareStack
from channels.routing import ProtocolTypeRouter, URLRouter
import chat.routing

application = ProtocolTypeRouter({
    # (http->django views is added by default)
    'websocket': AuthMiddlewareStack(
        URLRouter(
            chat.routing.websocket_urlpatterns	# 稍后建立chat的routing.py
        )
    ),
})
```

`channels_example/settings.py`

```python
# 绑定应用
ASGI_APPLICATION = "channels_example.routing.application"

# 设定消息队列，选用redis
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [("localhost", 6379)],
        },
    },
}
```

`channels_example/urls.py`

```python
from django.conf.urls import url, include
from django.contrib import admin

urlpatterns = [
    url(r'^admin/', admin.site.urls),
    url(r'^chat/', include('chat.urls')),
]
```

## 三. chat应用

在`channels_example/chat`目录下新建`consumers.py`。

相关链接：

[Consumers](https://channels.readthedocs.io/en/latest/topics/consumers.html) 

[Channel Layers](https://channels.readthedocs.io/en/latest/topics/channel_layers.html)

```python
from channels.generic.websocket import AsyncWebsocketConsumer
import json

# 异步websocket处理类
class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        # scope 包含了很多Django view中的request对象信息
        self.room_name = self.scope['url_route']['kwargs']['room_name']
        self.room_group_name = 'chat_%s' % self.room_name
        # 将新的连接加入到群组
        await self.channel_layer.group_add(
            self.room_group_name,
            # Channels的Consumer会自动提供self.channel_layer and self.channel_name属性
            self.channel_name
        )
        await self.accept()

    async def disconnect(self, close_code):
        # 连接关闭时调用
        # 将关闭的连接从群组中移除
        await self.channel_layer.group_discard(
            self.room_group_name,
            self.channel_name
        )

    async def receive(self, text_data=None, bytes_data=None):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']
        # 信息群发
        await self.channel_layer.group_send(
            self.room_group_name,
            {
                'type': 'chat_msg',
                'message': message
            }
        )

    async def chat_msg(self, event):
        message = event['message']
        await self.send(text_data=json.dumps({
            'message': message
        }))
```

在`channels_example/chat`目录下新建`routing.py`

```python
from django.conf.urls import url

from . import consumers

websocket_urlpatterns = [
    url(r'^ws/chat/(?P<room_name>[^/]+)/$', consumers.ChatConsumer),
]
```

在`channels_example/chat`目录下新建`urls.py`

```python
from django.conf.urls import url

from . import views

urlpatterns = [
    url(r'^(?P<room_name>[^/]+)/$', views.room, name='room'),
]
```

`chat/views.py`

```python
from django.shortcuts import render

# Create your views here.
from django.utils.safestring import mark_safe
import json


def room(request, room_name):
    return render(request, 'chat/room.html', {
        'room_name_json': mark_safe(json.dumps(room_name))
    })
```

`channels_example/chat/templates/chat/room.html`

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8"/>
    <title>Chat Room</title>
</head>
<body>
    <textarea id="chat-log" cols="100" rows="20"></textarea><br/>
    <input id="chat-message-input" type="text" size="100"/><br/>
    <input id="chat-message-submit" type="button" value="Send"/>
</body>
<script>
    var roomName = {{ room_name_json }};

    var chatSocket = new WebSocket(
        'ws://' + window.location.host +
        '/ws/chat/' + roomName + '/');

    chatSocket.onmessage = function(e) {
        var data = JSON.parse(e.data);
        var message = data['message'];
        document.querySelector('#chat-log').value += (message + '\n');
    };

    chatSocket.onclose = function(e) {
        console.error('Chat socket closed unexpectedly');
    };

    document.querySelector('#chat-message-input').focus();
    document.querySelector('#chat-message-input').onkeyup = function(e) {
        if (e.keyCode === 13) {  // enter, return
            document.querySelector('#chat-message-submit').click();
        }
    };

    document.querySelector('#chat-message-submit').onclick = function(e) {
        var messageInputDom = document.querySelector('#chat-message-input');
        var message = messageInputDom.value;
        chatSocket.send(JSON.stringify({
            'message': message
        }));

        messageInputDom.value = '';
    };
</script>
</html>
```