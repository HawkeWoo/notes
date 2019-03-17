---
title: aiohttp 客户端初体验
tags: 
  - aiohttp
  - 异步
categories:
  - aiohttp
---
<!-- more -->
#### 一个小例子

```python
import aiohttp
import asyncio

url_github = 'https://api.github.com/events'


async def test():
    async with aiohttp.ClientSession() as session:
        async with session.get(url_github) as resp:
            print(resp.status)
            print(await resp.text())


if __name__ == '__main__':
    coro = test()
    loop = asyncio.get_event_loop()
    task = asyncio.ensure_future(coro)
    loop.run_until_complete(task)
```

`ClientSession` 可以使用如下coroutine（协程）来发送http请求：

```python
session.post('http://httpbin.org/post', data=b'data')
session.put('http://httpbin.org/put', data=b'data')
session.delete('http://httpbin.org/delete')
session.head('http://httpbin.org/get')
session.options('http://httpbin.org/get')
session.patch('http://httpbin.org/patch', data=b'data')
```

**Note**：

不要对每个请求都创建一个session。大部分情况下只需要对每个应用创建一个session来负责所有的请求。一个session内部包含了一个连接池， Connection reusage 和 keep-alives  都是默认设置的。

#### 传递参数

```python
params = {'key1': 'value1', 'key2': 'value2'}
async with session.get('http://httpbin.org/get', params=params) as resp:
    ...
```

#### 响应内容和状态码

状态码：`resp.status`

响应文本：`await resp.text()`

二进制：`await resp.read()`

JSON: `await resp.json()`

#### JSON请求

传递json参数：

```python
async with aiohttp.ClientSession() as session:
    async with session.post(url, json={'test': 'object'})
```

指定其他json序列化器：

```python
import ujson

async with aiohttp.ClientSession(
        json_serialize=ujson.dumps) as session:
    await session.post(url, json={'test': 'object'})
```

#### 流响应

`read()`, `json()`, `text()`这些方法会在内存中加载整个响应，在下载大文件下直接加载所有响应可能不适合。

```python
async def download(url, filename, chunk_size=10):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            with open(filename, 'wb') as fd:
                while True:
                    chunk = await resp.content.read(chunk_size)
                    if not chunk:
                        break
                    fd.write(chunk)


if __name__ == '__main__':
    coro = download(url_github, 'github.txt')
    loop = asyncio.get_event_loop()
    task = asyncio.ensure_future(coro)
    loop.run_until_complete(task)
```

#### 更多的POST请求

当给post请求传递字典参数时，字典参数会自动转化为form：

```python
payload = {'key1': 'value1', 'key2': 'value2'}
async with session.post('http://httpbin.org/post',
                        data=payload) as resp:
    print(await resp.text())
```

```python
{
  ...
  "form": {
    "key2": "value2",
    "key1": "value1"
  },
  ...
}
```

如果不想传递form表单，可以通过传递二进制的参数，参数会直接post并且content-type 会默认设置为`application/octet-stream` 

```python
async with session.post(url, data=b'\x00Binary-data\x00') as resp:
```

`json`参数

```python
async with session.post(url, json={'example': 'test'}) as resp:
    ...
```

`text`参数

```python
async with session.post(url, text='Тест') as resp:
    ...
```

#### 文件上传

```python
async def upload(url, filename):
    async with aiohttp.ClientSession() as session:
        files = {'file': open(filename, 'rb')}
        async with session.post(url, data=files) as resp:
            print(resp.status)
            print(await resp.text())


if __name__ == '__main__':
    coro = upload('http://httpbin.org/post', 'github.txt')
    loop = asyncio.get_event_loop()
    task = asyncio.ensure_future(coro)
    loop.run_until_complete(task)
```

或者使用`aiohttp.FormData()`

```python
async def upload(url, filename):
    async with aiohttp.ClientSession() as session:
        data = aiohttp.FormData()
        data.add_field('file',
                       open(filename, 'rb'),
                       filename=filename,
                       content_type='text/plain')
        async with session.post(url, data=data) as resp:
            print(resp.status)
            print(await resp.text())
```

#### WebSockets

必须使用`aiohttp.ClientSession.ws_connect()`协程来进行websocket连接。它接受一个url参数并返回`ClientWebSocketResponse`，通过这个对象可以响应的方法和websocket服务端进行通信。

#### 超时

超时可以通过`ClientTimeout`来设置，aiohttp默认5min超时，意味着所有的操作都应该在5min内完成。

默认的timeout设置为：

```python
aiohttp.ClientTimeout(total=5*60, connect=None, sock_connect=None, sock_read=None)
```

超时值可以通过重写session的timeout参数或者请求的timeout参数来改变。

```python
timeout = aiohttp.ClientTimeout(total=60)
async with aiohttp.ClientSession(timeout=timeout) as session:
    ...
```

```python
async with session.get(url, timeout=timeout) as resp:
    ...
```

参数说明：

`total`：整个操作时间包括连接的建立，请求的发送和响应的读取。

`connect`: 从连接池获取连接的总超时，这个时间组成包括：一个新连接的建立或者当超过连接池连接数量限制时从连接池等待空闲连接的时间。

`sock_connect`: 一个新连接建立的时间，不包括从连接池获取空闲连接这种情况。

`sock_read`: 读取新数据的最大允许时间。

所有的字段均为`floates`，None和0都禁用特定的超时设置。