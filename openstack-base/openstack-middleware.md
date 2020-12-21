---
date: 2017-09-13T14:39:47+08:00
title: openstack middleware
---

## Background

我们知道openstack 处理一个api请求，会经过多个middleware的处理，最终到达处理函数。有时候我们会有这样的需求：
比如：拦截openstack的api请求，来做计费处理。
比如：拦截openstack的api，根据请求的类型来生成操作记录，展示给用户。本文来讨论开发middleware的步骤、优缺点。

## Deploy middleware

{{< code "code/middleware_example.py" >}}

## middleware Developer Doc

## 1. 安装与配置

### 1.1 安装

首先clone项目到本地:

```
git clone <repo_url>
```

安装pip:

```
yum install -y python-pip
```

切换到项目根目录，执行以下命令完成安装:

```sh
cd billingmiddleware
pip install .
```

安装后检查,执行以下命令:

```python
file /usr/lib/python2.7/site-packages/billingmiddleware/billing.py \
        | grep -q 'No such file or directory' \
        && echo "fail to install billing middleware" \
        || echo 'install sucessfully'
```
如果安装成功，会输出`install sucessfully`。


### 1.2 配置

Billing middleware与其它keystone middleware配置完全一样，只需要修改各个服务的paste配置即可。以nova为例，修改`/etc/nova/api-paste.ini`文件，在`keystone pipelines`下增加`billing`。

**注意：`billing`需要放在`authtoken`后面**

```
[composite:openstack_compute_api_legacy_v2]
use = call:nova.api.auth:pipeline_factory
noauth2 = cors compute_req_id faultwrap sizelimit noauth2 legacy_ratelimit osapi_compute_app_legacy_v2
keystone = cors compute_req_id faultwrap sizelimit authtoken keystonecontext billing audit legacy_ratelimit osapi_compute_app_legacy_v2
keystone_nolimit = cors compute_req_id faultwrap sizelimit authtoken keystonecontext billing audit osapi_compute_app_legacy_v2

[composite:openstack_compute_api_v21]
use = call:nova.api.auth:pipeline_factory_v21
noauth2 = cors compute_req_id faultwrap sizelimit noauth2 osapi_compute_app_v21
keystone = cors compute_req_id faultwrap sizelimit authtoken keystonecontext billing audit osapi_compute_app_v21

[composite:openstack_compute_api_v21_legacy_v2_compatible]
use = call:nova.api.auth:pipeline_factory_v21
noauth2 = cors compute_req_id faultwrap sizelimit noauth2 legacy_v2_compatible osapi_compute_app_v21
keystone = cors compute_req_id faultwrap sizelimit authtoken keystonecontext billing audit legacy_v2_compatible osapi_compute_app_v21
```

增加billing filter:

```
[filter:billing]
paste.filter_factory = billingmiddleware.billing:filter_factory
billing_wsgi_url = http://localhost:8774
pre_ignore_method = HEAD, TRACE
post_ignore_method = HEAD, TRACE
```

配置项如下：

* `paste.filter_factory`: 配置该中间件的模块地址，该配置项不需要修改。
* `billing_wsgi_url`: 配置计费模块网关地址，需要指定协议，如`http://`。
* `pre_ignore_method`: 配置发往OpenStack API之前需要忽略的请求方法，比如HEAD、TRACE等。
* `pre_ignore_url`: 配置发往OpenStack API之前需要忽略的请求URL,默认为空。
* `post_ignore_method`: 配置发往OpenStack API之后需要忽略的请求方法，比如HEAD、TRACE等。
* `post_ignore_url`: 配置发往OpenStack API之后需要忽略的请求URL。

### 1.3 重启服务

修改了paste配置，需要重启API服务:

```
systemctl restart openstack-nova-api
```

## 2 使用文档

计费网关必须通过POST方法接受数据，所有的源数据都已经转化为json格式并包装在body中。

* `request_id`: 请求id，用于关联请求前和请求后。
* `request_url`: 请求URL地址。
* `request_method`: 请求方法。
* `request_body`: 请求的数据。
* `request_headers`: 请求的头部。
* `request_params`: 请求的URL参数。
* `response_headers`: 响应头部信息。
* `response_text`: 响应返回的数据。
* `response_status`: 响应状态信息。

