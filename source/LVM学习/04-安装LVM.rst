04 安装lvm
==============================

默认centos6，7都是安装了lvm了的。如果没有安装可以使用如下命令安装。

.. code-block:: bash

    [root@centos7 ~]$ rpm -q lvm2                           # 查看lvm2是否安装了， 我使用的centos7，这是安装的了
    lvm2-2.02.171-8.el7.x86_64
    [root@centos7 ~]$ # yum -y install lvm2                 # 没有安装的话使用yum安装
