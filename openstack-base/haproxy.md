## haproxy

[文档官网](http://cbonte.github.io/haproxy-dconv/)

### 简介及作用

#### HAProxy 是什么

* TCP 代理：可从监听 socket 接受 TCP 连接，然后自己连接到 server，HAProxy 将这些 sockets attach 到一起，使通信流量可双向流动。
* HTTP 反向代理（在 HTTP 专用术语中，称为 gateway）：HAProxy 自身表现得就像一个 server，通过监听 socket 接受 HTTP 请求，然后与后端服务器建立连接，通过连接将请求转发给后端服务器。
* SSL terminator / initiator / offloader: 客户端 -> HAProxy 的连接，以及 HAProxy -> server 端的连接都可以使用 SSL/TLS
* TCP normalizer: 因为连接在本地操作系统处终结，client 和 server 端没有关联，所以不正常的 traffic 如 invalid packets, flag combinations, window advertisements, sequence numbers, incomplete connections(SYN floods) 不会传递给 server 端。这种机制可以保护脆弱的 TCP stacks 免遭协议上的攻击，也使得我们不必修改 server 端的 TCP 协议栈设置就可以优化与 client 的连接参数。
* HTTP normalizer: HAProxy 配置为 HTTP 模式时，只允许有效的完整的请求转发给后端。这样可以使得后端免遭 protocol-based 攻击。一些不规范的定义也被修改，以免在 server 端造成问题（eg: multiple-line headers，会被合并为一行）
* HTTP 修正工具：HAProxy 可以 modify / fix / add / remove / rewrite URL 及任何 request or response header。
* a content-based switch: 可基于内容进行转发。可基于请求中的任何元素转发请求或连接。因此可基于一个端口处理多种协议（http,https, ssh）
* a server load balancer: 可对 TCP 连接 和 HTTP 请求进行负载均衡调度。工作于 TCP 模式时，可对整个连接进行负载均衡调度；工作于 HTTP 模式时，可对 HTTP 请求进行调度。
* a traffic regulator: 可在不同的方面对流量进行限制，保护 server ，使其不超负荷，基于内容调整 traffic 优先级，甚至可以通过 marking packets 将这些信息传递给下层以及网络组件。
* 防御 DDos 攻击及 service abuse: HAProxy 可为每个 IP地址，URL，cookie 等维护大量的统计信息，并对其进行检测，当发生服务滥用的情况，采取一定的措施如：slow down the offenders, block them, send them to outdated contents, etc
* 是 network 的诊断的一个观察节点：根据精确记录细节丰富的日志，对网络诊断很有帮助
* an HTTP compression offloader：可自行对响应进行压缩，而不是让 server 进行压缩，因此对于连接性能较差的 client，或使用高延迟移动网络的 client，可减少页面加载时间。

#### HAProxy 不是什么

* 显式地 HTTP 代理，即不是浏览器用于访问 internet 的代理。Squid 可用于做这种事情。HAProxy 可安装在 Squid 的前端，提供负载均衡和高可用。
* 缓存代理：对于从服务器收到的内容，HAProxy 只能将其原样返回给 client，对 caching policy 不做任何干涉。对于这个任务， Varnish 是一个很好的选择。HAProxy 可安装在 Varnish 的前端提供 SSL offloading，以及可伸缩性（通过 smart load balancing）
* a data scrubber: HAProxy 不能修改请求或响应的 body 部分
* a web server: HAProxy 在启动时，它将自己隔离在一个 chroot 环境中，并抛弃了自己的特权，所以一旦启动之后，它不会进行任何对文件系统的访问。所以 HAProxy 不能以 web server 的方式工作。可作 web server 的是 Apache 或 Nginx，HAProxy 可安装在其前端提供提供负载均衡和高可用。
* a packet-based load balancer：基于 packet 的负载均衡调度器。HAProxy 不会看见 IP packet 或 UDP datagram，不能进行 NAT 转换，或者 less DSR。这些是下层的工作，比如 LVS 就做得很好，LVS 对 HAProxy 是完美的补充。


#### HAProxy 是如何工作的

HAProxy 由单线程的、事件驱动的、非阻塞的引擎， 以及一个非常快速的 I/O 层组合而成。拥有 “基于优先级” 的调度机制。

HAProxy 的设计理念是让数据尽可能快的移动。它在每一层设计了 bypass 机制，在不必要的时候，数据不会传递到上层，直接在下层进行传递。大部分的
处理都在 kernel 中完成，HAProxy 尽最大努力帮助 kernel 尽快完成工作，包括给一些 hints，或是当某些操作可稍后合并完成时，避免某些操作。

当 HAProxy 工作在 TCP 或者 HTTP close mode 时，HAProxy 消耗的 CPU 负载是 15%，内核消耗的负载是 85%。

当 HAProxy 工作在 HTTP keep-alive mode 时，HAProxy 消耗的 CPU 负载是 30%，内核消耗的负载是 70%。

一个 HAProxy 进程可以运行许多 proxy 实例；一个进程可容纳 300000 万个实例而运行良好。因此一般不需要启动多个进程。

HAProxy 被用来做 SSL offloader 时，可使用多进程。

运行 HAProxy 只需要一个 haproxy 可执行程序和一份配置文件即可运行。要记录日志，强烈建议配置好 syslog daemon 及 rotation 机制。配置文件在启动前被解析，HAProxy 尝试绑定所有的 监听 sockets，如果有地方失败就不会启动。一旦成功启动，HAProxy 不会再失效，也就是说不会有 runtime failures，一旦启动，它会工作到被关闭为止。
HAProxy 一旦启动，会做三件事情：
* 处理客户端接入的连接
* 周期性检查 server 的状态（健康检查）
* 与其他 haproxy 交换信息

处理客户端接入的连接，是目前为止最为复杂的工作，因为配置有太多的可能性，但总的说来有 9 个步骤：

* 配置实体 frontend 拥有监听 socket，HAProxy 从它的监听 socket 处接受客户端连接
* 根据 frontend 配置的规则，对连接进行处理。可能会拒绝一些连接，修改一些 headers，或是拦截连接，执行内部的小程序，比如统计页面，或者 CLI
* backend 是定义后端 servers，以及负载均衡规则的配置实体，frontend 完成上面的处理后将连接转发给 backend。
* 根据 backend 定义的规则，对连接进行处理
* 根据负载均衡规则对连接进行调度
* 根据 backend 定义的规则对 response data 进行处理
* 根据 frontend 定义的规则对 response data 进行处理
* 发起一个 log report，记录日志
* 在 HTTP 模式，回到第二步，等待新的请求，或者关闭连接。

frontend 和 backend 有时被认为是 half-proxy，因为他们对一个 end-to-end（端对端）的连接只关心一半：frontend 只关心 client，backend 只关心 server。

HAProxy 也支持 full proxy，通过对 frontend 和 backend 的准确联合来工作。

HAProxy 工作于 HTTP 模式时，配置被分裂为 frontend 和 backend 两个部分，因为任何 frontend 可能转发连接给 任何 backend。

HAProxy 工作于 TCP 模式时，实际上就是 full proxy 模式，配置中使用 frontend 和 backend 不能提供更多的好处，在 full proxy 模式，配置文件更有可读性。

### haproxy 会话保持的三种方法

#### session 知识

Session是由应用服务器维持的一个服务器端的存储空间，用户在连接服务器时，会由服务器生成一个唯一的SessionID,用该SessionID为标识符来存取服务器端的Session存储空间。而SessionID这一数据则是保存到客户端，用Cookie保存的，用户提交页面时，会将这一 SessionID提交到服务器端，来存取Session数据。
服务器也通过URL重写的方式来传递SessionID的值，因此不是完全依赖Cookie。如果客户端Cookie禁用，则服务器可以自动通过重写URL的方式来保存Session的值，并且这个过程对程序员透明。在后端应用服务器上php.ini 里几个session相关值的，可以进行简单设置
session.use_cookies = 1 #服务端和客户端交互session是通过cookie的方式 默认值
session.name = LXSYM #默认值是PHPSESSID 可以自行定义。比如LXSYM
session.cache_limiter = nocache #此设置确保对每个请求，在可能提供缓存的版本前，先请求发送到最初的服务器。
针对session数据推荐使用共享存储，实现方法很多。比如存于多个memcached中。

#### 实现haproxy 与客户端session一致的方法

- 用户IP 识别
haroxy 将用户IP经过hash计算后 指定到固定的真实服务器上。
配置指令 balance source (如: balance uri len 100)
- cookie 识别
haproxy 将WEB服务端发送给客户端的cookie中插入(或添加前缀)haproxy定义的后端的服务器COOKIE ID。
配置指令例举 cookie SESSION_COOKIE insert indirect nocache
可以使用firebug可以观察到用户的请求头的cookie信息
- session 识别
haproxy 将后端服务器产生的session和后端服务器标识存在haproxy中的一张表里。客户端请求时先查询这张表。
配置指令例举 appsession LXSYM len 64 timeout 5h request-learn
注意LXSYM这个值替换成 你的php.ini 里session.name的值。

### 配置及使用

```
# cat /etc/haproxy/haproxy.conf

global

    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

listen  web_proxy  *:8788
        option  persist
        balance roundrobin
        server  web1  192.178.1.4:8787 check
        server  web2  192.158.1.4:8787 check
        server  web3  192.168.1.8:8787 check

```

启动`haproxy`

`systemctl restart haproxy`

访问： `curl http://localhost:8788` 该请求会被转发到web1, web2, web3 上。

### 监控

```
listen stats
  bind 0.0.0.0:1936
  mode http
  stats enable
  stats realm Haproxy\ Statistics
  stats uri /
  stats auth admin:R00tme@1234
```
### 参数解释

1. `tune.bufsize  32768`

### our current configuration:

```
# This file managed by Puppet
global
  chroot  /var/lib/haproxy      # chroot运行路径，增加安全性
  daemon                        # 以守护进程的方式工作于后台
  group  haproxy                # 运行haproxy用户所属的组
  log  127.0.0.1 local2 info
  maxconn  4000                 # 默认的最大连接数
  pidfile  /var/run/haproxy.pid
  stats  socket /var/lib/haproxy/stats
  tune.bufsize  32768
  user  haproxy

defaults
  log  global
  maxconn  8000
  mode  http                    # 默认使用协议，可以为｛http|tcp|health｝http:是七层协议，tcp是四层, health:只返回ok
  option  redispatch            # ServerID对应的服务器宕机后，强制定向到其他运行正常的服务器
  retries  3                    # 3次连接失败则认为服务不可用
  timeout  http-request 10s     # 默认http请求超时时间
  timeout  queue 1m             # 默认队列超时时间
  timeout  connect 10s          # 默认连接超时时间
  timeout  client 5m            # 默认客户端超时时间
  timeout  server 5m            # 默认服务器端超时时间
  timeout  check 10s            # 心跳检测超时
```

### The options we used in haproxy.cfg:

#### mode

```
mode { tcp|http|health }
  Set the running mode or protocol of the instance
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  Arguments :
    tcp       The instance will work in pure TCP mode. A full-duplex connection
              will be established between clients and servers, and no layer 7
              examination will be performed. This is the default mode. It
              should be used for SSL, SSH, SMTP, ...

    http      The instance will work in HTTP mode. The client request will be
              analyzed in depth before connecting to any server. Any request
              which is not RFC-compliant will be rejected. Layer 7 filtering,
              processing and switching will be possible. This is the mode which
              brings HAProxy most of its value.

    health    The instance will work in "health" mode. It will just reply "OK"
              to incoming connections and close the connection. Alternatively,
              If the "httpchk" option is set, "HTTP/1.0 200 OK" will be sent
              instead. Nothing will be logged in either case. This mode is used
              to reply to external components health checks. This mode is
              deprecated and should not be used anymore as it is possible to do
              the same and even better by combining TCP or HTTP modes with the
              "monitor" keyword.

  When doing content switching, it is mandatory that the frontend and the
  backend are in the same mode (generally HTTP), otherwise the configuration
  will be refused.

  Example :
     defaults http_instances
         mode http

  See also : "monitor", "monitor-net"
```

#### tune.bufsize

```
tune.bufsize <number>
  Sets the buffer size to this size (in bytes). Lower values allow more
  sessions to coexist in the same amount of RAM, and higher values allow some
  applications with very large cookies to work. The default value is 16384 and
  can be changed at build time. It is strongly recommended not to change this
  from the default value, as very low values will break some services such as
  statistics, and values larger than default size will increase memory usage,
  possibly causing the system to run out of memory. At least the global maxconn
  parameter should be decreased by the same factor as this one is increased.
  If HTTP request is larger than (tune.bufsize - tune.maxrewrite), haproxy will
  return HTTP 400 (Bad Request) error. Similarly if an HTTP response is larger
  than this size, haproxy will return HTTP 502 (Bad Gateway).
```

#### tune.maxrewrite

```
tune.maxrewrite <number>
  Sets the reserved buffer space to this size in bytes. The reserved space is
  used for header rewriting or appending. The first reads on sockets will never
  fill more than bufsize-maxrewrite. Historically it has defaulted to half of
  bufsize, though that does not make much sense since there are rarely large
  numbers of headers to add. Setting it too high prevents processing of large
  requests or responses. Setting it too low prevents addition of new headers
  to already large requests or to POST requests. It is generally wise to set it
  to about 1024. It is automatically readjusted to half of bufsize if it is
  larger than that. This means you don't have to worry about it when changing
  bufsize.
```

#### [global] maxconn

```
maxconn <number>
  Sets the maximum per-process number of concurrent connections to <number>. It
  is equivalent to the command-line argument "-n". Proxies will stop accepting
  connections when this limit is reached. The "ulimit-n" parameter is
  automatically adjusted according to this value. See also "ulimit-n". Note:
  the "select" poller cannot reliably use more than 1024 file descriptors on
  some platforms. If your platform only supports select and reports "select
  FAILED" on startup, you need to reduce maxconn until it works (slightly
  below 500 in general).

```

#### [defaults] maxconn

```
maxconn <conns>
  Fix the maximum number of concurrent connections on a frontend
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  Arguments :
    <conns>   is the maximum number of concurrent connections the frontend will
              accept to serve. Excess connections will be queued by the system
              in the socket's listen queue and will be served once a connection
              closes.

  If the system supports it, it can be useful on big sites to raise this limit
  very high so that haproxy manages connection queues, instead of leaving the
  clients with unanswered connection attempts. This value should not exceed the
  global maxconn. Also, keep in mind that a connection contains two buffers
  of 8kB each, as well as some other data resulting in about 17 kB of RAM being
  consumed per established connection. That means that a medium system equipped
  with 1GB of RAM can withstand around 40000-50000 concurrent connections if
  properly tuned.

  Also, when <conns> is set to large values, it is possible that the servers
  are not sized to accept such loads, and for this reason it is generally wise
  to assign them some reasonable connection limits.

  By default, this value is set to 2000.

  See also : "server", global section's "maxconn", "fullconn"
```

#### [server] maxconn

```
maxconn <maxconn>
  The "maxconn" parameter specifies the maximal number of concurrent
  connections that will be sent to this server. If the number of incoming
  concurrent requests goes higher than this value, they will be queued, waiting
  for a connection to be released. This parameter is very important as it can
  save fragile servers from going down under extreme loads. If a "minconn"
  parameter is specified, the limit becomes dynamic. The default value is "0"
  which means unlimited. See also the "minconn" and "maxqueue" parameters, and
  the backend's "fullconn" keyword.

  Supported in default-server: Yes

```

#### option clitcpka

```
option clitcpka
no option clitcpka
  Enable or disable the sending of TCP keepalive packets on the client side
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  Arguments : none

  When there is a firewall or any session-aware component between a client and
  a server, and when the protocol involves very long sessions with long idle
  periods (eg: remote desktops), there is a risk that one of the intermediate
  components decides to expire a session which has remained idle for too long.

  Enabling socket-level TCP keep-alives makes the system regularly send packets
  to the other end of the connection, leaving it active. The delay between
  keep-alive probes is controlled by the system only and depends both on the
  operating system and its tuning parameters.

  It is important to understand that keep-alive packets are neither emitted nor
  received at the application level. It is only the network stacks which sees
  them. For this reason, even if one side of the proxy already uses keep-alives
  to maintain its connection alive, those keep-alive packets will not be
  forwarded to the other side of the proxy.

  Please note that this has nothing to do with HTTP keep-alive.

  Using option "clitcpka" enables the emission of TCP keep-alive probes on the
  client side of a connection, which should help when session expirations are
  noticed between HAProxy and a client.

  If this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  See also : "option srvtcpka", "option tcpka"
```

#### option httpchk

```
option httpchk
option httpchk <uri>
option httpchk <method> <uri>
option httpchk <method> <uri> <version>
  Enable HTTP protocol to check on the servers health
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  Arguments :
    <method>  is the optional HTTP method used with the requests. When not set,
              the "OPTIONS" method is used, as it generally requires low server
              processing and is easy to filter out from the logs. Any method
              may be used, though it is not recommended to invent non-standard
              ones.

    <uri>     is the URI referenced in the HTTP requests. It defaults to " / "
              which is accessible by default on almost any server, but may be
              changed to any other URI. Query strings are permitted.

    <version> is the optional HTTP version string. It defaults to "HTTP/1.0"
              but some servers might behave incorrectly in HTTP 1.0, so turning
              it to HTTP/1.1 may sometimes help. Note that the Host field is
              mandatory in HTTP/1.1, and as a trick, it is possible to pass it
              after "\r\n" following the version string.

  By default, server health checks only consist in trying to establish a TCP
  connection. When "option httpchk" is specified, a complete HTTP request is
  sent once the TCP connection is established, and responses 2xx and 3xx are
  considered valid, while all other ones indicate a server failure, including
  the lack of any response.

  The port and interval are specified in the server configuration.

  This option does not necessarily require an HTTP backend, it also works with
  plain TCP backends. This is particularly useful to check simple scripts bound
  to some dedicated ports using the inetd daemon.

  Examples :
      # Relay HTTPS traffic to Apache instance and check service availability
      # using HTTP request "OPTIONS * HTTP/1.1" on port 80.
      backend https_relay
          mode tcp
          option httpchk OPTIONS * HTTP/1.1\r\nHost:\ www
          server apache1 192.168.1.1:443 check port 80

  See also : "option ssl-hello-chk", "option smtpchk", "option mysql-check",
             "option pgsql-check", "http-check" and the "check", "port" and
             "inter" server options.

```

#### option httpclose

```
option httpclose
no option httpclose
  Enable or disable passive HTTP connection closing
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  Arguments : none

  By default HAProxy operates in keep-alive mode with regards to persistent
  connections: for each connection it processes each request and response, and
  leaves the connection idle on both sides between the end of a response and
  the start of a new request. This mode may be changed by several options such
  as "option http-server-close", "option forceclose", "option httpclose" or
  "option http-tunnel".

  If "option httpclose" is set, HAProxy will work in HTTP tunnel mode and check
  if a "Connection: close" header is already set in each direction, and will
  add one if missing. Each end should react to this by actively closing the TCP
  connection after each transfer, thus resulting in a switch to the HTTP close
  mode. Any "Connection" header different from "close" will also be removed.
  Note that this option is deprecated since what it does is very cheap but not
  reliable. Using "option http-server-close" or "option forceclose" is strongly
  recommended instead.

  It seldom happens that some servers incorrectly ignore this header and do not
  close the connection even though they reply "Connection: close". For this
  reason, they are not compatible with older HTTP 1.0 browsers. If this happens
  it is possible to use the "option forceclose" which actively closes the
  request connection once the server responds. Option "forceclose" also
  releases the server connection earlier because it does not have to wait for
  the client to acknowledge it.

  This option may be set both in a frontend and in a backend. It is enabled if
  at least one of the frontend or backend holding a connection has it enabled.
  It disables and replaces any previous "option http-server-close",
  "option forceclose", "option http-keep-alive" or "option http-tunnel". Please
  check section 4 ("Proxies") to see how this option combines with others when
  frontend and backend options differ.

  If this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  See also : "option forceclose", "option http-server-close" and
             "1.1. The HTTP transaction model".
```

#### http-server-close

```
option http-server-close
no option http-server-close
  Enable or disable HTTP connection closing on the server side
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  Arguments : none

  By default HAProxy operates in keep-alive mode with regards to persistent
  connections: for each connection it processes each request and response, and
  leaves the connection idle on both sides between the end of a response and
  the start of a new request. This mode may be changed by several options such
  as "option http-server-close", "option forceclose", "option httpclose" or
  "option http-tunnel". Setting "option http-server-close" enables HTTP
  connection-close mode on the server side while keeping the ability to support
  HTTP keep-alive and pipelining on the client side.  This provides the lowest
  latency on the client side (slow network) and the fastest session reuse on
  the server side to save server resources, similarly to "option forceclose".
  It also permits non-keepalive capable servers to be served in keep-alive mode
  to the clients if they conform to the requirements of RFC2616. Please note
  that some servers do not always conform to those requirements when they see
  "Connection: close" in the request. The effect will be that keep-alive will
  never be used. A workaround consists in enabling "option
  http-pretend-keepalive".

  At the moment, logs will not indicate whether requests came from the same
  session or not. The accept date reported in the logs corresponds to the end
  of the previous request, and the request time corresponds to the time spent
  waiting for a new request. The keep-alive request time is still bound to the
  timeout defined by "timeout http-keep-alive" or "timeout http-request" if
  not set.

  This option may be set both in a frontend and in a backend. It is enabled if
  at least one of the frontend or backend holding a connection has it enabled.
  It disables and replaces any previous "option httpclose", "option forceclose",
  "option http-tunnel" or "option http-keep-alive". Please check section 4
  ("Proxies") to see how this option combines with others when frontend and
  backend options differ.

  If this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  See also : "option forceclose", "option http-pretend-keepalive",
             "option httpclose", "option http-keep-alive", and
             "1.1. The HTTP transaction model".
```

#### forceclose

```
option forceclose
no option forceclose
  Enable or disable active connection closing after response is transferred.
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  Arguments : none

  Some HTTP servers do not necessarily close the connections when they receive
  the "Connection: close" set by "option httpclose", and if the client does not
  close either, then the connection remains open till the timeout expires. This
  causes high number of simultaneous connections on the servers and shows high
  global session times in the logs.

  When this happens, it is possible to use "option forceclose". It will
  actively close the outgoing server channel as soon as the server has finished
  to respond and release some resources earlier than with "option httpclose".

  This option may also be combined with "option http-pretend-keepalive", which
  will disable sending of the "Connection: close" header, but will still cause
  the connection to be closed once the whole response is received.

  This option disables and replaces any previous "option httpclose", "option
  http-server-close", "option http-keep-alive", or "option http-tunnel".

  If this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  See also : "option httpclose" and "option http-pretend-keepalive"
```

#### http-pretend-keepalive

```
option http-pretend-keepalive
no option http-pretend-keepalive
  Define whether haproxy will announce keepalive to the server or not
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  Arguments : none

  When running with "option http-server-close" or "option forceclose", haproxy
  adds a "Connection: close" header to the request forwarded to the server.
  Unfortunately, when some servers see this header, they automatically refrain
  from using the chunked encoding for responses of unknown length, while this
  is totally unrelated. The immediate effect is that this prevents haproxy from
  maintaining the client connection alive. A second effect is that a client or
  a cache could receive an incomplete response without being aware of it, and
  consider the response complete.

  By setting "option http-pretend-keepalive", haproxy will make the server
  believe it will keep the connection alive. The server will then not fall back
  to the abnormal undesired above. When haproxy gets the whole response, it
  will close the connection with the server just as it would do with the
  "forceclose" option. That way the client gets a normal response and the
  connection is correctly closed on the server side.

  It is recommended not to enable this option by default, because most servers
  will more efficiently close the connection themselves after the last packet,
  and release its buffers slightly earlier. Also, the added packet on the
  network could slightly reduce the overall peak performance. However it is
  worth noting that when this option is enabled, haproxy will have slightly
  less work to do. So if haproxy is the bottleneck on the whole architecture,
  enabling this option might save a few CPU cycles.

  This option may be set both in a frontend and in a backend. It is enabled if
  at least one of the frontend or backend holding a connection has it enabled.
  This option may be combined with "option httpclose", which will cause
  keepalive to be announced to the server and close to be announced to the
  client. This practice is discouraged though.

  If this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  See also : "option forceclose", "option http-server-close", and
             "option http-keep-alive"
```
It is recommended not to enable this option by default, because most servers will more efficiently
close the connection themselves after the last packet, and release its buffers slightly earlier.
Also, the added packet on the network could slightly reduce the overall peak performance.
However it is worth noting that when this option is enabled, haproxy will have slightly less
work to do. So if haproxy is the bottleneck on the whole architecture, enabling this option might
save a few CPU cycles.


#### option httplog

```
option httplog [ clf ]
  Enable logging of HTTP request, session state and timers
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  Arguments :
    clf       if the "clf" argument is added, then the output format will be
              the CLF format instead of HAProxy's default HTTP format. You can
              use this when you need to feed HAProxy's logs through a specific
              log analyser which only support the CLF format and which is not
              extensible.

  By default, the log output format is very poor, as it only contains the
  source and destination addresses, and the instance name. By specifying
  "option httplog", each log line turns into a much richer format including,
  but not limited to, the HTTP request, the connection timers, the session
  status, the connections numbers, the captured headers and cookies, the
  frontend, backend and server name, and of course the source address and
  ports.

  This option may be set either in the frontend or the backend.

  Specifying only "option httplog" will automatically clear the 'clf' mode
  if it was set by default.

  See also :  section 8 about logging.
```

#### option redispatch

```
option redispatch
no option redispatch
  Enable or disable session redistribution in case of connection failure
  May be used in sections:    defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  Arguments : none

  In HTTP mode, if a server designated by a cookie is down, clients may
  definitely stick to it because they cannot flush the cookie, so they will not
  be able to access the service anymore.

  Specifying "option redispatch" will allow the proxy to break their
  persistence and redistribute them to a working server.

  It also allows to retry last connection to another server in case of multiple
  connection failures. Of course, it requires having "retries" set to a nonzero
  value.

  This form is the preferred form, which replaces both the "redispatch" and
  "redisp" keywords.

  If this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  See also : "redispatch", "retries", "force-persist"
```

#### redis-check

```
option redis-check
  Use redis health checks for server testing
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  Arguments : none

  It is possible to test that the server correctly talks REDIS protocol instead
  of just testing that it accepts the TCP connection. When this option is set,
  a PING redis command is sent to the server, and the response is analyzed to
  find the "+PONG" response message.

  Example :
        option redis-check

  See also : "option httpchk"
```

#### option srvtcpka

```
option srvtcpka
no option srvtcpka
  Enable or disable the sending of TCP keepalive packets on the server side
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  Arguments : none

  When there is a firewall or any session-aware component between a client and
  a server, and when the protocol involves very long sessions with long idle
  periods (eg: remote desktops), there is a risk that one of the intermediate
  components decides to expire a session which has remained idle for too long.

  Enabling socket-level TCP keep-alives makes the system regularly send packets
  to the other end of the connection, leaving it active. The delay between
  keep-alive probes is controlled by the system only and depends both on the
  operating system and its tuning parameters.

  It is important to understand that keep-alive packets are neither emitted nor
  received at the application level. It is only the network stacks which sees
  them. For this reason, even if one side of the proxy already uses keep-alives
  to maintain its connection alive, those keep-alive packets will not be
  forwarded to the other side of the proxy.

  Please note that this has nothing to do with HTTP keep-alive.

  Using option "srvtcpka" enables the emission of TCP keep-alive probes on the
  server side of a connection, which should help when session expirations are
  noticed between HAProxy and a server.

  If this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  See also : "option clitcpka", "option tcpka"
```

#### option tcpka

```
option tcpka
  Enable or disable the sending of TCP keepalive packets on both sides
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  Arguments : none

  When there is a firewall or any session-aware component between a client and
  a server, and when the protocol involves very long sessions with long idle
  periods (eg: remote desktops), there is a risk that one of the intermediate
  components decides to expire a session which has remained idle for too long.

  Enabling socket-level TCP keep-alives makes the system regularly send packets
  to the other end of the connection, leaving it active. The delay between
  keep-alive probes is controlled by the system only and depends both on the
  operating system and its tuning parameters.

  It is important to understand that keep-alive packets are neither emitted nor
  received at the application level. It is only the network stacks which sees
  them. For this reason, even if one side of the proxy already uses keep-alives
  to maintain its connection alive, those keep-alive packets will not be
  forwarded to the other side of the proxy.

  Please note that this has nothing to do with HTTP keep-alive.

  Using option "tcpka" enables the emission of TCP keep-alive probes on both
  the client and server sides of a connection. Note that this is meaningful
  only in "defaults" or "listen" sections. If this option is used in a
  frontend, only the client side will get keep-alives, and if this option is
  used in a backend, only the server side will get keep-alives. For this
  reason, it is strongly recommended to explicitly use "option clitcpka" and
  "option srvtcpka" when the configuration is split between frontends and
  backends.

  See also : "option clitcpka", "option srvtcpka"
```

#### option tcplog

```
option tcplog
  Enable advanced logging of TCP connections with session state and timers
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  Arguments : none

  By default, the log output format is very poor, as it only contains the
  source and destination addresses, and the instance name. By specifying
  "option tcplog", each log line turns into a much richer format including, but
  not limited to, the connection timers, the session status, the connections
  numbers, the frontend, backend and server name, and of course the source
  address and ports. This option is useful for pure TCP proxies in order to
  find which of the client or server disconnects or times out. For normal HTTP
  proxies, it's better to use "option httplog" which is even more complete.

  This option may be set either in the frontend or the backend.

  See also :  "option httplog", and section 8 about logging.
```

#### timeout server

```
timeout server <timeout>
timeout srvtimeout <timeout> (deprecated)
  Set the maximum inactivity time on the server side.
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  Arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  The inactivity timeout applies when the server is expected to acknowledge or
  send data. In HTTP mode, this timeout is particularly important to consider
  during the first phase of the server's response, when it has to send the
  headers, as it directly represents the server's processing time for the
  request. To find out what value to put there, it's often good to start with
  what would be considered as unacceptable response times, then check the logs
  to observe the response time distribution, and adjust the value accordingly.

  The value is specified in milliseconds by default, but can be in any other
  unit if the number is suffixed by the unit, as specified at the top of this
  document. In TCP mode (and to a lesser extent, in HTTP mode), it is highly
  recommended that the client timeout remains equal to the server timeout in
  order to avoid complex situations to debug. Whatever the expected server
  response times, it is a good practice to cover at least one or several TCP
  packet losses by specifying timeouts that are slightly above multiples of 3
  seconds (eg: 4 or 5 seconds minimum). If some long-lived sessions are mixed
  with short-lived sessions (eg: WebSocket and HTTP), it's worth considering
  "timeout tunnel", which overrides "timeout client" and "timeout server" for
  tunnels.

  This parameter is specific to backends, but can be specified once for all in
  "defaults" sections. This is in fact one of the easiest solutions not to
  forget about it. An unspecified timeout results in an infinite timeout, which
  is not recommended. Such a usage is accepted and works but reports a warning
  during startup because it may results in accumulation of expired sessions in
  the system if the system's timeouts are not configured either.

  This parameter replaces the old, deprecated "srvtimeout". It is recommended
  to use it to write new configurations. The form "timeout srvtimeout" is
  provided only by backwards compatibility but its use is strongly discouraged.

  See also : "srvtimeout", "timeout client" and "timeout tunnel".
```

In TCP mode (and to a lesser extent, in HTTP mode), it is highly recommended that the client
timeout remains equal to the server timeout in order to avoid complex situations to debug.

#### timeout http-request

```
timeout http-request <timeout>
  Set the maximum allowed time to wait for a complete HTTP request
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  Arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  In order to offer DoS protection, it may be required to lower the maximum
  accepted time to receive a complete HTTP request without affecting the client
  timeout. This helps protecting against established connections on which
  nothing is sent. The client timeout cannot offer a good protection against
  this abuse because it is an inactivity timeout, which means that if the
  attacker sends one character every now and then, the timeout will not
  trigger. With the HTTP request timeout, no matter what speed the client
  types, the request will be aborted if it does not complete in time. When the
  timeout expires, an HTTP 408 response is sent to the client to inform it
  about the problem, and the connection is closed. The logs will report
  termination codes "cR". Some recent browsers are having problems with this
  standard, well-documented behaviour, so it might be needed to hide the 408
  code using "option http-ignore-probes" or "errorfile 408 /dev/null". See
  more details in the explanations of the "cR" termination code in section 8.5.

  Note that this timeout only applies to the header part of the request, and
  not to any data. As soon as the empty line is received, this timeout is not
  used anymore. It is used again on keep-alive connections to wait for a second
  request if "timeout http-keep-alive" is not set.

  Generally it is enough to set it to a few seconds, as most clients send the
  full request immediately upon connection. Add 3 or more seconds to cover TCP
  retransmits but that's all. Setting it to very low values (eg: 50 ms) will
  generally work on local networks as long as there are no packet losses. This
  will prevent people from sending bare HTTP requests using telnet.

  If this parameter is not set, the client timeout still applies between each
  chunk of the incoming request. It should be set in the frontend to take
  effect, unless the frontend is in TCP mode, in which case the HTTP backend's
  timeout will be used.

  See also : "errorfile", "http-ignore-probes", "timeout http-keep-alive", and
             "timeout client".
```

#### timeout http-keep-alive

```
timeout http-keep-alive <timeout>
  Set the maximum allowed time to wait for a new HTTP request to appear
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  Arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  By default, the time to wait for a new request in case of keep-alive is set
  by "timeout http-request". However this is not always convenient because some
  people want very short keep-alive timeouts in order to release connections
  faster, and others prefer to have larger ones but still have short timeouts
  once the request has started to present itself.

  The "http-keep-alive" timeout covers these needs. It will define how long to
  wait for a new HTTP request to start coming after a response was sent. Once
  the first byte of request has been seen, the "http-request" timeout is used
  to wait for the complete request to come. Note that empty lines prior to a
  new request do not refresh the timeout and are not counted as a new request.

  There is also another difference between the two timeouts : when a connection
  expires during timeout http-keep-alive, no error is returned, the connection
  just closes. If the connection expires in "http-request" while waiting for a
  connection to complete, a HTTP 408 error is returned.

  In general it is optimal to set this value to a few tens to hundreds of
  milliseconds, to allow users to fetch all objects of a page at once but
  without waiting for further clicks. Also, if set to a very small value (eg:
  1 millisecond) it will probably only accept pipelined requests but not the
  non-pipelined ones. It may be a nice trade-off for very large sites running
  with tens to hundreds of thousands of clients.

  If this parameter is not set, the "http-request" timeout applies, and if both
  are not set, "timeout client" still applies at the lower level. It should be
  set in the frontend to take effect, unless the frontend is in TCP mode, in
  which case the HTTP backend's timeout will be used.

  See also : "timeout http-request", "timeout client".
```
#### timeout client

```
timeout client <timeout>
timeout clitimeout <timeout> (deprecated)
  Set the maximum inactivity time on the client side.
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  Arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  The inactivity timeout applies when the client is expected to acknowledge or
  send data. In HTTP mode, this timeout is particularly important to consider
  during the first phase, when the client sends the request, and during the
  response while it is reading data sent by the server. That said, for the
  first phase, it is preferable to set the "timeout http-request" to better
  protect HAProxy from Slowloris like attacks. The value is specified in
  milliseconds by default, but can be in any other unit if the number is
  suffixed by the unit, as specified at the top of this document. In TCP mode
  (and to a lesser extent, in HTTP mode), it is highly recommended that the
  client timeout remains equal to the server timeout in order to avoid complex
  situations to debug. It is a good practice to cover one or several TCP packet
  losses by specifying timeouts that are slightly above multiples of 3 seconds
  (eg: 4 or 5 seconds). If some long-lived sessions are mixed with short-lived
  sessions (eg: WebSocket and HTTP), it's worth considering "timeout tunnel",
  which overrides "timeout client" and "timeout server" for tunnels, as well as
  "timeout client-fin" for half-closed connections.

  This parameter is specific to frontends, but can be specified once for all in
  "defaults" sections. This is in fact one of the easiest solutions not to
  forget about it. An unspecified timeout results in an infinite timeout, which
  is not recommended. Such a usage is accepted and works but reports a warning
  during startup because it may results in accumulation of expired sessions in
  the system if the system's timeouts are not configured either.

  This parameter replaces the old, deprecated "clitimeout". It is recommended
  to use it to write new configurations. The form "timeout clitimeout" is
  provided only by backwards compatibility but its use is strongly discouraged.

  See also : "clitimeout", "timeout server", "timeout tunnel",
             "timeout http-request".
```

#### timeout check

```
timeout check <timeout>
  Set additional check timeout, but only after a connection has been already
  established.

  May be used in sections:    defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  Arguments:
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  If set, haproxy uses min("timeout connect", "inter") as a connect timeout
  for check and "timeout check" as an additional read timeout. The "min" is
  used so that people running with *very* long "timeout connect" (eg. those
  who needed this due to the queue or tarpit) do not slow down their checks.
  (Please also note that there is no valid reason to have such long connect
  timeouts, because "timeout queue" and "timeout tarpit" can always be used to
  avoid that).

  If "timeout check" is not set haproxy uses "inter" for complete check
  timeout (connect + read) exactly like all <1.3.15 version.

  In most cases check request is much simpler and faster to handle than normal
  requests and people may want to kick out laggy servers so this timeout should
  be smaller than "timeout server".

  This parameter is specific to backends, but can be specified once for all in
  "defaults" sections. This is in fact one of the easiest solutions not to
  forget about it.

  See also: "timeout connect", "timeout queue", "timeout server",
            "timeout tarpit".
```

#### timeout connon

```
timeout connect <timeout>
timeout contimeout <timeout> (deprecated)
  Set the maximum time to wait for a connection attempt to a server to succeed.
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  Arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  If the server is located on the same LAN as haproxy, the connection should be
  immediate (less than a few milliseconds). Anyway, it is a good practice to
  cover one or several TCP packet losses by specifying timeouts that are
  slightly above multiples of 3 seconds (eg: 4 or 5 seconds). By default, the
  connect timeout also presets both queue and tarpit timeouts to the same value
  if these have not been specified.

  This parameter is specific to backends, but can be specified once for all in
  "defaults" sections. This is in fact one of the easiest solutions not to
  forget about it. An unspecified timeout results in an infinite timeout, which
  is not recommended. Such a usage is accepted and works but reports a warning
  during startup because it may results in accumulation of failed sessions in
  the system if the system's timeouts are not configured either.

  This parameter replaces the old, deprecated "contimeout". It is recommended
  to use it to write new configurations. The form "timeout contimeout" is
  provided only by backwards compatibility but its use is strongly discouraged.

  See also: "timeout check", "timeout queue", "timeout server", "contimeout",
            "timeout tarpit".
```

#### timeout queue

```
timeout queue <timeout>
  Set the maximum time to wait in the queue for a connection slot to be free
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  Arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  When a server's maxconn is reached, connections are left pending in a queue
  which may be server-specific or global to the backend. In order not to wait
  indefinitely, a timeout is applied to requests pending in the queue. If the
  timeout is reached, it is considered that the request will almost never be
  served, so it is dropped and a 503 error is returned to the client.

  The "timeout queue" statement allows to fix the maximum time for a request to
  be left pending in a queue. If unspecified, the same value as the backend's
  connection timeout ("timeout connect") is used, for backwards compatibility
  with older versions with no "timeout queue" parameter.

  See also : "timeout connect", "contimeout".
```

#### check

```
check
  This option enables health checks on the server. By default, a server is
  always considered available. If "check" is set, the server is available when
  accepting periodic TCP connections, to ensure that it is really able to serve
  requests. The default address and port to send the tests to are those of the
  server, and the default source is the same as the one defined in the
  backend. It is possible to change the address using the "addr" parameter, the
  port using the "port" parameter, the source address using the "source"
  address, and the interval and timers using the "inter", "rise" and "fall"
  parameters. The request method is define in the backend using the "httpchk",
  "smtpchk", "mysql-check", "pgsql-check" and "ssl-hello-chk" options. Please
  refer to those options and parameters for more information.

  Supported in default-server: No
```

#### inter & fastinter & downinter

```
inter <delay>
fastinter <delay>
downinter <delay>
  The "inter" parameter sets the interval between two consecutive health checks
  to <delay> milliseconds. If left unspecified, the delay defaults to 2000 ms.
  It is also possible to use "fastinter" and "downinter" to optimize delays
  between checks depending on the server state :

             Server state            |             Interval used
    ---------------------------------+-----------------------------------------
     UP 100% (non-transitional)      | "inter"
    ---------------------------------+-----------------------------------------
     Transitionally UP (going down), |
     Transitionally DOWN (going up), | "fastinter" if set, "inter" otherwise.
     or yet unchecked.               |
    ---------------------------------+-----------------------------------------
     DOWN 100% (non-transitional)    | "downinter" if set, "inter" otherwise.
    ---------------------------------+-----------------------------------------

  Just as with every other time-based parameter, they can be entered in any
  other explicit unit among { us, ms, s, m, h, d }. The "inter" parameter also
  serves as a timeout for health checks sent to servers if "timeout check" is
  not set. In order to reduce "resonance" effects when multiple servers are
  hosted on the same hardware, the agent and health checks of all servers
  are started with a small time offset between them. It is also possible to
  add some random noise in the agent and health checks interval using the
  global "spread-checks" keyword. This makes sense for instance when a lot
  of backends use the same servers.

  Supported in default-server: Yes
```

#### rise

```
rise <count>
  The "rise" parameter states that a server will be considered as operational
  after <count> consecutive successful health checks. This value defaults to 2
  if unspecified. See also the "check", "inter" and "fall" parameters.

  Supported in default-server: Yes
```

#### fall

```
fall <count>
  The "fall" parameter states that a server will be considered as dead after
  <count> consecutive unsuccessful health checks. This value defaults to 3 if
  unspecified. See also the "check", "inter" and "rise" parameters.

  Supported in default-server: Yes
```