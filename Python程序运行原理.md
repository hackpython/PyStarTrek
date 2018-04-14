# Python程序运行原理

本章主要回答：

+ Python程序运行时究竟发生了什么？
+ Python虚拟机做了什么工作？

从一个简单的例子看起，在同一个目录下创建两个文件，分别为myfun.py和test.py

```python
[myfun.py]
def add(a,b):
	return a+b
```

```python
[test.py]
import myfun

a = list(range(5))
a = 'ayuliao'

def func():
	a = 1
	b = 257
	print(a+b)

print(a)

if __name__ == '__main__':
	func()
	myfun.add(1,2)
```

通过python编译执行test.py这个程序

```python
python test.py
```

获得如下结果

```
ayuliao
258
```

这里使用的是python3.6，运行完这个程序后，会出现\__pycache\__这个文件夹，里面存放了经过python编译后的pyc文件

![](http://p6un02lk4.bkt.clouddn.com/mypy%E5%B8%83%E5%B1%80.png)

## python在背后做了什么？

python将.py文件看成一个module(模块)，这些module中有一个主module作为程序的入口，上面的实例中，主module就是test.py，因为我们使用了 python test.py 这个命令

执行 python test.py 后，python解释器(Cpython)会将test.py编译成一个字节码对象PyCodeObject，在Python中一切都是对象，python解释器编译出来的字节码也是对象，而我们看见的.pyc文件其实就是PyChodeObject这个字节对象在硬盘上的表现形式

?>一般说python解释器指的就是Cpython这个解释器，它有C语言实现，除了Cpython外，还有Jython解释器，有Java实现，PyPy解释器，由Python本身实现的解释器

在python代码运行期间，PyCodeObject对象只会存在内容中，只有到这个module下的python代码执行完后，才会将编译结果（PyCodeObject对象）保存到pyc文件中，下次运行时，就不需要编译了，直接加载到内存汇总就好，**pyc文件只是PyCodeObject对象在硬盘上的表现形式**

?>PyCodeObject对象包含了Python代码汇总所有的字符串、常量值、以及通过语法解析后编译生成的字节码指令。同时PyCodeObject还会存放这些字节码指令与原始代码行号之间的对应关系，这样当代码出现异常时，就可以指明代码在哪里报错了

我们可以通过dis这个python标准库来获得代码对应的字节码，这些字节码就是PyCodeObject对象中的内容（也是pyc文件中的内容）

```python
In [7]: import dis

In [8]: dis.dis(lambda x,y,z:(x+y)*z)
  1           0 LOAD_FAST                0 (x) # x进栈
              2 LOAD_FAST                1 (y) # y进栈
              4 BINARY_ADD                     # 栈上的数相加，也就是x与y相加
              6 LOAD_FAST                2 (z) # z进栈
              8 BINARY_MULTIPLY                # 栈上的数相乘，x与y相加的结果与z相乘
             10 RETURN_VALUE                   # 返回值，整个操作就是  xy+z*
```

之所以要通过dis库是因为pyc文件中存储的是二进制字节数据，无法直接阅读

我们可以通过下面的代码生成 test.py 文件中对应的字节码指令

```python
In [12]: test = open('test.py').read()

In [13]: co = compile(test, 'test.py', 'exec')

In [14]: import dis

In [15]: dis.dis(co)
```

结果如下：

```python
  1           0 LOAD_CONST               0 (0)
              2 LOAD_CONST               1 (None)
              4 IMPORT_NAME              0 (myfun)
              6 STORE_NAME               0 (myfun)

  3           8 LOAD_NAME                1 (list)
             10 LOAD_NAME                2 (range)
             12 LOAD_CONST               2 (5)
             14 CALL_FUNCTION            1
             16 CALL_FUNCTION            1
             18 STORE_NAME               3 (a)

  4          20 LOAD_CONST               3 ('ayuliao')
             22 STORE_NAME               3 (a)

  6          24 LOAD_CONST               4 (<code object func at 0x109e7a780, file "test.py", line 6>)
             26 LOAD_CONST               5 ('func')
             28 MAKE_FUNCTION            0
             30 STORE_NAME               4 (func)

 11          32 LOAD_NAME                5 (print)
             34 LOAD_NAME                3 (a)
             36 CALL_FUNCTION            1
             38 POP_TOP

 13          40 LOAD_NAME                6 (__name__)
             42 LOAD_CONST               6 ('__main__')
             44 COMPARE_OP               2 (==)
             46 POP_JUMP_IF_FALSE       66

 14          48 LOAD_NAME                4 (func)
             50 CALL_FUNCTION            0
             52 POP_TOP

 15          54 LOAD_NAME                0 (myfun)
             56 LOAD_ATTR                7 (add)
             58 LOAD_CONST               7 (1)
             60 LOAD_CONST               8 (2)
             62 CALL_FUNCTION            2
             64 POP_TOP
        >>   66 LOAD_CONST               1 (None)
             68 RETURN_VALUE
```

test.py中的代码总共就是15行，如下图：

![](http://p6un02lk4.bkt.clouddn.com/mypy15%E8%A1%8C.png)

上面编译出来的字节码其实也就是15个区块，每个区块都对应着一行代码

当test.py被python编译器编译后，Python虚拟机会从编译得到的PyCodeObject对象中依次读入每一条字节码指令，在**当前的上下文环境**中执行这条字节码，Python程序就这样跑起来了。

## pyc文件
大体明白了Python代码运行流程后，再来挖掘一些细节，如：

+ 为什么上面的代码分明运行的是test.py文件生成的却是myfun.py文件的pyc呢？
+ myfun.py文件对应的pyc再什么时候需要再次生成？

在一个pyc文件中，包含了三部分内容，分别是：

+ python的magic number（类似于python版本号）
+ pyc文件创建时间信息
+ PyCodeObject对象

magic number是Python定义的一个整数值，一般说，不同版本的python都会定义不同的magic number，目的是保证python的兼容性。因为不同版本的python生成的字节码可能不同，所以每次python运行时，解释器都要检查magic number是否与当前运行版本python的magic number不同，如果不做检查，python虚拟机执行时就可能会报错

pyc文件创建时间信息主要的作用就是判断pyc文件是否需要重新生成，在运行python代码时，如果python发现已经存在相应的pyc，就会去检查对应python文件的修改时间，如果pyc文件的创建时间与python文件的修改时间不同，说明python文件发生了变动，此时就会重新生成pyc文件

PyCodeObject对象就不多讲了

那为啥不生成test.py的pyc文件呢？

在test.py中我们使用了内置属性\__name\__，python程序在运行时，会自动为每个module都设置\__name\__属性，通常的作用就是这个module的名字，也就是文件名，唯一的例外就是主module，主module会的\__name\__会被设置为\__main\__。

利用这个特性，我们可以通过对不同的模块进行测试，当这个模块以主模块运行时，if \__name\__ == '\__main\__'下的代码就会执行，而当我们import该模块时，if \__name\__ == '\__main\__'下的代码**不会**执行

python只会对那些以后可能会被继续使用和载入的module生成pyc文件，python认为import指令对应的module属于这类，所以myfun.py生成了相应的pyc文件，而那些临时只使用一次的模块不会生成pyc，python将主模块当成这种类型的文件，这也就是test.py不会生成pyc文件的原因

通过下面命令来验证上面的说法

先删除\__pycache\__文件夹，然后再用python myfun.py命令运行myfun.py，将myfun.py当做主module时，看看是否会生成pyc文件，结果是没有生成

```bash
(anaconda3-4.4.0)  ~/Desktop/mypy > ls
__pycache__ myfun.py    test.py
(anaconda3-4.4.0)  ~/Desktop/mypy > rm -rf __pycache__
(anaconda3-4.4.0)  ~/Desktop/mypy > ls
myfun.py test.py
(anaconda3-4.4.0)  ~/Desktop/mypy > python myfun.py
(anaconda3-4.4.0)  ~/Desktop/mypy > ls
myfun.py test.py
```

## 小结
本章简单的介绍了python程序的运行原理，这些原理性的东西，除了阅读源码，还有看python源码解析书籍这条捷径

推荐：[Python源码剖析](https://book.douban.com/subject/3117898/)

参考：[谈谈 Python 程序的运行原理](https://www.restran.net/2015/10/22/how-python-code-run/)


