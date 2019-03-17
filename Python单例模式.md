---
title: Python单例模式
tags: 
  - python
  - 单例模式
categories:
  - python
---
#### 模块

```python
class SingletonModule(object):
    def func(self):
        pass

singleton = SingletonModule()
```

#### __new__

```python
class SingletonNew(object):

    def __init__(self, name):
        self._name = name

    def __new__(cls, *args, **kwargs):
        if not hasattr(cls, 'instance'):
            cls.instance = super().__new__(cls)
        return cls.instance
```

#### 装饰器

```python
def single_decorate(cls):
    instances = {}

    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance

@single_decorate
class SingletonDecorate(object):
    def __init__(self, name):
        self._name = name
```

#### 元类

```python
class SingletonMeta(type):
    __instance = None

    def __call__(cls, *args, **kwargs):
        if cls.__instance is None:
            cls.__instance = super().__call__(*args, **kwargs)
        return cls.__instance


class SingletonMetaTest(metaclass=SingletonMeta):

    def __init__(self, name):
        self._name = name
```

