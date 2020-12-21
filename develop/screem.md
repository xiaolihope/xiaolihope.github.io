GNU Screen是一款由GNU计划开发的用于命令行终端切换的自由软件。 用户可以通过该软件同时连接多个本地或远程的命令行会话，并在其间自由切换。
screen -ls -> 列出当前所有的session
screen -r yourname -> 回到yourname这个session
screen -d yourname -> 远程detach某个session
screen -d -r yourname -> 结束当前session并回到yourname这个session
http://www.cnblogs.com/mchina/archive/2013/01/30/2880680.html
https://www.ibm.com/developerworks/cn/linux/l-cn-screen/

http://www.cnblogs.com/mchina/archive/2013/01/30/2880680.html
http://blog.csdn.net/frankyanchen/article/details/7886502
http://blog.csdn.net/flying881114/article/details/5912990

screen -ls  //查看当前的有多少个screen
screen -r 6568//进入screen
ctrl -a ctrl-a 连续2遍,可以进入下一个window
或者ctrl-a +[0-9]顺序切换window
ctrl-a d 退出screen
screen -x 会话共享
ctrl-a n  切换到下一个 window
ctrl-a p  切换到前一个 window
ctrl-a k   强行关闭当前的 window
ctrl-a [ 进入copy mode 可以滚动窗口
SSH使用screen的时候出现了如下错误:Cannot open your terminal '/dev/pts/0' - please check.  
可以使用script命令来记录这个终端会话,
执行script /dev/null
今天用SSH登录学校的服务器下载数据（其实是从一台服务器转移数据到另一台服务器），因为数据量很大，估计个把小时才能下完，但是我马上就要去吃饭上课了，同组的学长就告诉我用screen这个命令。
大概意思就是：
1. 在SSH里输入screen -S name(name可以自己随便取，这个name代表你将要创建的session的名称)
2. 回车后就会创建一个新的session，在这个session里你可以敲入命令运行你想要运行的程序，in my case，就是ftp下载
3. 好了，现在数据已经在下载了，但是我马上要有事离开了，为了让下载继续进行而不会断开，按下ctrl+a+d, 你会看到你正在运行的程序被detach了，也就是被扔到后台去执行了，而你也回到了screen之前的状态。这时你可以logout SSH，然后放心去爱干嘛干嘛了。
4. 等我回来重新login SSH后，输入screen -r name，你就会回到这个下载session；你会看到数据在这段时间内又下载了很多，或者干脆已经下载完啦！
 
这只是一个简单的用法介绍，screen命令的详细介绍可以参照：
http://bjzero.blogbus.com/logs/30983025.html
 
 补充：
像我这种懒人，每次都忘记kill掉已经没用的screen session，结果一段时间之后就有一堆稀奇古怪名称的session，记也记不住。（可以用screen -ls 查看所有的session）
以下是两种kill session的方法：
1. attach to session之后，按ctrl+a （屏幕上暂时不会有任何反应），接着输入冒号“:”，然后输入quit， 回车，此session就被terminate了。
2. 如果想直接kill detached session, 输入命令
screen -S session_name -X quit

c-b Backward, PageUp
c-f Forward, PageDown
c-a [ 进入copy mode. 
Space  第一次按为标记区起点，第二次按为终点. 
Esc 结束 copy mode.
c-a ] Paste. 把刚刚在copy mode 选定的内容粘上

后台运行：
nohup <command> [argument…] &
