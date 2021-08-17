2017/11/06

# Openstack periodic_task 周期性任务

## 需求

在工作中遇到了这样的需求：需要不断的轮询数据库，来做一些操作。

## 解决方案一：添加`periodic_task`函数

在模块的`manager.py`中增加`periodic_task`装饰的周期性函数。
### 用法
```
@periodic_task.periodic_task(spacing=60,run_immediately=True)
def _instance_usage_audit(self, context):
    ...
```
将此装饰方法放到要周期执行的函数上
```
* spacing 每隔多长时间执行一次
* run_immediately 立即执行
```

## 解决方案二：`oslo_service`中的`loopingcall`

## 解决方案三
简化实现方式：用python 现有的模块以及技术来实现。

* celery, celery beat
* PythonRQ in combination with RQ Scheduler
* 

## References
* http://www.cnblogs.com/d-monkey/p/5299222.html
* http://www.cnblogs.com/yuhan-TB/p/4085074.html
* http://blog.csdn.net/halcyonbaby/article/details/23388891
* http://gtcsq.readthedocs.io/en/latest/openstack/periodic_task.html
* https://docs.openstack.org/oslo.service/latest/reference/loopingcall.html
* https://docs.openstack.org/oslo.service/latest/reference/periodic_task.html
* http://gtcsq.readthedocs.io/en/latest/openstack/periodic_task.html
