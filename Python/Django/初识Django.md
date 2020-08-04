## 设计模型

Django提供了[对象关系映射器](https://en.wikipedia.org/wiki/Object-relational_mapping)用来描述数据库结构，只需要Python代码即可。可以使用[数据-模型语句](https://docs.djangoproject.com/zh-hans/2.0/topics/db/models/)来描述数据模型：

```python
from django.db import models

class Reporter(models.Model):  # 首先要让我们的模型继承自models.Model类
    full_name = models.CharField(max_length=70)  # 然后就可以定义各个字段

    def __str__(self):
        return self.full_name

class Article(models.Model):
    pub_date = models.DateField()
    headline = models.CharField(max_length=200)
    content = models.TextField()
    reporter = models.ForeignKey(Reporter, on_delete=models.CASCADE)

    def __str__(self):
        return self.headline
```

## 应用数据模型

定义好自己的模型后，运行Django命令行工具来创建数据库表：

```powershell
python manage.py migrate
```

这个**migrate**命令会查找所有可用的模型，并将在数据库中创建那些不存在的表，注意建表时会自动提供一个**id**自增字段，同时还提供了可选的[丰富schema控制](https://docs.djangoproject.com/zh-hans/2.0/topics/migrations/)。

## 便捷的API来访问数据

创建完表后，就可以使用一些内置的[Python API](https://docs.djangoproject.com/zh-hans/2.0/topics/db/queries/)来访问数据库内的数据，这些API时即时创建的，而不用显示的生成代码：

```shell
# Import the models we created from our "news" app
>>> from news.models import Article, Reporter

# 返回所有的Reporter
>>> Reporter.objects.all()
<QuerySet []>

# 首先创建一个新的Reporter对象
>>> r = Reporter(full_name='John Smith')

# 然后调用对象的save方法，就可以将它保存到数据库中
>>> r.save()

# 保存完成后就可以访问它的各个属性了，这时候就是数据库的值
# 例如刚才定义的时候没有id属性，但现在有了
>>> r.id
1

# 数据已经存到数据库里了
>>> Reporter.objects.all()
<QuerySet [<Reporter: John Smith>]>

####### Django提供了大量的API用于数据库查找

# get方法，传入参数可过滤某一行
>>> Reporter.objects.get(id=1)
<Reporter: John Smith>
>>> Reporter.objects.get(full_name__startswith='John')
<Reporter: John Smith>
>>> Reporter.objects.get(full_name__contains='mith')
<Reporter: John Smith>
>>> Reporter.objects.get(id=2)
Traceback (most recent call last):
    ...
DoesNotExist: Reporter matching query does not exist.

# Create an article.
>>> from datetime import date
>>> a = Article(pub_date=date.today(), headline='Django is cool',
...     content='Yeah.', reporter=r)
>>> a.save()

# Now the article is in the database.
>>> Article.objects.all()
<QuerySet [<Article: Django is cool>]>

# Article objects get API access to related Reporter objects.
>>> r = a.reporter
>>> r.full_name
'John Smith'

# And vice versa: Reporter objects get API access to Article objects.
>>> r.article_set.all()
<QuerySet [<Article: Django is cool>]>

# The API follows relationships as far as you need, performing efficient
# JOINs for you behind the scenes.
# This finds all articles by a reporter whose name starts with "John".
>>> Article.objects.filter(reporter__full_name__startswith='John')
<QuerySet [<Article: Django is cool>]>

# Change an object by altering its attributes and calling save().
>>> r.full_name = 'Billy Goat'
>>> r.save()

# Delete an object with delete().
>>> r.delete()
```

## 动态管理接口

当模型定义完成后，Django就会自动生成一个专业的生产级[管理接口](https://docs.djangoproject.com/zh-hans/2.0/ref/contrib/admin/)：允许认证用户添加、更改和删除对象的Web站点。你只需简单地在admin站点上注册你的模型即可：

```python
from django.db import models

class Article(models.Model):
    pub_date = models.DateField()
    headline = models.CharField(max_length=200)
    content = models.TextField()
    reporter = models.ForeignKey(Reporter, on_delete=models.CASCADE)
```

```python
from django.contrib import admin

from . import models

admin.site.register(models.Article)		# 在站点上注册这个模型
```

它的设计理念时这样，通过开放某些模型的编辑权限给不同的用户，这样员工就可以只编辑自己的那一部分内容，来填充网站里的数据。

## 规划URLs

为了设计自己的[URLconf](https://docs.djangoproject.com/zh-hans/2.0/topics/http/urls/)，需要创建一个叫URLconf的Python模块，这是网站的目录，包含了一张URL和Python回调函数之间的映射表，这也有利于将Python代码与URL进行解耦。

```python
from django.urls import path

from . import views

# Django1.x里面是url函数
urlpatterns = [
    path('articles/<int:year>/', views.year_archive),
    path('articles/<int:year>/<int:month>/', views.month_archive),
    path('articles/<int:year>/<int:month>/<int:pk>/', views.article_detail),
]
```

上述代码将URL路径映射到了Pthon回调函数，当用户请求页面时，Django依次遍历路径，直至初次匹配到了请求的URL，这个过程时比较快的，因为路径在加载时就编译成了**正则表达**。

一旦有URL路径匹配成功，Django会调用相应的视图函数，每个视图函数会接受一个请求对象，其中包含了请求元信息，以及在匹配式中获取的参数值。例如，当用户请求了这样的 URL "/articles/2005/05/39323/"，Django 会调用 `news.views.article_detail(request, year=2005, month=5, pk=39323)`。

## 编写视图

试图函数的执行结果只可能有两种：返回一个包含请求页面元素的HttpResponse对象，或是抛出Http404这类异常。通常来说一个视图的工作是：从参数获取数据，装载一个模板，根据获取的数据对模板进行渲染。

```python
from django.shortcuts import render

from .models import Article

def year_archive(request, year):
    a_list = Article.objects.filter(pub_date__year=year)
    context = {'year': year, 'article_list': a_list}
    return render(request, 'news/year_archive.html', context) # render函数
```

这个例子使用了Django[模板系统](https://docs.djangoproject.com/zh-hans/2.0/topics/templates/)。

## 设计模板

上面的代码加载了**news/year_archive.html**模板，Django允许设置搜索模板路径，这样可最小化模板之间的冗余。在Django设置中，可通过[DIRS](https://docs.djangoproject.com/zh-hans/2.0/ref/settings/#std:setting-TEMPLATES-DIRS)参数指定一个路径列表用域检索模板。如果第一个路径不包含模板，则继续检查第二个，以此类推。