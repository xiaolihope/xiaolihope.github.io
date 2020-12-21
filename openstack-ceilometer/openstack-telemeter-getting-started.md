---
date: 2017-11-08T10:06:00+08:00
title: openstack telemetry getting started
---

Openstack Telemetry目前由四个项目组成如图 1 telemetry系统架构，Ceilometer，aodh，gnocchi和panko。
Ceilometer是云平台指标数据搜集服务
Aodh是云平台告警服务
Gnocchi是云平台Metric服务，为ceilometer metering metrics提供存储和查询服务，Aodh也开发了相应的alarm模块，支持gnocchi metrics alarm。Gnocchi提供了Grafana接口，可以提供可视化API，与ELK集成。Gnocchi可以作为单独的service提供服务。
但从目前了解到的资料来看，gnocchi尚未有太多生产环境采用。
Panko是从Newton版本才有的，目的是从ceilometer中剥离出Event部分，尚且不在设计范围。

![telemetry](../images/telemetry-archi.png)

## 监控数据搜集服务

Ceilometer服务主要用于数据收集，提供Event和meter数据搜集。

Event数据来源是openstack service 的notification，本设计不考虑event存储。

Ceilometer 服务主要包含:

* polling agent：定期轮询获取监控数据，通过libvirt接口或者snmp或者openstack service API。Polling agent又根据namespace做了metering的区分，分别对应不同的agent
* Central agent是通过opensack api 获取监控数据
* Compute agent同时hypervisor接口等获取用户虚拟机及compute 节点服务信息

![polling-agent](../images/polling-agent.png)

* notification agent：通过监听message queue，将message 转化为event和sample并且执行pipeline操作，消息的来源主要是openstack service的notification。注：openstack services需要enable notification的配置
* collector：收集Event和metering的数据存储到store，支持：
    * File
    * 数据库

| Driver | API querying | API statistics |
|---------|-------------|----------------|
| MongoDB | Yes  | Yes |
| MySQL | Yes | Yes |
| PostgreSQL | Yes | Yes|
| HBase| Yes | Yes, except groupby & selectable aggregates |

    * Gnocchi
    * http
    * api - 提供rest api，并提供查询统计数据

## Ceilometer measurements

Reference Document 附件中总结了mitaka版本Ceilometer支持数据统计项。

## 监控数据存储

Ceilometer监控数据存储支持多种plugin，例如，mongodb/hbase/mysql/gnocchi。CERN架构中就是使用的mongodb的模式。但是从社区反应来讲，mongodb作为数据存储具有很多问题，可参见讨论http://www.gossamer-threads.com/lists/openstack/operators/44584。

Gnocchi是Openstack Telemtry的Metric serivce，开始与2015年，目的在与解决ceilometer的监控数据存储的性能问题。

Gnocchi支持的存储driver有文件系统，Swift，s3，Ceph

官方建议使用ceph，gnocchi对ceph的数据存储做了很多优化

Gnocchi索引存储支持PostgreSQL (recommended)和MySQL (at least version 5.6.4)

可以使用现有的mysql server

 本节主要做了mongodb 和gnocchi作为ceilometer监控数据存储的对比测试

### gnocchi vs. Mongodb测试对比

Gnocchi + ceph vs. mongodb

 响应时间对比

Ceilometer sample 查询时间，跟获取的数据量成正比

Gnocchi 检索时间稳定在3~4s，检索时间主要是读取ceph obj，解析obj格式

## 告警服务
Aodh是从ceilometer中分离出来的子项目，只负责处理alarm，aodh数据库独立于ceilometer，根据官方文档将只支持SQL DB。

### 告警规则

Aodh告警规则支持的类型

Threshold：设置阈值类型的alarm

Combination：组合条件的alarm

gnocchi_resources_threshold

gnocchi_aggregation_by_metrics_threshold

gnocchi_aggregation_by_resources_threshold

gnocchi_aggregation_by_resources_threshold

### Aodh支持的notifier类型

Aodh支持如下类型的告警触发，需要CMP提供URL获取告警通知。

HTTP: 发送http request到指定的URL

HTTPS: 发送http request到指定的URL

LOG：notification日志

TEST：测试用处

Trust + HTTP: Keystone will be used as trust authentication

n URL定义格式 trust+http://trust-id@host/action

u Trustid：

TRUST+HTTPS

### Aodh Alarm 存储

Alarm存储支持SQL DB，选用现有DB mysql 即可

* MYSQL
* MYSQL+pymysql
* Postgresql
* Sqlite

### CERN telemetry参考架构

从cern分享2016年2月在欧洲operations meetup上分享的ceilometer参考架构（Kilo），metrics的存储使用的mongodb存储compute agent搜集的信息。其他的notification metering和其他的openstack service的数据都存储在HBase上。

![cern-archi](../images/cern-archi.png)