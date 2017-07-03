---
title: windows 10 64bit下安装Tensorflow+Keras
date: 2017-07-03 16:40:58
tags: [python,anaconda,AI,Tensorflow,Keras]
categories: [anaconda,python,AI,Tensorflow,Keras]
---

### 修改pip源 参考 [Python pip 国内镜像大全及使用办法][linkPythonPip国内镜像大全及使用办法]

官方文档 [Config file][linkConfigFile]

windows 全部用户需要在`%APPDATA%\pip\pip.ini`,当前用户在`%HOME%\pip\pip.ini`

```
[global]
index-url=http://mirrors.aliyun.com/pypi/simple/
[install]
trusted-host=mirrors.aliyun.com
```

### 安装Tensorflow

参考 [Installing TensorFlow on Windows][linkInstallingTensorflowOnWindows] 

```
# 切换到 python3.5 参考 详见另外一篇博文 https://anjia.ml/2017/07/02/anaconda-install-and-configurating-jupyter/#切换python版本

#打开Anaconda Prompt
(python35) C:\Users\xx> activate python35

#因为电脑无独显，所以安装`CPU-only`版本
(python35) C:\Users\xx> pip install --ignore-installed --upgrade https://storage.googleapis.com/tensorflow/windows/cpu/tensorflow-1.2.1-cp35-cp35m-win_amd64.whl 

(python35) C:\Users\xx>python
```

```python
>>> import tensorflow as tf
>>> hello = tf.constant('Hello, TensorFlow')
>>> sess = tf.Session()
2017-07-03 16:44:16.082952: W c:\tf_jenkins\home\workspace\release-win\m\windows\py\35\tensorflow\core\platform\cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use SSE instructions, but these are available on your machine and could speed up CPU computations.
2017-07-03 16:44:16.085175: W c:\tf_jenkins\home\workspace\release-win\m\windows\py\35\tensorflow\core\platform\cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use SSE2 instructions, but these are available on your machine and could speed up CPU computations.
2017-07-03 16:44:16.085590: W c:\tf_jenkins\home\workspace\release-win\m\windows\py\35\tensorflow\core\platform\cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use SSE3 instructions, but these are available on your machine and could speed up CPU computations.
2017-07-03 16:44:16.085952: W c:\tf_jenkins\home\workspace\release-win\m\windows\py\35\tensorflow\core\platform\cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use SSE4.1 instructions, but these are available on your machine and could speed up CPU computations.
2017-07-03 16:44:16.086312: W c:\tf_jenkins\home\workspace\release-win\m\windows\py\35\tensorflow\core\platform\cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use SSE4.2 instructions, but these are available on your machine and could speed up CPU computations.
2017-07-03 16:44:16.086634: W c:\tf_jenkins\home\workspace\release-win\m\windows\py\35\tensorflow\core\platform\cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use AVX instructions, but these are available on your machine and could speed up CPU computations.
2017-07-03 16:44:16.087014: W c:\tf_jenkins\home\workspace\release-win\m\windows\py\35\tensorflow\core\platform\cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use AVX2 instructions, but these are available on your machine and could speed up CPU computations.
2017-07-03 16:44:16.087363: W c:\tf_jenkins\home\workspace\release-win\m\windows\py\35\tensorflow\core\platform\cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use FMA instructions, but these are available on your machine and could speed up CPU computations.
>>> print(sess.run(hello))
b'Hello, TensorFlow'
```

如果要去掉`4-12`的警告信息，需要自己编译。详见 ["The TensorFlow library wasn't compiled to use SSE instructions, but these are available on your machine and could speed up CPU computations" in "Hello, TensorFlow!" program #7778](https://github.com/tensorflow/tensorflow/issues/7778)

### 安装Keras

参考 官方文档 [Installation][]  中文文档  [Keras安装和配置指南(Windows)][]

```bash
(python35) C:\Users\xx>pip install keras -U --pre
```

但是我安装一直报错，
```bash
Running setup.py bdist_wheel for scipy ... error
  Complete output from command {Anaconda3_home}\envs\python35\python.exe -u -c "import setuptools, tokenize;__file__='{AppData}\\Local\\Temp\\pip-build-mgdjtt1d\\scipy\\setup.py';f=getattr(tokenize, 'open', open)(__file__);code=f.read().replace('\r\n', '\n');f.close();exec(compile(code, __file__, 'exec'))" bdist_wheel -d {AppData}\Local\Temp\tmpb_od_dlvpip-wheel- --python-tag cp35:
  lapack_opt_info:
  lapack_mkl_info:
    libraries mkl_rt not found in ['{Anaconda3_home}\\envs\\python35\\lib', 'C:\\', '{Anaconda3_home}\\envs\\python35\\libs']
    NOT AVAILABLE

## ...

Command "{Anaconda3_home}\envs\python35\python.exe -u -c "import setuptools, tokenize;__file__='{AppData}\\Local\\Temp\\pip-build-mgdjtt1d\\scipy\\setup.py';f=getattr(tokenize, 'open', open)(__file__);code=f.read().replace('\r\n', '\n');f.close();exec(compile(code, __file__, 'exec'))" install --record {AppData}\Local\Temp\pip-htcraop7-record\install-record.txt --single-version-externally-managed --compile" failed with error code 1 in {AppData}\Local\Temp\pip-build-mgdjtt1d\scipy\
```

网上有建议通过 `pip install git+git://github.com/Theano/Theano.git` 从github直接下最新代码安装的，但是也是安装失败

我成功的方式

```
(python35) C:\Users\xx>conda install mingw libpython theano -y
(python35) C:\Users\xx>pip install keras
```

天朝网络不稳定，挺慢的，可以参考 另外一篇博文切换清华源 [https://anjia.ml/2017/07/02/anaconda-install-and-configurating-jupyter/#设置清华镜像源][清华镜像源]
```
(python35) C:\Users\xx>python
Python 3.5.3 |Continuum Analytics, Inc.| (default, May 15 2017, 10:43:23) [MSC v.1900 64 bit (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information.
>>> import keras
Using TensorFlow backend.
```
安装成功，默认后端是TensorFlow

[linkPythonPip国内镜像大全及使用办法]: http://blog.csdn.net/testcs_dn/article/details/54374849
[linkConfigFile]: https://pip.pypa.io/en/stable/user_guide/#config-file
[linkInstallingTensorflowOnWindows]: https://www.tensorflow.org/install/install_windows
[Installation]: https://keras.io/#installation
[Keras安装和配置指南(Windows)]: https://keras-cn.readthedocs.io/en/latest/for_beginners/keras_windows/
[清华镜像源]: https://anjia.ml/2017/07/02/anaconda-install-and-configurating-jupyter/#设置清华镜像源
