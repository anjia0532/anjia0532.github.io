---
title: 064-真正解决scrapy自动将header请求头大写问题
urlname: scrapy-capitalizes-request-headers
date: 2021-08-05 19:35:21 +0800
tags: [scrapy,python,爬虫]
categories: [python]
---

> 这是坚持技术写作计划（含翻译）的第 64 篇，定个小目标 999，每周最少 2 篇。

本文主要讲解如何真正解决 scrapy 将 header 请求头自动大写(`str.title()`)的问题

<!-- more -->

## 背景

搞了个小爬虫，命名参数都正常，但是被模目标网站识别了，用 requests 又都正常，问题出在 scrapy 没跑了
​

## 分析过程

用到的工具

- fiddler , charles , wireshark 任选一个抓包工具就行
- beyondcompare 等比对工具

分别用 request 和 scrapy 请求目标网站，url，参数，form 等都用一样的数据（排除类似随机数，时间戳，rsa 非对称加密等导致的数据不一致的问题）
​

以 fiddler 为例，点开抓包数据，选择 Raw 选项卡，复制到比对工具里,真实用的过程中，
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1628133543722-cbe3e3ff-c63c-4fb7-ab89-9c8ed70e8b7c.png#clientId=uedcf7eb9-2963-4&from=paste&height=664&id=u378f7054&margin=%5Bobject%20Object%5D&name=image.png&originHeight=664&originWidth=828&originalType=binary∶=1&size=635105&status=done&style=none&taskId=u70e9eb42-0fd1-4d36-9861-0c562a3a728&width=828)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1628133733213-d4c1e4e9-383e-4445-9b86-f434d2a16517.png#clientId=uedcf7eb9-2963-4&from=paste&height=231&id=u7f0f17c8&margin=%5Bobject%20Object%5D&name=image.png&originHeight=231&originWidth=1179&originalType=binary∶=1&size=30344&status=done&style=none&taskId=u62227fbe-5c16-47be-b627-b37d3a2b56a&width=1179)
找到问题了，scrapy 自动将 header 转换成大写了,看了下 scrapy 的源码， 问题出在
[https://github.com/scrapy/scrapy/blob/4d1ecc31c9bdb42638e8a1f85f7e7f83130f41f3/scrapy/http/headers.py#L15](https://github.com/scrapy/scrapy/blob/4d1ecc31c9bdb42638e8a1f85f7e7f83130f41f3/scrapy/http/headers.py#L15) 的 `key.title()`

```python
    def normkey(self, key):
        """Normalize key to bytes"""
        return self._tobytes(key.title())
```

开始以为解决起来比较简单，用了网上的一些办法，还是不行。后来又研究了下源码，结合网上的方案，终于搞定了
​

新建`Headers.py`,复制自 [https://github.com/scrapy/scrapy/blob/4d1ecc31c9bdb42638e8a1f85f7e7f83130f41f3/scrapy/http/headers.py](https://github.com/scrapy/scrapy/blob/4d1ecc31c9bdb42638e8a1f85f7e7f83130f41f3/scrapy/http/headers.py) ,只改第 15 行将 `self._tobytes(key.title())`改成`self._tobytes(key)`

```python
from w3lib.http import headers_dict_to_raw
from scrapy.utils.datatypes import CaselessDict
from scrapy.utils.python import to_unicode


class Headers(CaselessDict):
    """Case insensitive http headers dictionary"""

    def __init__(self, seq=None, encoding='utf-8'):
        self.encoding = encoding
        super().__init__(seq)

    def normkey(self, key):
        """Normalize key to bytes"""
        return self._tobytes(key)

    def normvalue(self, value):
        """Normalize values to bytes"""
        if value is None:
            value = []
        elif isinstance(value, (str, bytes)):
            value = [value]
        elif not hasattr(value, '__iter__'):
            value = [value]

        return [self._tobytes(x) for x in value]

    def _tobytes(self, x):
        if isinstance(x, bytes):
            return x
        elif isinstance(x, str):
            return x.encode(self.encoding)
        elif isinstance(x, int):
            return str(x).encode(self.encoding)
        else:
            raise TypeError(f'Unsupported value type: {type(x)}')

    def __getitem__(self, key):
        try:
            return super().__getitem__(key)[-1]
        except IndexError:
            return None

    def get(self, key, def_val=None):
        try:
            return super().get(key, def_val)[-1]
        except IndexError:
            return None

    def getlist(self, key, def_val=None):
        try:
            return super().__getitem__(key)
        except KeyError:
            if def_val is not None:
                return self.normvalue(def_val)
            return []

    def setlist(self, key, list_):
        self[key] = list_

    def setlistdefault(self, key, default_list=()):
        return self.setdefault(key, default_list)

    def appendlist(self, key, value):
        lst = self.getlist(key)
        lst.extend(self.normvalue(value))
        self[key] = lst

    def items(self):
        return ((k, self.getlist(k)) for k in self.keys())

    def values(self):
        return [self[k] for k in self.keys()]

    def to_string(self):
        return headers_dict_to_raw(self)

    def to_unicode_dict(self):
        """ Return headers as a CaselessDict with unicode keys
        and unicode values. Multiple values are joined with ','.
        """
        return CaselessDict(
            (to_unicode(key, encoding=self.encoding),
             to_unicode(b','.join(value), encoding=self.encoding))
            for key, value in self.items())

    def __copy__(self):
        return self.__class__(self)

    copy = __copy__

```

修改 scrapy 的`settings.py`文件,参考 twisted 的源码 [https://github.com/twisted/twisted/blob/022659ca88dca6361288cd26486942cba0ed77c4/src/twisted/web/http_headers.py#L261-L279](https://github.com/twisted/twisted/blob/022659ca88dca6361288cd26486942cba0ed77c4/src/twisted/web/http_headers.py#L261-L279)

```xml
# 忽略其他
from twisted.web.http_headers import Headers as TwistedHeaders

TwistedHeaders._caseMappings.update({
    b"aa": b"aa",
    b"aa-aa": b"aa-aa",
    b"aa-bb": b"AA-BB",
    b"aa-cc": b"AA-cc",
})

# 清空默认header
DEFAULT_REQUEST_HEADERS = {
}
```

修改使用代码

```python
from your.package import Headers

headers = {
    "aa": "a",
    "aa-aa": "aa-aa",
    "aa-bb": "AA-BB",
    "aa-cc": "AA-cc",
    # 千万别用 AA-BB或者AA-cc这样的，就是全部用小写，通过修改TwistedHeaders._caseMappings.update的映射来实现就行了
}

yield scrapy.FormRequest(url=url + "?" + parse.urlencode(params, quote_via=parse.quote),
             formdata=body, dont_filter=True, errback=self.error,headers=Headers(headers),
```

![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1628138380238-5fd4d26d-298e-4a66-a1c5-ac070780e63c.png#clientId=uedcf7eb9-2963-4&from=paste&height=307&id=u71c57cbc&margin=%5Bobject%20Object%5D&name=image.png&originHeight=307&originWidth=651&originalType=binary∶=1&size=183535&status=done&style=none&taskId=u663f7a5c-58a6-4f21-b942-4c7ea3fa68f&width=651)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/08/05/scrapy-capitalizes-request-headers)
- [我的掘金](https://juejin.cn/post/6993634847272992775/)
- [Scrapy capitalizes request headers](https://stackoverflow.com/a/57788982)
- [How to prevent scrapy from forcing the key of the request headers into uppercase? ](https://github.com/scrapy/scrapy/issues/3981)
