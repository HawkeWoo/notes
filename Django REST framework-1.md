---
title: Django REST framework 总结一
tags: 
  - Django
  - Django REST framework
categories:
  - Django
---
# Django REST framework

很久之前接触过这个框架，但是之前在都是用公司自己开发的api模块。现在重新接触下，温故而知新。顺便给有需要的人看看基本用法。

还是从demo入手。

[TOC]

## 一. 创建demo项目

```
# 创建一个新项目restful和一个单个应用api
python3 django-admin.py startproject restful
cd restful
python3 manage.py startapp api
# 创建超级用户，方便添加测试数据
python manage.py createsuperuser
```

## 二. 项目基本设置

`restful/settings.py`

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'api',
]
```

`restful/urls.py`

```python
from django.conf.urls import url, include
from django.contrib import admin

urlpatterns = [
    url(r'^admin/', admin.site.urls),
    url(r'^api/', include('api.urls')),
]
```

项目基本设置完，开始api应用的编写。

## 三. api应用编写

### 1. 编写model

`api/models.py`

```python
from django.db import models

# Create your models here.


class Author(models.Model):
    """作者"""
    name = models.CharField(max_length=30)

    def __str__(self):
        return self.name


class Address(models.Model):
    """地址"""
    name = models.CharField(max_length=50)

    def __str__(self):
        return self.name


class Publisher(models.Model):
    """出版社"""
    name = models.CharField(max_length=30)
    # 一个出版社只有一个地址
    address = models.OneToOneField(Address)

    def __str__(self):
        return self.name


class Book(models.Model):
    """书本"""
    name = models.CharField(max_length=30)
    # 一本书可能有多个作者，一个作者可能出版多本书
    authors = models.ManyToManyField(Author)
    # 一本书只能由一个出版社出版
    publisher = models.ForeignKey(Publisher)

    def __str__(self):
        return self.name
```

### 2. 新建序列化模块serializers

`api/serializers.py`

`rest_framework.serializers`有多个序列化类可以使用：`Serializer`，`ModelSerializer`，`HyperlinkedModelSerializer`，`ListSerializer`和`BaseSerializer`。个人觉得比较好用的是`ModelSerializer`

1. 它根据模型自动生成一组字段。
2. 它自动生成序列化器的验证器，比如unique_together验证器。
3. 它默认简单实现了`.create()`方法和`.update()`方法

```python
from rest_framework import serializers

from api.models import Author, Book, Address, Publisher


class AuthorSerializer(serializers.ModelSerializer):
    class Meta:
        model = Author
        fields = '__all__'
        # 可以自定义需要序列化的字段
        # fields = ('id', 'name')
        # or
        # fields = ('name')


class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = '__all__'


class AddressSerializer(serializers.ModelSerializer):
    class Meta:
        model = Address
        fields = '__all__'


class PublisherSerializer(serializers.ModelSerializer):
    class Meta:
        model = Publisher
        fields = '__all__'
```

### 3. 编写视图函数

`api/views.py`

```python
from rest_framework import generics

from api.models import Author, Book, Publisher, Address
from api.serializers import AuthorSerializer, BookSerializer, PublisherSerializer, AddressSerializer


class AuthorList(generics.ListCreateAPIView):
    queryset = Author.objects.all()
    serializer_class = AuthorSerializer


class BookList(generics.ListCreateAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer


class PublisherList(generics.ListCreateAPIView):
    queryset = Publisher.objects.all()
    serializer_class = PublisherSerializer


class AddressList(generics.ListCreateAPIView):
    queryset = Address.objects.all()
    serializer_class = AddressSerializer
```

### 4. 编写url模块

`api/urls.py`

```python
from django.conf.urls import url

from . import views


urlpatterns = [
    url(r'^authors/$', views.AuthorList.as_view(), name='authors'),
    url(r'^books/$', views.BookList.as_view(), name='books'),
    url(r'^addresses$/', views.AddressList.as_view(), name='addresses'),
    url(r'^publisher$/', views.PublisherList.as_view(), name='publishers'),
]
```

### 5. model加入到admin管理

`api/admin.py`

```python
from django.contrib import admin

# Register your models here.


from api.models import Author, Book, Address, Publisher


admin.site.register(Author)
admin.site.register(Book)
admin.site.register(Address)
admin.site.register(Publisher)
```

### 6. 测试

```python
HTTP 200 OK
Allow: GET, POST, HEAD, OPTIONS
Content-Type: application/json
Vary: Accept

[
    {
        "id": 1,
        "name": "薄荷糖",
        "publisher": 1,
        "authors": [
            1
        ]
    },
    {
        "id": 2,
        "name": "咖啡",
        "publisher": 1,
        "authors": [
            2
        ]
    },
    {
        "id": 3,
        "name": "柠檬茶",
        "publisher": 1,
        "authors": [
            1
        ]
    }
]
```

## 四. 进一步改进api应用

从上面测试可以看到，api返回的数据中嵌套的出版社`publisher`和`authors`都只是返回了实例的`id`，如果想返回详细的信息，只需要设置序列化模块中的`depth`。`depth`选项应该设置一个整数值，表明应该遍历的关联深度。 

另外还可以设置

`read_only_fields`选项：将所需要的字段指定为只读。 

`extra_kwargs`选项：快捷地在字段上指定任意附加的关键字参数。 

`api/serializers.py`

```python
class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = '__all__'
        depth = 1	# 新增设置
```

现在的结果是：

```python
HTTP 200 OK
Allow: GET, POST, HEAD, OPTIONS
Content-Type: application/json
Vary: Accept

[
    {
        "id": 1,
        "name": "薄荷糖",
        "publisher": {
            "id": 1,
            "name": "人民出版社",
            "address": 3
        },
        "authors": [
            {
                "id": 1,
                "name": "Tom"
            }
        ]
    },
    {
        "id": 2,
        "name": "咖啡",
        "publisher": {
            "id": 1,
            "name": "人民出版社",
            "address": 3
        },
        "authors": [
            {
                "id": 2,
                "name": "James"
            }
        ]
    },
    {
        "id": 3,
        "name": "柠檬茶",
        "publisher": {
            "id": 1,
            "name": "人民出版社",
            "address": 3
        },
        "authors": [
            {
                "id": 1,
                "name": "Tom"
            }
        ]
    }
]
```
## 五. 通用的视图类

### CreateAPIView

用于 **仅创建** 端点。

提供一个 `post` 方法处理程序。

### ListAPIView

用于 **只读** 端点以表示**模型实例集合** 。

提供一个 `get` 方法处理程序。

### RetrieveAPIView

用于**只读** 端点以表示**单个模型实例**。

提供一个 `get` 方法处理程序。

### DestroyAPIView

用于**只删除**端点以表示**单个模型实例**。

提供一个 `delete` 方法处理程序。

### UpdateAPIView

用于**只更新**端点以表示**单个模型实例**。

提供一个 `put`和`patch`方法处理程序。

### ListCreateAPIView

用于**读写端点**以表示**模型实例的集合**。

提供一个 `get` 和 `post` 方法的处理程序。

### RetrieveUpdateAPIView

用于 **读取或更新** 端点以表示 **单个模型实例**。

提供 `get`, `put` 和 `patch` 方法的处理程序。

### RetrieveDestroyAPIView

用于 **读取或删除** 端点以表示 **单个模型实例**。

提供 `get` 和 `delete` 方法的处理程序。

### RetrieveUpdateDestroyAPIView

用于 **读写删除** 端点以表示 **单个模型实例**。

提供 `get`, `put`, `patch` 和 `delete`方法的处理程序。

因为上面的通用视图是扩展`GenericAPIView`和`Mixins`类的，当需要一些定制的需求时，可以用以下方法覆盖：

- `perform_create(self, serializer)` - 在保存新对象实例时由 `CreateModelMixin` 调用。
- `perform_update(self, serializer)` - 在保存现有对象实例时由 `UpdateModelMixin` 调用。
- `perform_destroy(self, instance)` - 在删除对象实例时由 `DestroyModelMixin` 调用。