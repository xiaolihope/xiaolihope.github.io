## Screen 用法
GNU Screen是一款由GNU计划开发的用于命令行终端切换的自由软件。 用户可以通过该软件同时连接多个本地或远程的命令行会话，并在其间自由切换。
```
screen -ls -> 列出当前所有的session
screen -r yourname -> 回到yourname这个session
screen -r 6568//进入screen
screen -d yourname -> 远程detach某个session
screen -d -r yourname -> 结束当前session并回到yourname这个session
ctrl -a ctrl-a 连续2遍,可以进入下一个window
或者ctrl-a +[0-9]顺序切换window
ctrl-a d 退出screen
screen -x 会话共享
ctrl-a n  切换到下一个 window
ctrl-a p  切换到前一个 window
ctrl-a k   强行关闭当前的 window
ctrl-a [ 进入copy mode 可以滚动窗口

1. 在SSH里输入screen -S name(name可以自己随便取，这个name代表你将要创建的session的名称)
2. 回车后就会创建一个新的session，在这个session里你可以敲入命令运行你想要运行的程序
3. 按下ctrl+a+d, 你会看到你正在运行的程序被detach了，也就是被扔到后台去执行了，而你也回到了screen之前的状态。这时你可以logout SSH，然后放心去爱干嘛干嘛了。
4. 等我回来重新login SSH后，输入screen -r name
```
- http://www.cnblogs.com/mchina/archive/2013/01/30/2880680.html
- https://www.ibm.com/developerworks/cn/linux/l-cn-screen/
- http://www.cnblogs.com/mchina/archive/2013/01/30/2880680.html
- http://blog.csdn.net/frankyanchen/article/details/7886502
- http://blog.csdn.net/flying881114/article/details/5912990

SSH使用screen的时候出现了如下错误:Cannot open your terminal '/dev/pts/0' - please check.  
可以使用script命令来记录这个终端会话,
执行`script /dev/null`

这只是一个简单的用法介绍，screen命令的详细介绍可以参照：
http://bjzero.blogbus.com/logs/30983025.html
 
两种kill session的方法：
1. attach to session之后，按ctrl+a （屏幕上暂时不会有任何反应），接着输入冒号“:”，然后输入quit， 回车，此session就被terminate了。
2. 如果想直接kill detached session, 输入命令
screen -S session_name -X quit
```
c-b Backward, PageUp
c-f Forward, PageDown
c-a [ 进入copy mode. 
Space  第一次按为标记区起点，第二次按为终点. 
Esc 结束 copy mode.
c-a ] Paste. 把刚刚在copy mode 选定的内容粘上
```
后台运行：
`nohup <command> [argument…] &`
