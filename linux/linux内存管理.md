copy from http://bbs.chinaunix.net/thread-2018659-1-1.html
ref link: http://blog.chinaunix.net/uid-26611383-id-3761754.html 
好文！
这里引用计算机界一句无从考证的名言：“计算机系统里的任何问题都可以靠引入一个中间层来解决。”

ref link: http://www.cnblogs.com/autum/archive/2012/10/12/linuxmalloc.html
buffer and cache: http://www.linuxeye.com/Linux/1932.html
http://www.cnblogs.com/zhaoyl/p/3695517.html
http://www.linuxeye.com/Linux/1931.html

Linux 内存组成:
virtual memory
虚拟内存
SWAP
虚拟内存就是采用硬盘来对物理内存进行扩展，将暂时不用的内存页写到硬盘上而腾出更多的物理内存让有需要的进程来用。当这些内存页需要用的时候在从硬盘读 回内存。这一切对于用户来说是透明的。通常在Linux系统说，虚拟内存就是swap分区。在X86系统上虚拟内存被分为大小为4K的页。
host memory
物理内存
RAM
每一个进程启动时都会向系统申请虚拟内存（VSZ），内核同意或者拒就请求。当程序真正用到内存时，系统就它映射到物理内存。RSS表示程序所占的物理内存的大小。用ps命令我们可以看到进程占用的VSZ和RSS
cache/buffer
buffer: 磁盘缓存的大小
cache: 用于cpu 和内存直接的缓冲
difference: http://www.lxway.com/485614091.htm
swap

SLAB:
http://blog.csdn.net/rockrockwu/article/details/8005763
