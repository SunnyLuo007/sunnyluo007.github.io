---
layout: post
title: Python变量与作用域相关内置函数的应用
category: Python
tags: [python]
keywords: python,vars,locals,globals,dir,global,nonlocal
---
### 概述
今天来谈谈dir locals globals global和nonlocal的作用。  
这些内置函数都与变量的查找和访问有关，一般情况下，Python查找变量的顺序如下：
1. 当前函数的作用域
2. 任何外围作用域（如：包含当前函数的其他函数，非调用函数）
3. 包含当前代码的那个模块的作用域（也叫全局作用域）
4. 内置作用域（也就是包含len、str等buildin函数的那个作用域）
  
例1：
```py
x='global'
def outer():
    x = 'out'

    def inner():
        print(x)  #首先在inner范围内找，找不到。
                  #再找外围函数，即outer。找到x。
                  #如果outer内没有定义x，就会找到x='global'

    inner()
    print(x)

outer()  #结果输出： out out
```
例2：
```py
def outer():
    x = 'out'

    def inner():
        x = 'in' #如果在inner内没有定义过x，
                #则这里会新定义一个x变量，作用域为inner函数
        print(x)

    inner()
    print(x)

outer()  #结果输出：in  out
```
例3:
```py
__builtins__.x = 'builtin'  #在内置作用域定义变量x

def outer():

    def inner():
        print(x)

    inner()
    print(x)

outer()  #结果输出：builtin builtin
```
### dir
dir函数在不传参数的情况下，会返回一个列表，包含当前作用域内所有的属性名称。  

举例：

```py
def outer():

    def inner():
        x = 'out'
        print(dir())

    inner()

outer()  #输出['x']
```

```py
def outer():
    x = 'out'
    def inner():
        print(dir())

    inner()

outer()  #输出[]
```

在有参数的情况下：
- 如果参数是一个模块对象，则返回模块内属性的名称。
- 如果参数是type或类，则返回该类及其父类的属性名称。
- 如果参数是其他对象，在该对象定义了\_\_dir()\_\_方法的情况下，则返回\_\_dir()\_\_的返回值（必须为列表），否则返回该对象对应的类及其父类的属性名称。  

举例：

```py
class B(object):
    z = 'c'
    

class A(B):
    y = 'b'
    def __init__(self):
        x = 'a'

    def __dir__(self):
        return ['area', 'perimeter', 'location']


a = A()
b = B()

print(dir(A))  #输出类的共有属性和'y','z'
print(dir(b))  #输出类的共有属性和'z'
print(dir(a))  #输出__dir__返回值

结果：
['__class__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__le__', '__lt__', '__module__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', 'y', 'z']
['__class__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__le__', '__lt__', '__module__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', 'z']
['area', 'location', 'perimeter']
```
### locals和globals
globals总是返回当前模块内所有标识符（属性、变量等）的名称和值。  
举例：

```py
>>> x = 'a'
>>> print(globals())
{'__loader__': <class '_frozen_importlib.BuiltinImporter'>, '__doc__': None, '__package__': None, '__spec__': None, 'x': 'a', '__builtins__': <module 'builtins' (built-in)>, '__name__': '__main__', 'sys': <module 'sys' (built-in)>}
```
对应的，locals返回当前作用域内所有的标识符（属性、变量等）的名称和值：  

```py
print(locals())
def outer():
    # x = 'out'

    def inner():
        # nonlocal x
        x = 'in'
        print(locals())

    inner()
    print()

outer()
结果：
{'__doc__': None, '__spec__': None, '__name__': '__main__', '__package__': None, '__builtins__': <module 'builtins' (built-in)>, '__loader__': <_frozen_importlib_external.SourceFileLoader object at 0x7fc5a101b860>, '__cached__': None, '__file__': '/home/sunny/桌面/sp/tt.py'}
{'x': 'in'}
```

### global
这个最常用。局部变量需要修改**全局变量本身的值**的情况下必须先声明为global变量：

```py
a = 'x'
def fun():
    a = 'b'   #前面说过，这里其实是新建里一个变量
    print(a)

fun()
print(a)

结果：
b
x
```
```py
a = 'x'
def fun():
    global a
    a = 'b'
    print(a)

fun()
print(a)
结果：
b
x
```
### nonlocal
这是python3的一种特殊写法，能获取闭包内的数据。如果一个便量用nonlocal声明以后，在需要使用（包括修改变量的值）该变量的时候，程序会在上层作用域中查找该变量。  
**注意:**
nonlocal声明的变量不能延伸到模块级别，也就是不能使用全局变量，防止它污染全局作用域。这个函数其实是与global函数互为补充作用。

```py
x='golbal'

def outer():
    # nonlocal x  #启用这句会报错
    x = 'out'

    def inner():
        nonlocal x
        x = 'in'
        print(x)

    inner()
    print(x)

outer()

结果：
in
in
```

