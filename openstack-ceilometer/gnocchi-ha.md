---
date: 2017-11-08T09:59:43+08:00
title: gnocchi ha
---

## 部署模型

![gnocchi-ha](../images/gnocchi-ha.png)

## 准备rpm 安装包

```
openstack-gnocchi-api-3.1.3-1.el7.centos.x86_64
python-gnocchi-3.1.3-1.el7.centos.x86_64
openstack-gnocchi-common-3.1.3-1.el7.centos.x86_64
openstack-gnocchi-metricd-3.1.3-1.el7.centos.x86_64
openstack-gnocchi-indexer-sqlalchemy-3.1.3-1.el7.centos.x86_64
openstack-gnocchi-statsd-3.1.3-1.el7.centos.x86_64
## ssdb 安装包
ssdb-1.9.4-1.el7.centos.x86_64
python-ssdb-0.0.3-1.noarch
```

## ssdb部署
ssdb物理节点，需要事先准备好存储设备，并格式化，挂载/var 路径
需要两个网卡，数据复制最好能提供独立网卡
### ssdb服务安装
```
yum install ssdb-1.9.4-1.el7.centos.x86_64 python-ssdb-0.0.3-1.noarch
```
* 更新ssdb.conf配置文件
```
/etc/ssdb/ssdb.conf
# ssdb-server config
# MUST indent by TAB!

# relative to path of this file, directory must exists
work_dir = /var/lib/ssdb
pidfile = /var/lib/ssdb/ssdb.pid

server:

        # bind to public ip
        #ip: 0.0.0.0
        ip:
        port: 8888
        # format: allow|deny: all|ip_prefix
        # multiple allows or denys is supported
        #deny: all
        #allow: 127.0.0.1
        #allow: 192.168
        # auth password must be at least 32 characters
        #auth: very-strong-password
        #readonly: yes

replication:
        binlog: yes
        # Limit sync speed to *MB/s, -1: no limit
        sync_speed: -1
        slaveof:
                # to identify a master even if it moved(ip, port changed)
                # if set to empty or not defined, ip:port will be used.
                id: svc_2
                # sync|mirror, default is sync
                type: mirror
                # slave ssdb ip
                host:
                port: 8889

logger:
        level: info
        output: /var/log/ssdb/ssdb.log
        rotate:
                size: 1000000000

leveldb:
        # in MB
        cache_size: 500
        # in MB
        write_buffer_size: 64
        # in MB/s
        compaction_speed: 1000
        # yes|no
        compression: yes
```
* ssdb-server 服务启动
`systemctl start ssdb-server`
* 检查ssdb-server工作状态
`ssdb-cli`
### haproxy服务配置
```
listen ssdb
  bind 0.0.0.0:8888
  mode tcp
  balance source
  option tcplog
  option clitcpka
  option srvtcpka
  server ssdb-2 10.0.2.51:8888 check inter 5000 rise 2 fall 3 backup
  server ssdb-1 10.0.2.50:8888 check inter 10s fastinter 2s downinter 3s rise 3 fall 2
```
### 重启haproxy
`systemctl restart haproxy`
## gnocchi升级
### 停止ceilometer收集服务
```
# 所有的compute节点
systemctl stop openstack-ceilometer-compute
#所有的控制节点
systemctl stop openstack-ceilometer-collector
```
### 停止gnocchi服务
```
systemctl stop httpd
systemctl stop openstack-gnocchi-metricd

```
### 安装gnocchi 相关的rpm包
```
openstack-gnocchi-api-3.1.3-1.el7.centos.x86_64
python-gnocchi-3.1.3-1.el7.centos.x86_64
openstack-gnocchi-common-3.1.3-1.el7.centos.x86_64
openstack-gnocchi-metricd-3.1.3-1.el7.centos.x86_64
openstack-gnocchi-indexer-sqlalchemy-3.1.3-1.el7.centos.x86_64
openstack-gnocchi-statsd-3.1.3-1.el7.centos.x86_64
```
### 升级gnocchiclient
```
yum upgrade python-gnocchiclient
```
### gnocchi配置文件更新
```
[DEFAULT]
debug = False
verbose = True
log_dir = /var/log/gnocchi
policy_file = /etc/gnocchi/policy.json
[api]
pecan_debug = False
port = 8041
host = 0.0.0.0
max_limit = 1000
paste_config=/etc/gnocchi/api-paste.ini
auth_mode = keystone
[archive_policy]
default_aggregation_methods = max,min,sum,count,mean
[cors]
[cors.subdomain]
[database]
[indexer]
# mysql connection URL
# url = mysql+pymysql://gnocchi:gnocchi@127.0.0.1/gnocchi?charset=utf8
url =
[metricd]
workers = 24
[oslo_middleware]
[oslo_policy]
[statsd]
[incoming]
driver=ssdb
[storage]
driver = ssdb
ssdb_host =
ssdb_port =
file_basepath = /var/lib/gnocchi/
coordination_url = file:///var/lib/gnocchi/
metric_processing_delay = 10
metric_reporting_delay = 10
[service_credentials]
auth_type=password
auth_url=
project_name=services
project_domain_name=Default
username=gnocchi
user_domain_name=Default
password=
[keystone_authtoken]
auth_uri =
admin_tenant_name=
admin_user=
admin_password=
identity_uri=
```
### 升级gnocchi数据库
```
gnocchi-upgrade --debug --config-file /etc/gnocchi/gnocchi.conf

```
### 重启gnocchi服务
```
systemctl start openstack-gnocchi-metricd
systemctl start httpd
```
### 重启ceilometer收集服务
```
# 所有的compute节点
systemctl start openstack-ceilomter-compute
# 所有的控制节点
systemctl start openstack-ceilometer-collector
```
### gnocchi metricd高可用配置

gnocchi metricd 高可用模式，与cinder-volume 相同，需要使用pacemaker保证，始终有且仅有一个节点上的gnocchi metricd服务工作
