---
title: 006-Cobbler批量自动化部署CentOS/Ubuntu/Windows
urlname: cobbler
date: '2019-02-22 15:46:00 +0800'
tags:
  - pxe
  - dhcp
  - cobbler
  - centos
  - ubuntu
  - windows
  - tftp
categories: 运维
---

> 这是坚持技术写作计划（含翻译）的第 6 篇，定个小目标 999，每周最少 2 篇。

本文主要讲解通过 CentOS7.6 Minimal + Cobbler 自动化安装 CentOS,Ubuntu,Windows

## 准备

从阿里镜像站，下载  [CentOS-7-x86_64-Minimal-1810.iso](https://mirrors.aliyun.com/centos/7.6.1810/isos/x86_64/CentOS-7-x86_64-Minimal-1810.iso)  和  [ubuntu-16.04.5-desktop-amd64.iso](https://mirrors.aliyun.com/ubuntu-releases/releases/releases/16.04.5/ubuntu-16.04.5-desktop-amd64.iso) ，使用 VMware 创建一台 CentOS7 的虚拟机。

### 环境初始化

1.改为阿里源

```bash
# yum源
# 备份系统默认源
[root@localhost ~]# mkdir /etc/yum.repos.d/old && mv /etc/yum.repos.d/C* /etc/yum.repos.d/old/
[root@localhost ~]# yum clean all
# 设置阿里yum源
[root@localhost ~]# curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
[root@localhost ~]# curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo

# pip源(2.8.4的bug)
[root@localhost ~]# mkdir ~/.pip
[root@localhost ~]# cat > ~/.pip/pip.conf << EOF
[global]
trusted-host=mirrors.aliyun.com
index-url=https://mirrors.aliyun.com/pypi/simple/
EOF
```

2.配置 ssh

默认 ssh_config 启用了 DNS 解析，导致每次远程 ssh 时都特别慢

```bash
[root@localhost ~]# sed -i 's%#UseDNS yes%UseDNS no%' /etc/ssh/sshd_config
[root@localhost ~]# service sshd restart
```

~~3.~~[~~配置 SElinux~~](https://access.redhat.com/documentation/en-us/red_hat_network_satellite/5.3/html/reference_guide/ch-cobbler#s3-cobbler-reqs-security-selinux)
~~如果要在 Centos 上开启 Cobbler 的支持，需要用 root 用户运行 `setsebool` ，注意 `-P`  参数确保重启仍然生效。~~
~~同时需要配置 SELinux 上下文规则(CentOS 7 Minimal 默认不安装 `semanage` )，用于提供引导镜像。~~
按照红帽的 5.4 的教程，设置不生效，所以，禁用 SELinux

```bash
#//.. setsebool -P httpd_can_network_connect true
#//.. yum provides semanage
#//.. yum -y install policycoreutils-python.x86_64
#//.. semanage fcontext -a -t public_content_t "/var/lib/tftpboot/.*"
[root@localhost ~]# sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
[root@localhost ~]# setenforce 0
```

4.[配置防火墙](https://access.redhat.com/documentation/en-us/red_hat_network_satellite/5.3/html/reference_guide/ch-cobbler#s3-cobbler-reqs-security-iptables)

```bash
# TFTP
[root@localhost ~]# firewall-cmd --zone=public --add-port=69/tcp --permanent
[root@localhost ~]# firewall-cmd --zone=public --add-port=69/udp --permanent

# HTTPD
[root@localhost ~]# firewall-cmd --zone=public --add-port=80/tcp --permanent
[root@localhost ~]# firewall-cmd --zone=public --add-port=443/tcp --permanent

# Cobbler
[root@localhost ~]# firewall-cmd --zone=public --add-port=25150/tcp --permanent
[root@localhost ~]# firewall-cmd --zone=public --add-port=25150/udp --permanent

# Koan 如果未用到，可以不开放
[root@localhost ~]# firewall-cmd --zone=public --add-port=25151/tcp --permanent

# samba window安装需要用到samba,否则可以不开放
[root@localhost ~]# firewall-cmd --zone=public --add-port=139/tcp --permanent
[root@localhost ~]# firewall-cmd --zone=public --add-port=445/tcp --permanent
[root@localhost ~]# firewall-cmd --zone=public --add-port=137/udp --permanent
[root@localhost ~]# firewall-cmd --zone=public --add-port=138/udp --permanent

# 更新防火墙规则
[root@localhost ~]# firewall-cmd --reload

# 查看所有打开的端口
[root@localhost ~]# firewall-cmd --zone=public --list-ports
```

## 安装 Cobbler

### 安装依赖软件

```bash
[root@localhost ~]# yum install -y cobbler cobbler-web dhcp tftp-server pykickstart httpd xinetd
```

### 设置开机自启动

```bash
[root@localhost ~]# systemctl enable httpd
[root@localhost ~]# systemctl enable xinetd
[root@localhost ~]# systemctl enable rsyncd
[root@localhost ~]# systemctl enable tftp
[root@localhost ~]# systemctl enable cobblerd
```

### 启动服务

```bash
[root@localhost ~]# systemctl start httpd
[root@localhost ~]# systemctl start xinetd
[root@localhost ~]# systemctl start tftp
[root@localhost ~]# systemctl start cobblerd
```

### 检查配置

```bash
[root@localhost ~]# cobbler check
The following are potential configuration items that you may want to fix:

1 : The 'server' field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolvable hostname or IP for the boot server as reachable by all machines that will use it.
2 : For PXE to be functional, the 'next_server' field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boot server on the PXE network.
3 : SELinux is enabled. Please review the following wiki page for details on ensuring cobbler works correctly in your SELinux environment:
    https://github.com/cobbler/cobbler/wiki/Selinux
4 : change 'disable' to 'no' in /etc/xinetd.d/tftp
5 : Some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.
6 : debmirror package is not installed, it will be required to manage debian deployments and repositories
7 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to 'cobbler' and should be changed, try: "openssl passwd -1 -salt 'random-phrase-here' 'your-password-here'" to generate new one
8 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them

Restart cobblerd and then run 'cobbler sync' to apply changes.
```

1,2：
`cobbler_ip`  为 cobbler 主机 ip
`next_server`  是 dhcp 主机 ip，但是本实验 dhcp 和 cobbler 是一台

```bash
[root@localhost ~]# export cobbler_ip=192.168.0.12
[root@localhost ~]# sed -i "s%^server: 127.0.0.1%server: ${cobbler_ip}%g" /etc/cobbler/settings
[root@localhost ~]# sed -i "s%^next_server: 127.0.0.1%next_server: ${cobbler_ip}%g" /etc/cobbler/settings
```

3： 可以忽略，因为已经配置 SELinux 和防火墙了

4：开启 tftp 功能

```bash
[root@localhost ~]# sed -i '/disable\>/s/\<yes\>/no/' /etc/xinetd.d/tftp
```

5：下载 bootload

```bash
[root@localhost ~]# cobbler get-loaders
```

6：下载 ubuntu 本地包镜像（不装 ubuntu 的，可以不用改）

```bash
[root@localhost ~]# yum install -y debmirror
[root@localhost ~]# sed -i 's%^@dists="sid"%#@dists="sid"%g;s%@arches="i386"%#@arches="i386"%g' /etc/debmirror.conf
```

7：设置安装系统后的 root 密码

```bash
[root@localhost ~]# export root_pwd=$(openssl passwd -1 -salt `openssl rand 15 -base64` 'Abcd1234!@#$')
[root@localhost ~]# sed -i "s%^default_password_crypted.*%default_password_crypted: \"${root_pwd}\"%g" /etc/cobbler/settings
```

8：电源管理模块(非必选)，cman 和 ence-agents 二选一即可,此处忽略

其余修改：

```bash
[root@localhost ~]# sed -i "s%manage_dhcp: 0%manage_dhcp: 1%g" /etc/cobbler/settings
[root@localhost ~]# sed -i "s%pxe_just_once: 0%pxe_just_once: 1%g" /etc/cobbler/settings
```

修改 dhcp

```bash
[root@localhost ~]# vi /etc/cobbler/dhcp.template
\#仅列出修改过的部分
\......
subnet 192.168.0.0 netmask 255.255.255.0 {
     option routers             192.168.0.1;
     option domain-name-servers 192.168.0.1;
     option subnet-mask         255.255.255.0;
     range dynamic-bootp        192.168.0.100 192.168.0.200;
\......
```

重启服务

```bash
[root@localhost ~]# systemctl restart httpd
[root@localhost ~]# systemctl restart xinetd
[root@localhost ~]# systemctl restart tftp
[root@localhost ~]# systemctl restart cobblerd
```

再次校验,发现只有两条信息了，忽略即可。

```bash
[root@localhost ~]# cobbler check
The following are potential configuration items that you may want to fix:

1 : SELinux is enabled. Please review the following wiki page for details on ensuring cobbler works correctly in your SELinux environment:
    https://github.com/cobbler/cobbler/wiki/Selinux
2 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them

Restart cobblerd and then run 'cobbler sync' to apply changes.
```

执行 `cobbler sync`  同步信息

```bash
[root@localhost ~]# cobbler sync
task started: 2019-02-22_184309_sync
task started (id=Sync, time=Fri Feb 22 18:43:09 2019)
running pre-sync triggers
# ....忽略
running python trigger cobbler.modules.scm_track
running shell triggers from /var/lib/cobbler/triggers/change/*
*** TASK COMPLETE ***
```

### cobbler-web

#### 修复 2.8.4 bug

打开 `https://${cobbler_ip}/cobbler_web`  注意是 `https`  但是 2.8.4 有个 bug，会导致打开后报 500 `Internal Server Error`  错误 ,是因为   `django`  版本太高了 (参加 [cobbler 2.8.4/2.8.0 on centos 7 error ](https://github.com/cobbler/cobbler/issues/1959))

```bash
[root@localhost ~]# yum install -y python-pip
[root@localhost ~]# pip2.7 install -U django==1.9.13
[root@localhost ~]# systemctl restart cobblerd
```

#### 设置用户名密码

默认用户名密码是 cobbler/cobbler

```bash
[root@localhost ~]# htdigest -c /etc/cobbler/users.digest Cobbler cobbler # 后边这个是用户名
[root@localhost ~]# systemctl restart cobblerd
```

再次打开 `https://${cobbler_ip}/cobbler_web` 
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1550854050621-8cd8b769-08c5-457e-9e1f-e85d64c10479.png#align=left&display=inline&height=519&name=image.png&originHeight=519&originWidth=543&size=42554&status=done&width=543)

### 挂载镜像

通过 winscp,mobaxterm 等将 ubuntu 和 centos 镜像上传到 Cobbler 服务器上的 `/tmp/`  目录下,其中 `net.ifnames=0 biosdevname=0 noipv6`  是让网卡统一命名成 `eth0`

```bash
[root@localhost ~]# mount -t iso9660 -o loop /tmp/CentOS-7-x86_64-Minimal-1810.iso /mnt/
[root@localhost ~]# cobbler import --name=CentOS-7.6.1810-x86_64 --path=/mnt/ --arch=x86_64
[root@localhost ~]# cobbler profile edit --name=CentOS-7.6.1810-x86_64 --kopts='net.ifnames=0 biosdevname=0'
[root@localhost ~]# mount -t iso9660 -o loop /tmp/ubuntu-16.04.5-server-amd64.iso /mnt/
[root@localhost ~]# cobbler import --name=ubuntu-16.04.5-server-x86_64 --path=/mnt/ --arch=x86_64
```

### 测试 PXE 安装系统

在 vmware 创建两个虚拟机(选择空白盘),内存 2G，CPU2 核，磁盘 20G，创建完后，记得打个快照，后边做实验失败后，直接恢复即可。
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1550853771456-0e9b5af8-6bfe-44e0-83ed-22cf06deb895.png#align=left&display=inline&height=400&name=image.png&originHeight=400&originWidth=720&size=11329&status=done&width=720)
选择 CentOS，然后回车，系统将自动安装

## 优化配置

### CentOS ks

```bash
[root@localhost ~]# cobbler profile report --name CentOS-7.6.1810-x86_64
Name                           : CentOS-7.6.1810-x86_64
#//...忽略
Kickstart                      : /var/lib/cobbler/kickstarts/sample_end.ks
#//...忽略
#//复制一份ks，并且进行修改
[root@localhost ~]# cp /var/lib/cobbler/kickstarts/sample_end.ks /var/lib/cobbler/kickstarts/centos-7-6.ks
[root@localhost ~]# cobbler profile edit --name CentOS-7.6.1810-x86_64  --kickstart=/var/lib/cobbler/kickstarts/centos-7-6.ks
[root@localhost ~]# cp /var/lib/cobbler/kickstarts/sample.seed /var/lib/cobbler/kickstarts/ubuntu-16-4-5.seed
[root@localhost ~]# cobbler profile edit --name ubuntu-16.04.5-server-x86_64  --kickstart=/var/lib/cobbler/kickstarts/ubuntu-16-4-5.seed
```

具体 CentOS 的 ks 语法可以参考这里：[KICKSTART 语法参考](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/installation_guide/sect-kickstart-syntax)。另外可以参考  [运维工作笔记-Cobbler 配置文件](https://www.kancloud.cn/devops-centos/centos-linux-devops/392369)
具体 Ubuntu 的 Preseed 可以参考这里：[Preseed 语法参考](https://www.debian.org/releases/stable/amd64/apbs04.html.zh-cn)。

吐槽一下，cobbler 代码中检测到是 ubuntu 时，会自动将 ks 换成 url(强制走 preseed)，而用惯了 ks 的，用 preseed 还是很不习惯的，可以看一下  [Support selection of automatic installation file format in distros which allow it (Debian/Ubuntu allows kickstart and preseed) ](https://github.com/cobbler/cobbler/issues/1262)和  [Problems Provisioning Ubuntu with Cobbler and Kickstart Profiles](https://thornelabs.blog/posts/problems-provisioning-ubuntu-with-cobbler-and-kickstart-profiles.html)
此处贴一下  /var/lib/cobbler/kickstarts/ubuntu-16-4-5.seed

```bash
# Mostly based on the Ubuntu installation guide
# https://help.ubuntu.com/16.04/installation-guide/
# Debian sample
# https://www.debian.org/releases/stable/example-preseed.txt

## Part 1. Localization

# Preseeding only locale sets language, country and locale.
d-i debian-installer/locale string en_US

# Keyboard selection.
# Disable automatic (interactive) keymap detection.
d-i console-setup/ask_detect boolean false
d-i keyboard-configuration/toggle select No toggling
d-i keyboard-configuration/layoutcode string us
d-i keyboard-configuration/variantcode string

## Part 2. Network configuration
# netcfg will choose an interface that has link if possible. This makes it
# skip displaying a list if there is more than one interface.
#set $myhostname = $getVar('hostname',$getVar('name','cobbler')).replace("_","-")
d-i netcfg/choose_interface select auto
d-i netcfg/get_hostname string $myhostname

# If non-free firmware is needed for the network or other hardware, you can
# configure the installer to always try to load it, without prompting. Or
# change to false to disable asking.
# d-i hw-detect/load_firmware boolean true

## Part 3 NTP/Time
# NTP/Time Setup
d-i time/zone string Asia/Shanghai
d-i clock-setup/utc boolean true
d-i clock-setup/ntp boolean true
d-i clock-setup/ntp-server  string ntp1.aliyun.com

## Part 4. Mirror settings
# Setup the installation source
d-i mirror/country string manual
d-i mirror/http/hostname string $http_server
d-i mirror/http/directory string $install_source_directory
d-i mirror/http/proxy string

#set $os_v = $getVar('os_version','')
#if $breed == "ubuntu" and $os_v and ($os_v.lower()[0] > 'p' or $os_v.lower()[0] < 'd')
# Required at least for ubuntu 12.10+ , so test os_v is higher than precise and lower than drapper
d-i live-installer/net-image string http://$http_server/cobbler/links/$distro_name/install/filesystem.squashfs
#end if

# Suite to install.
# d-i mirror/suite string precise
# d-i mirror/udeb/suite string precise

# Components to use for loading installer components (optional).
#d-i mirror/udeb/components multiselect main, restricted

## Part 4. Partitioning

# Disk Partitioning
# Use LVM, and wipe out anything that already exists
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true
d-i partman-auto/method string lvm
d-i partman-lvm/device_remove_lvm boolean true
d-i partman-lvm/confirm boolean true
d-i partman-lvm/confirm_nooverwrite boolean true
d-i partman-md/device_remove_md boolean true
d-i partman-partitioning/confirm_write_new_label boolean true

# You can choose one of the three predefined partitioning recipes:
# - atomic: all files in one partition
# - home:   separate /home partition
# - multi:  separate /home, /usr, /var, and /tmp partitions
d-i partman-auto/choose_recipe select atomic

# If you just want to change the default filesystem from ext3 to something
# else, you can do that without providing a full recipe.
# d-i partman/default_filesystem string ext4

## Part 5. Account setup

# root account and password
d-i passwd/root-login boolean true
d-i passwd/root-password-crypted password $default_password_crypted

# skip creation of a normal user account.
d-i passwd/make-user boolean true
d-i passwd/user-fullname  string anjia
d-i passwd/username       string anjia
d-i passwd/user-password-crypted password $default_password_crypted

## Part 6. Apt setup

# You can choose to install restricted and universe software, or to install
# software from the backports repository.
# d-i apt-setup/restricted boolean true
# d-i apt-setup/universe boolean true
# d-i apt-setup/backports boolean true

# Uncomment this if you don't want to use a network mirror.
# d-i apt-setup/use_mirror boolean true

# Select which update services to use; define the mirrors to be used.
# Values shown below are the normal defaults.
# d-i apt-setup/services-select multiselect security
# d-i apt-setup/security_host string mirrors.aliyun.com
# d-i apt-setup/security_path string /ubuntu

$SNIPPET('preseed_apt_repo_config')

# Enable deb-src lines
# d-i apt-setup/local0/source boolean true

# URL to the public key of the local repository; you must provide a key or
# apt will complain about the unauthenticated repository and so the
# sources.list line will be left commented out
# d-i apt-setup/local0/key string http://local.server/key

# By default the installer requires that repositories be authenticated
# using a known gpg key. This setting can be used to disable that
# authentication. Warning: Insecure, not recommended.
# d-i debian-installer/allow_unauthenticated boolean true

## Part 7. Package selection
# Default for minimal
tasksel tasksel/first multiselect standard
# Default for server
# tasksel tasksel/first multiselect standard, web-server
# Default for gnome-desktop
# tasksel tasksel/first multiselect standard, gnome-desktop

# Individual additional packages to install
# wget is REQUIRED otherwise quite a few things won't work
# later in the build (like late-command scripts)
d-i pkgsel/include string ntp ssh wget

# Debian needs this for the installer to avoid any question for grub
# Please verify that it suit your needs as it may overwrite any usb stick
#if $breed == "debian"
d-i grub-installer/grub2_instead_of_grub_legacy boolean true
d-i grub-installer/bootdev string default
#end if

# Use the following option to add additional boot parameters for the
# installed system (if supported by the bootloader installer).
# Note: options passed to the installer will be added automatically.
d-i debian-installer/add-kernel-opts string $kernel_options_post

# Avoid that last message about the install being complete.
d-i finish-install/reboot_in_progress note

## Figure out if we're kickstarting a system or a profile
#if $getVar('system_name','') != ''
#set $what = "system"
#else
#set $what = "profile"
#end if

# This first command is run as early as possible, just after preseeding is read.
# d-i preseed/early_command string [command]
d-i preseed/early_command string wget -O- \
   http://$http_server/cblr/svc/op/script/$what/$name/?script=preseed_early_default | \
   /bin/sh -s

# This command is run immediately before the partitioner starts. It may be
# useful to apply dynamic partitioner preseeding that depends on the state
# of the disks (which may not be visible when preseed/early_command runs).
# d-i partman/early_command \
#       string debconf-set partman-auto/disk "\$(list-devices disk | head -n1)"

# This command is run just before the install finishes, but when there is
# still a usable /target directory. You can chroot to /target and use it
# directly, or use the apt-install and in-target commands to easily install
# packages and run commands in the target system.
# d-i preseed/late_command string [command]
d-i preseed/late_command string wget -O- \
   http://$http_server/cblr/svc/op/script/$what/$name/?script=preseed_late_default | \
   chroot /target /bin/sh -s
```

### 创建 snippets

安装完后，自动安装软件,参考  [Using template scripts for Debian and Ubuntu seeds](https://github.com/cobbler/cobbler/wiki/Using%20template%20scripts%20for%20Debian%20and%20Ubuntu%20seeds)

```bash
[root@localhost ~]# tee /var/lib/cobbler/snippets/ubuntu_apt_install_soft <<-'EOF'

apt-get update
apt-get install -y language-pack-zh-hans apt-transport-https ca-certificates software-properties-common  git ansible openssh-server vim curl htop iotop iftop ncdu

curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
apt-get update

apt-get install -y docker-ce=18.06.2~ce~3-0~ubuntu
apt-mark hold docker-ce
systemctl enable docker
EOF

## // d-i preseed/late_command 阶段执行
[root@localhost ~]# echo '$SNIPPET("ubuntu_apt_install_soft") >> /var/lib/cobbler/snippets/late_apt_repo_config
```



### 设置 ubuntu package repo

```bash
[root@localhost ~]# cobbler repo edit --name=ubuntu-16.04.5-server-x86_64 --arch=x86_64 --breed=apt --mirror=http://mirrors.aliyun.com/ubuntu --owners=admin --mirror-locally=False --apt-components='main universe' --apt-dists='xenial xenial-updates xenial-security'
[root@localhost ~]# cobbler profile edit --name=ubuntu-16.04.5-server-x86_64 --repos=ubuntu-16.04.5-server-x86_64
```

限于篇幅，下一篇，将介绍 Cobbler 安装 Windows

## 常见错误

```bash
[root@localhost ~]# cobbler check
cobblerd does not appear to be running/accessible: error(111, 'Connection refused')
```

原因：未启动相关服务

```bash
[root@localhost ~]# cobbler check
httpd does not appear to be running and proxying cobbler, or SELinux is in the way. Original traceback:
Traceback (most recent call last):
  File "/usr/lib/python2.7/site-packages/cobbler/cli.py", line 251, in check_setup
    s.ping()
  File "/usr/lib64/python2.7/xmlrpclib.py", line 1233, in __call__
    return self.__send(self.__name, args)
  File "/usr/lib64/python2.7/xmlrpclib.py", line 1591, in __request
    verbose=self.__verbose
  File "/usr/lib64/python2.7/xmlrpclib.py", line 1273, in request
    return self.single_request(host, handler, request_body, verbose)
  File "/usr/lib64/python2.7/xmlrpclib.py", line 1321, in single_request
    response.msg,
ProtocolError: <ProtocolError for 127.0.0.1:80/cobbler_api: 503 Service Unavailable>
```

未关闭防火墙

```bash
[root@localhost ~]# cobbler sync
received on stderr: Redirecting to /bin/systemctl restart dhcpd.service
Job for dhcpd.service failed because the control process exited with error code. See "systemctl status dhcpd.service" and "journalctl -xe" for details.

Exception occured: <class 'cobbler.cexceptions.CX'>
Exception value: 'cobbler trigger failed: cobbler.modules.sync_post_restart_services'
Exception Info:
  File "/usr/lib/python2.7/site-packages/cobbler/remote.py", line 82, in run
    rc = self._run(self)
   File "/usr/lib/python2.7/site-packages/cobbler/remote.py", line 181, in runner
    return self.remote.api.sync(self.options.get("verbose",False),logger=self.logger)
   File "/usr/lib/python2.7/site-packages/cobbler/api.py", line 763, in sync
    return sync.run()
   File "/usr/lib/python2.7/site-packages/cobbler/action_sync.py", line 144, in run
    utils.run_triggers(self.api, None, "/var/lib/cobbler/triggers/sync/post/*", logger=self.logger)
   File "/usr/lib/python2.7/site-packages/cobbler/utils.py", line 928, in run_triggers
    raise CX("cobbler trigger failed: %s" % m.__name__)

!!! TASK FAILED !!!
```

没改 dhcp 模板，导致 sync 同步出问题

```bash
[root@localhost ~]# cat /var/log/httpd/ssl_error_log
[Fri Feb 22 20:07:49.460442 2019] [:error] [pid 6910] [remote 127.0.0.1:204] mod_wsgi (pid=6910): Exception occurred processing WSGI script '/usr/share/cobbler/web/cobbler.wsgi'.
[Fri Feb 22 20:07:49.460559 2019] [:error] [pid 6910] [remote 127.0.0.1:204] Traceback (most recent call last):
[Fri Feb 22 20:07:49.460605 2019] [:error] [pid 6910] [remote 127.0.0.1:204]   File "/usr/share/cobbler/web/cobbler.wsgi", line 26, in application
[Fri Feb 22 20:07:49.460668 2019] [:error] [pid 6910] [remote 127.0.0.1:204]     _application = get_wsgi_application()
[Fri Feb 22 20:07:49.460684 2019] [:error] [pid 6910] [remote 127.0.0.1:204]   File "/usr/lib/python2.7/site-packages/django/core/wsgi.py", line 13, in get_wsgi_application
[Fri Feb 22 20:07:49.460723 2019] [:error] [pid 6910] [remote 127.0.0.1:204]     django.setup(set_prefix=False)
[Fri Feb 22 20:07:49.460737 2019] [:error] [pid 6910] [remote 127.0.0.1:204]   File "/usr/lib/python2.7/site-packages/django/__init__.py", line 22, in setup
[Fri Feb 22 20:07:49.460768 2019] [:error] [pid 6910] [remote 127.0.0.1:204]     configure_logging(settings.LOGGING_CONFIG, settings.LOGGING)
[Fri Feb 22 20:07:49.460781 2019] [:error] [pid 6910] [remote 127.0.0.1:204]   File "/usr/lib/python2.7/site-packages/django/conf/__init__.py", line 56, in __getattr__
[Fri Feb 22 20:07:49.460812 2019] [:error] [pid 6910] [remote 127.0.0.1:204]     self._setup(name)
[Fri Feb 22 20:07:49.460824 2019] [:error] [pid 6910] [remote 127.0.0.1:204]   File "/usr/lib/python2.7/site-packages/django/conf/__init__.py", line 41, in _setup
[Fri Feb 22 20:07:49.460852 2019] [:error] [pid 6910] [remote 127.0.0.1:204]     self._wrapped = Settings(settings_module)
[Fri Feb 22 20:07:49.460871 2019] [:error] [pid 6910] [remote 127.0.0.1:204]   File "/usr/lib/python2.7/site-packages/django/conf/__init__.py", line 110, in __init__
[Fri Feb 22 20:07:49.460900 2019] [:error] [pid 6910] [remote 127.0.0.1:204]     mod = importlib.import_module(self.SETTINGS_MODULE)
[Fri Feb 22 20:07:49.460911 2019] [:error] [pid 6910] [remote 127.0.0.1:204]   File "/usr/lib64/python2.7/importlib/__init__.py", line 37, in import_module
[Fri Feb 22 20:07:49.460953 2019] [:error] [pid 6910] [remote 127.0.0.1:204]     __import__(name)
[Fri Feb 22 20:07:49.460973 2019] [:error] [pid 6910] [remote 127.0.0.1:204]   File "/usr/share/cobbler/web/settings.py", line 89, in <module>
[Fri Feb 22 20:07:49.460995 2019] [:error] [pid 6910] [remote 127.0.0.1:204]     from django.conf.global_settings import TEMPLATE_CONTEXT_PROCESSORS
[Fri Feb 22 20:07:49.461043 2019] [:error] [pid 6910] [remote 127.0.0.1:204] ImportError: cannot import name TEMPLATE_CONTEXT_PROCESSORS
```

`pip2.7 install -U django==1.9.13`

## 参考资料

- [配置防火墙](https://access.redhat.com/documentation/en-us/red_hat_network_satellite/5.3/html/reference_guide/ch-cobbler#s3-cobbler-reqs-security-iptables)
- [cobbler 2.8.4/2.8.0 on centos 7 error](https://github.com/cobbler/cobbler/issues/1959)
- [KICKSTART 语法参考](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/installation_guide/sect-kickstart-syntax)
- [运维工作笔记-Cobbler 配置文件](https://www.kancloud.cn/devops-centos/centos-linux-devops/392369)
- [Preseed 语法参考](https://www.debian.org/releases/stable/amd64/apbs04.html.zh-cn)
- [Support selection of automatic installation file format in distros which allow it (Debian/Ubuntu allows kickstart and preseed) ](https://github.com/cobbler/cobbler/issues/1262)
- [Problems Provisioning Ubuntu with Cobbler and Kickstart Profiles](https://thornelabs.blog/posts/problems-provisioning-ubuntu-with-cobbler-and-kickstart-profiles.html)
- [Using template scripts for Debian and Ubuntu seeds](https://github.com/cobbler/cobbler/wiki/Using%20template%20scripts%20for%20Debian%20and%20Ubuntu%20seeds)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。

长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。
