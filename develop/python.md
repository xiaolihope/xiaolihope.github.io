## with...as
with ... as 是为了替换传统的 try... finally 
基本思想是with所求值的对象必须有一个__enter__()方法，一个__exit__()方法。

## yaml.sefe_dump

link : http://www.programcreek.com/python/example/3806/yaml.safe_dump
http://pyyaml.org/wiki/PyYAMLDocumentation

## unittest

openstack单元测试用组件一览:  http://blog.csdn.net/halcyonbaby/article/details/30344441
unittest framework: https://docs.python.org/2/library/unittest.html
python -m unittest -v test_module

testr: https://wiki.openstack.org/wiki/Testr

smalling test : https://wiki.openstack.org/wiki/SmallTestingGuide
mox test:        http://blog.csdn.net/quqi99/article/details/8533071

python34:
stack@devstack-test01:~/horizon$ sudo apt-get install -y python3.4-dev
stack@devstack-test01:~/horizon$ tox -epy34 -- notests

db type could not be determined
Just hit this. Used workaround:
 1299 rm .testrepository/times.dbm
 1300 tox -e py34
 1301 tox -e py27
                                              
run python34 test(tox -e py34):
error:
#include "pyconfig.h"
fix method:
sudo apt-get install python3-dev
sudo apt-get install python-dev

## python中的迭代器和生成器
http://stackoverflow.com/questions/10458437/what-is-the-difference-between-dict-items-and-dict-iteritems
What is the difference of items() and iteritems()?
dict.items(): Return a copy of the dictionary’s list of (key, value) pairs.
dict.iteritems(): Return an iterator over the dictionary’s (key, value) pairs.

## python中保留子
1. and 
2. exec
3. not
4. assert
5. finally
6. or
7. break
8. for
9. pass 
   
python 中pass的作用：
1） 空语句  do nothing
2） 保证格式完整
3） 保证语义完整
eg:
def test():
     pass
class Test():
     pass
10.  class
11. from 
12. print
13. continue
14. global
python 中可以通过global 将变量定义为全局变量(可以在函数内部改变变量值)。一个global语句可以同时定义多个变量。eg: global x,y,z
15. raise
16. def
17. if
18. return 
19. del
del 用于list 列表操作. 删除一个或者连续几个元素。 
>>> a = [1,2,3,4]
>>> a
[1, 2, 3, 4]
>>> del a[0]
>>> a
[2, 3, 4]
>>> del a[2:4]  #删除从第2个元素开始，到第四个元素为止的元素。包括头包括尾。
>>> a
[2, 3]
>>> del a
>>> a
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
NameError: name 'a' is not defined

20. import 
21. try
22. elif
23. in 
24. while
25. else
26. is 
27. with
with open("/tmp/foo.txt")
    as file:
        data = file.read()
  
with 所求值的对象必须有一个 _enter_()方法和一个_exit_()方法。
紧随with后面的语句被求值后，返回对象的_enter_()方法被调用。这个方法的返回值将赋值给as后面的变量。
当with后面的代码块被全部执行完之后，将调用前面返回对象的_exit_()方法。
http://blog.csdn.net/suwei19870312/article/details/23258495
http://zhoutall.com/archives/325

28. except 
1） try... excepe
eg: 
try: 
     a=b
     b=c
except Exception,ex:
     print Exception,":",ex

2） try... finally
try:
     block
finally: 
     block
这种语句不管try 有没有异常发生，都会执行finally 语句
3） try ... raise ... except
try:
     raise MyError
except MyError:
     print 'a error'
raise ValueError,'invalid argument'
4）用traceback 跟踪模块查看异常
5）采用sys模块回溯最后的异常
     
eg.
import sys
try:
     block
except:
     info=sys.exc_info()
     print info[0],":"info[1]
or
import sys
     tp,val,td=sys.exc_info()
sys.exc_info()返回值是一个tuple(type,value/message,traceback)


## python 中的 *args and **kwargs
这两个是python中的可变参数. *args表示任意多个无名参数. 它是一个tuple;
                                            **kwargs 表示关键字参数. 它是一个dict. 
并且同时使用*agrs和**kwargs时,*args参数必须要在**kwargs前面. 否则会报错: " SyntaxError: non-keyword arg after keyword arg "
当函数的参数不确定时，可以使用*args 和**kwargs，*args 没有key值，**kwargs有key值
*args可以当作可容纳多个变量组成的list


## python learning
1. Environment Setup in Windows
1) path : c:\python26
2)此时，还是只能通过"python *.py"运行python脚本，若希望直接运行*.py，只需再修改另一个环境变量PATHEXT:.PY;.PYM
3) 另外，在使用python的过程中，可能需要经常查看某个命令的帮助文档，如使用help('print')查看print命令的使用说明。默认安装的python无法查看帮助文档，尚需进行简单的配置：在python安装目录下，找到python25.chm，使用hh -decompile . python26.chm. 将其反编译出来，然后将其所在的目录加入到上面提到的PATH环境变量中即可。
4) 如何使Python解释器能直接import默认安装路径以外的第三方模块？为了能import默认安装路径以外的第三方的模块（如自己写的模块），需要新建PYTHONPATH环境变量，值为这个模块所在的目录.
2.Python Configuration in Eclipse
Help-->Install New Software.
http://pydev.org/updates 
1. 胶水语言
2. 框架：web django
3. 字节码：bytecode
4. 系统编写好的语句的程序文件称为：“模块”；能够直接执行的模块称为脚本（即程序的顶层文件,整个程序的人口）
5. 每一个文件可以叫做一个模块，可以被他们调用或者导入。
#！/usr/bin/python
import 
print platform.uname()
6. python程序可以分解为：模块、语句、表达式、和对象。
程序由模块（程序文件）构成
python中一切皆对象。
7. 面向过程：以指令为中心，由指令处理数据；如何组织代码解决问题。
    面向对象：以数据为中心，所有的处理代码都围绕数据展开：如何组织数据，如何设计数据结构，并且对此类数据多允许的处理操作。
python两者都支持。
dir(platform) 显示此模块内置的方法。
8. 安装方法
    1） 编译安装新版本到指定目录 
            configure --prefix
            make & ... 
            ipython ---是一个module 提供tab提示补全的功能。 ln -sv 
    2） pyenv (非常方便，install uninstall etc.)
http://blog.csdn.net/tiantiandjava/article/details/17242345  python install（升级旧版本）
Q：怎样更新Python版本？ 
1. 下载python2.7.5，保存到 /data/qtongmon/software
http://www.python.org/ftp/python/
2. 解压文件
tar xvf Python-2.7.5.tar.bz2
3. 创建安装目录
mkdir /usr/local/python27
4. 安装python
./configure --prefix=/usr/local/python27
make
make install
5. 修改老版本的ln指向（注意：这里修改后，可能会影响yum的使用）
mv /usr/bin/python /usr/bin/python2.4.3
ln -s /usr/local/python27/bin/python /usr/bin/python
安装成功！ 
9. 数据结构+算法=程序
python中最基本的数据结构是：序列（索引为编号），python有6中内建序列。
1） 列表
2）元组
3）字符串 (单双引号一样，必须用引号引起来)--不可变类型
id(name)  变量name在内存中的地址
type(name) 获得name的类型。
10. 对象引用（变量）
    python中将所有的数据存为内存对象。
    python中，变量实际上是指向内存对象的引用。变量也是保存在内存中。 （与java很相似，当内存对象的引用计数为0时，由垃圾回收机制回收。）
python中“=”用于将变量名与内存中的某对象绑定：如果对象事先存在，就直接进行绑定（一个内存空间可以有多个变量引用。）；否则，则有“=”创建引用的对象。
11. 变量命名规则
    区分字母大小写。 
命名惯例：
1）_x  不会被from module import * 语句导入
2） _x_ 是系统定义的变量名，对python解释器有特殊的意义。
3） __x 是类的本地变量。
4） 在交互模式下，_ 表示最近表达式的值。
变量名没有类型，对象才有类型。
12. 组合数据类型
    1） 序列类型：
        1）列表  [] --可变序列
        2）元组 () --不可变序列   （事实上，列表和元组并不真正存储数据，而是存放对象引用。）
        3）字符串  --可以切片
    2）集合
        没有次序
    3）字典 
        键值对。
len()获取长度
python对象可以具有其可以被调用的特定方法（函数）。
        
13. 逻辑操作符
    1）身份操作符
        is: 判定左端对象引用是否相同于右端对象引用。也可以与none进行比较
    2）比较
    3）成员操作符
        in  notin
14. 控制语句
    1) if  : 
        else:
    2) while :
    3) for ... in:
    
python中int类型是不可变的。
15. 输入输出
    pyton3: print()
    python2: print语句
    input()
    raw_input() --优先用这个。 a=raw_input()
    print "%d %f ..." %(variable1,vailable2,...)
    dir(__builtins__) #显示内置的函数或者命令。。。
    help(str) 
16. 函数的创建与调用。
    def functionName():
        suite
eg:
def printName(name):
    print(name)


## python pep8
Imports should be grouped in the following order:
1. standard library imports
2. related third party imports
3. local application/library specific imports
You should put a blank line between each group of imports.

## dict
1. dict get 方法的应用
获取字典值，有两种方法：
1. 通过dict['key'] 
2. 通过dict.get() 方法
dict.get(key, default=None)
get()方法返回给定键的值。如果键不可用，则返回默认值None。
在这里我们可以设置默认值为其它，当key 不存在时，则返回默认值。（省去了判断）
eg:
info = {'1':'first','2':'second','3':'third'}
number = raw_input('input type you number:')
print info.get(number,'error')
当输入为1,2,3之外的数字时，则返回error

## __init__.py
1. 每个包下面都有一个__init__.py 文件. 这个文件定义了包的属性和方法. 也可以是一个空文件,什么也不包含,但是必须存在. 当将一个包作为一个模块导入的时候,实际上导入了它的__init__.py 文件. 
当不包含此文件时, 仅仅就是普通个目录了,不能被导入或者包含其它的模块和嵌套包.
每个*.py文件可以作为一个module.
当使用from package1 import *时,需要在__init__.py中增加 __all__=['module_1','module_2']
