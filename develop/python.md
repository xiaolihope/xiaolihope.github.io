## yield
yield就是 return 返回一个值，并且记住这个返回的位置，下次迭代就从这个位置后开始。带yield关键字的函数是生成器。

## with...as
with ... as 是为了替换传统的 try... finally 
基本思想是with所求值的对象必须有一个__enter__()方法，一个__exit__()方法。
紧随with后面的语句被求值后，返回对象的_enter_()方法被调用。这个方法的返回值将赋值给as后面的变量。
当with后面的代码块被全部执行完之后，将调用前面返回对象的_exit_()方法。

## python生成器
yield
next
send()
close()
throw()

## python装饰器
使用场景：
- 操作日志记录
- 权限验证
- 函数参数类型验证
- 请求前数据格式转换
- 资源清理：数据的清理或添加

## [PyYAML](http://pyyaml.org/wiki/PyYAMLDocumentation)
PyYAML is a YAML parser and emitter for Python.

yaml.safe_dump 将一个python值转换为yaml格式文件

## python代码测试
- [tox](https://tox.readthedocs.io/en/latest/)
- [unittest framework](https://docs.python.org/2/library/unittest.html)
- [Testr](https://wiki.openstack.org/wiki/Testr)
- [smalling test](https://wiki.openstack.org/wiki/SmallTestingGuide)
- [mox test](http://blog.csdn.net/quqi99/article/details/8533071)

## items() and iteritems()
What is the difference of items() and iteritems()?
dict.items(): Return a copy of the dictionary’s list of (key, value) pairs.
dict.iteritems(): Return an iterator over the dictionary’s (key, value) pairs.

## python中pass的作用
- 空语句  do nothing
- 保证格式完整
- 保证语义完整

## python 中的 *args and **kwargs
这两个是python中的可变参数，用于函数参数个数不确定时。 `*args`表示任意多个无名参数. 它是一个tuple，用于容纳多个变量组成的list，
`**kwargs`表示关键字参数，它是一个dict。并且同时使用`*args`和`**kwargs`时,`*args`参数必须要在`**kwargs`前面，否则会报错: "SyntaxError: non-keyword arg after keyword arg"

## python垃圾回收机制
- 引用计数
- 标记清除，基于追踪回收（tracing GC）技术实现的垃圾回收算法。它分为两个阶段：第一阶段是标记阶段，GC会把所有的活动对象打上标记，第二阶段是把那些没有标记的对象非活动对象进行回收。
- 分代回收：分代回收是建立在标记清除技术基础之上的，是一种以空间换时间的操作方式。 Python将内存根据对象的存活时间划分为不同的集合，每个集合称为一个代，Python将内存分为了3“代”，
分别为年轻代（第0代）、中年代（第1代）、老年代（第2代），他们对应的是3个链表，它们的垃圾收集频率与对象的存活时间的增大而减小。新创建的对象都会分配在年轻代，年轻代链表的总数达到上限时，
Python垃圾收集机制就会被触发，把那些可以被回收的对象回收掉，而那些不会回收的对象就会被移到中年代去，依此类推，老年代中的对象是存活时间最久的对象，甚至是存活于整个系统的生命周期内。

## dict.get()
1. dict get 方法的应用
获取字典值，有两种方法：
1. 通过dict['key'] 
2. 通过dict.get() 方法
dict.get(key, default=None)
get()方法返回给定键的值。如果键不可用，则返回默认值None。
在这里我们可以设置默认值为其它，当key 不存在时，则返回默认值。（省去了判断）

## __init__.py
每个包下面都有一个__init__.py 文件. 这个文件定义了包的属性和方法. 也可以是一个空文件,什么也不包含,但是必须存在. 当将一个包作为一个模块导入的时候,实际上导入了它的__init__.py 文件.
当不包含此文件时, 仅仅就是普通个目录了,不能被导入或者包含其它的模块和嵌套包.
每个*.py文件可以作为一个module.当使用from package1 import *时,需要在__init__.py中增加 __all__=['module_1','module_2']


## __init__ and __new__
类在实例化的时候，先调用__new__,再调用__init__初始化.

python如何实现单例？
```
def __new__(cls, *args, **kwargs):
	if cls.__isinstance:  # 如果被实例化了
		return cls.__isinstance  # 返回实例化对象
	cls.__isinstance = object.__new__(cls)  # 否则实例化
	return cls.__isinstance  # 返回实例化的对象
```