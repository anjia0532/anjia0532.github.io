---
title: 062-ubuntu LVM不停机扩/缩容磁盘常见运维操作
urlname: ubuntu-lvm
date: '2021-04-27 20:12:21 +0800'
tags:
  - ubuntu
  - lvm
  - linux
  - 运维
categories:
  - linux
  - 运维
---

> 这是坚持技术写作计划（含翻译）的第 62 篇，定个小目标 999，每周最少 2 篇。

本文主要基于 ubuntu 讲解如何配置 lvm 的物理卷，逻辑卷，逻辑卷组，以及常见的扩容缩容操作。

<!-- more -->

> 物理存储介质（The physical media）

这里指系统的存储设备：硬盘，如：/dev/hda、/dev/sda 等等，是存储系统最低层的存储单元。

>

> 物理卷（physicalvolume）

物理卷就是指硬盘分区或从逻辑上与磁盘分区具有同样功能的设备(如 RAID)，是 LVM 的基本存储逻辑块，但和基本的物理存储介质（如分区、磁盘等）比较，却包含有与 LVM 相关的管理参数。

>

> 卷组（Volume Group）

LVM 卷组类似于非 LVM 系统中的物理硬盘，其由物理卷组成。可以在卷组上创建一个或多个“LVM 分区”（逻辑卷），LVM 卷组由一个或多个物理卷组成。

>

> 逻辑卷（logicalvolume）

LVM 的逻辑卷类似于非 LVM 系统中的硬盘分区，在逻辑卷之上可以建立文件系统(比如/home 或者/usr 等)。

>

> PE（physical extent）

每一个物理卷被划分为称为 PE(Physical Extents)的基本单元，具有唯一编号的 PE 是可以被 LVM 寻址的最小单元。PE 的大小是可配置的，默认为 4MB。

>

> LE（logical extent）

逻辑卷也被划分为被称为 LE(Logical Extents) 的可被寻址的基本单位。在同一个卷组中，LE 的大小和 PE 是相同的，并且一一对应。

引自 [Linx 卷管理详解 VG LV PV](https://blog.csdn.net/wuweilong/article/details/7565530)

## 操作步骤

从 esxi/vmware 添加磁盘，新加磁盘默认用`fdisk -l` 看不到，要么重启，要么 `echo "scsi add-single-device 32 0 2 0">/proc/scsi/scsi` 后再`fdisk -l`

参考 [虚拟机 VMware 新增硬盘无法识别问题](https://www.yuanmas.com/info/Glypko2ka2.html)

```bash
fdisk -l


Disk /dev/sdb: 100 GiB, 107374182400 bytes, 209715200 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

Disk /dev/sdc: 50 GiB, 53687091200 bytes, 104857600 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
# ...省略无用
```

发现有两块待用磁盘，具体操作如下

### 创建物理卷，逻辑卷组，逻辑卷

```bash
# 基于磁盘创建物理卷

> pvcreate /dev/sdc
  Physical volume "/dev/sdc" successfully created


 # 查看已有物理卷
 > pvdisplay

  --- Physical volume ---
  PV Name               /dev/sda5
  VG Name               rancher-252-vg
  PV Size               49.28 GiB / not usable 2.00 MiB
  Allocatable           yes
  PE Size               4.00 MiB
  Total PE              12616
  Free PE               4
  Allocated PE          12612
  PV UUID               9FdgPm-P5Ao-zK2Q-FEiA-aYOi-QBcM-PRVjIn

  "/dev/sdc" is a new physical volume of "50.00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sdc
  VG Name
  PV Size               50.00 GiB
  Allocatable           NO
  PE Size               0
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               FGkHWb-y0d8-iq4D-OeFd-JAh4-Dx2K-vp46W0


# 创建逻辑卷组
> vgcreate vg_data /dev/sdc
  Volume group "vg_data" successfully created

# 查看逻辑卷组
> vgdisplay
  --- Volume group ---
  VG Name               vg_data
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               50.00 GiB
  PE Size               4.00 MiB
  Total PE              12799
  Alloc PE / Size       0 / 0
  Free  PE / Size       12799 / 50.00 GiB
  VG UUID               8VgOL0-H0FU-ATx1-e9cF-NO2D-xffH-iDUqoD

  --- Volume group ---
  VG Name               rancher-252-vg
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               49.28 GiB
  PE Size               4.00 MiB
  Total PE              12616
  Alloc PE / Size       12612 / 49.27 GiB
  Free  PE / Size       4 / 16.00 MiB
  VG UUID               MWvlQB-lTEX-Svxu-wcTA-R6EC-wniE-RKpzGH

# 创建逻辑卷并分配所有空间
> lvcreate -l 100%VG -n lv_data vg_data
 Logical volume "lv_data" created.

# 格式化并挂载磁盘
# 格式化
mkfs.ext4 /dev/mapper/vg_data-lv_data
# 创建文件夹
mkdir -p /data
# 备份
cp /etc/fstab /etc/fstab.bak
# 弄成开机自动挂载
echo `blkid /dev/mapper/vg_data-lv_data | awk '{print $2}' | sed 's/\"//g'` /data ext4 defaults 0 0 >> /etc/fstab
# 现在挂载
mount /dev/mapper/vg_data-lv_data /data/

df -h
Filesystem                         Size  Used Avail Use% Mounted on
udev                               7.9G     0  7.9G   0% /dev
tmpfs                              1.6G  8.8M  1.6G   1% /run
/dev/mapper/rancher--252--vg-root   48G  2.0G   44G   5% /
tmpfs                              7.9G     0  7.9G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
tmpfs                              7.9G     0  7.9G   0% /sys/fs/cgroup
/dev/sda1                          720M   58M  626M   9% /boot
tmpfs                              1.6G     0  1.6G   0% /run/user/1000
/dev/mapper/vg_data-lv_data         50G   52M   47G   1% /data

# 发现已经挂载了一块50G的卷到了/data目录
```

### 卷扩容

向 50G 的 `/dev/mapper/vg_data-lv_data` 逻辑卷上增加一块 100G 盘(/dev/sdb)扩成 150G 空间

```bash
# 卸载
umount /data/

# 创建物理卷
pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created

# 将磁盘加到vg_data 逻辑组中
vgextend vg_data /dev/sdb

# 查询vg_data 显示为150G
vgdisplay vg_data
  --- Volume group ---
  VG Name               vg_data
  System ID
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               149.99 GiB
  PE Size               4.00 MiB
  Total PE              38398
  Alloc PE / Size       12799 / 50.00 GiB
  Free  PE / Size       25599 / 100.00 GiB
  VG UUID               8VgOL0-H0FU-ATx1-e9cF-NO2D-xffH-iDUqoD

# 扩容逻辑卷
lvextend -l +100%FREE /dev/mapper/vg_data-lv_data
  Size of logical volume vg_data/lv_data changed from 50.00 GiB (12799 extents) to 149.99 GiB (38398 extents).
  Logical volume lv_data successfully resized.

# 查看逻辑卷容量
lvdisplay /dev/mapper/vg_data-lv_data
  --- Logical volume ---
  LV Path                /dev/vg_data/lv_data
  LV Name                lv_data
  VG Name                vg_data
  LV UUID                02Xd3v-05hl-ZH2v-l3Qp-BkXE-ndbB-wbhdRu
  LV Write Access        read/write
  LV Creation host, time rancher-252, 2021-04-27 15:15:49 +0800
  LV Status              available
  # open                 1
  LV Size                149.99 GiB
  Current LE             38398
  Segments               2
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:2

# 重新计算逻辑卷大小
resize2fs  /dev/mapper/vg_data-lv_data
resize2fs 1.42.13 (17-May-2015)
Filesystem at /dev/mapper/vg_data-lv_data is mounted on /data; on-line resizing required
old_desc_blocks = 4, new_desc_blocks = 10
The filesystem on /dev/mapper/vg_data-lv_data is now 39319552 (4k) blocks long.

df -h
Filesystem                         Size  Used Avail Use% Mounted on
udev                               7.9G     0  7.9G   0% /dev
tmpfs                              1.6G  8.8M  1.6G   1% /run
/dev/mapper/rancher--252--vg-root   48G  2.0G   44G   5% /
tmpfs                              7.9G     0  7.9G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
tmpfs                              7.9G     0  7.9G   0% /sys/fs/cgroup
/dev/sda1                          720M   58M  626M   9% /boot
tmpfs                              1.6G     0  1.6G   0% /run/user/1000
/dev/mapper/vg_data-lv_data        148G   60M  141G   1% /data

# 检查文件完整性
e2fsck -f /dev/mapper/vg_data-lv_data
e2fsck 1.42.13 (17-May-2015)
第一步: 检查inode,块,和大小
第二步: 检查目录结构
第3步: 检查目录连接性
Pass 4: Checking reference counts
第5步: 检查簇概要信息
/dev/mapper/vg_data-lv_data: 11/9830400 files (0.0% non-contiguous), 664949/39319552 blocks

# 挂载
mount  /dev/mapper/vg_data-lv_data /data/
```

### 卷缩容

一般不会缩容，除非是，实在没剩余空间了，而另外一个文件夹更需要磁盘，将 A 缩容省出的空间给 B。

```bash
# 卸载
umount /data/

# 检查文件完整性
e2fsck -f /dev/mapper/vg_data-lv_data
e2fsck 1.42.13 (17-May-2015)
第一步: 检查inode,块,和大小
第二步: 检查目录结构
第3步: 检查目录连接性
Pass 4: Checking reference counts
第5步: 检查簇概要信息
/dev/mapper/vg_data-lv_data: 11/9830400 files (0.0% non-contiguous), 664949/39319552 blocks

# 缩容到50G
resize2fs  /dev/mapper/vg_data-lv_data 49G
resize2fs 1.42.13 (17-May-2015)
Resizing the filesystem on /dev/mapper/vg_data-lv_data to 12845056 (4k) blocks.
The filesystem on /dev/mapper/vg_data-lv_data is now 12845056 (4k) blocks long.

# 逻辑卷缩容到49G
lvreduce -L 50G /dev/mapper/vg_data-lv_data
  WARNING: Reducing active logical volume to 50.00 GiB
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce lv_data? [y/n]: y
  Size of logical volume vg_data/lv_data changed from 149.99 GiB (38398 extents) to 49.00 GiB (12544 extents).
  Logical volume lv_data successfully resized.

# 查看逻辑卷信息
lvdisplay /dev/mapper/vg_data-lv_data
  --- Logical volume ---
  LV Path                /dev/vg_data/lv_data
  LV Name                lv_data
  VG Name                vg_data
  LV UUID                02Xd3v-05hl-ZH2v-l3Qp-BkXE-ndbB-wbhdRu
  LV Write Access        read/write
  LV Creation host, time rancher-252, 2021-04-27 15:15:49 +0800
  LV Status              available
  # open                 0
  LV Size                50.00 GiB
  Current LE             12544
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:2

# 将/dev/sdb硬盘从逻辑卷组中移除
vgreduce vg_data /dev/sdb
  Removed "/dev/sdb" from volume group "vg_data"

# 查看信息
vgdisplay vg_data
  --- Volume group ---
  VG Name               vg_data
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  6
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               50.00 GiB
  PE Size               4.00 MiB
  Total PE              12799
  Alloc PE / Size       12544 / 49.00 GiB
  Free  PE / Size       255 / 1020.00 MiB
  VG UUID               8VgOL0-H0FU-ATx1-e9cF-NO2D-xffH-iDUqoD


# 挂载
mount  /dev/mapper/vg_data-lv_data /data/

df -h

Filesystem                         Size  Used Avail Use% Mounted on
udev                               7.9G     0  7.9G   0% /dev
tmpfs                              1.6G  8.8M  1.6G   1% /run
/dev/mapper/rancher--252--vg-root   48G  2.0G   44G   5% /
tmpfs                              7.9G     0  7.9G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
tmpfs                              7.9G     0  7.9G   0% /sys/fs/cgroup
/dev/sda1                          720M   58M  626M   9% /boot
tmpfs                              1.6G     0  1.6G   0% /run/user/1000
/dev/mapper/vg_data-lv_data         50G   52M   47G   1% /data
```

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/04/28/ubuntu-lvm/)
- [我的掘金](https://juejin.cn/post/6956034318401536031/)
- [Linx 卷管理详解 VG LV PV](https://blog.csdn.net/wuweilong/article/details/7565530)
- [虚拟机 VMware 新增硬盘无法识别问题](https://www.yuanmas.com/info/Glypko2ka2.html)
