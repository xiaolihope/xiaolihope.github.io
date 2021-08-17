2016/12/23

## oslo.messaging RPC 简介

### 概念
1.1 transport

处理消息传送的抽象层，根据rpc_backend的配置确定真正处理消息发送的drive。它能实现具体MQ的功能，但如何实现由其底层的driver决定，
你可以把它当做RabbitMQ,或是QPID，但你的操作都是针对transport的，底层是什么不用关心。
获取一个transport很简单，用oslo_messaging.get_transport()函数就行了，其定义如下：
oslo_messaging.get_transport(conf, url=None, allowed_remote_exmods=None, aliases=None)

1.2 target

对于message server, target代表了server监听的内容. 对于messge client, target代表了client要发送的目的地，
需要在其中指定binding key,server name。

target 的 _init_ 函数

```
def __init__(self, exchange=None, topic=None, namespace=None,
             version=None, server=None, fanout=None):
    self.exchange = exchange
    self.topic = topic
    self.namespace = namespace
    self.version = version
    self.server = server
    self.fanout = fanout
```
其中，

topic：就是binding key。如果多个consumer（在注释里也叫server）监听同一个queue，那么会使用round-robin来发配消息

exchange: 用于指定exchange，若没有，就用配置文件里的control_exchange

version：consumer（也就是server）是有版本号的，发送的message的版本要和consumer的版本兼容才行。版本号是major.mirror的形式。major号相同则兼容

fanout：表明消息会广播，忽略topic，只要consumer允许接收fanout消息即可

server：特定的consumer。相当于direct的转发了，就是发送者指定特定的consumer，然后由它处理，而不是像topic里的解释那样，用round-robin来分发消息

1.3 Listener

从MQ中不停地抽取消息, 存放到内部的list队列

1.4 Dispatcher

解析消息, 调用实际的endpoint / manager的业务函数来处理消息

1.5  Executor

负责把Listener和Dispatcher连接起来, 用listener来接受消息, 在用dispatcher来处理消息。有三种可用的Executor：blocking,eventlet,threading。


### 服务端（计算节点）
一个RPC server有多个endpoint，每个endpoint有多个供client远程调用的方法。要获取一个RPC server只需要调用oslo_messaging.get_rpc_server（），其定义如下：

```
oslo_messaging.get_rpc_server(transport, target, endpoints, executor='blocking', serializer=None)

# Construct an RPC server.
# The executor parameter controls how incoming messages will be received and dispatched. By default, the most simple executor is used - the blocking executor.
```

一个简单的有多个endpoint的RPC server例子如下：

```
cat rpc_server.py

from oslo_config import cfg
import oslo_messaging as messaging

# 封装方法，供cient进行RPC调用
class TestEndpoint(object):
    def demo_cast(self, ctxt, arg):
        print "<<< request cast_demo from: %s" % arg
    def demo_call(self, ctxt, arg):
        print "<<< request call_demo from: %s" % arg
        hostname = 'zll'
        result = ">>> response from: %s, result: OK" % hostname
        print result
        return result

# 封装方法，供cient进行RPC调用
class TestEndpoint2(object):
    def test(self, ctxt, arg):
        print 'test'

if __name__ == '__main__':
    # transport的一个简单的获取方法，将会根据user configuration选取一个合适的底层driver
    trans = messaging.get_transport(cfg.CONF)
    target = messaging.Target(topic = 'test', server = 'server1')

    endpoints = [
        TestEndpoint(),
        TestEndpoint2()
    ]


    server = messaging.get_rpc_server(trans, target, endpoints, executor='blocking')

    try:
        server.start()
        while True:
            time.sleep(1)
    except Exception:
        print("Stopping server")

    server.stop()
    server.wait()

```
### 客户端（控制节点）

```
cat client.py

from oslo_config import cfg
import oslo_messaging as messaging


if __name__ == '__main__':
    trans = messaging.get_transport(cfg.CONF)
    target = messaging.Target(topic = 'test' )
    client = messaging.RPCClient(transport=trans, target=target)
    print client.call(ctxt={}, method='demo_call', arg='host')

```

## RabbitMQ Heartbeat 机制

### 简介

一些网络故障中，操作系统可能需要很长的时间（Linux系统默认是11分钟）才能检测到，这可能导致数据丢失。为了避免这类问题，AMQP 0-9-1中提供heartbeat机制，这样应用程序就可以及时的发现断开的连接或完全不响应的对端。Heartbeat机制也可以避免RabbitMQ连接被中间的网络设备强制关闭。因为某些网络设备会自动关闭一段时间内没有传输任何数据包的网络连接。

### 实现方法

如果RabbitMQ服务端启用了heartbeat功能，RabbitMQ会通过connection.tune信令将heartbeat的超时间隔告诉告诉客户端，客户端可以根据需要重新设置该值，并通过connection.tune-ok信令将时间间隔告诉RabbitMQ，RabbitMQ会以客户端的时间作为该TCP连接上的heartbeat超时时间。
RabbitMQ收到客户端的connection.tune-ok信令之后，就好启动心跳检测。RabbitMQ会为每个TCP连接创建两个检测进程。一个进程周期性的发送heartbeat检测包给客户端；另一个进程则定时检测TCP上是否有数据接收，如果一段时间内没有收到任何数据，则判定为心跳超时，最终会关闭TCP连接。

### 配置方法

RabbitMQ只提供了一个参数来控制heartbeat的超时时间，参数应该写入配置文件 /etc/rabbitmq/rabbitmq.config，格式如下：

```
$ cat /etc/rabbitmq/rabbitmq.config
[
    ...
    {rabbit, [
        ....
        {heartbeat, 20},
        ....
    ]}
    ...
].
```
上面的配置中设置的heartbeat超时时间为20秒，RabbitMQ默认的超时时间是580秒。如果heartbeat超时时间为0，就是关闭heartbeat功能。
按照上面的配置，RabbitMQ大约每10（timeout/2）秒会发送一个heartbeat包给客户端，如果连续两次没有收到heartbeat包，RabbitMQ就会认为客户端不可达，TCP连接会关闭。当客户端检测到RabbitMQ因为heartbeat不可达，就需要重新建立连接。

### 参考资料

- https://www.rabbitmq.com/heartbeats.html
- http://my.oschina.net/hncscwc/blog/195343


## RabbitMQ 运维文档
## RabbitMQ 镜像队列机制

默认认情况下，RabbitMQ集群的queues通常只会在某个单独的节点上（客户端声明的节点）。这点与exchanges和bindings不同，它们会被创建在集群的所有节点上。为了实现RabbitMQ集群的高可用，常用的方法是采用镜像队列，或者是主从切换共享存储的方式。其中，镜像队列由于其几乎无缝的故障切换特性，较为得到青睐。

RabbitMQ镜像队列机制的设置较为简单，只需要声明queues为镜像队列模式，就可以在多个节点上建立镜像队列。每一个被镜像的queue包括一个主节点和多个从节点；如果主节点由于某种原因退出集群，创建时间最长的从队列将被选举为新的主节点队列（promoted to master）。由于镜像队列需要采用rabbitmq集群，对于容易出现网络分区的环境比如WAN，镜像队列机制难以正常工作。

### 镜像队列的配置

通过设置policy，可以将队列配置为镜像模式。由于policy可以随时设置，所以也支持将一个non-mirrored queue，动态修改为mirrored queue。需要说明的是，由于非镜像的队列无需实现镜像机制，所以其比镜像队列工作速率的更快。
镜像队列可以配置一下集中key，设置队列的工作模式：

| HA-mode|  HA-params |            Result             |
|--------|------------|-------------------------------|
|   all  |     无     |  在集群中的所有节点建立镜像队列集群 |
| exactly|    数量    |  将队列镜像到指定数目的RabbitMQ 节点上 |
| nodes  |   节点名    |   在指定的节点建立镜像队列        |

使用示例如下：

```
# 将名称以"ha."开头的节点镜像镜像到所有的节点
rabbitmqctl set_policy ha-all "^ha\." '{"ha-mode":"all"}'

# 将名称以"two"开头的节点镜像到任意两个节点
rabbitmqctl set_policy ha-two "^two\." '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'

# 将名称以"nodes."开头的节点镜像到nodeA和nodeB
rabbitmqctl set_policy ha-nodes "^nodes\." '{"ha-mode":"nodes","ha-params":["rabbit@nodeA", "rabbit@nodeB"]}'
```

### 镜像队列的实现

镜像队列模式下，对于publish操作，发往镜像队列的消息总是会被广播到所有的从节点，如果主队列宕机，在新的主队列被选出之后，客户端会将未被确认的publish消息，重新发送到从节点，所以在主队列切换时，RabbitMQ集群并不会丢失publish的消息。对于非publish的操作，从节点总是以相同的顺序，执行与主节点相同的操作，该类操作总是只需要发送到主节点，操作完成之后，主节点再广播操作的结果，比如在消费者完成对于主节点队列的消息消费之后，从节点会丢弃该消息。

通过这种工作方式，镜像队列能够最大限度地提高rabbitmq集群的高可用性，但是这种方式并不能降低集群的负载，因为所有的节点都需要处理全部的消息。另外需要说明的是，在rabbitmq节点异常宕机的情况下，由于消费操作的ACK消息可能丢失在客户端到主节点，或者主节点到镜像节点的广播过程中。所以，客户端在重新连到新的rabbitmq节点之后，仍然有可能会读取到重复的消息。

对于一个新加入集群的RabbitMQ节点，该节点只会同步发往该集群中的新消息，而并不会同步之前就在集群中的信息。随着时间的推移，消息的消费，新加入的节点最终回与master保持一致。因此，新增一个从节点，并不会提高本就在集群中的消息的冗余性。

当一个rabbitmq节点宕机时，队列启动时间最长的从节点将会被选举为新的队列主节点，因为该节点将会有更大的概率同步主节点的数据。宕机之后，宕机节点上的主队列将会转移到别的rabbitmq节点。对于一个RabbitMQ集群来说，最后关闭的节点将会是所有队列的主节点。由于目前没有好的方式能够比较新的主节点与重新加入集群的旧主节点，数据是否有分歧，所以当一个节点加入集群时，将会删除所有本地信息，包括声明为durable的内容。

### 镜像队列的参数优化

1. RabbitMQ 3.6.0之后，可以通过配置"ha-sync-batch-size"，设置消息同步时每个批次同步的消息数。如果该参数没有声明，则采用默认值1。设置较大的'ha-sync-batch-size'能够提高镜像队列的性能，但是设置时需要考虑到net_nicktime、带宽大小，以及消息的大小。

2. 为了避免数据的丢失，默认情况下，正常的RabbitMQ节点关闭操作，需要在所有镜像队列都完成同步的之后才能完成。但是，有时会出现从节点未完成同步，主节点就宕机的情况，比如节点宕机、网络中断等。因此，如果想保持镜像队列的高可用性而不是消息的完整性，可以考虑将"ha-promote-on-shutdown"从默认的"when-synced"修改为always。

### 其它

通过设置"nodes" policy可以将一个现有的主节点从RabbitMQ集群中删除，为了防止消息丢失，RabbitMQ将会保留该主节点，直至其他从节点中至少有一个完成数据同步（即使同步的时间很长）。

如果声明exclusive queue队列的client，由于某种原因中断了与rabbitmq-server的连接，RabbitMQ会删除该exclusive queue。所以exclusive queue无法被设置为镜像模式，也无法设置为durable模式。

可以执行如下命令，查看队列完成同步的从节点：

```
bash rabbitmqctl list_queues name slave_pids synchronised_slave_pids
```

镜像队列模式下，旧的队列主节点宕机之后，重新加入集群，会被视为一个刚加入集群的空节点。所以在对整个rabbitmq集群进行关闭操作之后，需要注意集群开启顺序，避免数据的丢失。

### 参考：

https://www.rabbitmq.com/ha.html

## RabbitMQ CLI 管理工具rabbitmqadmin

rabbitmqadmin 是由 rabbitmq_management 插件提供的，得启用此插件。

启动插件

```bash
# rabbitmq-plugins list
# rabbitmq-plugins enable rabbitmq_management
```

获取 rabbitmqadmin

```bash
# wget http://localhost:15672/cli/rabbitmqadmin
# chmod  +x rabbitmqadmin 
```
使用 rabbitmqadmin

```bash
# rabbitmqadmin --help
Usage
=====
  rabbitmqadmin [options] subcommand

Options
=======
--help, -h              show this help message and exit
--config=CONFIG, -c CONFIG
                        configuration file [default: ~/.rabbitmqadmin.conf]
--node=NODE, -N NODE    node described in the configuration file [default:
                        'default' only if configuration file is specified]
--host=HOST, -H HOST    connect to host HOST [default: localhost]
--port=PORT, -P PORT    connect to port PORT [default: 15672]
--path-prefix=PATH_PREFIX
                        use specific URI path prefix for the RabbitMQ HTTP API
                        (default: blank string) [default: ]
--vhost=VHOST, -V VHOST
                        connect to vhost VHOST [default: all vhosts for list,
                        '/' for declare]
--username=USERNAME, -u USERNAME
                        connect using username USERNAME [default: guest]
--password=PASSWORD, -p PASSWORD
                        connect using password PASSWORD [default: guest]
--quiet, -q             suppress status messages [default: True]
--ssl, -s               connect with ssl [default: False]
--ssl-key-file=SSL_KEY_FILE
                        PEM format key file for SSL
--ssl-cert-file=SSL_CERT_FILE
                        PEM format certificate file for SSL
--ssl-ca-cert-file=SSL_CA_CERT_FILE
                        PEM format CA certificate file for SSL
--ssl-disable-hostname-verification
                        Disables peer hostname verification
--format=FORMAT, -f FORMAT
                        format for listing commands - one of [raw_json, long,
                        pretty_json, kvp, tsv, table, bash] [default: table]
--sort=SORT, -S SORT    sort key for listing queries
--sort-reverse, -R      reverse the sort order
--depth=DEPTH, -d DEPTH
                        maximum depth to recurse for listing tables [default:
                        1]
--bash-completion       Print bash completion script [default: False]
--version               Display version and exit

More Help
=========

For more help use the help subcommand:

  rabbitmqadmin help subcommands  # For a list of available subcommands
  rabbitmqadmin help config       # For help with the configuration file
```

查看 exchanges

```bash
# rabbitmqadmin -u openstack -p <password> list exchanges
+-----------------------------------------------+---------+
|                     name                      |  type   |
+-----------------------------------------------+---------+
|                                               | direct  |
| amq.direct                                    | direct  |
| amq.fanout                                    | fanout  |
| amq.headers                                   | headers |
| amq.match                                     | headers |
| amq.rabbitmq.log                              | topic   |
| amq.rabbitmq.trace                            | topic   |
| amq.topic                                     | topic   |

```

查看queues

```bash
 # rabbitmqadmin -u openstack -p <password> list queues
+--------------------------------------------------------------------------------+----------+
|                                      name                                      | messages |
+--------------------------------------------------------------------------------+----------+
| cert                                                                           | 0        |
| cert.allinone-dev                                                              | 0        |
| cert_fanout_e72b52d1412f4c668d529f3d5255005e                                   | 0        |
| cinder-scheduler                                                               | 0        |
| cinder-scheduler.cinder                                                        | 0        |
| cinder-scheduler_fanout_3e7f75d0b5c7441c88aebdcaad081ad8                       | 0        |
| cinder-volume                                                                  | 0        |
| cinder-volume.cinder@ssd-ceph                                                  | 0        |
| cinder-volume_fanout_1ef24cb4931e4322bb8b24eab3c56f3e                          | 0        |
| compute                                                                        | 0        |
| compute.allinone-dev                                                           | 0        |
| compute_fanout_798bdeac39544944997efe7e0c35a96c                                | 0        |
... 
```

创建 queue

```
./rabbitmqadmin declare queue name=dixiaoli-trove  durable=true

```

创建binding

```
./rabbitmqadmin declare binding source=trove destination=dixiaoli-trove routing_key=notifications.*

./rabbitmqadmin list bindings | grep notifications

```

purge messages

```
./rabbitmqadmin purge queue name=dixiaoli-trove
```

get message

```
./rabbitmqadmin get queue=dixiaoli-trove requeue=false --format={raw_json, long, pretty_json, kvp, tsv, table, bash}
```
## References

[RabbitMQ-cli-rabbitmqadmin]

[RabbitMQ-cli-rabbitmqadmin]: http://soft.dog/2016/04/20/RabbitMQ-cli-rabbitmqadmin/

## rabbitmqctl

```
# rabbitmqctl list_queues --online name pid synchronised_slave_pids

--online:  List queues that are currently available (their master node is).
synchronised_slave_pids: If the queue is mirrored, this gives the IDs of the current slaves which are synchronised with the master - i.e. those which could take over from the master without message loss.

REF: http://www.rabbitmq.com/man/rabbitmqctl.1.man.html

for nova:
rabbitmqctl list_queues --online pid name synchronised_slave_pids | egrep "(cert|conductor|console|consoleauth| scheduler)"

for neutron:
rabbitmqctl list_queues --online pid name synchronised_slave_pids | egrep "(lbaas|l3|dhcp|metering|q-agent|q-plugin|q-reports|ipsec)"

for cinder:
rabbitmqctl list_queues --online pid name synchronised_slave_pids | grep cinder
```

## RabbitMQ 运维

### rabbitmq channel 过多

#### 故障现象

在rabbitmq server 端执行以下命令：

```
$ watch -n2 "rabbitmqctl list_connections channels name | sort -k1,1nr | head -20"
执行结果：
Every 2.0s: rabbitmqctl list_connections channels name | sort -k1,1nr | head -20                                                                           Mon Feb 29 10:16:45 2016
1011    10.3.0.35:32803 -> 10.3.0.44:5672
1823    10.3.0.35:33315 -> 10.3.0.44:5672
```
channel 链接数过多，从系统内部看，内存使用量大，系统负载高。

云平台现象：openstack 平台反应速度慢。

#### 故障原因

rabbitmq client端，在出现异常之后，不能正常的关闭channel，导致channel越来越多，每个channel都会占用系统资源，最后拖垮server端。

#### 影响范围

openstack整个平台,创建删除查询各种资源，反应缓慢。

#### 解决步骤

解决思路：

- 定位不断创建channal的connection；
- 通过connection找到channel；
- 通过channel找到consume；
- 通过consume找到openstack组件
- 重启openstack组件

这里我们通过rabbitmq自带的web管理界面进行定位。

step1：web 浏览器登录管理界面

IP：15672

step2： 定位不断创建channal的connection

```
$ watch -n2 "rabbitmqctl list_connections channels name | sort -k1,1nr | head -20"
执行结果：
Every 2.0s: rabbitmqctl list_connections channels name | sort -k1,1nr | head -20                                                                           Mon Feb 29 10:16:45 2016
1011    10.3.0.35:32803 -> 10.3.0.44:5672
1823    10.3.0.35:33315 -> 10.3.0.44:5672
```
"1011	10.3.0.35:32844 -> 10.3.0.44:5672" 表示："10.3.0.35:32803 -> 10.3.0.44:5672"这个connection，一共创建了1011个channel.

step3: web 端定位connection

connections -> filter: 10.1.0.35:32844

step4: 定位channel

通过channel 定位 openstack 组件

channel -> consumers -> fond the openstack services

step5: 重启定位到的 openstack 组件

#### 效果验证

```
 $ watch -n2 "rabbitmqctl list_connections channels name | sort -k1,1nr | head -20"
执行结果：
Every 2.0s: rabbitmqctl list_connections channels name | sort -k1,1nr | head -20                                                                           Mon Feb 29 10:16:45 2016
1   10.3.0.35:32803 -> 10.3.0.44:5672
1   10.3.0.35:33315 -> 10.3.0.44:5672
```

"1011 10.3.0.35:32844 -> 10.3.0.44:5672" 表示："10.3.0.35:32803 -> 10.3.0.44:5672"这个connection，一共创建了1个channel。这种表示，集群恢复正常。

#### 参考

https://bugs.launchpad.net/oslo.messaging/+bug/1406629


### rabbitmq queue 僵尸队列

#### 故障现象

一个cinder-scheduler 进程对应一个cinder-scheduler_fanout_{uuid}队列。我们现有 39 40 两个api 节点，含有两个cinder-scheduler进程，对应两个cinder-scheduler_fanout_{uuid}队列，如下所示，含有三个队列，说明有一个僵尸队列。（一般是堆积消息的队列为僵尸队列）

查询队列对应的comsume

```
$ rabbitmqctl list_consumers |grep cinder-scheduler_fanout_
cinder-scheduler_fanout_89ec88c1f9ce404089d17e68250505bb <rabbit@server-44.3.8145.28> 3 true 0 []
cinder-scheduler_fanout_ee07d2cb126c4378b99bf11007aa879b <rabbit@server-45.2.16539.60> 3 true 0 []
```

此时发现 cinder-scheduler_fanout_5720c0511f654740bb639de7282a3ed0 这个队列没有consume，确认此queue为僵尸队列。

#### 故障原因

一个组件突然断掉，当期再次加入集群中时 ，不是复用原先的队列，而是创建新的队列。而原来的队列依然绑定在exchange上，这样，从exchange 路由过来的消息依然会发送到老队列上，老队列上没有“consumer”与之对应，导致消息队列的堆积。

#### 故障影响

消息不断堆积，而无消费者消费消息，会消耗资源，从而触发rabbitmq的拥塞控制机制，降低生产者向rbbitmq发送消息的速率。直接感受是：云平台各种操作，反应变慢。

#### 解决步骤

直接删除此队列: `rabbitmqadmin delete queue name='xxxx'`

#### 验证效果

两个cinder scheduler 对应两个cinder-scheduler_fanout_{uuid}，观察一段时间后，如下图所示，说明僵尸队列删除成功。

```
$ rabbitmqctl list_queues |grep cinder-scheduler
cinder-scheduler    0
cinder-scheduler.cinder 0
cinder-scheduler_fanout_89ec88c1f9ce404089d17e68250505bb    0
cinder-scheduler_fanout_ee07d2cb126c4378b99bf11007aa879b    0
```

### rabbitmq 消息堆积

#### 故障现象

```
执行下面命令看看具体是哪个队列的消息数量超过500

# rabbitmqctl list_queues messages name | grep -v ^0
Listing queues ...
23 cinder-volume_fanout_c2424cb749ab4ef090dd08fd1c695693
62 notifications.error
36496 manila-scheduler_fanout_40fb9b27dd864b72a84c6404230a4726
66961 cinder-scheduler_fanout_594f38c357c149328e7d2006fd1ea0f0
```
上面的命令输出的第一列是消息数量，第二列是队列名称。

#### 故障原因

消费者假死，队列中消息无人消费。具体原因，还需定位具体组件，通过日志进行排查。

#### 故障范围

出问题openstack 的组件无法正常工作。

#### 解决步骤

首先，请参 rabbitmq queue僵尸队列 的解决方法，若不是僵尸队列，在查到相应的组件后，重启这个组件即可。

#### 效果验证

重启组件后，执行以下命令，可以观察到queue中，消息的数量会慢慢变少，直到合理的水平(小于五)。
`# rabbitmqctl list_queues messages name | grep -v ^0`

### rabbit 网络分区

#### 故障现象
```
# rabbitmqctl cluster_status
Cluster status of node 'rabbit@server-45' ...
[{nodes,[{disc,['rabbit@server-44','rabbit@server-45','rabbit@server-46']}]},
 {running_nodes,['rabbit@server-45']},
 {partitions,[{'rabbit@server-45',['rabbit@server-44','rabbit@server-46']}]}]
...done.
```

上面的例子中，查看这个字段：`{partitions,[{'rabbit@server-45',['rabbit@server-44','rabbit@server-46']}]}`

这个字段说明：RabbitMQ集群有三个节点，出现了网络两个分区，一个分区是server-45，另一个分区是 server-44 和 server-46；

#### 故障原因

 RabbitMQ cluster不能很好地处理Network Partition。RabbitMQ将queue、exchange、bindings等信息存储在Erlang的分布式数据库Mnesia中。所以出现Network partition时RabbitMQ的众多行为与Mnesia的行为密切相关。
    若某一node在一段时间内不能与另一node取得联系，则Mnesia认为未能与之取得联系的node宕掉了。若两个node彼此恢复联系了，但都曾以为对方宕掉了，则Manesia断定发生过Network partition。
    若发生了network partition，cluster中的双方（或多方）将独立存在，每一方都将认为其他方已经崩溃了。Queues、bindings、exchanges可以各自独立的创建、删除。对于Mirrored queues，处于不同network partition的每一方都会拥有各自的master，且各自独立的读写。（也可能发生其他诡异的行为）。若network partition恢复了，cluster的状态并不能自动恢复到network partition发生前的状态，直至采取措施进行修复。
以下为我们集群中常见的， 引发网络分区的原因。
网络不稳定；
 rabbitmq node虚拟机异常,但是异常后，快速恢复；

#### 故障范围

发生网络分区时，凡是使用rabbitmq的组件，表现诡异：一会可用，一会不可用。这种情况下，我们的监控会发现侧问题，并会向我们的监控人员发送报警信息。

#### 解决步骤

此类问题，解决思路是，把各自独立的节点，重新组合在一起。我们的做法是：

- 选取一个主节点，然后让其他出现分区的节点以此节点为基础，加入集群
- 在从节点依次执行：/etc/init.d/rabbitmq stop （新部署集群，可执行reboot）
- 在从节点依次执行：/etc/init.d/rabbitmq start

这里，按照以上步骤执行，有几个关键点：

- 主节点的选取
- rabbitmq node无法启动（“/etc/init.d/rabbitmq start”命令失败）
- rabbitmq node 启动后，依然为一个单独分区

1. 主节点的选取

在部分集群中，我们虽然使用了三个节点组成集群，但是实际使用的只是一个节点。针对这种情况，以使用的节点为主，重启其他节点。如果使用了三个节点，我们还需进行判断。
判断是否只是使用了一个节点：
rabbitmq 节点执行以下命令：
```
grep rabbit_hosts /etc/nova/nova.conf
------------------------------------
命令执行结果：a
rabbit_hosts=10.23.4.44
命令执行结果：b
rabbit_hosts=l4.0.zhsh.ustack.in:5672
命令执行结果：c
rabbit_hosts=10.50.0.44:5672,10.50.0.45:5672,10.50.0.46:5672
```
命令执行结果为 a ，10.23.4.44即为我们选取的主节点；
命令执行结果为 b，(a)；
命令执行结果为 c，(b);

(a) haproxy后端rabbitmq node主节点的判断

在 haproxy 节点执行：
```
cat /etc/haproxy/haproxy.cfg
--------------------------------
listen rabbitmq_proxy
  bind 0.0.0.0:5672
  mode  tcp
  balance  source
  option  tcpka
  option  tcplog
  server server-44 10.3.0.44:5672  check inter 5000 rise 2 fall 3
```
以上命令，主要查看 rabbitmq_proxy 字段server中的内容。在此例中，haproxy转发到的rabbitmq node为server-44。server-44 即为我们选取的主节点。若此处配置了三个server，go to (b)


(b) 三个rabbitmq node中选取主节点

如果openstack 使用了三个节点作为rabbitmq的主节点。我们分别在担当rabbitmq cluster node 角色的44，45，46三个节点执行以下命令：

```
lsof -nP -i:5672 |wc -l
```
选取其中的最大值作为rabbitmq 的主节点。

2. rabbitmq node 无法启动

遇到这种情况，在无法启动的rabbitmqmq node 执行以下命令，让node下次无条件的启动。`rabbitmqctl force_boot`

新集群部署时，在rabbitmq server 停掉之前，更改cookie会导致rabbitmq server无法启动，执行一下命令删除之前配置，重置配置。`rm /var/lib/rabbitmq/mnesia/rabbit/* -f`

3. rabbitmq node 重新加入cluster

确认节点处于运行状态：`/etc/init.d/rabbit-server status`

重新加入集群:

```
rabbit2$ rabbitmqctl stop_app
rabbit2$ rabbitmqctl join_cluster rabbit@server-44
rabbit2$ rabbitmqctl start_app
```

#### 自动化解决

rabbitmq一共有三种处理方式自动的恢复分区：ignore，autoheal，pause_minority。

默认的处理方式是ignore，即什么也不做。
 1. ignore为默认的处理方式，即什么也不做
 2, autoheal的处理方式：简单来讲就是当网络分区恢复后，rabbitmq各分区彼此进行协商，分区中客户端连接数最多的为胜者，其余的全部会进行重启，这样就会恢复到同步的状态了
 3. pause_minority的处理方式：rabbitmq节点感知集群中其他节点down掉时，会判断自己在集群中处于多数派还是少数派，也就是判断与自己形成集群的节点个数在整个集群中的比例是否超过一半。如果是多数派，则正常工作，如果是少数派，则会停止rabbit应用并不断检测直到自己成为多数派的一员后再次启动rabbit应用。注意：这种处理方式集群通常由奇数个节点组成。在CAP中，优先保证了CP

我们新部署的集群，默认使用autoheal这种处理方式。其配置如下：
```
cat /etc/rabbitmq/rabbitmq.config
% This file managed by Puppet
% Template Path: rabbitmq/templates/rabbitmq.config
[
  {rabbit, [
    {vm_memory_high_watermark, 0.8},
    {vm_memory_high_watermark_paging_ratio, 0.75},
    {cluster_nodes, {['rabbit@server-44', 'rabbit@server-45', 'rabbit@server-46'], disc}},
    {cluster_partition_handling, autoheal},
    {default_user, <<"guest">>},
    {default_pass, <<"guest">>}
  ]},
  {kernel, [
  ]}
].
% EOF
```
#### 效果验证

```
# rabbitmqctl cluster_status
Cluster status of node 'rabbit@server-44' ...
[{nodes,[{disc,['rabbit@server-44','rabbit@server-45','rabbit@server-46']}]},
 {running_nodes,['rabbit@server-45','rabbit@server-46','rabbit@server-44']},
 {partitions,[]}]
...done.
```
若执行命令后，如上所示，说明集群恢复正常。

#### rabbitmq usage

有的时候我们需要监听rabbitmq exchange的消息，可以通过以下代码来实现：

```
import pika
import json

credentials = pika.PlainCredentials('openstack', '43347ac7d1de4ef4')
params = pika.ConnectionParameters(host='192.168.17.13',credentials=credentials)
connection = pika.BlockingConnection(params)
channel = connection.channel()

exchange_name = 'neutron'
queue_name = channel.queue_declare(exclusive=True, durable=True).method.queue
binding_key = 'notifications.info'

channel.exchange_declare(exchange=exchange_name, durable=True, type='topic')
channel.queue_bind(exchange=exchange_name,
queue=queue_name,
routing_key=binding_key)

print ' [*] Waiting for logs. To exit press CTRL+C'

def callback(ch, method, properties, body):
    b= json.loads(body)
    print b['event_type'], b['payload']
# for key,value in b.iteritems():
# print key,':',value

channel.basic_consume(callback,queue=queue_name,no_ack=True)
channel.start_consuming()
```