05 LVM操作
=========================================

5.1 初始化磁盘或者磁盘分区
--------------------------------------------------------

运行pvcreate 在整个磁盘上

.. code-block:: bash

    # pvcreate /dev/hdb

运行pvcreate 在分区上

.. code-block:: bash

    # pvcreate /dev/hdb1

.. attention:: 不建议在整个硬盘上创建lvm，没有分区信息容易被其他系统识别为干净的磁盘。

5.2 创建卷组
-------------------------------------------------------------------------

创建卷组需要在指定的物理卷上面的。

.. code-block:: bash

    # vgcreate my_volume_group /dev/hda1 /dev/hdb1 

创建卷组是可以指定卷大小的，默认是32M。

5.3 激活一个卷组
---------------------------------------------------------------

如果我们重启系统或者关闭了卷组就需要激活它。

.. code-block:: bash

    # vgchange -a y my_volume_group

5.4 移除一个卷组
----------------------------------------------------------------

移除卷组需要先停掉卷组的。

.. code-block:: bash

    # vgchange -a n my_volume_group
    # vgremove my_volume_group

5.5 添加物理卷到卷组中
------------------------------------------------------------------------

.. code-block:: bash

    # vgextend my_volume_group /dev/hdc1

5.6 从一个卷组中移除一个物理卷
-----------------------------------------------------------------------------

移除物理卷前必须确保空间是足够的，使用pvdisplay查看

.. code-block:: bash

    # pvdisplay /dev/hda1

    --- Physical volume ---
    PV Name               /dev/hda1
    VG Name               myvg
    PV Size               1.95 GB / NOT usable 4 MB [LVM: 122 KB]
    PV#                   1
    PV Status             available
    Allocatable           yes (but full)
    Cur LV                1
    PE Size (KByte)       4096
    Total PE              499
    Free PE               0
    Allocated PE          499
    PV UUID               Sd44tK-9IRw-SrMC-MOkn-76iP-iftz-OVSen7

如果没啥问题了，可以执行vgreduce去移出物理卷。

.. code-block:: bash

    # vgreduce my_volume_group /dev/hda1

5.7 创建一个逻辑卷
--------------------------------------------------------

创建一个1500M的名字为testlv的lv

.. code-block:: bash

    # lvcreate -L 1500M -n testlv testvg

5.8 移除一个逻辑卷
--------------------------------------------

逻辑卷的移除需要先取消挂载的。

.. code-block:: bash

    # umount /dev/myvg/homevol
    # lvremove /dev/myvg/homevol

5.9 扩展一个逻辑卷
--------------------------------------------

扩展主要有2种方式，一种是直接指定扩展到多大，一种是在原有基础上扩展多少。

方案1： 

.. code-block:: bash

    # lvextend -L12G /dev/myvg/homevol

方案2：

.. code-block:: bash

    # lvextend -L+1G /dev/myvg/homevol

默认情况下，我们使用lvextend是扩展了逻辑卷的大小，但是文件系统的大小还是原来的值。 需要同步下，不同文件系统命令有所不同。

.. note:: lvextend有个-r的选项，如果指定可以完成自动同步。

*ext2/ext3*

.. code-block:: bash

    # umount /dev/myvg/homevol/dev/myvg/homevol
    # resize2fs /dev/myvg/homevol
    # mount /dev/myvg/homevol /home

*reiserfs* 

.. code-block:: bash

    # resize_reiserfs -f /dev/myvg/homevol

*xfs*

    # xfs_growfs /home

*jfs*

.. code-block:: bash

    # mount -o remount,resize /home

.. attention::  扩展xfs和jfs文件系统的时候，不能指定挂载设备，必须指定挂载点。


5.10 缩减一个逻辑卷
-----------------------------------------------------------------

缩减逻辑卷是有风险的，如果处理不当容易丢掉数据的。

*ext2/ext3*

.. code-block:: bash

    # umount /home
    # resize2fs /dev/myvg/homevol 524288
    # lvreduce -L-1G /dev/myvg/homevol
    # mount /home

*reiserfs* 

.. code-block:: bash

    # umount /home
    # resize_reiserfs -s-1G /dev/myvg/homevol
    # lvreduce -L-1G /dev/myvg/homevol
    # mount -treiserfs /dev/myvg/homevol /home

*xfs*

没办法缩减

*jfs*

没办法缩减

5.11 迁移一个老化的的磁盘。
-----------------------------------------------------------

如果有一个磁盘已经老化了，但是有大量数据在上面。那就需要迁移了，迁移前确保卷组的空间够使用。

.. code-block:: bash

    # pvmove /dev/hdb
    # vgreduce dev /dev/hdb

.. note:: pvmore是迁移数据，很慢的，如果你想获取迁移的详细信息，使用-v选项即可。

5.12 迁移一个老化的的磁盘到新的替代磁盘上。
-----------------------------------------------------------

如果有一个磁盘已经老化了，但是有大量数据在上面。新买了一个同样大小的盘用作替换的。

.. code-block:: bash

    # pvcreate /dev/sdf
    # vgextend dev /dev/sdf
    # pvmove /dev/hdb /dev/sdf
    # vgreduce dev /dev/hdb

5.13 使用快照备份
------------------------------------------------------

.. code-block:: bash

    # lvcreate -L592M -s -n dbbackup /dev/ops/databases 
    # mkdir /mnt/ops/dbbackup
    # mount /dev/ops/dbbackup /mnt/ops/dbbackup
    # tar -cf /dev/rmt0 /mnt/ops/dbbackup
    # umount /mnt/ops/dbbackup
    # lvremove /dev/ops/dbbackup 

.. attention:: xfs文件系统在挂载的时候需要指定--nouuid ro 选项才可以。

5.14 迁移卷组到另外一个系统上
----------------------------------------------------------

.. code-block:: bash

    # unmount /mnt/design/users                     # 卸载设备
    # vgchange -an design                           # 关闭卷组
    # vgexport design                               # 导出逻辑卷
    # 去除磁盘插入新的系统上面去
    # pvscan                                        # 扫描pv                   
    # vgimport design                               # 导入卷组
    # vgchange -ay design                           # 激活卷组
    # mkdir -p /mnt/design/users                    # 创建挂载点目录
    # mount /dev/design/users /mnt/design/users     # 挂载

.. warning:: 迁移到其他系统之前请确保名字和目标的不同，使用rename去重命名，且使用sync同步几次，确保数据完全写入到迁移的盘上面。

