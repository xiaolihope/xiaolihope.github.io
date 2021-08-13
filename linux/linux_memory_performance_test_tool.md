几个内存性能测试工具

## 1. memtester  memtest86+
压力测试; 

捕获内存错误和一直处于很高或者很低的坏位.  

进行的项目有: 随机值\异或比较\减法\乘法\除法\与或 给定测试内存的大小和次数.

## 2. LMbench

## 3. STREAM
内存带宽测试。

进行4种操作的测试: add, copy, triad(加/减/乘 结合起来), scale (乘法运算)

## 4. stress 
`apt-get install stress (a tool that can make the cpu stress to 100%)`

## 5. mbw
用来测试应用程序进行内存拷贝操作所能达到的带宽.
 
