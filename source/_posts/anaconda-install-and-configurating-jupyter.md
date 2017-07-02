---
title: 安装anaconda并且配置jupyter notebook
date: 2017-07-02 10:59:18
tags:[python,anaconda,jupyter,ipython]
categories: [anaconda,jupyter]
---

业余时间，偶尔接触了python，感觉python很优雅，遂研究一下。[基于elk报警器elastalert的微信企业号插件](https://github.com/anjia0532/elastalert-wechat-plugin/blob/master/wechat_qiye_alert.py)

之前一直用的[sublime text 3][linkSublimeText3] , 但是对于控制台输入(2.x raw_input,3.x input)支持不太好，虽然可以通过`sublimeREPL`->`python`->`execfile(filepath)`实现，但是无疑更繁琐(可以使用sublime 的key bindings，定义快捷键来触发，但是还是觉得繁琐)，而且使用`sublime+python`切换python版本也不方便(网上很多资料是基于python2.x)，但是python3的文章资料也越来越多，学习时经常需要切换很不方便

经过一番搜索，最后决定使用[Anaconda][Anaconda] Anaconda是Python众多发行版中非常适用于科学计算的版本，里面已经集成了很多优秀的科学计算Python库,开源且免费，全平台支持:linux,mac,windows;支持python 2.x，3.x,Anaconda集成了[jupyter notebook][linkJupyterNotebook] ，可以使用 [try it in your browser][linkTryItInYourBrowser] 进行体验。

<!-- more -->

## 安装anaconda

官方安装包 [https://www.continuum.io/downloads][anacondaDownloads] ,但是国内比较慢，可以使用[清华镜像][qinghua] ,从 [https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/][anacondaQingHuaDownloads] 下载安装包。目前(2017-07-02)最新的是 `Anaconda3-4.4.0-*` 

我下载的是windows 64位版[Anaconda3-4.4.0-Windows-x86_64.exe][linkWindows64]（如果用于机器学习(e.g. [Tensorflow][]) 建议使用Linux系统，具体参见 [Keras安装和配置指南(Windows)][keras_windows]）。

同时推荐 [李金][lijin-THU]的 [《中文 Python 笔记》][notes-python] ,github 打开.ipynb 较慢，推荐使用[NbViewer][] 查看



### 切换python版本

参考 [Managing Python][linkManagingPython] 或者 [Anaconda多环境多版本python配置指导][5465452]

打开 `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Anaconda3 (64-bit)` 运行 `Anaconda Prompt`

#### 设置清华镜像源
更多可参阅 [conda 使用清华大学开源软件镜像][linkConda使用清华大学开源软件镜像] 或者 [清华镜像][qinghua]
```bash
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --set show_channel_urls yes
```

#### 修改windows下jupyter默认路径
参考 stackoverflow 上 [how to change jupyter start folder?][linkHowToChangeJupyterStartFolder?] 的回答

1. 打开 `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Anaconda3 (64-bit)` 运行 `Anaconda Prompt`
2. 运行`jupyter notebook --generate-config`
3. 会生成一个默认配置文件，`C:\Users\{用户名}\.jupyter\jupyter_notebook_config.py`
4. 修改`#c.NotebookApp.notebook_dir = ''` 为 `c.NotebookApp.notebook_dir = '你的默认路径'`
5. 打开`C:\Users\{用户名}\Anaconda3\Scripts`
6. 右键单击`jupyter-notebook.exe`并发送到`桌面快捷方式` 
7. 在桌面上找到该快捷方式，`右键`->`属性`->`更改图标(C)...`->`{Anaconda3_home}\Menu\jupyter.ico`
8. 双击运行，会自动打开默认浏览器。
![](http://ww1.sinaimg.cn/large/afaffa71ly1fh5hpxapcxj20w3080q3f.jpg)

输入

```python
print('hello jupyter')
```
按 <kbd>Ctrl</kbd>+<kbd>Enter</kbd> 运行，结果如下
![](http://ww1.sinaimg.cn/large/afaffa71ly1fh5hpxb7fej20w00683yr.jpg)

具体快捷键，参见 `Help` -> `Keyboard Shortcuts`
![](http://ww1.sinaimg.cn/large/afaffa71ly1fh5htjiup4j210d0nzjty.jpg)

#### 创建python2.7环境

```bash
conda create -n python27 python=2.7 -y
activate python27
```
#### 设置jupyter 2.7,3.6共存
参考 [Anaconda3 Python 3 和 2 in Jupyter Notebook共存方法][linkAnaconda3Python3和2InJupyterNotebook共存方法]
```
conda install ipykernel -y
```

复制`${Anaconda3_home}\share\jupyter\kernels\python3` 并重命名为`${Anaconda3_home}\share\jupyter\kernels\python27`，编辑`${Anaconda3_home}\share\jupyter\kernels\python27\kernel.json`
```
{
 "argv": [
  "${Anaconda3_home}\\envs\\python27\\python.exe",
  "-m",
  "ipykernel_launcher",
  "-f",
  "{connection_file}"
 ],
 "display_name": "Python 27",
 "language": "python"
}
```

注意，修改`display_name`为自定义名称，`argv`第一行中路径用`\\`替代`\`

![jupyter-change-kernel](http://ww1.sinaimg.cn/large/afaffa71ly1fh5h0mqjg8j20ha08ujrt.jpg)

在`cell`中输入
```python
import sys 
sys.version
```

切换不同python版本 按<kbd>Ctrl</kbd>+<kbd>Enter</kbd>运行 查看版本，e.g. 上图中的`3.6.1`,因为 Tensorflow官方文档说windows只支持 [3.5.x][] ，故而又装了一个3.5.3的环境

#### jupyter作为公开服务使用(云IDE)

参考 [Running a notebook server][linkRunningANotebookServer] ，使用[nssm][linkWindows10CreatorsUpdate]将jupyter设置为开机自启动服务

1. 打开 `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Anaconda3 (64-bit)` 运行 `Anaconda Prompt`
2. 切换回anaconda默认环境`activate root`
3. 创建密码 `jupyter notebook password` [Preparing a hashed password][linkPreparingAHashedPassword]
4. 修改`C:\Users\{用户名}\.jupyter\jupyter_notebook_config.py`中`c.NotebookApp.ip = '*'`,`c.NotebookApp.open_browser = False`
5. 重启 jupyter ,打开 http://{ip}:8888, 提示输入密码，输入密码即可登录
6. 注册为服务 下载[nssm][linkWindows10CreatorsUpdate] 注意，如果之前用过nssm，建议升级到 nssm 2.24-101-g897c7ad 版本，详见 [Windows 10 Creators Update][linkWindows10CreatorsUpdate] 
7. `{nssm_home}\win64\nssm.exe install jupyter {Anaconda3_home}\Scripts\jupyter-notebook.exe --config=C:\Users\{用户名}\.jupyter\jupyter_notebook_config.py`
8. `{nssm_home}\win64\nssm.exe start jupyter'
9. 浏览器打开 http://ip:8888 输入密码登录

注意，nssm默认使用`LOCALSYSTEM `账号操作，而jupyter默认读取`~\.jupyter`(`~\`是当前登录用户文件夹)，可以使用`nssm set <servicename> ObjectName <username> <password>` 使用指定用户，这样就不需要`--config=C:\Users\{用户名}\.jupyter\jupyter_notebook_config.py` 参数了，具体详见 [Usage][] 和 [Managing services from the command line][linkManagingServicesFromTheCommandLine]

[linkSublimeText3]: http://www.sublimetext.com/3
[Anaconda]: https://www.continuum.io
[linkJupyterNotebook]: http://jupyter.org/
[linkTryItInYourBrowser]: https://try.jupyter.org/
[qinghua]: https://mirrors.tuna.tsinghua.edu.cn/help/anaconda/
[anacondaDownloads]: https://www.continuum.io/downloads
[anacondaQingHuaDownloads]: https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/
[Tensorflow]: http://tensorflow.org/
[keras_windows]: https://keras-cn.readthedocs.io/en/latest/for_beginners/keras_windows/
[linkWindows64]: https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/Anaconda3-4.4.0-Windows-x86_64.exe
[notes-python]: https://github.com/lijin-THU/notes-python
[lijin-THU]: https://github.com/lijin-THU
[NbViewer]: http://nbviewer.jupyter.org/github/lijin-THU/notes-python/blob/master/index.ipynb 
[linkManagingPython]: https://conda.io/docs/py2or3.html
[5465452]: http://www.cnblogs.com/harvey888/p/5465452.html
[linkConda使用清华大学开源软件镜像]: http://blog.csdn.net/u010570551/article/details/54291507
[linkAnaconda3Python3和2InJupyterNotebook共存方法]: https://segmentfault.com/a/1190000008585746
[3.5.x]: https://www.tensorflow.org/install/install_windows#installing_with_native_pip
[linkHowToChangeJupyterStartFolder?]: https://stackoverflow.com/a/44463707/7001350
[linkRunningANotebookServer]: http://jupyter-notebook.readthedocs.io/en/latest/public_server.html
[linkPreparingAHashedPassword]: http://jupyter-notebook.readthedocs.io/en/latest/public_server.html#preparing-a-hashed-password
[linkWindows10CreatorsUpdate]: http://www.nssm.cc/download
[linkManagingServicesFromTheCommandLine]: http://www.nssm.cc/commands
[Usage]: http://www.nssm.cc/usage
