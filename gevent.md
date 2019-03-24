[TOC]

# gevent

## 例子

```python
import time
import gevent


def func(i):
    print('func{0} start'.format(i))
    gevent.sleep(3)
    print('func{0} done'.format(i))


s = time.time()
coroutines = []
for i in range(3):
    # 创建一个新的greenlet协程对象, 并放到coroutines内
    coroutines.append(gevent.spawn(func, i=i))
# 等待所有greenlets全部执行完毕, 而g.join()是等待此单个协程执行完毕后
gevent.joinall(coroutines)
e = time.time()
print('gevent cost: ', e-s)
```

运行结果：

```shell
func0 start
func1 start
func2 start
func0 done
func1 done
func2 done
gevent cost:  3.0052270889282227
```

## 简介

​        Gevent是一个基于coroutine的网络库，底层以libev或者libuv事件循环为基础，并在上层通过greenlet来提供同步的API。

​	libev、libuv是用c语言实现的异步事件库。异步事件库本质上是提供异步事件通知（*Asynchronous Event Notification*，*AEN*）的。异步事件通知机制就是根据发生的事件，调用相应的回调函数进行处理。

​	一个 “greenlet” 是一个小型的独立伪线程。可以把它想像成一些栈帧，栈底是初始调用的函数，而栈顶是当前greenlet的暂停位置。你使用greenlet创建一堆这样的堆栈，然后在他们之间跳转执行。跳转必须显式声明的：一个greenlet必须选择要跳转到的另一个greenlet，这会让前一个挂起，而后一个在此前挂起处恢复执行。不同greenlets之间的跳转称为切换(switching) 。

总的来说，Gevent特点如下：

- 基于libev或libuv的快速事件循环。
- 基于greenlets的轻量级执行单位。
- 重用Python标准库概念的API (例如e [events](http://www.gevent.org/api/gevent.event.html#gevent.event.Event) 和[queues](http://www.gevent.org/api/gevent.queue.html#gevent.queue.Queue)).
- [协作式sockets，并支持SSL](http://www.gevent.org/api/index.html#networking)
- [协作式DNS查询](http://www.gevent.org/dns.html) 通过 threadpool, dnspython, or c-ares执行.
- [猴子补丁](http://www.gevent.org/intro.html#monkey-patching) 让第三方模块变成协作式
- TCP/UDP/HTTP servers
- Subprocess support (through [gevent.subprocess](http://www.gevent.org/api/gevent.subprocess.html#module-gevent.subprocess))
- Thread pools

## 进阶

### 常见函数

####spawn

```python
spawn = Greenlet.spawn

@classmethod
def spawn(cls, *args, **kwargs):
  """
    spawn(function, *args, **kwargs) -> Greenlet

    Create a new :class:`Greenlet` object and schedule it to run ``function(*args, **kwargs)``.
    This can be used as ``gevent.spawn`` or ``Greenlet.spawn``.

    The arguments are passed to :meth:`Greenlet.__init__`.

    .. versionchanged:: 1.1b1
    If a *function* is given that is not callable, immediately raise a :exc:`TypeError`
    instead of spawning a greenlet that will raise an uncaught TypeError.
  """
  g = cls(*args, **kwargs)
  g.start()
	return g
 
def start(self):
  """Schedule the greenlet to run in this loop iteration"""
	if self._start_event is None:
		_call_spawn_callbacks(self)
		self._start_event = self.parent.loop.run_callback(self.switch)
```

由源码看到，spwan会创建一个Greenlet的对象，并且在事件循环中调度它运行。

#### join

```python
def join(self, timeout=None):
    """
    join(timeout=None) -> None

    Wait until the greenlet finishes or *timeout* expires. Return
    ``None`` regardless.
    """
    if self.ready():
        return

    switch = getcurrent().switch # pylint:disable=undefined-variable
    self.rawlink(switch)
    try:
        t = Timeout._start_new_or_dummy(timeout)
        try:
            result = self.parent.switch()
            if result is not self:
                raise InvalidSwitchError('Invalid switch into Greenlet.join(): %r' % (result, ))
        finally:
            t.cancel()
    except Timeout as ex:
        self.unlink(switch)
        if ex is not t:
            raise
    except:
        self.unlink(switch)
        raise
```

join函数会阻塞greenlet对象直到其运行结束或者超过timeout的时间阈值。

#### joinall

```python
def joinall(greenlets, timeout=None, raise_error=False, count=None):
    """
    Wait for the ``greenlets`` to finish.

    :param greenlets: A sequence (supporting :func:`len`) of greenlets to wait for.
    :keyword float timeout: If given, the maximum number of seconds to wait.
    :return: A sequence of the greenlets that finished before the timeout (if any)
        expired.
    """
    if not raise_error:
        return wait(greenlets, timeout=timeout, count=count)

    done = []
    for obj in iwait(greenlets, timeout=timeout, count=count):
        if getattr(obj, 'exception', None) is not None:
            if hasattr(obj, '_raise_exception'):
                obj._raise_exception()
            else:
                raise obj.exception
        done.append(obj)
    return done
```

阻塞所有的greenlet对象直到其运行完或者运行时间超过timeout设置的事件阈值

#### get

```python
def get(self, block=True, timeout=None):
    """
    get(block=True, timeout=None) -> object

    Return the result the greenlet has returned or re-raise the
    exception it has raised.

    If block is ``False``, raise :class:`gevent.Timeout` if the
    greenlet is still alive. If block is ``True``, unschedule the
    current greenlet until the result is available or the timeout
    expires. In the latter case, :class:`gevent.Timeout` is
    raised.
    """
    if self.ready():
        if self.successful():
            return self.value
        self._raise_exception()
    if not block:
        raise Timeout()

    switch = getcurrent().switch # pylint:disable=undefined-variable
    self.rawlink(switch)
    try:
        t = Timeout._start_new_or_dummy(timeout)
        try:
            result = self.parent.switch()
            if result is not self:
                raise InvalidSwitchError('Invalid switch into Greenlet.get(): %r' % (result, ))
        finally:
            t.cancel()
    except:
        # unlinking in 'except' instead of finally is an optimization:
        # if switch occurred normally then link was already removed in _notify_links
        # and there's no need to touch the links set.
        # Note, however, that if "Invalid switch" assert was removed and invalid switch
        # did happen, the link would remain, causing another invalid switch later in this greenlet.
        self.unlink(switch)
        raise

    if self.ready():
        if self.successful():
            return self.value
```

get能够获取greenlet对象的运行结果，同样的value也能获得其运行结果。

### 协程间通信

#### Event

用于协程间通信，即程序中的其一个协程需要通过判断某个协程的状态来确定自己下一步的操作，就用到了event对象。

Event处理的机制：全局定义了一个“Flag”，默认为False。如果“Flag”值为 False，那么当程序执行 event.wait 方法时就会阻塞，如果“Flag”值为True，那么event.wait 方法时便不再阻塞。gevent内event的is_set()、isSet()、ready()都可以返回event的Flag值。

event主要提供了三个方法wait()、clear()、set()。set() 和 clear() 都是用来改变event的Flag状态的。wait会阻塞当前协程。

```python
def set(self):
    """
    Set the internal flag to true.

    All greenlets waiting for it to become true are awakened in
    some order at some time in the future. Greenlets that call
    :meth:`wait` once the flag is true will not block at all
    (until :meth:`clear` is called).
    """
    self._flag = True
    self._check_and_notify()

def clear(self):
    """
    Reset the internal flag to false.

    Subsequently, threads calling :meth:`wait` will block until
    :meth:`set` is called to set the internal flag to true again.
    """
    self._flag = False

def wait(self, timeout=None):
    """
    Block until the internal flag is true.

    If the internal flag is true on entry, return immediately. Otherwise,
    block until another thread (greenlet) calls :meth:`set` to set the flag to true,
    or until the optional timeout occurs.

    When the *timeout* argument is present and not ``None``, it should be a
    floating point number specifying a timeout for the operation in seconds
    (or fractions thereof).

    :return: This method returns true if and only if the internal flag has been set to
        true, either before the wait call or after the wait starts, so it will
        always return ``True`` except if a timeout is given and the operation
        times out.

    .. versionchanged:: 1.1
        The return value represents the flag during the elapsed wait, not
        just after it elapses. This solves a race condition if one greenlet
        sets and then clears the flag without switching, while other greenlets
        are waiting. When the waiters wake up, this will return True; previously,
        they would still wake up, but the return value would be False. This is most
        noticeable when the *timeout* is present.
    """
    return self._wait(timeout)
```

```python
import gevent
from gevent.event import Event

evt = Event()


def setter():
    print('setter start')
    # evt.set()		---> 1	
    gevent.sleep(3)
    print('setter done')
		# evt.set()		---> 2

def waiter(i):
    print('waiter{0} start'.format(i))
    evt.wait()
    print('waiter{0} done'.format(i))


gevent.joinall(
    [
        gevent.spawn(setter),
        gevent.spawn(waiter, 1),
        gevent.spawn(waiter, 2),
        gevent.spawn(waiter, 3)
    ]
)
# 在1处调用evt.set()，waiter协程均在setter sleep之前就唤醒，
# 因此water协程在setter协程运行完前就结束运行
# setter start
# waiter1 start
# waiter1 done
# waiter2 start
# waiter2 done
# waiter3 start
# waiter3 done
# setter done

# 在2处调用evt.set()，waiter协程的done输出会一直阻塞，
# 直到setter协程sleep完把evt的flag值设置为True，因此
# waiter的print done会在setter的print done后执行。
# setter start
# waiter1 start
# waiter2 start
# waiter3 start
# setter done
# waiter1 done
# waiter2 done
# waiter3 done
```

#### AsyncResult

类似 gevent.event.Event，但在调用 set() 方法时可以传递消息，等待的 greenlet 可以通过 wait() 或 get() 方法读取 set() 设置的值。

#### Queue

队列Queue可以用它的put()和get()方法来存取队列中的元素。gevent的队列对象可以让greenlet协程之间安全的访问。put()和get()方法都是阻塞式的，它们都有非阻塞的版本：put_nowait()和get_nowait()。如果调用get()方法时队列为空，则抛出”gevent.queue.Empty”异常。

#### BoundedSemaphore

信号量可以用来限制协程并发的个数。它有两个方法，acquire()和release()。顾名思义，acquire就是获取信号量，而release就是释放。当所有信号量都已被获取，那剩余的协程就只能等待任一协程释放信号量后才能得以运行。如果信号量个数为1，那就等同于同步锁。

### 局部变量

Gevent允许指定局部于greenlet上下文的数据。 在内部，它被实现为以greenlet的getcurrent()为键， 在一个私有命名空间寻址的全局查找。其作用域限制在当前协程内，当其他协程要访问该变量时，就会抛出异常。不同协程间可以有重名的本地变量，而且互相不影响。

### 协程池

通过 gevent.pool.Group 和 gevent.pool.Pool 可以管理一组 greenlets，前者没有容量限制，后者可设置 Pool 中 greenlet 的最大数量。当容器中的 greenlet 任务完成了，容器会自动将其从中除去。对于 Pool，容器中的 greenlet 数超过设置的最大值时，会将执行权交给 hub greenlet 进行调度。

### 猴子补丁

Python中标准库存在阻塞的情况，gevent提供了”monkey.patch_all()”方法将所有标准库都替换成非阻塞。

```python
from gevent import monkey
monkey.patch_socket()
```

## 引用

[gevent]: http://www.gevent.org/
[gevent介绍]: http://python.jobbole.com/87181/
[libevent、libev、libuv对比]: https://blog.csdn.net/lijinqi1987/article/details/71214974
[greenlet]: https://www.cnblogs.com/Security-Darren/p/4167961.html
[greenlet]: https://www.cnblogs.com/alan-babyblog/p/5353312.html
[greenlet官方文档]: https://greenlet.readthedocs.io/en/latest/

