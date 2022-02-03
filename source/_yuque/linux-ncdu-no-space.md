---
title: Linux磁盘空间占满问题快速排雷
urlname: linux-ncdu-no-space
date: '2019-02-14 14:36:00 +0800'
tags:
  - linux
  - 运维
  - devops
  - 排雷
  - 磁盘
  - ncdu
categories: 运维
---

情人节一大早就接到报警，一台测试服务器磁盘满了，这很程序员。

<!-- more -->

## 磁盘排雷三连

反手一个 `df`  先看是否是真满了(参考 [df(1) - Linux man page](https://linux.die.net/man/1/df) )
需要注意，如果磁盘空间未满，但是仍然报 `No space left on device` ,需要执行  `df -i`  排查 inode

```bash
$ df -h
文件系统        容量  已用  可用 已用% 挂载点
udev             47G     0   47G    0% /dev
tmpfs           9.4G  620M  8.8G    7% /run
/dev/sda2       1.1T  1.1T    0G    100% /
tmpfs            47G  8.7M   47G    1% /dev/shm
tmpfs           5.0M  4.0K  5.0M    1% /run/lock
tmpfs            47G     0   47G    0% /sys/fs/cgroup
/dev/sda1       511M  1.2M  510M    1% /boot/efi
tmpfs           9.4G  108K  9.4G    1% /run/user/1000
tmpfs           9.4G   40K  9.4G    1% /run/user/0

$ df -i
文件系统          Inode 已用(I)  可用(I) 已用(I)% 挂载点
udev           12277823     525 12277298       1% /dev
tmpfs          12285255    1368 12283887       1% /run
/dev/sda2      71041024 2250210 68790814       4% /
tmpfs          12285255      83 12285172       1% /dev/shm
tmpfs          12285255       5 12285250       1% /run/lock
tmpfs          12285255      17 12285238       1% /sys/fs/cgroup
/dev/sda1             0       0        0        - /boot/efi
tmpfs          12285255      44 12285211       1% /run/user/1000
tmpfs          12285255      15 12285240       1% /run/user/0

```

通过 `df -h`  只能看出磁盘满了，但是看不出每个文件夹的大小，所以需要使用 `du -ahd1` ,如果文件不是很多，很大，一般速度还能接受，但是今天执行相当慢，所以 `Ctrl+C`  中止。
此处简单说明一下 `-ahd1`  的意思(可以通过 `man du`  或者 `du --help`  自行查阅帮助文档,参考 [du(1) - Linux man page](https://linux.die.net/man/1/du))

```
$ du --help
用法：du [选项]... [文件]...
　或：du [选项]... --files0-from=F

  -a, --all             write counts for all files, not just directories (所有文件，不止目录)

  -h, --human-readable  print sizes in human readable format (e.g., 1K 234M 2G) (易读单位，会损失精度)

  -d, --max-depth=N     print the total for a directory (or file, with --all) (只扫描一层目录)
                          only if it is N or fewer levels below the command
                          line argument;  --max-depth=0 is the same as
                          --summarize

```

```bash
$ du -ahd1 /
1.6M	/dev
4.0K	/mnt
4.0K	/lib64
455M	/boot
0	/sys
0	/vmlinuz.old
689G	/data
370M	/tmp
15M	/etc
8.0K	/media
du: 无法访问'/proc/24390/task/24390/fd/4': 没有那个文件或目录
du: 无法访问'/proc/24390/task/24390/fdinfo/4': 没有那个文件或目录
du: 无法访问'/proc/24390/fd/3': 没有那个文件或目录
du: 无法访问'/proc/24390/fdinfo/3': 没有那个文件或目录
^C
# 慢的要死，Ctrl+C 终止
```

如果被删除的文件 `df -h`  快满了，而 `du -ahd1`  却很小，往往是文件被删除，而文件句柄没释放导致的,祭出 `lsof | grep deleted ` ，解决办法，要么 kill 掉 pid，释放句柄(治本)，要么就 ` > /path/to/deleted/file`  把内容覆盖掉(治标)。当然还有别的玩法，比如，不小心 `rm -rf /`  了，先别着急跑路，万一 `lsof | grep deleted`  还存在的，都还有救，约等于 windows 下的回收站的作用。 参考[ lsof(8) - Linux man page](https://linux.die.net/man/8/lsof)

```bash
$ lsof -i | grep deleted
# 当然也不快
```

## [ncdu](https://dev.yorhel.nl/ncdu)

针对 `du -d1`  大文件场景下的龟速表现，有人开发了 ncdu,以 ubuntu 为例

```
# 从APT安装(版本较旧目前是v1.11)
$ sudo apt-get update
$ sudo apt-get install -y ncdu

# 从源码编译安装(此处是1.14版本)
$ sudo apt-get install -y libncurses5-dev # 如果不安装会报 configure: error: required header file not found
$ wget https://dev.yorhel.nl/download/ncdu-1.14.tar.gz
$ tar zxf ncdu-1.14.tar.gz
$ cd ncdu-1.14
$ ./configure --prefix=/usr
$ make && make install
```

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1550123205304-ae5bfcbc-8c91-4f91-ab15-843953ed6701.png#align=left&display=inline&height=397&name=image.png&originHeight=397&originWidth=779&size=37521&width=779)

参考官方文档 [Ncdu Manual](https://dev.yorhel.nl/ncdu/man)

## 额外

通过 man 查询命令时，手册中会带有数字(例如 `du(1)` , `lsof(8)` )，这代表的是手册的不同部分，可以通过 `man man`  或者  [Linux man pages](https://linux.die.net/man/)  来查看

```
MANUAL SECTIONS
    The standard sections of the manual include:

    1      User Commands
    2      System Calls
    3      C Library Functions
    4      Devices and Special Files
    5      File Formats and Conventions
    6      Games et. al.
    7      Miscellanea
    8      System Administration tools and Daemons

    Distributions customize the manual section to their specifics,
    which often include additional sections.
```

## 参考资料

- [What do the numbers in a man page mean?](https://unix.stackexchange.com/a/3587)
- [df(1) - Linux man page](https://linux.die.net/man/1/df)
- [du(1) - Linux man page](https://linux.die.net/man/1/du)
- [lsof(8) - Linux man page](https://linux.die.net/man/8/lsof)
- [Ncdu Manual](https://dev.yorhel.nl/ncdu/man)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。

长期招聘，Java 程序员，大数据工程师，运维工程师。
