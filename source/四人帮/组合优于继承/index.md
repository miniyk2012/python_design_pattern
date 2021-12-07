
# 组合优于继承原则

源自于[四人帮](../index.rst)

> <center>Verdict</center>
> 在Python和其他编程语言中，这一伟大的原则鼓励软件架构师摆脱面向对象，转而享受基于对象编程的简单实践.

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

https://python-patterns.guide/gang-of-four/composition-over-inheritance/


