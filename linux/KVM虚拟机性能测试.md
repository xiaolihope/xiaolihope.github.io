性能测试工具

### 测试工具

性能分析工具:profiling

基准测试工具:benchmark

### 标准

评价一个系统给的性能标准:
- 响应时间:
- 吞吐量: 一次性能测试过程中网络上传数据量的总和
- 并发用户数
- 资源占用率: 客户端和服务器端物理资源占用情况


###  app 类型

CPU密集型 (eg. 科学计算)

网络I/O密集型 (eg. web服务)

磁盘I/O密集型(eg.数据库服务)

内存密集型 (eg. 缓存服务)

衡量KVM虚拟化性能最直接的方法就是:
将准备实施虚拟化的系统中运行的应用程序迁移到KVM虚拟客户机中试运行.
如果性能良好且稳定, 则可以考虑真正实施系统的虚拟化.


###  测试方法:

step1: 在非虚拟化的原生系统(Native)中执行某个基准测试程序

step2: 将该测试程序放到与原生系统配置相近的虚拟客户机中执行

step3: 对比在虚拟化和非虚拟化环境中该测试程序执行的性能. 实验环境: 修改宿主机上的参数.
原生系统是指物理机 客户机系统是指宿主机上的虚机. 两者的配置要一样!

CPU: tools: SPECCPU2006 SPECjbb2005 UnixBench SysBench link

www.spec.org 测试程序(CPU密集型):
    
[ubuntu tools]:http://www.howtogeek.com/111617/how-to-benchmark-your-linux-system-3-open-source-benchmarking-tools/
[kvm forum]:https://www.linux-kvm.org/page/KVM_Forum