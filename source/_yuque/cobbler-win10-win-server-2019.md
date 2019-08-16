
---

title: 007-Cobbler批量自动化部署Windows10和Server 2019

date: 2019-02-22 15:46:00 +0800

tags: [pxe,dhcp,cobbler,centos,ubuntu,windows,tftp,win10,server-2019]

categories: 运维

---

> 这是坚持技术写作计划（含翻译）的第7篇，定个小目标999，每周最少2篇。


本文主要讲解通过CentOS7.6 Minimal + Cobbler 自动化安装CentOS,Ubuntu,Windows 10 和 Windows Server 2019。

请注意，一般安装windows是用[MDT](https://docs.microsoft.com/zh-cn/windows/deployment/deploy-windows-mdt/deploy-a-windows-10-image-using-mdt)或者WDS居多，毕竟是巨硬自己家的，而且WDT还支持分布式镜像传输（主要是巨硬家的OS，动辄超过4G，万兆网卡也会卡啊）。本文不涉及到WDT或者WDS相关操作，感兴趣的可自行百度或者msdn。

<a name="424a2ad8"></a>
## 准备

- [Windows ADK](https://docs.microsoft.com/zh-cn/windows-hardware/get-started/adk-install#winADK) (分别下载 [Download the Windows ADK for Windows 10, version 1809](https://go.microsoft.com/fwlink/?linkid=2026036) 和 [Download the Windows PE add-on for the ADK](https://go.microsoft.com/fwlink/?linkid=2022233))
- 下载 [Windows 10 (business edition), version 1809 (Updated Feb 2019) (x64) - DVD (Chinese-Simplified)](http://msdn.itellyou.cn/)[ ](http://msdn.itellyou.cn/)
- 下载 [Windows Server 2019 (x64) - DVD (Chinese-Simplified)](http://msdn.itellyou.cn/)

注意，adk的两个都要下载，这俩都是引导包，真正的安装程序会由这俩软件进行下载。其中WinPE需要用到5G左右的磁盘空间，简直不能忍受。。。<br />msdn i tell u 堪称良心站，是windows装机神站啊，不过，没有直达页面挺不爽。为了防止下错，特意截图。<br />![Snipaste_2019-02-25_22-18-07.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1551104465714-66f42f29-fbcf-476f-b2e7-b1f3a6fb319b.png#align=left&display=inline&height=365&name=Snipaste_2019-02-25_22-18-07.png&originHeight=573&originWidth=1171&size=143092&status=done&width=746)

<a name="9639c173"></a>
## 安装ADK和WinPE
我已经装过，且忘记截图了，这是事后补图，只需要勾选必须的就行<br />![Snipaste_2019-02-25_22-32-48.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1551105181245-e9b47d7d-e1b3-48f8-918e-42cdff955406.png#align=left&display=inline&height=548&name=Snipaste_2019-02-25_22-32-48.png&originHeight=548&originWidth=746&size=36472&status=done&width=746)![Snipaste_2019-02-25_22-32-20.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1551105181277-4477f2fc-1cfe-414c-9178-c30ae74e725c.png#align=left&display=inline&height=548&name=Snipaste_2019-02-25_22-32-20.png&originHeight=548&originWidth=746&size=52141&status=done&width=746)

安装完后，以管理员身份打开部署和映像工具环境

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1551105055491-d302cb36-bb44-4359-8fb5-c7342e9854fa.png#align=left&display=inline&height=425&name=image.png&originHeight=425&originWidth=613&size=193121&status=done&width=613)

定制Win 10 PE 

```bash
copype amd64 C:\winpe

Dism /mount-image /imagefile:C:\winpe\media\sources\boot.wim /index:1 /mountdir:C:\winpe\mount

echo net use z: \\192.168.0.253\share >> C:\winpe\mount\Windows\System32\startnet.cmd
echo z:\win\setup.exe /unattend:z:\win\win10_x64_bios_auto.xml >> C:\winpe\mount\Windows\System32\startnet.cmd

Dism /unmount-image /mountdir:C:\winpe\mount /commit
MakeWinPEMedia /ISO C:\winpe C:\winpe\winpe_win10_amd64.iso
```

1. 本地生成winpe文件目录
1. dism 挂载 winpe的启动文件到winpe的mount目录
1. 将启动命令硬编码写死到winpe的startnet.cmd文件里
1. 无人值守安装
1. 卸载winpe的挂载（一定要执行，否则直接强制删除文件夹会出一些稀奇古怪的问题）
1. 制作win10镜像，名为 winpe_win10_amd64.iso

第三步的硬编码是无奈之举，因为要想挂载共享文件夹，必须要知道smb主机，但是这个参数又很难传递进来。<br />如果是U盘启动，可以写死U盘路径，大不了插上U盘后，手动改卷标(当然因为U盘挂载顺序不一致，可以通过for循环A-Z盘，挨个盘访问某个文件名，如果存在，即认为此盘是自己U盘，设置环境变量)。而网上说的，startnet.cmd调用另外一个bat，多是基于这个原理。

而如果PXE要达到跟上述要求，动态设置smb主机，要么写死域名，然后劫持或者配置域名，加上bat文件，在winpe启动时，通过startnet.cmd下载，并执行。要么找办法，看看能不能在启动时，传入参数（目前我还没找到），当然还可以用MDT方案，看着比PXE+无人应答文件简单很多。

<a name="d36faa4b"></a>
## 配置Cobbler Server

<a name="42b542e5"></a>
### 导入Cobbler
使用WinScp 等工具，将 winpe_win10_amd64.iso 上传到 Cobbler 服务器上

```bash
[root@localhost ~]# cobbler distro add --name=windows_10_x64 --kernel=/var/lib/tftpboot/memdisk --initrd=/root/winpe_win10_amd64.iso --kopts="raw iso"
[root@localhost ~]# touch /var/lib/cobbler/kickstarts/winpe.xml
[root@localhost ~]# cobbler profile add --name=windows_10_x64 --distro=windows_10_x64 --kickstart=/var/lib/cobbler/kickstarts/winpe.xml
```


<a name="b15fbfd6"></a>
### 创建自动应答文件
直接从 [Windows Answer File Generator#win10_x86_64](http://www.windowsafg.com/win10x86_x64.html) 通过简单配置后，下载即可（只支持简单操作，比如，装系统，格式化磁盘，设置密码等）。当然也可以使用 【Windows系统映像管理器】，不过挺难用的，具体用法可以参考 [How to create an unattended installation of Windows 10](https://www.windowscentral.com/how-create-unattended-media-do-automated-installation-windows-10)。也可以通过MDT简化操作。

但是如果使用直接生成的，有点问题，即使页面设置了安装语言，但是仍旧需要手动选择，经过多方研究，发现<br />主要卡在UILanguage和Inputlocale上，全写zh-CN无效。
```xml
<?xml version="1.0" encoding="utf-8"?>
<component 此处忽略>
  <SetupUILanguage>
    <UILanguage>en-US</UILanguage>
  </SetupUILanguage>
  <InputLocale>0804:{81D4E9C9-1D3B-41BC-9E6C-4B40BF79E35E}{FA550B04-5AD7-411f-A5AC-CA038EC515D7}</InputLocale> 
  <SystemLocale>zh-CN</SystemLocale>
  <UILanguage>zh-CN</UILanguage>
  <UILanguageFallback>zh-CN</UILanguageFallback>
  <UserLocale>zh-CN</UserLocale>
</component>

```

另外就是安装密钥,统一替换为 VK7JG-NPHTM-C97JM-9MPGT-3V66T

下面是我的应答文件，仅做参考。

```bash
<!--*************************************************
Windows 10 Answer File Generator
Created using Windows AFG found at:
;http://www.windowsafg.com

Installation Notes
Location: zh-CN
Notes: Enter your comments here...
**************************************************-->
<?xml version="1.0" encoding="utf-8"?>
<unattend
    xmlns="urn:schemas-microsoft-com:unattend">
    <settings pass="windowsPE">
        <component name="Microsoft-Windows-International-Core-WinPE" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <SetupUILanguage>
                <UILanguage>en-US</UILanguage>
            </SetupUILanguage>
            <InputLocale>0804:{81D4E9C9-1D3B-41BC-9E6C-4B40BF79E35E}{FA550B04-5AD7-411f-A5AC-CA038EC515D7}</InputLocale>
            <SystemLocale>zh-CN</SystemLocale>
            <UILanguage>zh-CN</UILanguage>
            <UILanguageFallback>zh-CN</UILanguageFallback>
            <UserLocale>zh-CN</UserLocale>
        </component>
        <component name="Microsoft-Windows-International-Core-WinPE" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <SetupUILanguage>
                <UILanguage>en-US</UILanguage>
            </SetupUILanguage>
            <InputLocale>0804:{81D4E9C9-1D3B-41BC-9E6C-4B40BF79E35E}{FA550B04-5AD7-411f-A5AC-CA038EC515D7}</InputLocale>
            <SystemLocale>zh-CN</SystemLocale>
            <UILanguage>zh-CN</UILanguage>
            <UILanguageFallback>zh-CN</UILanguageFallback>
            <UserLocale>zh-CN</UserLocale>
        </component>
        <component name="Microsoft-Windows-Setup" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <DiskConfiguration>
                <Disk wcm:action="add">
                    <CreatePartitions>
                        <CreatePartition wcm:action="add">
                            <Order>1</Order>
                            <Type>Primary</Type>
                            <Size>100</Size>
                        </CreatePartition>
                        <CreatePartition wcm:action="add">
                            <Extend>true</Extend>
                            <Order>2</Order>
                            <Type>Primary</Type>
                        </CreatePartition>
                    </CreatePartitions>
                    <ModifyPartitions>
                        <ModifyPartition wcm:action="add">
                            <Active>true</Active>
                            <Format>NTFS</Format>
                            <Label>System Reserved</Label>
                            <Order>1</Order>
                            <PartitionID>1</PartitionID>
                            <TypeID>0x27</TypeID>
                        </ModifyPartition>
                        <ModifyPartition wcm:action="add">
                            <Active>true</Active>
                            <Format>NTFS</Format>
                            <Label>OS</Label>
                            <Letter>C</Letter>
                            <Order>2</Order>
                            <PartitionID>2</PartitionID>
                        </ModifyPartition>
                    </ModifyPartitions>
                    <DiskID>0</DiskID>
                    <WillWipeDisk>true</WillWipeDisk>
                </Disk>
            </DiskConfiguration>
            <ImageInstall>
                <OSImage>
                    <InstallTo>
                        <DiskID>0</DiskID>
                        <PartitionID>2</PartitionID>
                    </InstallTo>
                    <InstallToAvailablePartition>false</InstallToAvailablePartition>
                </OSImage>
            </ImageInstall>
            <UserData>
                <AcceptEula>true</AcceptEula>
                <FullName>AnJia</FullName>
                <Organization>AnJia</Organization>
                <ProductKey>
                    <Key>VK7JG-NPHTM-C97JM-9MPGT-3V66T</Key>
                </ProductKey>
            </UserData>
        </component>
        <component name="Microsoft-Windows-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <DiskConfiguration>
                <Disk wcm:action="add">
                    <CreatePartitions>
                        <CreatePartition wcm:action="add">
                            <Order>1</Order>
                            <Type>Primary</Type>
                            <Size>100</Size>
                        </CreatePartition>
                        <CreatePartition wcm:action="add">
                            <Extend>true</Extend>
                            <Order>2</Order>
                            <Type>Primary</Type>
                        </CreatePartition>
                    </CreatePartitions>
                    <ModifyPartitions>
                        <ModifyPartition wcm:action="add">
                            <Active>true</Active>
                            <Format>NTFS</Format>
                            <Label>System Reserved</Label>
                            <Order>1</Order>
                            <PartitionID>1</PartitionID>
                            <TypeID>0x27</TypeID>
                        </ModifyPartition>
                        <ModifyPartition wcm:action="add">
                            <Active>true</Active>
                            <Format>NTFS</Format>
                            <Label>OS</Label>
                            <Letter>C</Letter>
                            <Order>2</Order>
                            <PartitionID>2</PartitionID>
                        </ModifyPartition>
                    </ModifyPartitions>
                    <DiskID>0</DiskID>
                    <WillWipeDisk>true</WillWipeDisk>
                </Disk>
            </DiskConfiguration>
            <ImageInstall>
                <OSImage>
                    <InstallTo>
                        <DiskID>0</DiskID>
                        <PartitionID>2</PartitionID>
                    </InstallTo>
                    <InstallToAvailablePartition>false</InstallToAvailablePartition>
                </OSImage>
            </ImageInstall>
            <UserData>
                <AcceptEula>true</AcceptEula>
                <FullName>AnJia</FullName>
                <Organization>AnJia</Organization>
                <ProductKey>
                    <Key>VK7JG-NPHTM-C97JM-9MPGT-3V66T</Key>
                </ProductKey>
            </UserData>
        </component>
    </settings>
    <settings pass="offlineServicing">
        <component name="Microsoft-Windows-LUA-Settings" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <EnableLUA>false</EnableLUA>
        </component>
    </settings>
    <settings pass="offlineServicing">
        <component name="Microsoft-Windows-LUA-Settings" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <EnableLUA>false</EnableLUA>
        </component>
    </settings>
    <settings pass="generalize">
        <component name="Microsoft-Windows-Security-SPP" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <SkipRearm>1</SkipRearm>
        </component>
    </settings>
    <settings pass="generalize">
        <component name="Microsoft-Windows-Security-SPP" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <SkipRearm>1</SkipRearm>
        </component>
    </settings>
    <settings pass="specialize">
        <component name="Microsoft-Windows-International-Core" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <InputLocale>0804:{81D4E9C9-1D3B-41BC-9E6C-4B40BF79E35E}{FA550B04-5AD7-411f-A5AC-CA038EC515D7}</InputLocale>
            <SystemLocale>zh-CN</SystemLocale>
            <UILanguage>zh-CN</UILanguage>
            <UILanguageFallback>zh-CN</UILanguageFallback>
            <UserLocale>zh-CN</UserLocale>
        </component>
        <component name="Microsoft-Windows-International-Core" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <InputLocale>0804:{81D4E9C9-1D3B-41BC-9E6C-4B40BF79E35E}{FA550B04-5AD7-411f-A5AC-CA038EC515D7}</InputLocale>
            <SystemLocale>zh-CN</SystemLocale>
            <UILanguage>zh-CN</UILanguage>
            <UILanguageFallback>zh-CN</UILanguageFallback>
            <UserLocale>zh-CN</UserLocale>
        </component>
        <component name="Microsoft-Windows-Security-SPP-UX" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <SkipAutoActivation>true</SkipAutoActivation>
        </component>
        <component name="Microsoft-Windows-Security-SPP-UX" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <SkipAutoActivation>true</SkipAutoActivation>
        </component>
        <component name="Microsoft-Windows-SQMApi" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <CEIPEnabled>0</CEIPEnabled>
        </component>
        <component name="Microsoft-Windows-SQMApi" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <CEIPEnabled>0</CEIPEnabled>
        </component>
        <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <ComputerName>AnJia-PC</ComputerName>
            <ProductKey>VK7JG-NPHTM-C97JM-9MPGT-3V66T</ProductKey>
        </component>
        <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <ComputerName>AnJia-PC</ComputerName>
            <ProductKey>VK7JG-NPHTM-C97JM-9MPGT-3V66T</ProductKey>
        </component>
    </settings>
    <settings pass="oobeSystem">
        <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <AutoLogon>
                <Password>
                    <Value></Value>
                    <PlainText>true</PlainText>
                </Password>
                <Enabled>true</Enabled>
                <Username>AnJia</Username>
            </AutoLogon>
            <OOBE>
                <HideEULAPage>true</HideEULAPage>
                <HideOEMRegistrationScreen>true</HideOEMRegistrationScreen>
                <HideOnlineAccountScreens>true</HideOnlineAccountScreens>
                <HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>
                <NetworkLocation>Work</NetworkLocation>
                <SkipUserOOBE>true</SkipUserOOBE>
                <SkipMachineOOBE>true</SkipMachineOOBE>
                <ProtectYourPC>1</ProtectYourPC>
            </OOBE>
            <UserAccounts>
                <LocalAccounts>
                    <LocalAccount wcm:action="add">
                        <Password>
                            <Value></Value>
                            <PlainText>true</PlainText>
                        </Password>
                        <Description>AnJia</Description>
                        <DisplayName>AnJia</DisplayName>
                        <Group>Administrators</Group>
                        <Name>AnJia</Name>
                    </LocalAccount>
                </LocalAccounts>
            </UserAccounts>
            <RegisteredOrganization>AnJia</RegisteredOrganization>
            <RegisteredOwner>AnJia</RegisteredOwner>
            <DisableAutoDaylightTimeSet>false</DisableAutoDaylightTimeSet>
            <FirstLogonCommands>
                <SynchronousCommand wcm:action="add">
                    <Description>Control Panel View</Description>
                    <Order>1</Order>
                    <CommandLine>reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\ControlPanel" /v StartupPage /t REG_DWORD /d 1 /f</CommandLine>
                    <RequiresUserInput>true</RequiresUserInput>
                </SynchronousCommand>
                <SynchronousCommand wcm:action="add">
                    <Order>2</Order>
                    <Description>Control Panel Icon Size</Description>
                    <RequiresUserInput>false</RequiresUserInput>
                    <CommandLine>reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\ControlPanel" /v AllItemsIconView /t REG_DWORD /d 0 /f</CommandLine>
                </SynchronousCommand>
                <SynchronousCommand wcm:action="add">
                    <Order>3</Order>
                    <RequiresUserInput>false</RequiresUserInput>
                    <CommandLine>cmd /C wmic useraccount where name="AnJia" set PasswordExpires=false</CommandLine>
                    <Description>Password Never Expires</Description>
                </SynchronousCommand>
            </FirstLogonCommands>
            <TimeZone>China Standard Time</TimeZone>
        </component>
        <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <AutoLogon>
                <Password>
                    <Value></Value>
                    <PlainText>true</PlainText>
                </Password>
                <Enabled>true</Enabled>
                <Username>AnJia</Username>
            </AutoLogon>
            <OOBE>
                <HideEULAPage>true</HideEULAPage>
                <HideOEMRegistrationScreen>true</HideOEMRegistrationScreen>
                <HideOnlineAccountScreens>true</HideOnlineAccountScreens>
                <HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>
                <NetworkLocation>Work</NetworkLocation>
                <SkipUserOOBE>true</SkipUserOOBE>
                <SkipMachineOOBE>true</SkipMachineOOBE>
                <ProtectYourPC>1</ProtectYourPC>
            </OOBE>
            <UserAccounts>
                <LocalAccounts>
                    <LocalAccount wcm:action="add">
                        <Password>
                            <Value></Value>
                            <PlainText>true</PlainText>
                        </Password>
                        <Description>AnJia</Description>
                        <DisplayName>AnJia</DisplayName>
                        <Group>Administrators</Group>
                        <Name>AnJia</Name>
                    </LocalAccount>
                </LocalAccounts>
            </UserAccounts>
            <RegisteredOrganization>AnJia</RegisteredOrganization>
            <RegisteredOwner>AnJia</RegisteredOwner>
            <DisableAutoDaylightTimeSet>false</DisableAutoDaylightTimeSet>
            <FirstLogonCommands>
                <SynchronousCommand wcm:action="add">
                    <Description>Control Panel View</Description>
                    <Order>1</Order>
                    <CommandLine>reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\ControlPanel" /v StartupPage /t REG_DWORD /d 1 /f</CommandLine>
                    <RequiresUserInput>true</RequiresUserInput>
                </SynchronousCommand>
                <SynchronousCommand wcm:action="add">
                    <Order>2</Order>
                    <Description>Control Panel Icon Size</Description>
                    <RequiresUserInput>false</RequiresUserInput>
                    <CommandLine>reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\ControlPanel" /v AllItemsIconView /t REG_DWORD /d 0 /f</CommandLine>
                </SynchronousCommand>
                <SynchronousCommand wcm:action="add">
                    <Order>3</Order>
                    <RequiresUserInput>false</RequiresUserInput>
                    <CommandLine>cmd /C wmic useraccount where name="AnJia" set PasswordExpires=false</CommandLine>
                    <Description>Password Never Expires</Description>
                </SynchronousCommand>
            </FirstLogonCommands>
            <TimeZone>China Standard Time</TimeZone>
        </component>
    </settings>
</unattend>
```

<a name="727af9d3"></a>
### 配置samba
在Cobbler上执行
<a name="0125c65d"></a>
#### 安装samba
```bash
[root@localhost ~]# yum install samba -y
```
<a name="6f1032ee"></a>
#### 修改smb config

```bash
[root@localhost ~]# vi /etc/samba/smb.conf
 
# /etc/samba/smb.conf
[global]
log file = /var/log/samba/log.%m
max log size = 5000
security = user
guest account = nobody
map to guest = Bad User
load printers = yes
cups options = raw
 
[share]
comment = share directory目录
path = /smb/
directory mask = 0755
create mask = 0755
guest ok=yes
writable=yes
```

<a name="9b9fd94f"></a>
#### 启动smb服务

```bash
[root@localhost ~]# service smb start
[root@localhost ~]# systemctl enable smb
```

<a name="6e499122"></a>
#### 挂载win10系统
通过winscp等软件将 cn_windows_10_business_edition_version_1809_updated_sept_2018_x64_dvd_84ac403f.iso 上传到cobbler服务器上,并将创建的应答文件，上传到cobbler `/smb/win/win10_x64_bios_auto.xml` 

```bash
[root@localhost ~]# mkdir -p /smb/win
[root@localhost ~]# mount -o loop,ro /tmp/cn_windows_10_business_edition_version_1809_updated_sept_2018_x64_dvd_84ac403f.iso /mnt/
[root@localhost ~]# cp -r /mnt/* /smb/win
[root@localhost ~]# umount /mnt/
```

<a name="a23d1d1f"></a>
### 自动化安装Windows10
从vmware创建一台内存4G，cpu2核，磁盘60G的空盘，win10虚拟机，然后开机。记得选BIOS，别选UEFI。<br />![Snipaste_2019-02-26_00-05-00.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1551110723753-c7785e92-60be-4a90-a606-96bc92306aaa.png#align=left&display=inline&height=400&name=Snipaste_2019-02-26_00-05-00.png&originHeight=400&originWidth=720&size=5025&status=done&width=720)

![Snipaste_2019-02-25_23-44-27.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1551110627345-af185562-5a0f-4b4c-98ef-68974880a6a3.png#align=left&display=inline&height=560&name=Snipaste_2019-02-25_23-44-27.png&originHeight=768&originWidth=1024&size=19684&status=done&width=746)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1551112320786-fc312bb1-ce6c-4771-864f-e8900a1aed58.png#align=left&display=inline&height=768&name=image.png&originHeight=768&originWidth=1024&size=233953&status=done&width=1024)

至于如何激活，参考  [vlmcsd搭建KMS服务器，成功激活Server 2019数据中心版本，全网应是我首发](http://bbs.pcbeta.com/viewthread-1796113-1-1.html)

<a name="31f87578"></a>
## Windows Server 2019
因为2019用的也是1809版本的，所以制作步骤一样的，在此不再赘述。
<a name="35808e79"></a>
## 参考资料

- [使用Cobbler批量部署Linux和Windows：Windows系统批量安装（三）](https://www.cnblogs.com/pluse/p/8508538.html)
- [WINPE镜像制作-startnet.cmd详解](https://blog.csdn.net/greless/article/details/52815716)
- [Windows 中的默认输入配置文件（输入区域设置）](https://msdn.microsoft.com/zh-cn/library/windows/hardware/dn898524(v=vs.85).aspx)
- [Answer files (unattend.xml)](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/update-windows-settings-and-scripts-create-your-own-answer-file-sxs)
- [Windows Answer File Generator#win10_x86_64](http://www.windowsafg.com/win10x86_x64.html)
- [WinPE: Mount and Customize](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-mount-and-customize)
- [Components](https://docs.microsoft.com/en-us/windows-hardware/customize/desktop/unattend/components-b-unattend)
- [How to create an unattended installation of Windows 10](https://www.windowscentral.com/how-create-unattended-media-do-automated-installation-windows-10)
-  [vlmcsd搭建KMS服务器，成功激活Server 2019数据中心版本，全网应是我首发](http://bbs.pcbeta.com/viewthread-1796113-1-1.html)

<a name="fb674066"></a>
## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。

长期招聘，Java程序员，大数据工程师，运维工程师，前端工程师。


