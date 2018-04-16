# 使用Python编写简单ORM

ORM（Object Relational Mapping）对象关系映射

很多web框架中都有ORM，它的核心作用有两点

+ 将对象映射成SQL语句，一个对象对应数据库中的一张表，对象中的属性对应表中的字段
+ 通过对象的形式让框架内部进行相应SQL语句的转换，一定程度上避免的SQL注入攻击

当然如果web使用了ORM，那么它就会比使用原始的SQL语句慢，因为多了一层将对象转换未SQL的过程

我们可以使用Python中的元类来实现一个简单的ORM框架，简单看一下Django ORM的源码，发现它也是通过python元类来实现的

## 自顶向下设计

在程序设计中，有种设计思想叫自顶向下进行设计，从顶部开始思考ORM框架如何实现。那么一个框架的最顶部就是用户如何使用，所以我们编写一段用户使用ORM框架的代码，假定用户这样使用我们的编写的ORM框架，ORM框架中要实现什么内容。

```python
# 定义一个类，对应数据库中的一张表
class User(Model):
    # 表名
    class Meta:
        table='users'
    # 创建bigint型字段
    id = IntegerField()
    # 创建varchar型字段
    name = StringField(max_length=50)
    email = StringField(max_length=50, default='ayuliao@xx.com')
    password = StringField(max_length=50)

u = User(id=234, name='ayuliao',email='123',password='123456')
u.save()
```

假定用户通过上面的代码使用我们的ORM框架

他定义了一个类User，继承了Model类（Model类是ORM框架提供的基类），在User类中，嵌套了Meta类，表明User类对应到数据库中表的表名

在User类中，他还定义了id、name、email、password四个属性，这里属性使用了IntegerField类和StringField类，分别表示创建bigint型字段和varchar型字段

User类定义完后，他希望实例化User类后，可以直接调用save()方法对实例化时传入的数据进行保存

接着就来逐步实现ORM框架，让上面的代码可以正常使用数据库

## 定义IntegerField类和StringField类

先来定义上面代码中使用的IntegerField类和StringField类，看回上妈的使用代码

```python
id = IntegerField()
email = StringField(max_length=50, default='ayuliao@xx.com')
```

IntegerField类表示在数据库中创建bigint类型的字段，字段名使用变量名，这里就是id
StringField类表示在数据库中创建varchar类型的字段，因为varchar类型的字段在创建时需要指明字段长度，所有我们希望可以使用max_length参数来表示varchar字段的长度，我们还希望可以通过defalue来设置varchar字段的默认值

下面开写，先定义出所有xxxxField类的基类Field，在Field类中定义出column_type属性表示数据库中字段的类型，max_length属性表示数据库中字段的长度，default表示字段的默认值

```python
class Field(object):
    def __init__(self,column_type, max_length, **kwargs):
        self.column_type = column_type # 字段类型
        self.max_length = max_length # 字段长度
        self.default = None # 字段默认值
        if kwargs:
            # 是否有某属性，有就为该属性赋值，其实就是赋值给default
            for k,v in kwargs.items():
                if hasattr(self,k):
                    setattr(self,k,v)
    
    # 定义打印类时显示的内容           
    def __str__(self):
        return '<%s>'%(self.__class__.__name__)
```

定义好了基类后再定义出StringField类和IntegerField类就比较简单了，其实就是**多态**，不同处在于column_type属性和max_length不同

```python
class StringField(Field):
    def __init__(self, max_length, **kwargs):
        # column_type：类型varchar，max_length：长度
        super().__init__(column_type='varchar({})'.format(max_length),max_length=max_length,**kwargs)


class IntegerField(Field):
    def __init__(self, **kwargs):
        super().__init__(column_type='bigint', max_length=8)
```

## 定义ModelMetaclass类和Model类

在上面使用代码中，User类继承了Model类

```python
class User(Model):
```

按照逻辑，接下来，就要实现Model类了，但在实现Model类前，我们先要实现ModelMetaclass这个元类，如果你忘记了元类相关的知识，可以看一下[python元类](http://hackpython.com/PyStarTrek/#/python%E4%B8%AD%E7%9A%84%E5%85%83%E7%B1%BB?id=python%E4%B8%AD%E7%9A%84%E5%85%83%E7%B1%BB)

ModelMetaclass元类的作用就是定义了Model类或Model子类(User类)的创建行为

当Model类要创建时，触发了if name=='Model'下面的代码

当User类(Model子类)要创建时，就会执行除if name=='Model'外的代码

ModelMetaclass类的具体代码如下：

```python
class ModelMetaclass(type):
    def __new__(cls, name, bases, attrs):
        if name == 'Model':
            # 使用type动态创建一个Model的实例
            return type.__new__(cls, name,bases,attrs)

        mappings = dict()
        # 将属性存放到dict中
        for k,v in attrs.items():
            # 必须是Field类
            if isinstance(v, Field):
                mappings[k] = v
        # 删除一下相应的值，元类是用于创建类的，所以这里的v其实是内存地址
        for k in mappings.keys():
            attrs.pop(k)
        # 保存属性和列的映射关系
        attrs['__mappings__'] = mappings
        # 保存表名，如果定义的类没用table属性，那么就将表名设置成类名
        attrs['__table__'] = attrs.get('Meta').table or name
        # 使用type创建类的实例对象
        return type.__new__(cls, name, bases, attrs)
```

上面代码的逻辑比较简单，从attrs中获得相应的属性，将这些属性存到mappings这个dict中

![](http://p6un02lk4.bkt.clouddn.com/mappings_orm.png)

然后通过pop方法将attrs中已经加到mappings这个dict中的值删除，元类的作用是创建类，从上图也可以看出dict中id、email这些key对应的值其实都是内存地址，删除attrs中值的目的是避免后面使用getattrs()方法出现意想不到的错误

实现好了ModelMetaclass元类，接着就可以写Model类了

```python
from functools import reduce
class Model(dict, metaclass=ModelMetaclass):
    def __init__(self,**kw):
        super(Model, self).__init__(**kw)

    # 动态获取属性
    def __getattr__(self, item):
        try:
            return self[item]
        except:
            raise AttributeError(r"'Model' 对象没有%s属性"%item)

    def __setattr__(self, key, value):
        self[key] = value

    def save(self):
        fields = []
        params = []
        # 在元类的attrs中
        for k,v in self.__mappings__.items():
            fields.append(k)
            # getattr(self,k,v.default)获得类中的属性k，如果不存在，则返回v.defalut
            params.append(getattr(self,k,v.default))
        sql = 'insert into {} ({}) values ({})'.format(self.__table__, self.join(fields), self.join(params))

        print('SQL:%s'%sql)

    # join函数，可以处理数字等非字符串
    def join(self, attrs, pattern=','):
        return reduce(lambda x,y:'{}{}{}'.format(x,pattern,y),attrs)
```

上面代码的大致逻辑就是从__mappings__中取出相应的值，通过getattr获得类中key对应的值，然后构建出一个sql语句

简单说一下getattr()方法：

当 u = User(id=234, name='ayuliao',email='123',password='123456') 这行代码被执行时，因为User是Model的子类，而Model类又绑定了ModelMetaclass元类，所用User在实例化时，ModelMetaclass元类中的\__new\__方法先与Model类的\__init\__方法被调用。

在ModelMetaclass元类中，使用了**attrs.pop()**方法删除attrs中相应的值，这是因为

+ 1.当u.save()被执行时，在save()方法中getattr()会先从类的属性或父类的属性中找相应的值
+ 2.当在类的属性或父类的属性中找不到相应的值时，才会通过\__gettattrs\__方法来查找
+ 3.如果没用使用attrs.pop()方法将id、name、email、password这些关键字删除，getattr()方法可以直接从类中获得相应的值，这些值在上面也提及了，都是内存地址，而不是我们需要的具体数值
+ 4.所以使用attrs.pop()删除那些关键字，这样就会调用\__gettattrs\__方法来查找，在\__gettattrs\__方法中，使用了self[item]，因为类的作用是创建实例对象，所以这里的self其实就是User类的实例u，通过实例就可以获得 u = User(id=234, name='ayuliao',email='123',password='123456')这行代码中传入的值


到这里就将一个简单的ORM编写完成了，将上面的代码都放在一个python文件中，运行起来，效果如下

```python
SQL:insert into users (id,password,name,email) values (234,123456,ayuliao,123)
```

获得具体的SQL后，就可以调用Python链接数据库的库，将这段SQL作为输入，在数据库中创建相应的表，插入相应的字段，而这个过程对用户来说都是透明的，它只见到的定义了一个User类

## 为什么要使用元类？

仔细观察上面ModelMetaclass类和Model类的代码，感觉ModelMetaclass类的作用似乎就是将User类中定义的元素存到mappings这个dict中，然后赋值给attrs['__mappings__']，接着就让Model类来使用了。

为什么要让User类中的元素过一遍mappings这个dict呢？

删除元类，似乎也可以实现需求，代码如下：

```python
class Field(object):

    def __init__(self, name, column_type):
        self.name = name
        self.column_type = column_type

    def __str__(self):
        return '<%s:%s>' % (self.__class__.__name__, self.name)


class StringField(Field):
    def __init__(self, name):
        super(StringField, self).__init__(name, 'varchar(100')


class IntegerField(Field):
    def __init__(self, name):
        super(IntegerField, self).__init__(name, 'bigint')


class Model(dict):
    def __init__(self, **kw):
        super(Model, self).__init__(**kw)

    def save(self):
        fields = []
        params = []
        args = []
        for k in self.keys():
            fields.append(k)
            params.append('?')
            args.append(self[k])
        sql = 'insert into %s (%s) values (%s)' % (self.__class__.__name__, ','.join(fields), ','.join(params))
        print('SQL: %s' % sql)
        print('ARGS: %s' % str(args))


class User(Model):
    # 定义类的属性到列的映射：
    id = IntegerField()
    name = StringField()
    email = StringField()
    password = StringField()


# 创建一个实例：
u = User(id=12345, name='Michael', email='test@orm.org', password='my-pwd')
# 保存到数据库
u.save()
```

运行结果：

```python
SQL: insert into User (password,email,id,name) values (?,?,?,?)
ARGS: ['my-pwd', 'test@orm.org', 12345, 'Michael']
```

删去了元类，依旧达到了效果，似乎还简单了很多

如果使用上面的代码来实现ORM框架就失去了ORM框架的意义了，代码中User类完全就是一个壳，完全可以将User类中的内容删除，改成pass，代码照样正常运行，其实也就说明，SQL语句的实现完全取决于实例化User类时传入的参数，这份代码中的Field类其实没啥用处，而**一个正常的ORM框架一般都会使用Field类做参数检查、非法参数过滤**

## 小结
可以去翻看一下Django ORM源码相关的博客、文章，理解真正实用的ORM框架在编写时要考虑哪些细节，本章只是实习了一个非常简易的ORM，要让ORM可以在真实项目中稳定运行，还是要多学点，共勉！



