---
title: 055-手把手教你如何抓取高德全部POI-基于ArcGIS 渔网分割
urlname: python-gaode-arcgis-poi
date: 2021-01-19 22:35:21 +0800
tags: [arcmap,arcgis,python,gis]
categories: [python]
---

> 这是坚持技术写作计划（含翻译）的第 55 篇，定个小目标 999，每周最少 2 篇。

工作原因需要获取本地全部 poi，直接调用高德 api 会有数量限制（理论上超过 1000 条就容易丢失部分数据，而且这种丢失很隐蔽，难以校验）

最终经过一番周折，基于网上资料进行了整理

1. 注册高德账号，获取高德 key
1. 下载安装 ArcMap/ArcGIS
1. 将抓取区域等距分割并导出多边形经纬度
1. 基于切割后的经纬度重新轮高德 api

<!-- more -->

## 前置准备

[注册高德账号，并创建应用，获得 key](https://console.amap.com/dev/key/app) ，这个看官网文档或者网上自己搜就行，不多说了

获取行政区划点位，[https://lbs.amap.com/api/webservice/guide/api/district](https://lbs.amap.com/api/webservice/guide/api/district)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611042302548-dc404b4c-1826-41c8-a3c3-1fb831cf48b1.png#align=left&display=inline&height=852&margin=%5Bobject%20Object%5D&name=image.png&originHeight=852&originWidth=1088&size=100689&status=done&style=none&width=1088)
复制出 polyline,用任何一个现代化编辑器(vs code,notepad++等)将 `;`  替换成换行符,在第一行插入 `x,y`  保存成 `jn.csv`

根据 [ArcMap 0 （ArcGIS10.2 安装(完善版--能解决常见问题)）](https://www.cnblogs.com/9587cgq/p/12839911.html) 下载并安装 ArcMap 10.2,并破解，但别汉化（后边点集转线的时候会报错）

## 获取分割后的坐标点位

### 导入点集

File->Add Data->Add XY Data...
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611043072640-9ec08bc2-8691-4d13-9785-9d43d65c22c4.png#align=left&display=inline&height=433&margin=%5Bobject%20Object%5D&name=image.png&originHeight=433&originWidth=483&size=42845&status=done&style=none&width=483)
建议如果不会用 arcmap 切换文件夹的，直接将文件放入 `文档\ArcGIS\`  下,自动带入 `x,y`  字段，点 OK 即可
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611043097101-beb77c05-16a9-43a0-848c-62baecebac57.png#align=left&display=inline&height=572&margin=%5Bobject%20Object%5D&name=image.png&originHeight=572&originWidth=503&size=35207&status=done&style=none&width=503)
出现行政区划点图像了
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611043189285-c26fb55e-11f4-40cc-ae67-c9e3ef8bb444.png#align=left&display=inline&height=730&margin=%5Bobject%20Object%5D&name=image.png&originHeight=730&originWidth=853&size=50063&status=done&style=none&width=853)

### 点集转线

ArcToolbox -> Data Management Tools -> Features ->Points To Line
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611043427769-ab358a11-d225-40a3-be4e-41bfd2ffb8c0.png#align=left&display=inline&height=1038&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1038&originWidth=614&size=80463&status=done&style=none&width=614)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611043701076-f83b8fd4-1009-46ba-b60a-778f0ea41370.png#align=left&display=inline&height=823&margin=%5Bobject%20Object%5D&name=image.png&originHeight=823&originWidth=1644&size=62475&status=done&style=none&width=1644)

### 使用渔网功能等分切割行政区域

![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611044119970-f885f090-fd79-408a-a1e3-018f808e084b.png#align=left&display=inline&height=859&margin=%5Bobject%20Object%5D&name=image.png&originHeight=859&originWidth=914&size=102418&status=done&style=none&width=914)
cell size width/height 这俩别选，因为高德导入的是火星坐标，不属于标准 gps 坐标，ArcGis 不识别

所以只能自行计算，假设我需要将济南切割成 3km X 3km 的小格子，参考 [经纬度 1 度等于多少米](http://blog.sciencenet.cn/blog-3417521-1191829.html)，经度 1 度=85.39km，纬度 1 度 = 大约 111km。那就拿出计算器算呗，(Right-Left)*85.39/3=49.78826191,取整用 50 就行，同理 (Top-Bottom)*111/3=57.060475,取整用 57 就行，我截图用的 58，懒得换了。

注意别选 `Create Label Points(optional)`  选项,因为用不到

### 将行政区划填充颜色

ArcToolbox -> Data Management Tools -> Features -> Feature To Polygon
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611044297987-02707df1-94bc-442e-8f27-fba55130eafa.png#align=left&display=inline&height=825&margin=%5Bobject%20Object%5D&name=image.png&originHeight=825&originWidth=924&size=104278&status=done&style=none&width=924)

### 取渔网和行政区划交集（擦除行政区划外的渔网网格部分）

![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611044640073-7f7f3e4a-2031-4f30-b3b7-2e196ef41b2a.png#align=left&display=inline&height=901&margin=%5Bobject%20Object%5D&name=image.png&originHeight=901&originWidth=1657&size=165875&status=done&style=none&width=1657)
将其导出后边要用
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611045920107-b7008594-3043-41d5-a126-b3138774ba00.png#align=left&display=inline&height=473&margin=%5Bobject%20Object%5D&name=image.png&originHeight=473&originWidth=820&size=63091&status=done&style=none&width=820)

### 获取网格点位的经纬度

参考大佬的文章 [Python3 爬取高德地图 POI](https://blog.csdn.net/GISer_kaifang/article/details/88603006) ，下载作者开发的 渔网对角坐标获取工具
链接：链接：[https://pan.baidu.com/s/11nwWHDf75fSVTxs323opFQ](https://pan.baidu.com/s/11nwWHDf75fSVTxs323opFQ)
提取码：rduv
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611045650929-80a75c83-ab60-455e-b311-d2826a4bdbd9.png#align=left&display=inline&height=440&margin=%5Bobject%20Object%5D&name=image.png&originHeight=440&originWidth=731&size=78324&status=done&style=none&width=731)
获取对角坐标
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611045997156-140ee0e6-ec6d-427f-b1ca-7c4114643df6.png#align=left&display=inline&height=506&margin=%5Bobject%20Object%5D&name=image.png&originHeight=506&originWidth=840&size=81016&status=done&style=none&width=840)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611046100753-61a007a2-7218-4d67-a54f-e824287a8758.png#align=left&display=inline&height=312&margin=%5Bobject%20Object%5D&name=image.png&originHeight=312&originWidth=691&size=56885&status=done&style=none&width=691)

## 抓取高德 POI

98%都是抄自 [Python3 爬取高德地图 POI](https://blog.csdn.net/GISer_kaifang/article/details/88603006) 代码，只是稍微处理了下从 dict 获取字段防止抛异常部分，以及加了省字段

```python
import requests
import json
from pymongo import MongoClient
import time

client = MongoClient('localhost',27017)
db = client.POI_Jinanshi
collection = db.table_1
polygon_list = list()

with open("jn2.txt", 'r', encoding='UTF-8') as txt_file:
    for each_line in txt_file:
        if each_line != "" and each_line != "\n":
            fields = each_line.split("\n")
            polygon = fields[0]
            polygon_list.append(polygon)

def getjson(polygon, page):
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.67 Safari/537.36'
    }
    pa = {
        'key': 'xxxxxxxxxxxxxxxxxxxxxx', #从控制台申请
        'polygon': polygon,
        'types':'970000|990000',    #不要过多
        'city':'0531',
        'offset': 20,
        'page': page,
        'extensions': 'all',
        'output': 'JSON'
    }
    r = requests.get('https://restapi.amap.com/v3/place/polygon?', params=pa, headers=headers)
    decodejson = json.loads(r.text)
    return decodejson

for each_polygon in polygon_list:
    not_last_page = True
    page = 1
    while not_last_page:
        decodejson = getjson(each_polygon, page)
        print(decodejson)
        count = decodejson.get('count',0)
        print(each_polygon, page)
        if decodejson['pois']:
            for eachone in decodejson['pois']:
                data={
                    'name':eachone.get('name',None), #POI名称
                    'types':eachone.get('type',None), #POI所属类别
                    'address':eachone.get('address',None), #POI地址
                    'location':eachone.get('location',None), #POI坐标
                    'city':eachone.get('cityname',None), #城市
                    'county':eachone.get('adname',None), #区县
                    'province':eachone.get('pname',None), # 省份
                    'count':count, # 条数
                    'polygon':each_polygon # 多边形经纬度，便于后边再次抓取
                }
                collection.insert_one(data)
                time.sleep(0.2)
            page += 1
        else:
            not_last_page = False
```

![](https://cdn.nlark.com/yuque/0/2021/png/226273/1611046647751-625a4ad4-5eb9-48a6-9ee2-ee60ea988f03.png#align=left&display=inline&height=825&margin=%5Bobject%20Object%5D&originHeight=825&originWidth=1380&size=0&status=done&style=none&width=1380)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html?ka=search_list_jname_2_blank&lid=ak5J7ypLUb7.search.2) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/01/19/python-gaode-arcgis-poi/)
- [我的掘金](https://juejin.cn/post/6919393638149521415/)
- [ArcMap 0 （ArcGIS10.2 安装(完善版--能解决常见问题)）](https://www.cnblogs.com/9587cgq/p/12839911.html)
- [Python3 爬取高德地图 POI](https://blog.csdn.net/GISer_kaifang/article/details/88603006)
- [python 获取高德 poi](https://blog.csdn.net/weixin_47796965/article/details/108372378)
- [经纬度 1 度等于多少米](http://blog.sciencenet.cn/blog-3417521-1191829.html)
