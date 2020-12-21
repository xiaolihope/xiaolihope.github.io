---
date: 2017-11-08T09:50:00+08:00
title: 基于时间序列数据的存储
---

## 时间序列（time series）数据

时间序列（time series）数据是一系列有序的数据。通常是等时间间隔的采样数据。时间序列存储最简单的定义就是数据格式里包含timestamp字段的数据。
时间序列数据在查询时，对于时间序列总是会带上一个时间范围去过滤数据。同时查询的结果里也总是会包含timestamp字段。

## 时间序列数据如何存储
时间序列数据的难点是实时数据量庞大，写入速度需要特别快。

## Borgmon
谷歌的监控组件Borgmon在内存中设计了一个基于内存的数据库，这个数据库存储设计的每一个数据点包含一对键值(timestamp, value)，
并且数据点按照时间轴一次排列形成一维列表，通过加入以name=value 汇聚的标签来排列组合，把时间序列数据变为多维数据表结构如下图。
在实践中，数据会被存在一个固定大小的Borgmon内存中，通过定期的垃圾收集机制清除过时的时间序列数据。

注意，内存中的数据大小正好被有意设计为可以实时查询的时间区间。对于谷歌数据中心来说，其时间区间是12小时，
需要17GB 大小的内存来存储数据点为24字节，每间隔1分钟收集，总存储量位一百万个数据点。Borgmon定期的把内存中过期的数据清除出来并存到时间数据库(TSDB)中。
Borgmon是支持查询TSDB 数据库的。

Screen Shot 2017-03-15 at 10.53.40.png

## Prometheus

Prometheus 是SoundCloud开发的开源的系统监控报警工具集。
数人云就用到了此开源组件，他们的监控报警架构如图：

OpenStack研发 > 基于时间序列数据的存储 > Screen Shot 2017-03-15 at 10.45.00.png

cAdvisro 作为一个后台运行的程序，收集，聚合，处理并导出容器运行时的信息。

Prometheus 从 cAdvisor的HTTP接口采集容器运行时的信息，保存在内部的存储里，利用PromQL对时序数据进行查询展示和设置报警。报警信息推送到 Alertmanager

Alerta 是一个用户友好的报警可视化展示工具，用于展示和管理从Alertmanager推送过来的报警数据。

## Prometheus 架构

OpenStack研发 > 基于时间序列数据的存储 > Screen Shot 2017-03-15 at 13.53.37.png

## 关于Prometheus 的一些实践：
Prometheus: A practical guide to alerting at scale
A practical guide to monitoring and alerting with time series at scale

##TSDB List
关于时间序列数据库有很多：

- DalmatinerDB
- InfluxDB
- Prometheus
- Riak-TS
- OpenTSDB
- KairosDB
- Elasticsearch
- Druid
- Blueflood
- Graphite (Whisper)
- Atlas
- KairosDB
- RRDtool
- Chronix odsc-chronix-afastandefficienttimeseriesstoragebasedonapachesolr-160614101631.pdf

## 参考

- SRE工程实践 | 基于时间序列数据的报警是一种怎样的体验？
- SRE系列教程 | 基于时间序列数据的监控实践
- https://prometheus.io/
- https://github.com/prometheus/prometheus
- Time Series DB List

# SSDB

## SSDB介绍及特性
SSDB是一个 NoSQL 数据库, 用于替代 Redis。

### 特性

- 替代 Redis 数据库, Redis 的 100 倍容量

- LevelDB 网络支持, 使用 C/C++ 开发

- Redis API 兼容, 支持 Redis 客户端

- 适合存储集合数据

- 客户端 API 支持的语言包括: C++, PHP, Python, Java, Go

- 持久化的队列服务

- 主从复制, 负载均衡

支持 key-string, key-zset, key-hashmap 三种数据类型。

### SSDB使用

**安装**

    wget --no-check-certificate https://github.com/ideawu/ssdb/archive/master.zip
    unzip master
    cd ssdb-master
    make
    $optional, install ssdb in /usr/local/ssdb
    sudo make install

**启动**

    # start master
    ./ssdb-server ssdb.conf

    # or start as daemon
    ./ssdb-server -d ssdb.conf

**Python使用**

Python API
类型	开发者	Repositiy	Note

内置	ideawu	SSDB.py	建议使用

pyssdb	ideawu	pyssdb	A SSDB Client Library for Python

**Usage**

    >>> import pyssdb
    >>> c = pyssdb.Client()
    >>> c.set('key', 'value')
    1
    >>> c.get('key')
    'value'
    >>> c.hset('hash', 'item', 'value')
    1
    >>> c.hget('hash', 'item')
    'value'
    >>> c.hget('hash', 'not exist') is None
    True
    >>> c.incr('counter')
    1
    >>> c.incr('counter')
    2
    >>> c.incr('counter')
    3
    >>> c.keys('a', 'z', 1)
    ['counter']
    >>> c.keys('a', 'z', 10)
    ['counter', 'key']

**ssdb-cli**

    Connect to a SSDB server:

    $ /usr/local/ssdb/ssdb-cli -h 127.0.0.1 -p 8888
    ssdb (cli) - ssdb command line tool.
    Copyright (c) 2012-2013 ideawu.com

    'h' or 'help' for help, 'q' to quit.

    ssdb 127.0.0.1:8888>

    ssdb 127.0.0.1:8899> info
    version
        1.8.0
    links
        1
    total_calls
        4
    dbsize
        1829
    binlogs
        capacity : 10000000
        min_seq  : 1
        max_seq  : 74
    replication
        client 127.0.0.1:55479
            type     : sync
            status   : SYNC
            last_seq : 73
    replication
        slaveof 127.0.0.1:8888
            id         : svc_2
            type       : sync
            status     : SYNC
            last_seq   : 73
            copy_count : 0
            sync_count : 44
    leveldb.stats
                         Compactions
        Level  Files Size(MB) Time(sec) Read(MB) Write(MB)
        --------------------------------------------------
          0        0        0         0        0         0
          1        1        0         0        0         0

    25 result(s) (0.001 sec)

**SSDB GUI tools**

    phpssdbadmin - A SSDB GUI tool like phpmyadmin.
    FastoNoSQL - A crossplatform SSDB, Redis, Memcached GUI too

**主从复制（Replication**

支持三种方式：Master-Slave、Master-Master、Multiple Masters

**Master-Slave**

    #server 1
    replication:
        slaveof:
    #server 2
     replication:
        slaveof:
            id: svc_1
            # sync|mirror, default is sync
            type: sync
            # use ip for older version
            #ip: 127.0.0.1
            # use host since 1.9.2
            host: localhost
            port: 8888

**Master-Master**

    #server1
     replication:
        slaveof:
            id: svc_2
            # sync|mirror, default is sync
            type: mirror
            # use ip for older version
            #ip: 127.0.0.1
            # use host since 1.9.2
            host: localhost
            port: 8889
    #server 2
     replication:
        slaveof:
            id: svc_1
            # sync|mirror, default is sync
            type: mirror
            # use ip for older version
            #ip: 127.0.0.1
            # use host since 1.9.2
            host: localhost
            port: 8888

**Multiple Masters**

这种架构下，每个节点都是其它节点的slave节点。

     replication:
        slaveof:
            id: svc_1
            # sync|mirror, default is sync
            type: mirror
            # use ip for older version
            #ip: 127.0.0.1
            # use host since 1.9.2
            host: localhost
            port: 8888
        slaveof:
            id: svc_2
            # sync|mirror, default is sync
            type: mirror
            # use ip for older version
            #ip: 127.0.0.1
            # use host since 1.9.2
            host: localhost
            port: 8889
        # ... more slaveof

**检查复制同步状态**

    ssdb 127.0.0.1:8899> info
    binlogs
        capacity : 10000000
        min_seq  : 1
        max_seq  : 74
    replication
        client 127.0.0.1:55479
            type     : sync
            status   : SYNC
            last_seq : 74
    replication
        slaveof 127.0.0.1:8888
            id         : svc_2
            type       : sync
            status     : SYNC
            last_seq   : 10023
            copy_count : 0
            sync_count : 44
对于master 节点来说, binlogs.max_seq 意味着最新一个写（更新／删除）操作的序号, replication.client.last_seq意味着发到slave节点上的最后一个操作的序号。所以判断slave是否已经同步到了master的依据是在SYNC的时候查看binlogs.max_seq 和 replication.client.last_seq 是否相等。

**HA**

SSDB 通过Replication可以实现（几乎）实时拷贝同步。大部分人选用Master-Slaves架构。

对于应用程序而言，要让应用程序，写操作只访问master节点，读操作访问master/slave 节点。

当master down时，需要控制程序将所有请求发给其中一个slave节点，并删除down的master node和down的slave node，修改了master节点之后要相应地重新修改 replication 架构。

⚠️ 已经删除的master节点和slave节点的数据必须从disk删除，并且不能再用。

这种方式实现的HA，高度依赖人工决策和操作。

**SSDB方案**

方案一：Twemproxy + SSDB

方案二：lvs + SSDB(双主模式)
 http://www.tuicool.com/articles/NRvEBb

方案三：keepalived + SSDB
 https://yq.aliyun.com/articles/69200

 https://github.com/ideawu/ssdb/issues/713

方案四：Haproxy + SSDB

**SSDB 性能分析**

**测试工具**

ssdb-bench

**参考**

- http://ssdb.io/zh_cn/
- http://ssdb.io/docs/
- http://ssdb.io/docs/replication.html
- https://github.com/ideawu/ssdb
- 入门基础: https://github.com/ideawu/ssdb-docs/blob/master/pdf/SSDB%E5%85%A5%E9%97%A8%E5%9F%BA%E7%A1%80.pdf
- 单实例每天支撑上亿个请求的ssdb: http://www.ideawu.net/blog/archives/736.html
- SSDB 使用：http://blog.csdn.net/stonehigher125/article/details/39202589
- 关于ssdb 备份的问题：https://github.com/ideawu/ssdb/issues/713