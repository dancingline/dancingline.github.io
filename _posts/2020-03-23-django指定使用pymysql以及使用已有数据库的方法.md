---
layout: post
title: "Windows进程通信之匿名管道"
date: 2020-03-23
excerpt: 
tags: [django, mysql, python]
feature: 
comments: false
---



# django指定使用pymysql以及使用已有数据库的方法

## 指定使用pymysql连接数据库

现在django官方建议使用的连接MySQL的方法是mysqlclient，但是直接pip的话会报错，这东西有C的运行库依赖，需要在机器上装一个MySQL，不想装的话就可以指定使用pymysql。

首先在工程的`setting.py`里面把`DATABASE`改成使用MySQL，这里需要注意使用的用户需要有数据库的很多权限。

然后在`init.py`里面加上上面的内容

```python
import pymysql
pymysql.install_as_MySQLdb()
```

然后执行`migrate`的时候大概率会报两个错

```shell
django.core.exceptions.ImproperlyConfigured: mysqlclient 1.3.13 or newer is required; you have 0.9.3.

AttributeError: ‘str’ object has no attribute ‘decode’
```

这时候需要去python安装路径下的`site-packages/django/db/backends/mysql/`这个文件夹里修改`base.py`和`opeartions.py`两个文件

```python
# 在base.py里注释35和36行的内容
if version < (1, 3, 13):
    raise ImproperlyConfigured('mysqlclient 1.3.13 or newer is required; you have %s.' % Database.__version__)
    
# 在operations.py里把146行的decode改为encode
if query is not None:
            query = query.decode(errors='replace')
```

## 使用已有的数据库

这个在[官方文档](https://docs.djangoproject.com/zh-hans/3.0/howto/legacy-databases/)里给出了方法:

配置了数据库相关内容以后，执行

```shell
python manage.py inspectdb
```

将产生的python代码放入`models.py`，然后migrate就行了