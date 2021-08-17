2017/11/02
# openstack 日志格式和收集内容
## 预备知识
Openstack的日志由oslo.log统一管理，用法也很简单，导入oslo.log模块即可:
```
from oslo_log import log as logging
LOG = logging.getLogger(__name__)
...
LOG.debug(...)
LOG.info(...)
LOG.warning(...)
LOG.error(...)
LOG.exception(...)
...
```
注意，除了日志的路径、debug、verbose(废弃)外，所有关于日志的配置都从组件的配置中分离出来,有单独的配置文件管理，配置文件由log_config_append指定。配置Demo如下：
```
# nova_sample.conf
[loggers]
keys = root, nova

[handlers]
keys = stderr, stdout, watchedfile, syslog, fluent, null

[formatters]
keys = context, default, fluent

[logger_root]
level = WARNING
handlers = null

[logger_nova]
level = INFO
handlers = stderr
qualname = nova

[logger_amqp]
level = WARNING
handlers = stderr
qualname = amqp

[logger_amqplib]
level = WARNING
handlers = stderr
qualname = amqplib

[logger_sqlalchemy]
level = WARNING
handlers = stderr
qualname = sqlalchemy
# "level = INFO" logs SQL queries.
# "level = DEBUG" logs SQL queries and results.
# "level = WARNING" logs neither.  (Recommended for production systems.)

[logger_boto]
level = WARNING
handlers = stderr
qualname = boto

# NOTE(mikal): suds is used by the vmware driver, removing this will
# cause many extraneous log lines for their tempest runs. Refer to
# https://review.openstack.org/#/c/219225/ for details.
[logger_suds]
level = INFO
handlers = stderr
qualname = suds

[logger_eventletwsgi]
level = WARNING
handlers = stderr
qualname = eventlet.wsgi.server

[handler_stderr]
class = StreamHandler
args = (sys.stderr,)
formatter = context

[handler_stdout]
class = StreamHandler
args = (sys.stdout,)
formatter = context

[handler_watchedfile]
class = handlers.WatchedFileHandler
args = ('nova.log',)
formatter = context

[handler_syslog]
class = handlers.SysLogHandler
args = ('/dev/log', handlers.SysLogHandler.LOG_USER)
formatter = context

[handler_fluent]
class = fluent.handler.FluentHandler
args = ('openstack.nova', 'localhost', 24224)
formatter = fluent

[handler_null]
class = logging.NullHandler
formatter = default
args = ()

[formatter_context]
class = oslo_log.formatters.ContextFormatter

[formatter_default]
format = %(message)s

[formatter_fluent]
class = oslo_log.formatters.FluentFormatter
```
以上是需要对日志进行定制化时的配置，通常情况我们使用默认配置就够了。因此文档后续都假定我们使用的默认配置。
## 日志格式
### 日志前缀
日志前缀包含了日志内容以外的部分，由logging_context_format_string配置，默认值为%(asctime)s.%(msecs)03d %(process)d %(levelname)s %(name)s [%(request_id)s %(user_identity)s] %(instance)s%(message)s。
其中:

* asctime，时间戳: 默认格式yyyy-mm-dd HH:MM:SS.xxx（xxx为毫秒)，可以通过log_date_format配置，如%Y-%m-%d %H:%M:%S。
* process: 整型值，进程PID。
* levelname，等级名称: 如INFO、DEBUG、ERROR等。
* name，模块名: 日志产生的模块名，由模块的__name__决定。如nova.osapi_compute.wsgi.server。
* Request ID/user_identity：这个是对应API请求的request id以及请求的用户，用于跟踪请求，格式是[req-id user_id project_id]，如[req-a9e3ae4f-38a0-4cb9-94b8-363b98629814 24d97e40fec0477bb75fdd8660a7b66f 5219429288634d579d1da8a8539cf43c - - ], 用[-]表示。
* instance: nova-compute特有，instance的uuid，格式为instance_format="[instance: %(uuid)s] ",通常该项为空。

## API日志格式
API日志主要记录API请求，方便审计、排错。以nova为例，API日志格式由wsgi_log_format配置项决定，默认值为:
```
wsgi_log_format=%(client_ip)s "%(request_line)s" status: %(status_code)s len: %(body_length)s time: %(wall_seconds).7f
```
注意`wsgi_log_format`不是一条完整的日志格式，而是日志的后半部分，完整的日志还要包含前面提到的日志前缀。
其中:

* client_ip是API请求的来源IP。
* request_line即请求的API,格式为"请求方法 请求路径 协议版本"（引号也包含)，如"POST /v2/7be9709cb4a0478e92bc66f2f98c2cfe/os-server-external-events HTTP/1.1"。
* status_code状态返回码，格式为status: xxxx，如status: 200。
* body_length，body长度，格式为len: xxx，比如len: 2057。
* wall_seconds，API响应时间，数据格式为7位浮点数，如time: 0.0006649，单位为秒。

完整的日志格式为日志前缀+API日志, demo如下：
```
2017-01-16 11:51:13.322 20502 INFO nova.osapi_compute.wsgi.server [req-a9e3ae4f-38a0-4cb9-94b8-363b98629814 24d97e40fec0477bb75fdd8660a7b66f 5219429288634d579d1da8a8539cf43c - - -] 10.0.2.34 "GET /v2/5219429288634d579d1da8a8539cf43c/servers/ed5768f4-0582-4d9d-9eba-2a5115628215 HTTP/1.1" status: 200 len: 2057 time: 0.1698310
```
## 一般日志格式
一般的日志格式比较简单，直接由日志前缀+日志内容组成，可通过logging_default_format_string配置，默认值为%(asctime)s.%(msecs)03d %(process)d %(levelname)s %(name)s [-] %(instance)s%(message)s。需要注意的是一条日志可能会拆分成多条，根据时间戳区别。以下是一个日志example:
```
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task [req-f5d223da-b40c-4a02-9632-900b5ab499a1 - - - - -] Error during ComputeManager.update_available_resource
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task Traceback (most recent call last):
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task   File "/usr/lib/python2.7/site-packages/oslo_service/periodic_task.py", line 220, in run_periodic_tasks
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task     task(self, context)
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task   File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 6486, in update_available_resource
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task     use_slave=True)
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task   File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 6522, in _get_compute_nodes_in_db
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task     use_slave=use_slave)
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task   File "/usr/lib/python2.7/site-packages/oslo_versionedobjects/base.py", line 174, in wrapper
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task     args, kwargs)
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task   File "/usr/lib/python2.7/site-packages/nova/conductor/rpcapi.py", line 240, in object_class_action_versions
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task     args=args, kwargs=kwargs)
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task   File "/usr/lib/python2.7/site-packages/oslo_messaging/rpc/client.py", line 158, in call
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task     retry=self.retry)
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task   File "/usr/lib/python2.7/site-packages/oslo_messaging/transport.py", line 90, in _send
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task     timeout=timeout, retry=retry)
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task   File "/usr/lib/python2.7/site-packages/oslo_messaging/_drivers/amqpdriver.py", line 470, in send
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task     retry=retry)
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task   File "/usr/lib/python2.7/site-packages/oslo_messaging/_drivers/amqpdriver.py", line 461, in _send
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task     raise result
2017-01-16 08:05:53.757 40213 ERROR oslo_service.periodic_task RemoteError: Remote error: DBConnectionError (_mysql_exceptions.OperationalError) (2013, 'Lost connection to MySQL server at \'reading initial communication packet\', system error: 0 "Internal error/check (Not system error)"') [SQL: u'SELECT 1']
```
## 日志收集

### Keystone

* 收集API请求。
* keystone.common.wsgi的WARNING,记录认证错误信息，需要开启insecure_debug模式。参考关于insecure_debug模式。

### Glance

* 收集api请求，eventlet.wsgi.server，包括字段参考Nova。
* 所有的ERROR日志。

### Nova

* 收集api请求，包括的字段为时间戳、client IP、日志等级、模块名称、请求方法、请求路径、响应时间、日志内容。
* ERROR日志，所有组件都需要收集，用于故障排查。
* 计算节点,nova.compute.manager日志，记录了该计算节点的所有虚拟机操作,收集字段为所有的前缀字段加上instance uuid、action。这里我们可以找出reboot或者shutdown慢的原因，soft-reboot or hard-reboot？
* 计算节点nova.virt.libvirt.driver日志，记录了所有libvirt操作，收集字段为所有的前缀字段加上instance uuid、action。虚拟机操作失败时有用。
* scheduler节点，收集nova.filters日志，记录调度流程,调度失败时哪个filter没有过。创建虚拟机、迁移虚拟机出错时需要。

### Cinder

* 收集api请求，参考Nova。
* 所有的ERROR日志。
* volume节点，收集cinder.volume.manager、cinder.volume.flows.manager.xxxx日志，记录volume行为。

### Neutron

* 收集api请求，参考Nova。
* 所有的ERROR日志，warning日志。
* 网络节点，计算节点，ovs-vswitchd.log的WARN。

### Heat

### Ceilometer

* 所有ERROR日志

### Aodh

* api 请求，参考nova
* 所有ERROR日志

### Gnocchi
* 所有ERROR日志
