# python中的元类

## \__new\__方法

元类Metaclass是在python2.2中引入的概念，主要作用就是**定制类的创建行为**，注意创建行为这个描述，一说创建，就会想到\__new\__()方法，大家可能对\__init\__()方法非常熟悉，而\__new\__()方法不常用，其实\__new\__()和\__init\__()方法的区别也是面试中常问的问题，这里简单了解一下，不深入去讲。

\__new\__()方法是类实例化时第一个被调用的方法，当\__new\__()执行完后，python解释器才会调用我们熟悉的\__init\__()方法，其实\__new\__()方法正是创建这个类实例的方法，通过\__new\__()方法创建出类实例后，再利用这个实例来调用\__init\__()方法。

按照python官方文档的说法，\__new\__()方法主要有两个作用：
（1）当你继承一些不可变的class时（如int,str,tuple)，提供一个自定义这些类的实例化过程的方法
（2）实现自定义元类metaclass

## 什么是元类

现在你应该明白了\__new\__()方法了，但是依旧没有搞明白啥是元类？**元类其实就是用于创建类的**，在python中类也是一个对象，我们可以通过这些对象(类）在去创建对象(类实例)，元类就是用于创建类这个对象的，可以将元类理解为类的类:-D

因为类实质上也是一个python对象，所以你可以做下面操作：
（1）你可以将它赋值给一个变量
（2）你可以拷贝它
（3）你可以为它增加属性
（4）你可以将它作为函数参数进行传递

## type方法动态创建类对象

在python中类除了是通过class这个关键字静态创建外，还可以通过type()方法动态创建一个类，虽然type()方法常被我们用于查看一个对象的类型，但它确实拥有动态创建对象的能力，它的使用方式如下

```python
type(类名, 父类的元组（针对继承的情况，可以为空），包含属性的字典（名称和值）)
```

通过简单代码实际体会一下，先用class关键字创建一个类，在用type()方法动态创建一个类

```python
class ayu():
    hello = 'Hello ayu!'

a = ayu()
print(a.hello)

# 动态创建AYU类
AYU = type('AYU',(object,),{'hello':'Hello My Love!'})
A = AYU()
print(A.hello)
```

再次体会一下类实例、类、元类之间的关系

```python
print(a.__class__)
print(ayu.__class__)
print(type.__class__)

输出结果：
<class '__main__.ayu'>
<class 'type'>
<class 'type'>
```

a是ayu类的实例，ayu是type的实例，type的类也是type，在python中type就是默认的最基层的元类，所有自定义的元类都继承它

## 使用元类

明白了元类是什么，那怎么使用它呢？下面我们通过元类来自定义出一个调用ayuliao()方法和author属性的list类

先自定义一个元类，自定义时要继承默认元类type，然后重写其中的\__new\__()方法则可

```python
class ListMetaclass(type):
    '''
    __new__()方法接收到的参数依次是：
    当前准备创建的类的对象；
    类的名字；
    类继承的父类集合；
    类的方法集合。
    '''
    def __new__(cls, name, bases, attrs):
        attrs['ayuliao'] = lambda self, value: self.append(value)
        attrs['author'] = 'ayuliao'
        return type.__new__(cls, name, bases, attrs)
```

在\__new\__方法中添加了ayuliao方法和author属性

接着定义一个类，该类继承list类，还需注意的就是添加了metaclass参数

```python
class MyList(list, metaclass=ListMetaclass):
    pass
```

当我们在定义类是使用了metaclass参数传入一个元类，python的元类黑魔法就生效了，使用一下MyList这个类

```python
L = MyList()
L.ayuliao(123)
L.ayuliao(345)
print(L)
print(L.author)

输出如下：
[123, 345]
ayuliao
```

到这里，元类就算介绍完了，仔细思考一下，似乎没啥用处，你要自定义出方法或属性，直接在类中定义就好了，没必要绕一圈去元类创建这些方法和属性。

## 元类的使用情景

元类一般用于类似这样的需求，如下

+ 构造一个动物类(父类)
+ 构造一个狗的子类和鸟的子类
+ 构造一个像狗一样吃东西，像鸟一样运动的新物种

**简单说就是混合同父类的子类中的方法，得到一个新的类**

先定义出动物父类、狗子类和鸟子类

```python
class Animal(object):
    def eat(self):
        print("吃东西像[Animal]")

    def move(self):
        print("运动想[Animal]")


class Dog(Animal):
    def eat(self):
        print("吃东西像[Dog]")

    def move(self):
        print("运动想[Dog]")

class Bird(Animal):
    def eat(self):
        print("吃东西像[Bird]")

    def move(self):
        print("运动想[Bird]")
```

上面这个需求，如果你单纯的使用类继承，就会造成要么狗子类的方法将鸟子类的方法覆盖，要么就反过来，鸟子类的方法覆盖狗子类中的方法，如下：

```python
class NewSpecies(Bird,Dog):
    pass

ns = NewSpecies()
ns.eat()
ns.move()

输入结果：

吃东西像[Bird]
运动想[Bird]
```

所以要创建一个新物种，让它既有狗吃东西的特征又有鸟运动的特征，就可以使用元类，当然蠢方法就是将不同的方法重写到新类中，当代码量多时，很难维护，这里我们用元类来实现一个这个需求

定义出一个元类，重新实现一下eat方法和move方法

```python
class AnimalMetaclass(type):
    def __new__(cls, *args, **kwargs):
        name, bases,attr = args[:3]
        #bases是一个tuple，存放着子类继承的父类
        dog, bird = bases

        def eat(self):
            dog.eat(self)

        def move(self):
            bird.move(self)

        attr['eat'] = eat
        attr['move'] = move
        
        return type(name,bases,attr)
```

定义代码比较简单，需要注意的就是，在\__new\__方法中重新定义的eat()方法和move()方法需要传入self

通过AnimalMetaclass元类再定义出NewAnimal类

```python
class NewAnimal(Dog,Bird, metaclass=AnimalMetaclass):
    pass

na = NewAnimal()
na.eat()
na.move()

输入结果：
吃东西像[Dog]
运动想[Bird]
```

元类除了用于上面的常见，最常见的可能就是实现ORM了，Django框架中的ORM就使用了元类

## 小结
其实这篇文章一开始定的标题是要介绍如何使用元类来自己定义ORM框架的，但是光元类的内容就写了不少，所以如何编写自己的ORM就留到下一篇文章来写了。

还需要提一下就是，除非你要使用python自己开发一个框架，大部分情况下是不必使用元类的，类的继承和多态用好就可以解决大部分问题了，不要为了让代码有所谓的高级感而绕圈写代码:-D


