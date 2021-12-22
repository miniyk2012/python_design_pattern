
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

或者至少这就是其核心思想. 标准库的logging模块实际上更加[复杂](https://docs.python.org/3/howto/logging.html#logging-flow). 例如除了logger所有的过滤器外, 每个处理器可以携带自己的过滤器列表. 每个处理器还指定了一个最低的消息"级别", 比如INFO或WARN. 相当令人困惑的是, 这个级别既不是由处理器本身也不是由它的任何过滤器来执行, 而是由深埋在logger中的if语句来执行, 该logger是遍历这些处理器的. 因此整个设计有点混乱.

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

请注意, 知道我们最后一个设计, 过滤器才真正以其应有的简单性闪亮登场. 这是首次它只接受一个字符串, 只返回一个判定. 之前所有的设计要么是将过滤功能隐藏在日志类中, 要么是让过滤器承担额外的职责, 而不是简单地执行一个判定. 

事实上，"log"这个词已经从过滤器类的名称中完全删除了, 而且有一个非常重要的原因: 它再也不与任何特定日志有关. `TextFilter`现在完全可以在任何涉及到字符串的情况下重用. 最后与日志的具体概念解耦, 过滤器变得更容易测试和维护.

同样, 就像所有组合优于继承的解决方案一样, 类在运行时组合，而不需要任何继承.

```python
f = TextFilter('Error')
h = FileHandler(sys.stdout)
logger = Logger([f], [h])

logger.log('Ignored: this will not be logged')
logger.log('Error: this is important')
```

```bash
Error: this is important
```

这里有一个关键的教训:像组合优于继承这样的设计原则, 最终要比Adapter或Decorator这样的个别模式更重要. 始终遵循原则, 而不要总是被限制在官方列表中选择模式. 我们现在得出的设计比以前的任何一个设计都更灵活, 也更容易维护, 尽管前面的设计是基于官方的四人帮模式, 但这个最终的设计却不是. 有时候, 是的, 你会发现一个现有的设计模式完全适合你的问题 -- 但如果不是, 那只要你比它们好就行.

## 诡计: "if"语句

我怀疑上面的代码让许多读者感到惊奇. 对于一个典型的Python程序员来说, 如此大量地使用类可能看起来完全是矫揉造作的 --这是一种试图使1980年代的旧观念与现代Python相关的尴尬做法.

当一个新的设计需求出现时, 典型的Python程序员真的会去写一个新的类吗？没有！"简单比复杂好". 如果一个if语句就可以, 为什么要增加一个类呢？单个记录器类可以逐渐增加条件, 直到它处理所有与我们之前的例子相同的情况.

```python
# Each new feature as an “if” statement.

class Logger:
    def __init__(self, pattern=None, file=None, sock=None, priority=None):
        self.pattern = pattern
        self.file = file
        self.sock = sock
        self.priority = priority

    def log(self, message):
        if self.pattern is not None:
            if self.pattern not in message:
                return
        if self.file is not None:
            self.file.write(message + '\n')
            self.file.flush()
        if self.sock is not None:
            self.sock.sendall((message + '\n').encode('ascii'))
        if self.priority is not None:
            syslog.syslog(self.priority, message)

# Works just fine.

logger = Logger(pattern='Error', file=sys.stdout)

logger.log('Warning: not that important')
logger.log('Error: this is important')
```

```bash
Error: this is important
```

你可能认识到这个例子是你在实际应用中遇到的比较典型的Python设计实践.

if语句的方法并非完全没有好处. 这个类的所有可能的行为都可以在从上到下阅读代码的过程中掌握. 参数列表可能看起来很冗长，但是由于Python的可选关键字参数, 对这个类的大多数调用不需要提供所有的四个参数.

(这个类确实只能处理一个文件和一个套接字, 但这是为了可读性而附带进行的简化. 我们可以很容易地将文件和套接字的参数转变为文件和套接字的列表).

鉴于每个Python程序员都能很快学会if, 但要花更多的时间来理解类, 依靠最简单的机制来实现一个功能, 似乎是一个明显的胜利. 但是让我们来平衡一下这种诱惑，明确说明躲避组合大于继承所带来的损失:

1. **局部性**. 重组代码以使用if语句, 对于可读性来说并不是一个无懈可击的胜利. 如果你的任务是改进或调试一个特定的功能 -- 比如说支持写入套接字 -- 你会发现你无法在一处地方阅读它的代码. 该单一功能背后的代码散落在初始化方法的参数列表, 初始化方法的实现和log()方法之间.

2. **可删除性**. 好的设计的一个未被重视的特性是它使删除功能变得容易. 也许只有大型和成熟的Python应用程序的老手才会强烈地体会到删除代码对项目健康的重要性. 在我们基于类的解决方案中, 我们可以通过删除SocketHandler类和它的单元测试, 在应用程序不再需要它的时候, 轻而易举地删除像记录日志到套接字的功能. 相比之下, 从if语句森林中删除socket功能不仅需要谨慎, 以避免破坏相邻的代码, 而且还提出了一个尴尬的问题: 如何处理初始化器中的socket参数. 它可以被删除吗？如果我们需要保持位置参数列表的一致性就不行 -- 我们需要保留这个参数, 但如果它被使用就会引发一个异常.

3. **死忘代码分析**. 与上一点有关的实事是, 当我们使用组合优于继承时, 死亡代码分析器可以轻松检测到代码库中最后一次使用SocketHandler的时间点. 但对于"你可以删除所有关于套接字输出的属性和if语句", 死亡代码分析往往无能为. 这是因为除了知道调用初始化方法时传的sock为None外, 对于其他我们一无所知.

4. **测试**. 我们的测试提供了关于代码健康状况的诸多信号, 其中一种最强烈的是, 在到达被测行之前要运行多少行不相关的代码. 如果测试可以简单地启动一个SocketHandler实例，传递给它一个开启的套接字，并要求它发出一个消息, 那么测试打日志到套接字这样的功能就很容易. 除了与该功能相关的代码外, 没有其他代码运行. 但是在我们的if语句森林中测试套接字日志, 至少要运行三倍的代码行数. 仅仅为了测试其中的一个功能，就必须正确组合多个特性来构造logger, 这是一个非常重要的警告信号, 在这个小例子中可能看起来微不足道, 但随着系统的扩大，这一点变得至关重要.

5. **效率**. 我故意把这一点放在最后, 因为可读性和可维护性通常是更重要的问题. 但是if语句森林的设计问题还表现在该方法的低效率上. 即使你想要简单的把未经过滤的日志输出到一个文件中, 每一条消息也都将被迫运行所有可能使用到的特性的if语句. 相比之下, 组合技术只运行你所组装的功能的代码.

由于所有这些原因, 我表明了从软件设计的角度来看, if语句森林的表面简单性在很大程度上是一种错觉. 将logger的代码自上而下阅读的能力, 是以其他几种概念上的花费为代价的, 这些花费会随着代码库的大小而急剧增长.

## 诡计: 多继承

Some Python projects fall short of practicing Composition Over Inheritance because they are tempted to dodge the principle by means of a controversial feature of the Python language: multiple inheritance.

一些Python项目没有实践"组合优于继承"原则, 因为它们想通过Python语言的一个有争议的特性来回避这个原则: 多重继承。

让我们回到我们最初的例子, `FilteredLogger`和`SocketLogger`是基类`Logger`的两个不同子类. 在一个只支持单一继承的语言中, `FilteredSocketLogger`将不得不选择从SocketLogger或FilteredLogger继承, 然后不得不重复另一个类的代码.

但python支持多重继承, 因此新`FilteredSocketLogger`类可以同时继承`SocketLogger`h和`FilteredLogger`.

```python
# Our original example’s base class and subclasses.

class Logger(object):
    def __init__(self, file):
        self.file = file

    def log(self, message):
        self.file.write(message + '\n')
        self.file.flush()

class SocketLogger(Logger):
    def __init__(self, sock):
        self.sock = sock

    def log(self, message):
        self.sock.sendall((message + '\n').encode('ascii'))

class FilteredLogger(Logger):
    def __init__(self, pattern, file):
        self.pattern = pattern
        super().__init__(file)

    def log(self, message):
        if self.pattern in message:
            super().log(message)

# A class derived through multiple inheritance.

class FilteredSocketLogger(FilteredLogger, SocketLogger):
    def __init__(self, pattern, sock):
        FilteredLogger.__init__(self, pattern, None)
        SocketLogger.__init__(self, sock)

# Works just fine.

logger = FilteredSocketLogger('Error', sock1)
logger.log('Warning: not that important')
logger.log('Error: this is important')

print('The socket received: %r' % sock2.recv(512))
```

```bash
The socket received: b'Error: this is important\n'
```

这与我们的装饰器模式解决方案有几处惊人的相似之处: 

* 每种输出都有一个logger类(而不是适配器模式那样, 直接写文件和通过适配器不写文件这样的不对称).
* 消息保留了由调用者提供的精确值(而不是适配器那样习惯于通过添加换行来替换为文件特定值).
* 过滤器和记录器是对称的，因为它们都实现了相同的方法`log()`(除了装饰器模式外, 我们的其他解决方案是过滤器类提供一种方法, 而输出类提供另一种方法).
* 过滤器不尝试自己产生输出, 但如果一条消息在过滤过程中幸存下来, 就会把输出的任务推迟到其他代码.

这些与我们先前的装饰器解决方案的密切相似性意味着我们可以将其与新代码进行比较，从而在组合和多重继承之间进行异常鲜明的比较. 让我们用一个问题进一步突出重点:

如果我们做单元测试, 那么我们有多大把握能对logger和过滤器想做彻底的测试?

1. 装饰器例子的成功是由于只依赖每个类的公共行为: `LogFilter`提供了一个`log()`方法, 该方法反过来调用它所包装的对象上的log()(测试可以用一个假记录器简单地做验证), 这些对象也都有可工作的`log()`方法. 只要我们的单元测试验证了这两个公共行为，我们就能确保组合后的行为正确.

    相比之下, 多重继承依赖的那些行为, 无法通过简单实例化类来验证. `FilteredLogger`的公共行为是, 它提供了一个`log()`方法, 既能过滤又能写入文件. 但多重继承实例并不仅仅取决于该公共行为, 而是取决于该行为在内部是如何实现的. 尽管下面两种实现都能做单元测试, 但如果用`super()`那么`log()`方法会延迟到它的基类来调用, 那么多重继承将起作用, 反之该方法自己`write()`到文件(译者注: 但多重继承后未必能保证行为正确).

    因此，测试套件必须超越单元测试, 在多重继承的类上测试 -- 或者用monkey patch来验证`log()`调用了`super().log()` -- 以保证多重继承在为了的开发中继续正常工作.

2. 


https://python-patterns.guide/gang-of-four/composition-over-inheritance/



