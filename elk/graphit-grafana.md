---
date: 2017-04-06T11:44:09+08:00
title: graphit grafana
---

## Install grafana

```bash
wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana-4.1.2-1486989747.x86_64.rpm
yum localinstall grafana-4.1.2-1486989747.x86_64.rpm
```

## Install graphit

```bash
# install some development header
yum install python-devel cairo-devel libffi-devel

#!/bin/bash

/opt/graphite/bin/carbon-cache.py start
# in this step, mayby you need to run `pip install service_identity

# getting metrics into garphite

echo "foo.bar 1 `date +%s`" | nc localhost 2003

# metrics with statsd
# metrics with collectd

yum install collectd

# configure collectd config

# start grafana
$ systemctl daemon-reload
$ systemctl start grafana-server
$ systemctl status grafana-server
systemctl enable grafana-server.service
`
/opt/graphite/bin/carbon-cache.py start
# in this step, mayby you need to run `pip install service_identity

# getting metrics into garphite

echo "foo.bar 1 `date +%s`" | nc localhost 2003

# metrics with statsd
# metrics with collectd

yum install collectd

# configure collectd config

# start grafana
$ systemctl daemon-reload
$ systemctl start grafana-server
$ systemctl status grafana-server
systemctl enable grafana-server.service
```

## Send data

```bash
#!/bin/bash

CARBON_HOST="localhost"
CARBON_PORT="2003"
METRIC_NAME="debug.jdixon.procs.bad_java_app"
TIMESTAMP=`date +%s`

NUMBER_OF_WAYWARD_PROCS=`ps auwx | grep httpd | wc -l`

echo ${METRIC_NAME} ${NUMBER_OF_WAYWARD_PROCS} ${TIMESTAMP} | nc ${CARBON_HOST} ${CARBON_PORT}
```