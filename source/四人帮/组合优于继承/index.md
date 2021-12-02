
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

类的数量会随着*m*和*n*几何级数增加. 这就是是"类的扩散"和"子类的爆炸", 是四人帮想避免的问题.

The solution is to recognize that a class responsible for both filtering messages and logging messages is too complicated. In modern Object Oriented practice, it would be accused of violating the “Single Responsibility Principle.”

But how can we distribute the two features of message filtering and message output across different classes?

解决办法是认识到一个既负责过滤消息又负责记录消息的类太复杂了. 在现代面向对象的实践中, 它将被指责为违反了"单一责任原则".

但是，我们怎样才能将消息过滤和消息输出这两个功能分布在不同的类中呢？

## 方法一: 适配器模式

https://python-patterns.guide/gang-of-four/composition-over-inheritance/