
# 组合优于继承原则

源自于[四人帮](../index.rst)

> <center>Verdict</center>
> 在Python和其他编程语言中，这一伟大的原则鼓励软件架构师摆脱直面对象, 转而享受基于对象编程的简单实践.


在四人帮中, 该原则如此重要, 以至标为艺术字体:

*组合优于继承*

考虑一个设计问题，看看该原则是如何通过几个经典的四人帮设计模式来实现. 每个设计模式都装配简单的类, 不使用继承，它们被组装成一个优雅的运行时解决方案.

## 问题: 子类爆炸

继承作为一种设计策略有一个显著缺点, 一个类经常需要同时沿不同轴线的特殊化，这导致了四人帮在他们的Bridge章节中所说的"类的扩散"，以及在他们的Decorator章节中所说的"为支持各种组合导致子类的爆炸".

Python的标准库logging模块遵循了组合优于继承的原则，所以我们将其作为例子. 想象一下当需要增加新log发送目的地时, logger的子类逐渐增加的过程.

```python
import sys
import syslog

# The initial class.

class Logger(object):
    def __init__(self, file):
        self.file = file

    def log(self, message):
        self.file.write(message + '\n')
        self.file.flush()

# Two more classes, that send messages elsewhere.

class SocketLogger(Logger):
    def __init__(self, sock):
        self.sock = sock

    def log(self, message):
        self.sock.sendall((message + '\n').encode('ascii'))

class SyslogLogger(Logger):
    def __init__(self, priority):
        self.priority = priority

    def log(self, message):
        syslog.syslog(self.priority, message)
```

当这第一个设计轴被另一个设计轴所交织时，问题就出现了. 让我们想象一下，现在需要对日志信息进行过滤--一些用户只想看到带有"Error"的消息，于是开发一个Logger的子类:

```python
# New design direction: filtering messages.

class FilteredLogger(Logger):
    def __init__(self, pattern, file):
        self.pattern = pattern
        super().__init__(file)

    def log(self, message):
        if self.pattern in message:
            super().log(message)

# It works.

f = FilteredLogger('Error', sys.stdout)
f.log('Ignored: this is not important')
f.log('Error: but you want to see this')
```

陷阱已经埋下, 若应用日志不写入文件, 而是由socket发送, 并且需要过滤, 那么它就会迅速膨胀. 已存在的类没法覆盖这个case. 若开发者继续进行子类化，并创建一个结合了两个类特征的FilteredSocketLogger，那么子类爆炸就开始了. 

或许开发者很幸运, 因为没有更多的组合了. 但总的来讲, 应用会产生6个类:

```
Logger            FilteredLogger
SocketLogger      FilteredSocketLogger
SyslogLogger      FilteredSyslogLogger
```

类的数量会随着*m*和*n*几何级数增加. 这就是是"类的扩散"和"子类爆炸", 是四人帮想避免的问题.

解决办法是认识到一个既负责过滤消息又负责记录消息的类太复杂了. 在现代面向对象的实践中, 它将被指责为违反了"单一责任原则".

但是，我们怎样才能将消息过滤和消息输出这两个功能分布在不同的类中呢？

## 方法1: 适配器模式

一个解决方案是适配器模式：原本的logger类不需要改进，因为任何输出消息的机制都可以被包装成"文件"对象, 传给logger.

1. 因此我们保持原有的`Logger`.
2. 我们也保持`FilteredLogger`不变.
3. 不是创建专门的目的地子类, 而是创建不同输出目的地的"文件"适配器，然后将该适配器传递给logger.

下面是另外2种输出的适配器:

```python
import socket

class FileLikeSocket:
    def __init__(self, sock):
        self.sock = sock

    def write(self, message_and_newline):
        self.sock.sendall(message_and_newline.encode('ascii'))

    def flush(self):
        pass

class FileLikeSyslog:
    def __init__(self, priority):
        self.priority = priority

    def write(self, message_and_newline):
        message = message_and_newline.rstrip('\n')
        syslog.syslog(self.priority, message)

    def flush(self):
        pass
```

python鼓励鸭子类型, 因此适配器只需提供正确的方法--例如我们的适配器就免于继承它们所包装的类或模拟的文件类型. 它们也无须重新实现真正文件的全部十几个方法. 就像如果你只要鸭子叫，那么鸭子会不会走路并不重要, 我们的适配器只需要实现两个文件方法(译者注: write和flush), 它们是`Logger`真正使用的方法.

这样一来"子类爆炸"就避免了! Logger对象和适配器对象可以在运行时自由组合与匹配.

```python
sock1, sock2 = socket.socketpair()

fs = FileLikeSocket(sock1)
logger = FilteredLogger('Error', fs)
logger.log('Warning: message number one')
logger.log('Error: message number two')

print('The socket received: %r' % sock2.recv(512))
```

```bash
The socket received: b'Error: message number two\n'
```

注意，上面的`FileLikeSocket`类只是为了举例--在现实中, 该适配器是内置在Python的标准库中. 只需调用socket的[makefile()](https://docs.python.org/3/library/socket.html#socket.socket.makefile)方法, 就可以得到一个完整的适配器, 使套接字似乎是一个文件.

## 方法2: 桥接模式

桥接模式将一个类的行为分隔开来, 调用者看到的是外部"抽象"对象, 内部则是"实现"对象. 如果我们决定将过滤功能归为"抽象"类, 而输出归为"实现"类(也许有点武断), 那么就能将桥接模式应用到logging例子中.

就和适配器一样, 现在有一组单独的类来负责write. 但是不必再改变输出类来匹配Python文件对象的接口--这需要在logger添加一个换行, 有时还需要再次删除--我们现在可以自己定义封装类的接口.

So let’s design the inner “implementation” object to accept a raw message, rather than needing a newline appended, and reduce the interface to only a single method emit() instead of also having to support a flush() method that was usually a no-op.

因此让我们定义一个接收原始消息的内部实现, 不需要添加换行, 并自定义接口只有一个emit()方法而不需要flush()方法.

```python
# The “abstractions” that callers will see.

class Logger(object):
    def __init__(self, handler):
        self.handler = handler

    def log(self, message):
        self.handler.emit(message)

class FilteredLogger(Logger):
    def __init__(self, pattern, handler):
        self.pattern = pattern
        super().__init__(handler)

    def log(self, message):
        if self.pattern in message:
            super().log(message)

# The “implementations” hidden behind the scenes.

class FileHandler:
    def __init__(self, file):
        self.file = file

    def emit(self, message):
        self.file.write(message + '\n')
        self.file.flush()

class SocketHandler:
    def __init__(self, sock):
        self.sock = sock

    def emit(self, message):
        self.sock.sendall((message + '\n').encode('ascii'))

class SyslogHandler:
    def __init__(self, priority):
        self.priority = priority

    def emit(self, message):
        syslog.syslog(self.priority, message)
```

抽象对象和实现对象可以在运行时自由组合.

```python
handler = FileHandler(sys.stdout)
logger = FilteredLogger('Error', handler)

logger.log('Ignored: this will not be logged')
logger.log('Error: this is important')
```

```bash
Error: this is important
```

这比适配器更对称. 过去文件输出是Logger的原生部分, 若是非文件输出需要适配一个额外的类. 现在, 一个具备功能的logger总是由一个抽象和一个实现组合而成.

子类爆炸再一次被避免了, 因为两种类在运行时被组合在一起, 不需要任何一个类被扩展.

## 方法3: 装饰器模式

要是我们想给一个logger添加2个过滤器怎么办? 前面的方法都不支持多个过滤器, 即1个过滤优先级, 一个过滤关键字.

如果我们让过滤器和handler提供相同的接口，即都有一个log()方法，那么我们就得到了装饰器模式.

```python
class FileLogger:
    def __init__(self, file):
        self.file = file

    def log(self, message):
        self.file.write(message + '\n')
        self.file.flush()

class SocketLogger:
    def __init__(self, sock):
        self.sock = sock

    def log(self, message):
        self.sock.sendall((message + '\n').encode('ascii'))

class SyslogLogger:
    def __init__(self, priority):
        self.priority = priority

    def log(self, message):
        syslog.syslog(self.priority, message)

# The filter calls the same method it offers.

class LogFilter:
    def __init__(self, pattern, logger):
        self.pattern = pattern
        self.logger = logger

    def log(self, message):
        if self.pattern in message:
            self.logger.log(message)
```

过滤代码首次被移到任何特定的日志类之外. 相反，它现在是一个独立的功能，可以包裹任何我们想要的日志类.

和前两种方法一样, 过滤器可以在运行时与输出类组合, 而无需构建新的复杂的类.
```python
log1 = FileLogger(sys.stdout)
log2 = LogFilter('Error', log1)

log1.log('Noisy: this logger always produces output')

log2.log('Ignored: this will be filtered out')
log2.log('Error: this is important and gets printed')
```

```bash
Noisy: this logger always produces output
Error: this is important and gets printed
```

而且由于装饰器的对称性, 它们提供了与它们所包装类相同的接口 -- 我们可以把好多filter叠在一起作用在一个log上!

```python
log3 = LogFilter('severe', log2)

log3.log('Error: this is bad, but not that bad')
log3.log('Error: this is pretty severe')
```

```bash
Error: this is pretty severe
```

但是请注意这个设计的对称性会被打破: 虽然过滤器可以被叠在一起，但是输出过程不能被组合或堆叠. 日志消息仍然只有一个输出.

## 方法4: 四人帮之外的模式

python的logging模块想要更加灵活: 单个日志消息流不仅支持多个过滤器, 也支持多个输出. 基于其他语言中日志模块的设计 -- 见[PEP282](https://www.python.org/dev/peps/pep-0282/)的"Influences"部分的主要灵感 -- Python日志模块实现了自己的组合优于继承模式。

1. 调用者与之交互的Logger类本身并不实现过滤或输出. 相反它维护一个过滤器列表和一个处理器(handler)列表.
2. 对于每条日志消息, logger都会调用其每个过滤器. 如果有任何过滤器拒绝, 该消息就会被丢弃。
3. 对于每条被所有过滤器接受的日志消息, 日志记录器会遍历每个输出处理器, 要求每一个处理器发射该消息.

或者至少这就是其核心思想. 标准库的logging模块实际上更加复杂. 例如除了logger所有的过滤器外, 每个处理器可以携带自己的过滤器列表. 每个处理器还指定了一个最低的消息"级别", 比如INFO或WARN. 相当令人困惑的是, 这个级别既不是由处理器本身也不是由它的任何过滤器来执行, 而是由深埋在logger中的if语句来执行, 该logger是遍历这些处理器的. 因此整个设计有点混乱.

但我们可以借鉴标准库logging的基本思路 -- 一个logger的消息既有多个过滤器又有得多个输出 -- 来完全解耦过滤器类和处理器类.

```python
# There is now only one logger.

class Logger:
    def __init__(self, filters, handlers):
        self.filters = filters
        self.handlers = handlers

    def log(self, message):
        if all(f.match(message) for f in self.filters):
            for h in self.handlers:
                h.emit(message)

# Filters now know only about strings!

class TextFilter:
    def __init__(self, pattern):
        self.pattern = pattern

    def match(self, text):
        return self.pattern in text

# Handlers look like “loggers” did in the previous solution.

class FileHandler:
    def __init__(self, file):
        self.file = file

    def emit(self, message):
        self.file.write(message + '\n')
        self.file.flush()

class SocketHandler:
    def __init__(self, sock):
        self.sock = sock

    def emit(self, message):
        self.sock.sendall((message + '\n').encode('ascii'))

class SyslogHandler:
    def __init__(self, priority):
        self.priority = priority

    def emit(self, message):
        syslog.syslog(self.priority, message)
```

https://python-patterns.guide/gang-of-four/composition-over-inheritance/



