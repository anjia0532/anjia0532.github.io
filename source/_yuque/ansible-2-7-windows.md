
---

title: Ansible2.7批量管理Windows

date: 2019-02-14 00:58:00 +0800

tags: [ansible,运维,devops,windows,ansible-playbook]

categories: 运维

---

<a name="59683fbc"></a>
## 简述

<a name="8e1b944f"></a>
### 背景
Ansible是一款轻量级的开源的自动化运维工具，支持linux和windows(只支持client，并且部分模块)，利用Ansible可以简单批量的配置系统，安装软件，或者更高级的运维任务（比如滚动升级）。

Ansible之类的运维工具对运维工作进行抽象及规范，能够极大的降低运维难度。本文只是为了演示如何通过ansible 的各模块对windows进行传输文件,管理账号,执行脚本等批量自动化管理工作

<!-- more -->

<a name="12267079"></a>
### 实验环境
| 类型 | 系统 | ip |
| --- | --- | --- |
| Server | Ubuntu Server 16.04.5 LTS X64 | 192.168.0.22 |
| Client | Windows Server 2008 R2 SP1 | 192.168.0.23 |


**注意：**

如果是实验目的，建议用Vmware，并且在关键操作时备份快照（比如，刚装完环境，升级完PS和.Net后），这样能够及时，干净的还原现场，节省每次重装系统导致的时间浪费

<a name="d29617ad"></a>
## Windows 被控端(Ansible Client)

Ansible 只支持Powershell 4.0及以上(用3.0会报 [Process is terminated due to StackOverflowException.](https://github.com/ansible/ansible/issues/10825))，所以要求最低要求Win7,或者Win server 2008,详见 [Ansible Doc host requirements](https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html#host-requirements)

<a name="399fa47e"></a>
### [升级PowerShell 和.Net Framework](https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html#id2)

官方文档要求升级至ps3.0即可，但是实验发现，3.0会报错

`Win+R`  -> `PowerShell`  打开PowerShell

输入 `$PSVersionTable` 查看 `PSVersion` 确保大于等于4.0(PowerShell 4.0),以及 `CLRVersion` 大于等于4.0(.NET Framework 4.0) ,如果版本过低，则执行下面代码，直接复制到PowerShell 执行即可,建议使用5.1(参考 [hotfixv4.trafficmanager.net dont work](https://github.com/jborean93/ansible-windows/issues/14) 和 [安装和配置 WMF 5.1](https://docs.microsoft.com/zh-cn/powershell/wmf/5.1/install-configure))

**注意：**

- 注意用户名密码改成管理员的用户名密码
- 确保能够访问外网
- PowerShell以管理员模式打开
- PS3不能直升PS5，需要卸载PS3或者保存 PSModulePath
- 安装PS5需要先打最新SP补丁
- PS5要求 .NET Framework 不低于 4.5.2
- 安装成功后会自动重启服务器，注意别影响其他服务

```bash
$url = "https://raw.githubusercontent.com/jborean93/ansible-windows/master/scripts/Upgrade-PowerShell.ps1"
$file = "$env:temp\Upgrade-PowerShell.ps1"
$username = "管理员用户名"
$password = "管理员密码"

(New-Object -TypeName System.Net.WebClient).DownloadFile($url, $file)
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Force

# PowerShell 版本，只能用 3.0, 4.0 和 5.1 
&$file -Version 5.1 -Username $username -Password $password -Verbose
```

重启后，再次打开PS ,输入 `$PSVersionTable` 查看版本

<a name="fadbd8e5"></a>
### 安装WinRM，并且配置监听

```
## 配置WinRM

$url = "https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1"
$file = "$env:temp\ConfigureRemotingForAnsible.ps1"
(New-Object -TypeName System.Net.WebClient).DownloadFile($url, $file)
powershell.exe -ExecutionPolicy ByPass -File $file
```

<a name="1994c2f8"></a>
## Linux主控端(Ansible Server)

<a name="401f96f4"></a>
### 安装Ansible和pywinrm

参考 [Installing the Control Machine](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-the-control-machine)

```bash
$ sudo apt-get update
$ sudo apt-get install -y software-properties-common
$ sudo apt-add-repository --yes --update ppa:ansible/ansible
$ sudo apt-get install -y ansible
$ ansible --version
ansible 2.7.7
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/dist-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.12 (default, Nov 12 2018, 14:36:49) [GCC 5.4.0 20160609]

$ pip install "pywinrm>=0.3.0"
```

<a name="a9796771"></a>
### 配置Inventory

参考 [Working with Inventory](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html)

默认是 Inventory `/etc/ansible/hosts` ,此处改为手动指定，Ansible的Inventory支持ini格式和yaml格式，本文采用yaml格式

```bash
$ mkdir ansible
$ tee ansible/test_hosts.yaml <<-'EOF'
winserver:
  hosts: 
    192.168.0.23:
  vars:
    ansible_user: Administrator
    ansible_password: 密码
    ansible_connection: winrm
    ansible_port: 5986
    ansible_winrm_server_cert_validation: ignore

EOF
```

<a name="ea275a70"></a>
### 探测主机是否存活

```bash
$ ansible -i ansible/test_hosts.yaml winserver -m win_ping
 192.168.0.23 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
```

如果报错

```bash
TASK [Gathering Facts] *******************************************************************************************************************************************************************************************************************************************************
fatal: [192.168.0.23]: UNREACHABLE! => {"changed": false, "msg": "ssl: HTTPSConnectionPool(host='192.168.0.23', port=5986): Max retries exceeded with url: /wsman (Caused by SSLError(SSLError(\"bad handshake: Error([('SSL routines', 'tls_process_server_certificate', 'certificate verify failed')],)\",),))", "unreachable": true}
```

降级一下 `pip install "pywinrm==0.2.2"`

更多实验，参见参考资料中的两篇51cto中的博文

<a name="7d486fc7"></a>
## 可用的Windows模块

根据 [What modules are available?](https://docs.ansible.com/ansible/latest/user_guide/windows_faq.html#what-modules-are-available) 绝大部分Module都是针对Linux编写的，大部分在windows下不能正常使用，有一些专用的windows module 使用ps编写的，可用在windows下使用，详细列表参见 [Windows modules](https://docs.ansible.com/ansible/latest/modules/list_of_windows_modules.html#windows-modules)

<a name="fae94bcd"></a>
## [Windows常见问题](https://docs.ansible.com/ansible/latest/user_guide/windows_faq.html#windows-frequently-asked-questions)

官方文档中列举了Ansible windows常见问题，建议仔细阅读

<a name="53001b41"></a>
## Ansible Playbook

Ansible配合[playbook](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html#about-playbooks)食用更佳，上述中的 `ansible` 命令是[adhoc命令模式](https://docs.ansible.com/ansible/latest/user_guide/intro_adhoc.html)，类似在bash中手动执行命令，多用于简单，并且不需要复用场景，而 `ansible-playbook` 类似 bash脚本，但是更强大，适合复杂任务编排场景。

考虑后期出一个playbook的文章

推荐配合[vscode](https://code.visualstudio.com/)的[ansible](https://marketplace.visualstudio.com/items?itemName=vscoss.vscode-ansible) 插件(github repo 地址 [https://github.com/VSChina/vscode-ansible](https://github.com/VSChina/vscode-ansible))

效果如下所示，比idea的ansible强大很多。

![](https://cdn.nlark.com/yuque/0/2019/gif/226273/1556186451366-4149328b-fa7d-439d-9761-0194293d9352.gif#align=left&display=inline&height=546&originHeight=724&originWidth=990&size=0&status=done&width=746)

<a name="35808e79"></a>
## 参考资料

- [官方文档](https://docs.ansible.com/)
- [官方文档-windows指南](https://docs.ansible.com/ansible/latest/user_guide/windows.html)
- [官方文档-windows指南-winrm步骤](https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html#winrm-setup)
- [Ansible批量远程管理Windows主机(部署与配置)](http://blog.51cto.com/7424593/2174156)
- [ansible自动化管理windows系统实战](http://blog.51cto.com/dyc2005/2064746)
- [安装和配置 WMF 5.1](https://docs.microsoft.com/zh-cn/powershell/wmf/5.1/install-configure)
- [hotfixv4.trafficmanager.net dont work](https://github.com/jborean93/ansible-windows/issues/14)

<a name="fb674066"></a>
## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。

长期招聘，Java程序员，大数据工程师，运维工程师。

