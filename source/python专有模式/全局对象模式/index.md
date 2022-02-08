# 全局对象模式

> <center>Verdict</center>
> 像其它几种脚本语言一样, Python将每个模块的外层解析为普通代码. 未缩进的赋值语句, 表达式, 甚至循环和条件语句都将在导入模块时执行. 这为用常量和数据结构来补充模块的类和函数提供了极好的机会, 调用者会发现它们很有用 -- 但这是个双面刃: 可变的全局对象会与边缘代码发生耦合, 而 I/O 操作带来严重的导入耗时和副作用.

每个python模块都是隔离的命名空间. `json`模块可以提供`loads()`函数, 而不会与`pickle`模块的`loads()`函数冲突, 替换或覆盖.

隔离的命名空间对一门编程语言的易用性至关重要. 如果python的模块不隔离, 你将无法通过将注意力集中在你面前的模块上来阅读或编写Python代码 -- 一行代码可能会使用或意外地与标准库中其他地方定义的名字或你所安装的第三方模块冲突. 如果一个第三方模块的新版本定义了一个新的全局变量, 并与你的模块发生冲突, 那么升级该模块可能会破坏你的整个程序. 被迫在没有命名空间的语言中编码的程序员很快就会发现自己必须在全局名称中加入前缀, 后缀和额外的标点符号, 以避免落入命名发生冲突的绝望境地中.

尽管每个函数和类都是一个对象 -- 在Python中, 所有东西都是一个对象 -- 但模块全局模式更专指在模块的全局层面上被命名的一般对象实例.

有两种模式使用了模块全局值，其重要性足以分2篇文章来介绍:

https://python-patterns.guide/python/module-globals/#the-constant-pattern

Prebound Methods are generated when a module builds an object and then assigns one or more of the object’s bound methods to names at the module’s global level. The names can be used to call the methods later without needing to find the object itself.
While a Sentinel Object doesn’t have to live in a module’s global namespace — some sentinel objects are defined as class attributes, while others are private and live inside of a closure — many sentinels, both in the Standard Library and beyond, are defined and accessed as module globals.

