---
title: selectors
tags: 
  - selectors
  - python
  - 网络编程
categories:
  - python
---
# Selectors

[TOC]

## 一. 网络编程基础

socket模块

```python
sk = socket(socket.AF_INET, socket.SOCK_STREAM, 0)
```

```
参数一：地址簇
socket.AF_INET      	表示IPV4(默认)
socket.AF_INET6     	表示IPV6
socket.AF_UNIX      	只能用于单一的Unix系统进程间的通信

参数二：类型
socket.SOCK_STREAM      流式socket for TCP（默认）
socket.SOCK_DGRAM       数据格式socket,for UDP
socket.SOCK_RAW         原始套接字，普通的套接字无法处理ICMP,IGMP等网络报文，可以通过IP_HDRINCL套接字选项由用户构造IP头
socket.SOCK_RDM         是一种可靠的UDP形式，即保证交付数据报但不保证顺序，SOCK_RDM用来提供对原始协议的低级访问，在需要执行某些特殊操作时使用，如发送ICMP报文，SOCK_RAM通常仅限于高级用户或管理员运行的程序使用socket.SOCK_SEQPACKET   可靠的连续数据包服务

参数三：协议
默认与特定地址家族相关的协议，如果是0 则系统就会根据地址格式和套接类别，自动选择一个合适的协议
```

### server端

- 建立socket对象
- 设置socket选项
- 绑定到一个端口
- 监听连接

```python
def server():
    ip_port = ('127.0.0.1', 9000)
    # 默认socket.AF_INET, socket.SOCK_STREAM
    s = socket.socket()
    s.bind(ip_port)
    # 指明在服务器实际处理连接的时候，允许有多少个等待的连接在队列中等待
    s.listen(5)
    conn, addr = s.accept()
    while True:
        # 最多每次接受1024字节
        data = conn.recv(1024)
        print("recv data from client: {0}".format(data))
        if str(data, encoding='utf8') == 'exit':
            break
    conn.close()
```

### client端

```python
def client():
    ip_port = ('127.0.0.1', 9000)
    s = socket.socket()
    s.connect(ip_port)
    while True:
        data = input(">>: ").strip()
        if len(data) == 0:
            continue
        s.send(bytes(data, encoding='utf8'))
        if data == 'exit':
            break
    s.close()
```

## 二. selectors模块

​        `selectors`主要应用于非阻塞的网络编程中，实现了高效的I/O多路复用，该模块定义了一个`BaseSelector`的抽象基类，并包含了`SelectSelector`，`PollSelector`, `EpollSelector`, `DevpollSelector`, `KqueueSelector`的子类。通过`DefaultSelector`可以获取到当前平台的最有效的selector。

`select`：通过`select`用来监视多个文件描述符的数组，当`select`返回后，该数组中就绪的文件描述符便会被内核修改标志位，使得进程可以获得这些文件描述符从而进行后续的读写操作。 `select`是跨平台的，但是缺点在于单个进程能够监视的文件描述符的数量存在最大限制，在Linux上一般为1024，不过可以通过修改宏定义甚至重新编译内核的方式提升这一限制。 另外，`select`所维护的存储大量文件描述符的数据结构，随着文件描述符数量的增大，其复制的开销也线性增长。同时，由于网络响应时间的延迟使得大量TCP连接处于非活跃状态，但调用`select`会对所有socket进行一次线性扫描，所以这也浪费了一定的开销。 

`poll`：和select在本质上没有多大差别，但是poll没有最大文件描述符数量的限制。 poll同样会把大量文件描述符数组整体复制于用户态和内核态的地址空间之间，而不管这些文件描述符是否就绪，所以开销随着文件描述符数量的增加而线性增大。 poll和select支持水平触发。

`epoll`：支持水平触发和边缘触发 。`epoll`采用基于事件的就绪通知方式。在`select`/`poll`中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而`epoll`事先通过注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用时便得到通知。 

### 基本介绍

#### selectors常量

|    常量     |   含义    |
| :---------: | :-------: |
| EVENT_READ  | 可读（1） |
| EVENT_WRITE | 可写（2） |

#### SelectorKey类

是一个命名元组，用来关联一个`fileobj`和他的`fd`，`events mask`和`data`。

|  变量   |                             含义                             |
| :-----: | :----------------------------------------------------------: |
| fileobj |                       已注册的文件对象                       |
|   fd    |                          文件描述符                          |
| events  |                      文件对象等待的事件                      |
|  data   | 注册一个文件对象是绑定的data ，可选。比如可以用存放客户端的session ID |

#### register

```python
abstractmethod register(fileobj, events, data=None) 
```

作用：注册一个文件对象。 

参数：

`fileobj`：需要监控的文件对象，可以是一个整型的文件描述符或者一个包含`fileno()`方法的对象

`events`：`event Mask`，`EVENT_READ`或`EVENT_WRITE`

`data`：an opaque object 

返回：`SelectorKey`实例

#### unregister

```python
abstractmethod unregister(fileobj) 
```

作用：注销一个已经注册过的文件对象

参数：

`fileobj`：需要监控的文件对象

返回：`SelectorKey`实例

#### modify

```python
modify(fileobj, events, data=None) 
```

作用：修改一个已经注册的文件对象的`events`或者`data`

参数：

`fileobj`：需要监控的文件对象，可以是一个整型的文件描述符或者一个包含`fileno()`方法的对象

`events`：`event Mask`，`EVENT_READ`或`EVENT_WRITE`

`data`：an opaque object 

返回：`SelectorKey`实例

#### select

```python
abstractmethod select(timeout=None) 
```

作用：返回一个`(key, events)`的元组。

`key`是一个`SelectorKey`类的实例，相当于就绪的文件对象

`events` 就是 `event Mask`（`EVENT_READ`或`EVENT_WRITE`,或者二者的组合)

#### close

```python
close()
```

作用：关闭selector。

这个必须要调用来确保资源正确被释放。

#### get_key

```python
get_key(fileobj) 
```

作用：返回一个已注册的文件对象关联的`key`

返回：返回文件对象相关联的`SelectorKey`对象

### selectors实现server端

```python
import socket
import selectors

sel = selectors.DefaultSelector()


def accept(sock, mask):
    conn, addr = sock.accept()
    print('accepted', conn, 'from', addr)
    conn.setblocking(False)
    sel.register(conn, selectors.EVENT_READ, read)


def read(conn, mask):
    data = conn.recv(1024)
    if data:
        print('recv data: ', str(data, encoding='utf8'))
        conn.send(data)
    else:
        print('closing', conn)
        sel.unregister(conn)
        conn.close()


def server():
    sock = socket.socket()
    sock.bind(('localhost', 9003))
    sock.listen(100)
    sock.setblocking(False)
    sel.register(sock, selectors.EVENT_READ, accept)

    while True:
        events = sel.select()
        for key, mask in events:
            callback = key.data
            callback(key.fileobj, mask)
```