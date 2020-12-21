---
title: python py27 debug
date: 2019-08-21
---

Debug test


https://wiki.openstack.org/wiki/Testr

 # source .tox/py27/bin/activate
 # testr list-tests <magnum.tests.unit.api.controllers.v1.test_cluster.TestPost.test_create_cluster_without_lb_without_fip> > my-list
 # python -m testtools.run discover --load-list my-list
 # deactivate

And wherever you set your pdb.set_trace() will break into the debugger


https://wiki.openstack.org/wiki/Testr
source .tox/py27/bin/activate
testr list-tests test_case_名称 > my-list
在对应的test case代码中打入断点
python -m testtools.run discover --load-list my-list
