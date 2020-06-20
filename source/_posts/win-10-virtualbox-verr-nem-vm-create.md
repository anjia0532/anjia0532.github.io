
---

title: 042-解决win10 VirtualBox无法启动(VERR_NEM_VM_CREATE_FAILED)

date: 2019-08-26 19:35:21 +0800

tags: [虚拟机,kvm,vagrant,virtualbox,python]

categories: python

---

> 这是坚持技术写作计划（含翻译）的第42篇，定个小目标999，每周最少2篇。


最近将win10从1809升级到1903，结果自己的VirtualBox无法启动，经过一番google，问题已解决。

python+vagrant+virtualbox系列文章<br /> 

- [036-win10搭建python的linux开发环境(pycharm+vagrant+virtualbox)](https://juejin.im/post/5d3a55ece51d454f71439dd2) 
- [037-vagrant启动(up)后自动同步文件(rsync-auto)](https://juejin.im/post/5d562b5e5188252d43756db8) 
- [040-解决Linux使用virtualbox共享文件夹问题](https://juejin.im/post/5d5695056fb9a06afd6600f0)
- [042-解决win10 VirtualBox无法启动(VERR_NEM_VM_CREATE_FAILED)](https://juejin.im/post/5d63869a51882559c41612c6)
- [043-解决vagrant访问virtualbox共享文件夹报无权限问题(Permission denied)](https://juejin.im/post/5d6493d6e51d456206115a2c)

<!-- more -->

我的运行错误日志忘记保存了，从网上找的类似的

```
00:00:03.418689 VMSetError: F:\tinderbox\win-6.0\src\VBox\VMM\VMMR3\NEMR3Native-win.cpp(1463) int __cdecl nemR3NativeInitAfterCPUM(struct VM *); rc=VERR_NEM_VM_CREATE_FAILED
00:00:03.418732 VMSetError: Call to WHvSetupPartition failed: ERROR_SUCCESS (Last=0xc000000d/87)
00:00:03.418771 NEM: Destroying partition 00000000013f29b0 with its 0 VCpus...
00:00:03.548429 ERROR [COM]: aRC=E_FAIL (0x80004005) aIID={872da645-4a9b-1727-bee2-5585105b9eed} aComponent={ConsoleWrap} aText={Call to WHvSetupPartition failed: ERROR_SUCCESS (Last=0xc000000d/87) (VERR_NEM_VM_CREATE_FAILED)}, preserve=false aResultDetail=-6805
00:00:03.548750 Console: Machine state changed to 'PoweredOff'
00:00:03.558813 Power up failed (vrc=VERR_NEM_VM_CREATE_FAILED, rc=E_FAIL (0X80004005))
00:00:04.060139 GUI: UIMachineViewNormal::resendSizeHint: Restoring guest size-hint for screen 0 to 800x600
00:00:04.060177 ERROR [COM]: aRC=E_ACCESSDENIED (0x80070005) aIID={ab4164db-c13e-4dab-842d-61ee3f0c1e87} aComponent={DisplayWrap} aText={The console is not powered up}, preserve=false aResultDetail=0
00:00:04.060407 GUI: Aborting startup due to power up progress issue detected...
```
> 代码节选自 [https://forums.virtualbox.org/viewtopic.php?f=6&t=92260](https://forums.virtualbox.org/viewtopic.php?f=6&t=92260)


解决办法是禁用Hyper-V。<br /><kbd>Win</kbd>+<kbd>R</kbd> -> cmd -> <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>Enter</kbd> -> `bcdedit /set hypervisorlaunchtype off` -> 重启电脑 -> 启动vbox

<a name="fb674066"></a>
## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。<br />长期招聘，Java程序员，大数据工程师，运维工程师，前端工程师。

<a name="35808e79"></a>
## 参考资料

- [我的博客](https://anjia0532.github.io/2019/08/26/win-10-virtualbox-verr-nem-vm-create)
- [我的掘金](https://juejin.im/post/5d63869a51882559c41612c6)
- [vbox: WHvSetupPartition failed: ERROR_SUCCESS (VERR_NEM_VM_CREATE_FAILED) (Hyper-V conflict) #4587](https://github.com/kubernetes/minikube/issues/4587)
- [VERR_NEM_VM_CREATE_FAILED after Activating Hyper-V features in Windows 10](https://www.virtualbox.org/ticket/18687)

