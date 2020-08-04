## *args和 **kwargs

这两者主要用于函数定义，可将不定数量的参数传递给一个函数：

- `*args`：用来发送一个**非键值对**的可变数量的参数列表给一个函数

- `**kwargs`：将不定长度的**键值对**作为参数传递给一个函数。

```python
def test_var_args(f_arg, *argv):
    print("first normal arg:", f_arg)
    for arg in argv:
        print("another arg through *argv:", arg)

test_var_args('yasoob', 'python', 'eggs', 'test')
# 输出
first normal arg: yasoob
another arg through *argv: python
another arg through *argv: eggs
another arg through *argv: test

def greet_me(**kwargs):
    for key, value in kwargs.items():
        print("{0} == {1}".format(key, value))


greet_me(name="yasoob", sex="man", age=16)  # 可传递任意长度的键值对
# 输出
name == yasoob
age == 16
sex == man
```

## 生成器

### 可迭代对象

Python中任意的对象，只要它定义了可以返回一个迭代器的`__iter__`方法，或者定义了可以支持下标索引的`__getitem__`方法，那么它就是一个可迭代对象。简单说，**可迭代对象就是能提供迭代器的任意对象**。

### 迭代器

任意对象，只要定义了`next`(Python2) 或者`__next__`方法，它就是一个迭代器。

### 迭代

用简单的话讲，它就是从某个地方（比如一个列表）取出一个元素的过程。当我们使用一个循环来遍历某个东西时，这个过程本身就叫迭代。

### 生成器

生成器也是一种迭代器，但是你只能对其迭代一次。**这是因为它们并没有把所有的值存在内存中，而是在运行时生成值**。你通过遍历来使用它们，要么用一个“for”循环，要么将它们传递给任意可以进行迭代的函数和结构。**大多数时候生成器是以函数来实现的**。然而，它们并不返回一个值，而是`yield`(暂且译作“生出”)一个值。这里有个生成器函数的简单例子：

```python
def generator_function():
    for i in range(10):
        yield i

for item in generator_function():  # 使用for循环来访问迭代器
    print(item)

```

生成器最佳应用场景是：**你不想同一时间将所有计算出来的大量结果集分配到内存当中**，特别是结果集里还包含循环。许多Python 2里的标准库函数都会返回列表，而Python 3都修改成了返回生成器，因为生成器占用更少的资源。来看一个计算斐波那契数列的生成器：

```python
# generator version
def fibon(n):
    a = b = 1
    for i in range(n):
        yield a   # 计算完成后，生成a（类似于返回a）
        a, b = b, a + b

for x in fibon(1000000):
    print(x)
```

python有一个内置的函数`next()`，它允许我们获取一个序列的下一个元素。

```python
def generator_function():
    for i in range(3):
        yield i

gen = generator_function()
print(next(gen))  # 将生成器作为参数传递给next()函数，或者调用生成器.next()方法
print(next(gen))
print(next(gen))
```

而对于python中的字符串我们也可以进行迭代访问：

```python
my_string = "Yasoob"   # 字符串是一个可迭代对象
my_iter = iter(my_string)  # 使用iter方法，将迭代对象作为参数，返回一个迭代器
next(my_iter)   # 将迭代器作为参数传给next()函数，就可以访问迭代对象里的元素
# Output: 'Y'
```

## Map，Filter和Reduce

### Map

`map`会将一个函数映射到一个输入列表的所有元素上：

```python
map(function_to_apply, list_of_inputs)  # 第一个是函数，第二个是元素
```

例如求一个列表中所有元素的平方，可以使用下面的方法：

```python
nums = [1, 2, 3, 4, 5]
squared = list(map(lambda x : x**2, nums))
```

大多数时候，我们使用匿名函数(lambdas)来配合`map`，我们不仅可以将一个元素列表传递进去，也可以将一个函数列表传递进去，例如：

```python
def multiply(x):
        return (x*x)
def add(x):
        return (x+x)

funcs = [multiply, add]
for i in range(5):
    value = map(lambda x : x(i), funcs)  # 这里相当于对于列表中每个函数，都调用它，并以i为参数
    print list(value)

# Output:
# [0, 0]
# [1, 2]
# [4, 4]
# [9, 6]
# [16, 8]
```

### Filter

`filter`过滤列表中的元素，并且返回一个由所有符合要求的元素所构成的列表，`符合要求`即函数映射到该元素时返回值为True。

```python
number_list = list(range(-5, 5))
less_than_zero = list(filter(lambda x : x<0, number_list))
```

### Reduce？？

当需要对一个列表进行一些计算并返回结果时，`Reduce` 是个非常有用的函数。举个例子，当你需要计算一个整数列表的乘积时，可以使用`reduce`来完成。

```python
from functools import reduce
product = reduce( (lambda x, y: x * y), [1, 2, 3, 4] )

# Output: 24
```

## 三元运算符

三元运算符通常在Python里被称为条件表达式，这些表达式基于真(true)/假(false)的条件判断，可以看一下伪代码：

```python
# 如果条件condition为真，返回condition_is_true 否则返回condition_is_false
condition_is_true if condition else condition_is_false
```

真正使用如下：

```python
is_fat = True
state = "fat" if is_fat else "not fat"
```

## 装饰器

简单地说：**他们是修改其他函数的功能的函数**。

### 一切皆对象

能看到一个函数也是一个对象，当不带括号时可以赋值给其他变量，当带括号时代表要调用这个函数。而且从下面的例子能看出来，`hi`和`greet`相当于都指向了一个方法，删除一个变量时并不影响另一个变量对该方法的调用。

```python
def hi(name="yasoob"):
    return "hi " + name

print(hi())
# output: 'hi yasoob'

# 不带括号，则可以赋值给某个变量
greet = hi

# 对这个变量加上括号后，就可以调用哪个方法了
print(greet())
# output: 'hi yasoob'

# 如果我们删掉旧的hi函数，则不能调用hi了
del hi
print(hi())
#outputs: NameError

print(greet())  # 但greet可以继续调用
#outputs: 'hi yasoob'
```

### 在函数中定义函数

在Python中我们可以在一个函数中定义另一个函数，但要注意我们在外面是不能访问一个函数里面的函数的。也就是说我们访问不到函数`hi()`里面的`greet()`和`welcome()`。

```python
def hi(name="yasoob"):
    print("now you are inside the hi() function")

    def greet():
        return "now you are in the greet() function"

    def welcome():
        return "now you are in the welcome() function"

    print(greet())
    print(welcome())
    print("now you are back in the hi() function")

hi()
#output:now you are inside the hi() function
#       now you are in the greet() function
#       now you are in the welcome() function
#       now you are back in the hi() function

# 上面展示了无论何时你调用hi(), greet()和welcome()将会同时被调用。
# 然后greet()和welcome()函数在hi()函数之外是不能访问的，比如：

greet()
#outputs: NameError: name 'greet' is not defined
```

### 从函数中返回函数

如上面所说我们不能访问一个函数内部的函数，但可以将内部的函数返回出来，因为一切皆对象：

```python
def hi(name="yasoob"):
    def greet():
        return "now you are in the greet() function"

    def welcome():
        return "now you are in the welcome() function"

    if name == "yasoob":
        return greet
    else:
        return welcome

a = hi()
print(a)
#outputs: <function greet at 0x7f2143c01500>

#上面清晰地展示了`a`现在指向到hi()函数中的greet()函数
#现在试试这个

print(a())
#outputs: now you are in the greet() function
```

### 将函数作为参数传递给另一个函数

```python
def hi():
    return "hi yasoob!"

def doSomethingBeforeHi(func):  # 可以将一个函数传递进来
    print("I am doing some boring work before executing hi()")
    print(func())

doSomethingBeforeHi(hi)
#outputs:I am doing some boring work before executing hi()
#        hi yasoob!
```

### 第一个装饰器

```python
def a_new_decorator(a_func):  
    def wrapTheFunction():  # 将传进来的方法进行包装，并返回这个方法
        print "I'm doing some boring work before executing a_func()"
        a_func()
        print "I am doing some boring work after executing a_func()"

    return wrapTheFunction

def a_function_requiring_decoration():
    print "I am the function which needs some decoration to remove my foul smell"

a_function_requiring_decoration()

# 我们得到了一个包装后的方法，其实就是wrapTheFunction
a_function_requiring_decoration = a_new_decorator(a_function_requiring_decoration)

a_function_requiring_decoration()
```

当然我们可以使用@符号来使代码简短些，就是将装饰器作为一个注解修饰要被包装的方法：

```python
@a_new_decorator  # 装饰器修饰被包装的方法
def a_function_requiring_decoration():
    print "I am the function which needs some decoration to remove my foul smell"

# 上面的定义就等价于下面这种了
a_function_requiring_decoration = a_new_decorator(a_function_requiring_decoration)
```

但如果我们运行下面这个方法会存在一个问题：

```python
print(a_function_requiring_decoration.__name__)
# Output: wrapTheFunction
```

能理解，因为本身返回的确实是函数`wrapTheFunction`，这里的函数被warpTheFunction替代了，**它重写了我们函数的名字和注释文档(docstring)**。而Python提供了一个简单的函数来解决这个问题，即functools.wraps。

```python
from functools import wraps

def a_new_decorator(a_func):
    @wraps(a_func)  # 表明要包装传进来的函数
    def wrapTheFunction():
        print "I'm doing some boring work before executing a_func()"
        a_func()
        print "I am doing some boring work after executing a_func()"

    return wrapTheFunction

@a_new_decorator
def a_function_requiring_decoration():
    print "I am the function which needs some decoration to remove my foul smell"

print a_function_requiring_decoration.__name__
# Output: a_function_requiring_decoration
```

> 注意：@wraps接受一个函数来进行装饰，并加入了复制函数名称、注释文档、参数列表等等的功能。这可以让我们在装饰器里面访问在装饰之前的函数的属性。

















## Global和Return

当我们需要访问函数中返回的值时，可以使用`global`关键字，然后在函数内部进行赋值后，在外面就可以访问到了。例如：

```python
# 首先，是没有使用global变量
def add(value1, value2):
    result = value1 + value2

add(2, 4)
print(result)  # 访问不到result，因为作用域的关系

# 输出
Traceback (most recent call last):
  File "", line 1, in
    result
NameError: name 'result' is not defined

# 现在我们运行相同的代码，不过是在将result变量设为global之后
def add(value1, value2):
    global result  # 设置为global后，函数内部赋值后在外部就可以访问到了
    result = value1 + value2

add(2, 4)
print(result)
6
```

如果一个函数要返回多个值时，当然仍然可以使用`global`关键字，但其实python中是可以直接返回多个变量的，例如：

```python
def profile():
    name = "Danny"
    age = 30
    return name, age
```

> 当然应该尽量避免使用global变量，不然滥用的话很容易引起变量名字的冲突。

## 对象变动 Mutation

Python中存在可变(**mutable**)与不可变(**immutable**)两种数据类型，简单的说：

- 可变(mutable)意味着"可以被改动"
- 而不可变(immutable)的意思是“常量(constant)”。

```python
foo = ['hi']
print(foo)
# Output: ['hi']

bar = foo
bar += ['bye']
print(foo)
# Output: ['hi', 'bye']
```

每当你将一个变量赋值为另一个可变类型的变量时，对这个数据的任意改动会同时反映到这两个变量上去。新变量只不过是老变量的一个别名而已。**这个情况只是针对可变数据类型**。

## slots魔法

在Python中，每个类都有**实例属性**。默认情况下Python用一个**字典**来保存一个对象的实例属性。这非常有用，因为它允许我们**在运行时去设置任意的新属性**。

然而，对于有着已知属性的小类来说，它可能是个瓶颈。**这个字典浪费了很多内存。Python不能在对象创建时直接分配一个固定量的内存来保存所有的属性**。因此如果你创建许多对象（我指的是成千上万个），它会消耗掉很多内存。 不过还是有一个方法来规避这个问题。这个方法需要使用`__slots__`来告诉Python不要使用字典，而且只给一个固定集合的属性分配空间。

- 不使用 `__slots__`:

```python
class MyClass(object):
    def __init__(self, name, identifier):
        self.name = name
        self.identifier = identifier
        # self.set_up()
```

- 使用`__slots__`：

```python
class MyClass(object):
    __slots__ = ['name', 'identifier']  # 提前定义__slots__
    def __init__(self, name, identifier):
        self.name = name
        self.identifier = identifier
        # self.set_up()
```

而后者会将内存占用率减少很多，且在`PyPy`默认已经做了这些优化。

## 虚拟环境 Virtualenv ？？？

 当我们安装了某个版本的Python后（例如2.7），则所有第三方的包都会被`pip`安装到Python27的`site-packages`目录下。

`Virtualenv` 是一个工具，它能够帮我们创建一个独立(隔离)的Python环境。使用`virtualenv`针对每个程序创建独立（隔离）的Python环境，而不是在全局安装所依赖的模块。要安装它，只需要在命令行中输入以下命令：

```powershell
$ pip install virtualenv
```

然后假设我们要开发一个新的项目，且需要一套独立的Python环境，可以这样做：

1. 创建项目MyProject目录，并进入到此目录下
2. 创建一个独立的Python环境，命名为`venv`：





## 容器Collections ???

Python附带一个模块，它包含许多容器数据类型，名字叫作`collections`。我们将讨论的是：

- defaultdict
- counter
- deque
- namedtuple
- enum.Enum (包含在Python 3.4以上)

### defaultdict

与`dict`类型不同，你不需要检查**key**是否存在：

```python
from collections import defaultdict

colours = (('Yasoob', 'Yellow'),
    ('Ali', 'Blue'),
    ('Arham', 'Green'),
    ('Ali', 'Black'),
    ('Yasoob', 'Red'),
    ('Ahmed', 'Silver'),
)

favourite_colours = defaultdict(list) # 表示字典值的类型是个列表

for name, colour in colours:
    favourite_colours[name].append(colour)  # 因为值是个列表，所以是append

print favourite_colours
```

另一种重要的是例子就是：当你在一个字典中对一个键进行嵌套赋值时，如果这个键不存在，会触发`keyError`异常。 `defaultdict`允许我们用一个聪明的方式绕过这个问题。

首先我分享一个使用`dict`触发`KeyError`的例子，然后提供一个使用`defaultdict`的解决方案。

```python
some_dict = {}
some_dict['colours']['favourite'] = "yellow"

## 异常输出：KeyError: 'colours'
```

解决方案：

```python
import collections
tree = lambda: collections.defaultdict(tree)
some_dict = tree()
some_dict['colours']['favourite'] = "yellow"

## 运行正常
```

### counter

Counter是一个计数器，它可以帮助我们针对某项数据进行计数。比如它可以用来计算每个人喜欢多少种颜色：

```python
from collections import Counter

colours = (
    ('Yasoob', 'Yellow'),
    ('Ali', 'Blue'),
    ('Arham', 'Green'),
    ('Ali', 'Black'),
    ('Yasoob', 'Red'),
    ('Ahmed', 'Silver'),
)

favs = Counter(name for name, colour in colours)
print(favs)

## 输出:
## Counter({
##     'Yasoob': 2,
##     'Ali': 2,
##     'Arham': 1,
##     'Ahmed': 1
##  })
```

### deque

deque提供了一个双端队列，你可以从头/尾两端添加或删除元素。要想使用它，首先我们要从`collections`中导入`deque`模块：

## 枚举Enumerate

枚举(`enumerate`)是Python内置函数，它允许我们遍历数据并自动计数：

```python
for counter, value in enumerate(some_list):  # 当对访问对象使用了enumerate时，会生成一个索引
    print(counter, value)
```

而且`enumerate`也接受一些可选参数：

```python
my_list = ['apple', 'banana', 'grapes', 'pear']
for c, value in enumerate(my_list, 1):  # 参数1表示刚才的索引从1开始，如果是5则是5，6，7，8
    print c, value


# 输出:
(1, 'apple')
(2, 'banana')
(3, 'grapes')
(4, 'pear')
```

当然因为使用了`enumerate`后生成了一个索引，那么我们可以创建包含索引的元组列表，例如：

```python
my_list = ['apple', 'banana', 'grapes', 'pear']
counter_list = list(enumerate(my_list, 1))
print(counter_list)
# 输出: [(1, 'apple'), (2, 'banana'), (3, 'grapes'), (4, 'pear')]
```

## 对象自省

自省(introspection)，在计算机编程领域里，是指在运行时来判断一个对象的类型的能力。Python中所有一切都是一个对象，而且我们可以仔细勘察那些对象。

### dir

它返回一个列表，列出了一个对象所拥有的属性和方法。

```python
my_list = [1, 2, 3]
dir(my_list)
# Output: ['__add__', '__class__', '__contains__', '__delattr__', '__delitem__',
# '__delslice__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__',
# '__getitem__', '__getslice__', '__gt__', '__hash__', '__iadd__', '__imul__',
# '__init__', '__iter__', '__le__', '__len__', '__lt__', '__mul__', '__ne__',
# '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__reversed__', '__rmul__',
# '__setattr__', '__setitem__', '__setslice__', '__sizeof__', '__str__',
# '__subclasshook__', 'append', 'count', 'extend', 'index', 'insert', 'pop',
# 'remove', 'reverse', 'sort']
```

上面的自省给了我们一个列表对象的所有方法的名字。如果我们运行`dir()`而不传入参数，那么它会返回**当前作用域的所有名字**。例如下面的打印了当前域的一些属性，还有所有已经定义了的变量。

```python
['__builtins__', '__doc__', '__file__', '__name__', '__package__', 'c', 'colour', 'colours', 'defaultdict', 'favourite_colours', 'my_list', 'name', 'value']
```

### type和id

`type`函数返回一个对象的类型

```python
print(type(''))
# Output: <type 'str'>

print(type([]))
# Output: <type 'list'>

print(type({}))
# Output: <type 'dict'>

print(type(dict))
# Output: <type 'type'>

print(type(3))
# Output: <type 'int'>
```

`id()`函数返回任意不同种类对象的唯一ID

```python
name = "Yasoob"
print(id(name))
# Output: 139972439030304
```

### inspect模块

`inspect`模块也提供了许多有用的函数，来获取活跃对象的信息。比方说，你可以查看一个对象的成员，只需运行：

```python
import inspect
print(inspect.getmembers(str))
# Output: [('__add__', <slot wrapper '__add__' of ... ...
```

## 推导式Comprehension

推导式（又称解析式）是Python的一种独有特性，它是可以从一个数据序列构建另一个新的数据序列的结构体，共有三种推导：

- 列表(`list`)推导式

- 字典(`dict`)推导式

- 集合(`set`)推导式

### 列表推导式

列表推导式（又称列表解析式）提供了一种简明扼要的方法来创建列表。 它的结构是在一个中括号里包含一个**表达式**，然后是一个`for`语句，然后是0个或多个`for`或者`if`语句。**那个表达式可以是任意的，意思是你可以在列表中放入任意类型的对象**。返回结果将是一个新的列表，在这个以`if`和`for`语句为上下文的表达式运行完成之后产生。

```python
variable = [out_exp for out_exp in input_list if out_exp == 2]
```

例如：

```python
multiples = [i for i in range(30) if i % 3 is 0]
print(multiples)
# Output: [0, 3, 6, 9, 12, 15, 18, 21, 24, 27]
```

列表推导式可以很方便的创建一个列表，例如下面两种创建列表的方式：

```python
squared = []
for x in range(10):
	squared.append(x**2)

squared = [x**2 for x in range(10)]
```

### 字典推导式

其实和列表推导式是类似的：

```python
mcase = {'a':10, 'b':34, 'A':7, 'Z':3}

mcase_frequency = {
    # 对于每个key，将其小写作为新key，并将原字典内的大写和小写的值的和作为值
	k.lower(): mcase.get(k.lower(), 0) + mcase.get(k.upper(), 0)
	for k in mcase.keys()
}

# mcase_frequency == {'a': 17, 'z': 3, 'b': 34}
```

还可以快速的对换一个字典的键和值：

```python
{v: k for k,v in some_dict.items()}
```

### 集合推导式

和列表推导式类似，只不过它们使用大括号`{}`：

```python
squared = {x**2 for x in [1, 1, 2]}
print(squared)
# Output: {1, 4}
```

## 异常

最基本的术语里我们知道了`try/except`从句。可能触发异常产生的代码会放到`try`语句块里，而处理异常的代码会在`except`语句块里实现：

```python
try:
    file = open('test.txt', 'rb')
except IOError as e:
    print('An IOError occurred. {}'.format(e.args[-1]))
```

### 处理多个异常

我们有三种方法来处理多个异常。

- 把所有可能发生的异常放到一个元组里：

```python
try:
    file = open('test.txt', 'rb')
except (IOError, EOFError) as e:    # 所有异常放到一个元组里了
    print("An error occurred. {}".format(e.args[-1]))
```

- 对每个单独的异常在单独的`except`语句块中处理：

```python
try:
    file = open('test.txt', 'rb')
except EOFError as e:
    print("An EOF error occurred.")
    raise e
except IOError as e:
    print("An error occurred.")
    raise e
```

- 捕获所有异常：

```python
try:
    file = open('test.txt', 'rb')
except Exception:
    # 打印一些异常日志，如果你想要的话
    raise
```

### finally从句

包裹到`finally`从句中的代码不管异常是否触发都将会被执行。这可以被用来在脚本执行之后做清理工作。

```python
try:
    file = open('test.txt', 'rb')
except IOError as e:
    print('An IOError occurred. {}'.format(e.args[-1]))
finally:  # 即使发生异常也会执行finally块中语句
    print("This would be printed whether or not an exception occurred!")

# Output: An IOError occurred. No such file or directory
# This would be printed whether or not an exception occurred!
```

### try/else从句

我们常常想在没有触发异常的时候执行一些代码。这可以很轻松地通过一个`else`从句来达到。而如果`else`从句也抛出异常的话，是不会被`except`从句捕获的。

```python
try:
    print('I am sure no exception is going to occur!')
except Exception:
    print('exception')
else:
    # 这里的代码只会在try语句里没有触发异常时运行,
    # 但是这里的异常将 *不会* 被捕获
    print('This would only run if no exception occurs. And an error here '
          'would NOT be caught.')
finally:
    print('This would be printed in every case.')

# Output: I am sure no exception is going to occur!
# This would only run if no exception occurs.
# This would be printed in every case.
```

`else`从句只会在没有异常的情况下执行，**而且它会在`finally`语句之前执行**。

## lambda表达式

`lambda`表达式是一行函数。如果你不想在程序中对一个函数使用两次，你也许会想用lambda表达式，它们和普通的函数完全一样。看一下它的原型：

```python
lambda 参数:操作(参数)
```

看一些例子：

```python
add = lambda x, y : x + y  # 接受两个参数，并返回两者之和
```

列表排序：

```python
a = [(1,2), (4,1), (9,10), (13,-3)]
a.sort(key=lambda x : x[1])   # 对于每个元素，取它的第二个

print a
# Output: [(13, -3), (4, 1), (1, 2), (9, 10)]
```

列表并行排序？？？：

```python
data = zip(list1, list2)
data = sorted(data)
list1, list2 = map(lambda t: list(t), zip(*data))
```

## 一行式

看一些简单的例子，它们都是一些一行式的Python命令



简易Web Server：若要通过网络快速共享文件，可以进入到要共享文件的目录下并在命令行中运行下面代码：

```powershell
# Python 2
python -m SimpleHTTPServer

# Python 3
python -m http.server

Serving HTTP on 0.0.0.0 port 8000 ...
127.0.0.1 - - [28/Jun/2020 14:16:54] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [28/Jun/2020 14:16:54] code 404, message File not found
127.0.0.1 - - [28/Jun/2020 14:16:54] "GET /favicon.ico HTTP/1.1" 404 -
127.0.0.1 - - [28/Jun/2020 14:16:56] "GET /leetcode/ HTTP/1.1" 200 -
```



漂亮的打印：可以在Python REPL漂亮的打印出列表和字典：

```python
from pprint import pprint

my_dict = {'name': 'Yasoob', 'age': 'undefined', 'personality': 'awesome'}
pprint(my_dict)
```

这种方法在字典上更为有效。此外，如果你想快速漂亮的从文件打印出json数据，那么你可以这么做：

```python
cat file.json | python -m json.tool
```

当然还有很多一行式，可以参考[Python官方文档](https://wiki.python.org/moin/Powerful%20Python%20One-Liners)

## For - Else

`for`循环还有一个`else`从句，我们大多数人并不熟悉。**这个`else`从句会在循环正常结束时执行。这意味着，循环没有遇到任何`break` **。

有个常见的构造是跑一个循环，并查找一个元素。如果这个元素被找到了，我们使用`break`来中断这个循环。有两个场景会让循环停下来。

- 第一个是当一个元素被找到，`break`被触发。
- 第二个场景是循环结束。  

但我们并不知道到底是哪个原因让循环停止，当然可以设置一个标记来标识到底是哪个原因，另一个方法就是使用`else`从句：

```python
for item in container:
    if search_something(item):
        # Found it!
        process(item)
        break
else:   # 执行到这里说明for循环是正常结束，而不是被break掉
    # Didn't find anything..
    not_found_in_container()
```

例如下面的程序会找出2到10之间的数字的因子，但如果没有找到说明它是个质数，我们就可以利用一个`else`来实现没找到的逻辑：

```python
for n in range(2, 10):
    for x in range(2, n):
        if n%x == 0:
            print n, ' equals ', x, '*', n/x
            break
    else:   # 说明没有找到使n%x == 0的值
        print n, ' is a prime number'
```

## 使用C扩展？？



## Open函数

`open`函数可以打开一个文件



## 协程

Python中的协程和生成器很相似但又稍有不同。主要区别在于：

- 生成器是数据的生产者
- 协程则是数据的消费者

## 函数缓存

函数缓存允许我们**将一个函数对于给定参数的返回值缓存起来**。 当一个I/O密集的函数被频繁使用相同的参数调用的时候，函数缓存可以节约时间。 在Python 3.2版本以前我们只有写一个自定义的实现。

```python
from functools import wraps

def memoize(function):
    memo = {}
    @wraps(function)
    def wrapper(*args):
        if args in memo:
            return memo[args]
        else:
            rv = function(*args)
            memo[args] = rv
            return rv
    return wrapper

@memoize
def fibonacci(n):
    if n < 2: return n
    return fibonacci(n-1) + fibonacci(n-2)

print fibonacci(25)
```

能看出来，本质上还是使用了一个装饰器，只是再装饰器里面维护了一个`dict`，当某个参数对应的值没有被缓存起来先，先缓存起来再返回；否则直接返回已经缓存的数据。

## 上下文管理器

### 基于类的实现

一个上下文管理器的类，最起码要定义`__enter__`和`__exit__`方法。可以先尝试写一个开启文件的上下文管理器：

```python
class File(object):
    def __init__(self, file_name, method):
        self.file_obj = open(file_name, method)

    def __enter__(self):
        return self.file_obj

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.file_obj.close()
```

通过定义`__enter__`和`__exit__`方法，我们可以在`with`语句里使用它：

```python
with File('demo.txt', 'w') as opened_file:
    opened_file.write('Hola!')
```

我们的`__exit__`函数接受三个参数。这些参数对于每个上下文管理器类中的`__exit__`方法都是必须的。我们来谈谈在底层都发生了什么。

1. `with`语句先暂存了`File`类的`__exit__`方法
2. 然后它调用`File`类的`__enter__`方法
3. `__enter__`方法打开文件并返回给`with`语句
4. 打开的文件句柄被传递给`opened_file`参数
5. 我们使用`.write()`来写文件
6. `with`语句调用刚才暂存的`__exit__`方法
7. `__exit__`方法关闭了文件

### 处理异常

要注意到`__exit__`方法的这三个参数：`type`, `value`和`traceback`。

在第4步和第6步之间，如果发生异常，Python会将异常的`type`,`value`和`traceback`**传递给`__exit__`方法**。 它让`__exit__`方法来决定如何关闭文件以及是否需要其他步骤。所以我们来看一下当异常发生时，`with`语句会采取哪些步骤：

1. 把异常的`type`，`value`和`traceback`传递给`__exit__`方法
2. 让`__exit__`方法来处理异常
3. 如果`__exit__`返回的是True，那么这个异常就被优雅地处理了。
4. 如果`__exit__`返回的是**True以外的任何东西，那么这个异常将被`with`语句抛出**。

### 基于生成器的实现

我们还可以使用**装饰器**和**生成器**来实现上下文管理器。Python有个`contextlib`模块专门用于这个目的。我们可以使用一个生成器函数来实现一个上下文管理器，而不是使用一个类：

```python
from contextlib import contextmanager

@contextmanager
def open_file(name):
    f = open(name, 'w')
    yield f
    f.close()
```

让我们小小地剖析下这个方法。 

1. Python解释器遇到了`yield`关键字。因为这个缘故它创建了一个生成器而不是一个普通的函数。 
2. 因为这个装饰器，`contextmanager`会被调用并传入函数名（`open_file`）作为参数。 
3. `contextmanager`函数返回一个以`GeneratorContextManager`对象封装过的生成器。
4. 这个`GeneratorContextManager`被赋值给`open_file`函数，我们实际上是在调用`GeneratorContextManager`对象。

然后就可以使用这个新生成的上下文管理器了：

```python
with open_file('some_file') as f:
    f.write('hola!')
```

#### 实现原理



```python
class GeneratorContextManager(object):

    def __init__(self, gen):
        self.gen = gen

    def __enter__(self):
        try:
            return self.gen.next()
        except StopIteration:
            raise RuntimeError("generator didn't yield")

    def __exit__(self, type, value, traceback):
        if type is None:
            try:
                self.gen.next()   # 调用生成器的next()
            except StopIteration:
                return
            else:
                raise RuntimeError("generator didn't stop")
        else:
            if value is None:
                # Need to force instantiation so we can reliably
                # tell if we get the same exception back
                value = type()
            try:
                self.gen.throw(type, value, traceback)
                raise RuntimeError("generator didn't stop after throw()")
            except StopIteration, exc:
                # Suppress the exception *unless* it's the same exception that
                # was passed to throw().  This prevents a StopIteration
                # raised inside the "with" statement from being suppressed
                return exc is not value
            except:
                # only re-raise if it's *not* the exception that was
                # passed to throw(), because __exit__() must not raise
                # an exception unless __exit__() itself failed.  But throw()
                # has to raise the exception to signal propagation, so this
                # fixes the impedance mismatch between the throw() protocol
                # and the __exit__() protocol.
                #
                if sys.exc_info()[1] is not value:
                    raise


def contextmanager(func):
    @wraps(func)
    def helper(*args, **kwds):
        return GeneratorContextManager(func(*args, **kwds))
    return helper
```

