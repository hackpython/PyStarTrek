# python中的闭包

较为深入的讲讲python中的闭包，涉及的内容

1.变量的作用域
2.nonlocal关键字
3.闭包及其原理

## 变量作用域

要理解python中的闭包，离不开对python中变量作用域的理解。

在python中，变量主要分为：局部变量和全局变量

局部变量：该变量的作用范围只在定义了该变量的类内或只在定义了该变量的函数内，在范围外使用该变量会报错

全局变量：顾名思义，全局变量的作用范围是整个python文件，从一个全局变量被定义后，后面的任何方法和类都可以直接使用该变量

一个简单的例子:

```python
In [10]: def ayu(a):
    ...:     print(a)
    ...:     print(b)
    ...:

In [11]: ayu(1)
1
---------------------------------------------------------------------------
NameError                                 Traceback (most recent call last)
<ipython-input-11-f6a49eaafea5> in <module>()
----> 1 ayu(1)

<ipython-input-10-bee8bdc6a8e4> in ayu(a)
      1 def ayu(a):
      2     print(a)
----> 3     print(b)
      4

NameError: name 'b' is not defined
```

定义了一个方法ayu，其中有变量a和变量b，变量a是局部变量，通过参数传值的方式赋值，而变量b是全局变量，因为在代码中没有为其赋值，所有报出“变量名为b的变量没有定义”的错误

只需要在ayu方法外定义一下变量b就好了

```python
In [12]: b = 6

In [13]: ayu(1)
1
6
```

此时定义了全局变量b，再运行ayu方法就不会报错了

此时简单修改一下ayu方法，再次运行

```python
In [17]: b = 6

In [18]: def ayu(a):
    ...:     print(a)
    ...:     print(b)
    ...:     b = 666
    ...:

In [19]: ayu(123)
123
---------------------------------------------------------------------------
UnboundLocalError                         Traceback (most recent call last)
<ipython-input-19-48be8501c1c7> in <module>()
----> 1 ayu(123)

<ipython-input-18-a571fe8e284b> in ayu(a)
      1 def ayu(a):
      2     print(a)
----> 3     print(b)
      4     b = 666
      5

UnboundLocalError: local variable 'b' referenced before assignment
```

观察上面代码，我只是在ayu方法末尾添加了一个赋值语句，为全局变量b赋一个新的值，结果在运行ayu方法时就报错了，仔细看报错信息，说“局部变量'b'在赋值之前引用”

讲道理，print(b)应该会正常执行打印出6才对，因为我们在ayu方法前已经为全局变量b赋值为6了，而且报错信息说b是局部变量。

但现实世界是不讲道理的。简单解释一下这种现象，python编译器在编译ayu函数的定义体时，它判断b是局部变量，因为在ayu函数中给b进行了赋值操作，可以通过dis函数生成字节码来验证这种判断

```python
In [20]: from dis import dis

In [21]: dis(ayu)
  2           0 LOAD_GLOBAL              0 (print)
              3 LOAD_FAST                0 (a) #局部变量a
              6 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
              9 POP_TOP

  3          10 LOAD_GLOBAL              0 (print)
             13 LOAD_FAST                1 (b) #编译器将b视为局部变量，与a一样
             16 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
             19 POP_TOP

  4          20 LOAD_CONST               1 (666)
             23 STORE_FAST               1 (b)
             26 LOAD_CONST               0 (None)
             29 RETURN_VALUE
```

为了让编译器明白其实我们想让变量b成为全局变量，就需要动用global关键字，直截了当的告诉编译器，变量b是全局变量，这样编译器就不会将变量b偷偷的转成局部变量了

```python
In [22]: b = 6

In [23]: def ayu(a):
    ...:     global b
    ...:     print(a)
    ...:     print(b)
    ...:     b = 666
    ...:

In [24]: ayu(123)
123
6
```

同样可以通过dis函数来验证一波

```python
In [25]: dis(ayu)
  3           0 LOAD_GLOBAL              0 (print)
              3 LOAD_FAST                0 (a) # 局部变量
              6 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
              9 POP_TOP

  4          10 LOAD_GLOBAL              0 (print)
             13 LOAD_GLOBAL              1 (b) # 全局变量
             16 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
             19 POP_TOP

  5          20 LOAD_CONST               1 (666)
             23 STORE_GLOBAL             1 (b)
             26 LOAD_CONST               0 (None)
             29 RETURN_VALUE
```

## 闭包

首先要有几点共识

+ 1.python中一切皆对象，函数也是对象，类也是对象
+ 2.函数和类是有作用域的，两者不同之处在于，函数的作用域会随着的函数的作用域在函数调用时创建，在函数执行完后，其相应的作用域会被释放，而类的作用域则在python程序执行时创建，一般程序执行完后其作用域才释放

而闭包的作用简单来讲就是延伸了作用域的函数，函数是否是匿名函数没有关系，关键是**该函数能不能访问函数定义体外的非全局变量。**

一个简单的例子：

```python
In [29]: def divide2():
    ...:     num = []
    ...:
    ...:     def func(v):
    ...:         num.append(v)
    ...:         return sum(num)/2
    ...:
    ...:     return func
    ...:

In [30]: d = divide2()

In [31]: d(4)
Out[31]: 2.0

In [32]: d(8)
Out[32]: 6.0
```

divide2()方法的作用是累加此前传入的数，并将其除以2的结果返回，该方法其实就是一个典型的闭包

为什么它是个闭包？关注点应该在于，num是divide2()函数的局部变量，在函数初始化时，我们定义了num=[]，那么当divide2()函数调用执行完后，其作用域应该随着函数执行完成而被释放才对（共识1），为什么下次调用时，计算出了的值还是受到上一次的影响，d(4)的结果为2，d(8)的结果为6，按道理，divide2()函数的作用域在d(4)执行完后，已经释放了，在调用d(8)时再创建新的作用域，此前作用域中参数传入的4应该不起作用了，d(8)的结果为4才对，但是程序执行的d(8)却为6，说明此前执行d(4)时传入的4还在。

再次体会到现实是不讲道理的，其核心原因在于，num是个自由变量

?>自由变量：表示未在本地作用域中绑定的变量，一个技术术语

直观理解，看图

![bibao_1]()

图中，func函数的闭包延伸到了该函数的作用域之外，包含了自由变量num，自由变量不会随作用域释放而销毁，封闭的闭包包裹自由变量，有点像类包裹着类属性一样，只要在闭包的范围内，就可以访问该闭包中的自由变量

深入点，我们可以从函数的\__code\__属性中获得编译后函数定义体中保存的局部变量和自由变量，而自由变量的值其实被存储在cell对象的cell_contents属性中

```python
In [40]: d.__code__.co_varnames # 编译后函数的局部变量
Out[40]: ('v',)

In [41]: d.__code__.co_freevars # 编译后函数的自由变量
Out[41]: ('num',)

In [42]: d.__closure__ # __closure__中各元素对应这co_freevars的自由变量
Out[42]: (<cell at 0x10d6a4be8: list object at 0x10d737488>,)

In [43]: d.__closure__[0].cell_contents # 查看自由变量num中的值
Out[43]: [4, 8]
```

简单讲：闭包是一种函数，它会将定义函数时存在的自由变量绑定起来形成一个包裹，包裹内的自由变量不会随着函数的作用域变动，就算函数的作用域被释放了，包裹内的自由变量已经活的好好的，只要在闭包范围内，闭包中绑定的自由变量都可以使用

## nonlocal关键字

但是如果自由变量是常量的话，情况会不同吗？会不同，同样给你个不讲理的报错

这里将list类型的自由变量num改成int型的自由变量sum，sum是数值类型，数值是不可变的

```python
In [44]: def addall():
    ...:     sum = 0
    ...:     def add(v):
    ...:         sum += 1
    ...:         return sum
    ...:     return add
    ...:

In [45]: a = addall()

In [46]: a(4)
---------------------------------------------------------------------------
UnboundLocalError                         Traceback (most recent call last)
<ipython-input-46-331eaaf84436> in <module>()
----> 1 a(4)

<ipython-input-44-b667aca388e6> in add(v)
      2     sum = 0
      3     def add(v):
----> 4         sum += 1
      5         return sum
      6     return add

UnboundLocalError: local variable 'sum' referenced before assignmen
```

因为将自由变量改成不可变的数值类型，程序就报错了，而且又说sum是局部变量

原因其实简单，因为sum是不可变类型，当python解释器，执行到sum += 1时，因为sum中的值不可变，那么赋值语句其实就创建了一个新的sum（创建了数值2这个变量，将sum变量中的内存地址改为数值2的内存地址），这个赋值操作将sum从自由变量打回了局部变量的身份，而前面的num变量是list类型，使用的是num.append()方法，没有对num变量进行赋值操作，所以它的身份一直是自由变量（利用了list类型是可变变量的特性）

不止是数值类型，字符串、元组等不可变类型都会因为其只可读不可改的特性，在闭包中重新赋值时被打回局部变量的身份（隐式创建局部变量），为了解决这个问题，python3引入了nonlocal关键字，作用就是把变量标记为自由变量，直接了当告诉解释器，该变量的身份是自由变量，别偷偷又把它改回局部变量。

有了nonlocal，即使对不可变类型做赋值操作，也会将创建的新变量自动变成自由变量。

```python
In [48]: def addall():
    ...:     sum = 0
    ...:     def add(v):
    ...:         nonlocal sum #声明为自由变量
    ...:         sum += 1
    ...:         return sum
    ...:     return add
    ...:

In [49]: a = addall()

In [50]: a(6)
Out[50]: 1

In [51]: a(8)
Out[51]: 2
```

那python2咋办？它没有nonlocal关键字。在python2上要实现这样的效果，只能采取折中的方法，就是将不可能变量存储为可变对象，如存到list或dist中，然后再使用闭包，此时闭包绑定的事可变对象，就会有上面的问题。

## 小节

通篇阅读后，发现，闭包的作用跟类很像，闭包将自由变量包裹在一起，延伸了函数的作用域，使函数可以使用闭包范围内的自由变量，而不必拘束于自己的作用域内，简单讲就是将父函数的变量与内部函数做绑定，让绑定变量后的函数在即使生成闭包函数的父函数已经释放也依旧可以使用闭包中的自由变量。

很像类（父函数）实例化（创建闭包）的过程。类也是对象，它由元类实例化而来，类实例绑定了其中的属性元素，就算类作用域消失了，类实例依旧存在，类一般在python执行时就创建了，在程序执行完后其作用域才释放（共识2），那么闭包会比类占用更少的系统资源，也更加轻巧。

当然我认为很少有人会考虑这方面的资源占用，生活愉快:-D





