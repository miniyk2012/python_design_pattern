# 全局对象模式

> <center>Verdict</center>
> 像其它几种脚本语言一样, Python将每个模块的外层解析为普通代码. 未缩进的赋值语句, 表达式, 甚至循环和条件语句都将在导入模块时执行. 这为用常量和数据结构来补充模块的类和函数提供了极好的机会, 调用者会发现它们很有用 -- 但这是个双刃剑: 可变的全局对象会与边缘代码发生耦合, 而 I/O 操作带来严重的导入耗时和副作用.
<br>

每个python模块都是隔离的命名空间. `json`模块可以提供`loads()`函数, 而不会与`pickle`模块的`loads()`函数冲突, 替换或覆盖.

隔离的命名空间对一门编程语言的易用性至关重要. 如果python的模块不隔离, 你将无法通过将注意力集中在你面前的模块上来阅读或编写Python代码 -- 一行代码可能会使用或意外地与标准库中其他地方定义的名字或你所安装的第三方模块冲突. 如果一个第三方模块的新版本定义了一个新的全局变量, 并与你的模块发生冲突, 那么升级该模块可能会破坏你的整个程序. 被迫在没有命名空间的语言中编码的程序员很快就会发现自己必须在全局名称中加入前缀, 后缀和额外的标点符号, 以避免落入命名发生冲突的绝望境地中.

尽管每个函数和类都是一个对象 -- 在Python中, 所有东西都是一个对象 -- 但模块全局模式更专指在模块的全局层面上被命名的一般对象实例.

有两种模式使用了模块全局值，其重要性足以另外写2篇文章来介绍:

https://python-patterns.guide/python/module-globals/#the-constant-pattern

* [预绑定方法](https://python-patterns.guide/python/prebound-methods/): 模块创建出对象并把对象的一个或多个方法赋值给模块的全局名称. 这些名称可以在今后被调用, 而不需要找到对象本身.

* [哨兵对象](https://python-patterns.guide/python/sentinel-object/): 尽管一些哨兵对象可以是类属性甚至可以是私有的或处于闭包里, 但标准库或其他三方库的大多数哨兵对象定义在模块的全局命名空间里并且可以被访问.

本文覆盖一些其他常见的全局对象例子.

## 常量模式

模块经常将有用的数字, 字符串和其他值赋给其全局范围内的名字. 标准库包括许多这样的赋值, 我们可以从中摘录几个例子.

```python
January = 1                   # calendar.py
WARNING = 30                  # logging.py
MAX_INTERPOLATION_DEPTH = 10  # configparser.py
SSL_HANDSHAKE_TIMEOUT = 60.0  # asyncio.constants.py
TICK = "'"                    # email.utils.py
CRLF = "\r\n"                 # smtplib.py
```

记住称之为"常量"只是说对象本身不可改变, 但名称仍然可以被重新分配.

```python
import calendar
calendar.January = 13
print(calendar.January)
```
```bash
13
```
名称也可以被删除:
```python
del calendar.January
print(calendar.January)
```
```bash
Traceback (most recent call last):
  ...
AttributeError: module 'calendar' has no attribute 'January'
```
除了整数、浮点数和字符串之外, 常量还包括不可变的容器, 如元祖和不可变集合:
```python
all_errors = (Error, OSError, EOFError)  # ftplib.py
bytes_types = (bytes, bytearray)         # pickle.py
DIGITS = frozenset("0123456789")         # sre_parse.py
```
更专门的不可变数据类型也可以作为常量:
```python
_EPOCH = datetime(1970, 1, 1, tzinfo=timezone.utc)  # datetime
```
在极少数情况下, 明显不打算修改的模块全局常量还是会使用可变数据结构. 在`frozenset`发明之前的代码中, 普通的可变集很常见. 例如至今仍在使用的字典, 因为标准库没有提供不可变字典.
```python
# socket.py
_blocking_errnos = { EAGAIN, EWOULDBLOCK }
```
```python
# locale.py
windows_locale = {
  0x0436: "af_ZA", # Afrikaans
  0x041c: "sq_AL", # Albanian
  0x0484: "gsw_FR",# Alsatian - France
  ...
  0x0435: "zu_ZA", # Zulu
}
```

常量通常是作为重构引入的: 程序员注意到相同的值`60.0`反复出现在他们的代码中, 因此为该值引入了一个常量`SSL_HANDSHAKE_TIMEOUT`. 现在, 每次使用这个名字都会付出在全局范围内进行搜索的轻微代价, 但这被一些优势所平衡. 常量的名称现在记录了值的含义, 提高了代码的可读性. 常量的赋值语句现在提供了一个单一的位置, 将来可以在那里编辑这个值，而不需要在代码中寻找每个使用`60.0`的地方.

这些优点是非常重要的, 有时甚至会为一个只使用一次的值引入常量, 将一个隐藏在代码深处的名字提升到全局可见.

有些程序员把常量赋值放在靠近使用它们的代码的地方; 有些程序员则把所有常量放在文件的顶部. 除非常量被放在离代码很近的地方，以至于人们总是能看到它, 否则把常量放在模块的顶部会更友好, 便于那些还没有把编辑器配置为支持跳转定义的读者参考.

另一种常量不是对内的 -- 针对模块本身的代码, 而是向外的 -- 作为模块暴露的API的一部分. 像`logging`模块中的`WARNING`这样的常量为调用者提供了常量的优势: 代码将更加可读, 常量的值可以在以后调整, 而不需要每个调用者动他们的代码.

你可能期望一个供模块自己使用而不非供调用者使用的常量总是以下划线开始, 以标记它为私有. 但是Python程序员在标记常量为私有的记号并不一致, 这也许是因为永远锁定一个常量比永远锁定一个辅助函数或类的API的代价要小.

## 导入的时间花销

Sometimes constants are introduced for efficiency, to avoid recomputing a value every time code is called. For example, even though math operations involving literal numbers are in fact optimized away in all modern Python implementations, developers often still feel more comfortable making it explicit that the math should be done at import time by assigning the result to a module global:

有时引入常量是为了提高效率，以避免每次调用代码时重新计算值. 例如, 尽管在所有的现代Python实现中, 涉及字面数字的数学运算实际上已经被优化掉了,  但开发者仍喜欢这样的数在导入时计算并赋值给一个模块全局值:

```python
# zipfile.py
ZIP_FILECOUNT_LIMIT = (1 << 16) - 1
```

当数学表达式很复杂时, 指定一个名称也能增强代码的可读性.

再比如, 存在着一些特殊的浮点值, 它们不能在Python中写成字面值; 它们只能通过向float类型传递一个字符串来生成. 为了避免每次需要这样的值时都用`'nan'`或`'inf'`来调用`float()`, 模块通常只创建一次这样的值并将其设置为模块全局值.

```python
# encoder.py
INFINITY = float('inf')
```

https://python-patterns.guide/python/module-globals/#the-constant-pattern

