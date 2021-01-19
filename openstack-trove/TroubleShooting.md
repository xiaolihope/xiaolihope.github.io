# Trouble Shooting
date: 2017-05-10T10:39:30+08:00

## 1. Trove instance stuck in BUILD state

处于build状态的instance，不允许delete操作，可以通过修改db中的状态为active后再删除，or 直接删除数据库。
`trove force-delete`
## 2. 调试步骤

```
Step1: 登录 guest vm，首先查看 guest agent 错误日志，发现没有/var/log/trove/trove-guestagent.log，于是查看 /var/log/upstart 看看之前发生了什么，并找到/var/log/upstart/trove-guest.log
a. 检查securitygroup rule，确认有22 规则
b. ssh -i key ubuntu@ip key 为integration 目录下的key
C. 阅读trove 错误日志
a) trove api(apache httpd), task manage 和 trove conductor log
B) guest vm
要查看的log有两部分：
 系统和用户数据库的日志文件
通过trove guest agent 生成的日志文件
在ubuntu 中，系统日志文件通常在：/var/log/syslog 下
／var/log/upstart 下的文件以及由数据库生成的日志文件一般也位于/var/log下。
例如，mysql 日志文件在/var/log/mysql
如果在启动guest agent 之前有错误，则通常在/var/log/upstart/下可以找到。
由guest agent生成的日志文件在/var/log/trove下，并且默认名字为：trove-guestagent.log
修改 trove guest agent 代码之后，重启服务：service trove-guest restart
```
## 3. trove show 没有返回instance ip
```
在trove.conf中设置：network_label_regex = ^NETWORK_NAME$
```
## 4. swift list error
```
# swift list
Authorization Failure. Authorization failed: (http://10.0.13.47/identity/auth/tokens): The resource could not be found. (HTTP 404)
```
## 5. 在创建instance，security group 必须要开启22端口吗？

不需要开启。默认是不开启的。如果运维人员需要登陆到vm里，可以自己增加22规则。
## 6. trove replica

当有副本处于build状态重视，进行主从切换，导致所有的节点都变成error。—need to debug.
## 7. failover error
```
|                         | Traceback (most recent call last):                                                              |
|                         |   File "/opt/stack/trove/trove/taskmanager/manager.py", line 173, in promote_to_replica_source  |
|                         |     replicas)                                                                                   |
|                         |   File "/opt/stack/trove/trove/taskmanager/manager.py", line 103, in _promote_to_replica_source |
|                         |     master_ips = old_master.detach_public_ips()                                                 |
|                         |   File "/opt/stack/trove/trove/taskmanager/models.py", line 1357, in detach_public_ips          |
|                         |     floating_ips = self._get_floating_ips()                                                     |
|                         |   File "/opt/stack/trove/trove/taskmanager/models.py", line 1348, in _get_floating_ips          |
|                         |     for ip in self.nova_client.floating_ips.list():                                             |
|                         | AttributeError: 'Client' object has no attribute 'floating_ips'                                 |
https://review.openstack.org/#/c/447724/
最新的novaclient code的问题。替换成ocata stable版本就可以了。
```
## 8. trove dns support
https://blueprints.launchpad.net/trove/+spec/implement-dns-management-drive-for-trove

## 9. 创建percona 实例的时候，发现mysql 服务起不来
通过查看syslog，看到out of memory.

## 10. 创建instance实例，guest 连不上rabbitmq
```
nc -z  0  %IP%    %PORT% -vvv
修改allinone上iptables 规则：
iptables -I INPUT -p tcp --dport 5672 -j ACCEPT
或者直接修改文件：/etc/sysconfig/iptables，注意加到最上面，然后service iptables restart
```
## 11. trove 创建副本的时候，会讲master guest vm shutdown 然后start吗？
## 12. create slave instance error:
```
An error occurred while deleting a bad replication snapshot from instance ff0b53eb-754d-48f8-9ece-eb13ff0e6e78.     |

|                         | Backup 5db73df3-bcbd-4a05-bb54-26b37238a3ca cannot be deleted because it is running.                                |

| fault_date              | 2017-06-05T02:37:44                                                                                                 |

| fault_details           | Server type: taskmanager                                                                                            |

|                         | Traceback (most recent call last):                                                                                  |

|                         |   File "/opt/stack/trove/trove/taskmanager/manager.py", line 384, in create_instance                                |

|                         |     locality)                                                                                                       |

|                         |   File "/opt/stack/trove/trove/taskmanager/manager.py", line 349, in _create_instance                               |

|                         |     backup_id, volume_type, modules)                                                                                |

|                         |   File "/opt/stack/trove/trove/taskmanager/manager.py", line 312, in _create_replication_slave                      |

|                         |     replica_number=replica_number)                                                                                  |

|                         |   File "/opt/stack/trove/trove/taskmanager/models.py", line 665, in get_replication_master_snapshot                 |

|                         |     self._log_and_raise(e_delete, msg_delete, err)                                                                  |

|                         |   File "/opt/stack/trove/trove/taskmanager/models.py", line 958, in _log_and_raise                                  |

|                         |     raise TroveError(message=full_message)                                                                          |

|                         | TroveError:                                                                                                         |

|                         |     An error occurred while deleting a bad replication snapshot from instance ff0b53eb-754d-48f8-9ece-eb13ff0e6e78. |

|                         | Backup 5db73df3-bcbd-4a05-bb54-26b37238a3ca cannot be deleted because it is running.
The reason is that the backup status for master is "NEW", the backup creation is not successfully. the root cause is "backup quota exceed"
```

# trove 几点经验

## 镜像制作

1. 手工制作:http://docs.openstack.org/developer/trove/dev/building_guest_images.html
2. 使用devstack 制作好的镜像

之前经验：

1. mysql和redis用了4个月，基本满足需求
2. 推荐使用debian的系统，在512M内存下表现比centos要好。
3. 创建一个trove实例=nova创建一个虚拟机+cinder创建一个卷，在数据库中很多数据是重复的，需要考虑数据一致性，出现过数据不一致的情况（主要出现在trove删实例之后，nova那边没删掉）
4. 实例的22端口必须开启，用来开机传输配置文件，怎么样防止用户连上
5. n版本的trove mysql 不支持集群只支持ms模式，redis支持集群，不支持mysql集群，用户很难忍，
6. 默认配额较少，需要修改，位置在 trove/common/cfg.py
7. trove 既要需要直连rabbit，又需要连接用户虚拟机， 需要考虑的安全问题，以及网络设计问题，防止rabbit的信息被用户截获。
