
---

title: 011-openresty的maxminddb插件

date: 2019-03-14 23:26:00 +0800

tags: [nginx,openresty,maxminddb,maxmind,ipip]

categories: 运维

---

> 这是坚持技术写作计划（含翻译）的第11篇，定个小目标999，每周最少2篇。


本文主要介绍我之前基于openresty写的maxminddb的解析插件 -- [anjia0532/lua-resty-maxminddb](https://github.com/anjia0532/lua-resty-maxminddb) (已开源)。主要用途是根据ip获取地理位置。国内精确度不如国内[ipip.net](https://www.ipip.net/) ，但是胜在免费。在精确度要求不高的场景，还是可以用的。

如果要用ipip.net的lua库，可以参考官方的 [ipipdotnet/ipdb-luajit](https://github.com/ipipdotnet/ipdb-luajit)

<a name="b0ff454a"></a>
## 前提条件
<a name="OpenResty"></a>
### OpenResty
```bash
# import our GPG key:
wget -qO - https://openresty.org/package/pubkey.gpg | sudo apt-key add -

# for installing the add-apt-repository command
# (you can remove this package and its dependencies later):
sudo apt-get -y install software-properties-common

# add the our official APT repository:
sudo add-apt-repository -y "deb http://openresty.org/package/ubuntu $(lsb_release -sc) main"

# to update the APT index:
sudo apt-get update

sudo apt-get install openresty
```

<a name="62a25ba8"></a>
### maxmind/libmaxminddb && maxmind/geoipupdate
```bash
sudo add-apt-repository ppa:maxmind/ppa
sudo apt update
sudo apt install libmaxminddb0 libmaxminddb-dev mmdb-bin geoipupdate
```

<a name="a259fd57"></a>
### 配置 geoipupdate

```bash
sudo tee /etc/GeoIP.conf <<-'EOF'
# The following AccountID and LicenseKey are required placeholders.
# For geoipupdate versions earlier than 2.5.0, use UserId here instead of AccountID.
AccountID 0
LicenseKey 000000000000

# Include one or more of the following edition IDs:
# * GeoLite2-City - GeoLite 2 City
# * GeoLite2-Country - GeoLite2 Country
# For geoipupdate versions earlier than 2.5.0, use ProductIds here instead of EditionIDs.
EditionIDs GeoLite2-City GeoLite2-Country
EOF

sudo /usr/local/bin/geoipupdate
```


<a name="68692f19"></a>
## 安装和使用 lua-resty-maxminddb

<a name="e655a410"></a>
### 安装
```bash
opm get anjia0532/lua-resty-maxminddb
```

<a name="2651f0d5"></a>
### 配置openresty

```bash
  local cjson = require 'cjson'
  local geo = require 'resty.maxminddb'
  if not geo.initted() then
      geo.init("/path/to/GeoLite2-City.mmdb")
  end
  local res,err = geo.lookup(ngx.var.arg_ip or ngx.var.remote_addr) --support ipv6 e.g. 2001:4860:0:1001::3004:ef68

  if not res then
      ngx.log(ngx.ERR,'failed to lookup by ip ,reason:',err)
  end

  ngx.say("full :",cjson.encode(res))
  if ngx.var.arg_node then
     ngx.say("node name:",ngx.var.arg_node," ,value:", cjson.encode(res[ngx.var.arg_node] or {}))
  end
```

<a name="db06c78d"></a>
### 测试

```bash
#ipv4
curl localhost/ip=114.114.114.114&node=city
#ipv6
#curl localhost/ip=2001:4860:0:1001::3004:ef68&node=country
full :{"city":{"geoname_id":1799962,"names":{"en":"Nanjing","ru":"Нанкин","fr":"Nankin","pt-BR":"Nanquim","zh-CN":"南京","es":"Nankín","de":"Nanjing","ja":"南京市"}},"subdivisions":[{"geoname_id":1806260,"names":{"en":"Jiangsu","fr":"Province de Jiangsu","zh-CN":"江苏省"},"iso_code":"32"}],"country":{"geoname_id":1814991,"names":{"en":"China","ru":"Китай","fr":"Chine","pt-BR":"China","zh-CN":"中国","es":"China","de":"China","ja":"中国"},"iso_code":"CN"},"registered_country":{"geoname_id":1814991,"names":{"en":"China","ru":"Китай","fr":"Chine","pt-BR":"China","zh-CN":"中国","es":"China","de":"China","ja":"中国"},"iso_code":"CN"},"location":{"time_zone":"Asia\/Shanghai","longitude":118.7778,"accuracy_radius":50,"latitude":32.0617},"continent":{"geoname_id":6255147,"names":{"en":"Asia","ru":"Азия","fr":"Asie","pt-BR":"Ásia","zh-CN":"亚洲","es":"Asia","de":"Asien","ja":"アジア"},"code":"AS"}}
node name:city ,value:{"geoname_id":1799962,"names":{"en":"Nanjing","ru":"Нанкин","fr":"Nankin","pt-BR":"Nanquim","zh-CN":"南京","es":"Nankín","de":"Nanjing","ja":"南京市"}}
```

格式化一下

```javascript
full: {
    "city": {
        "geoname_id": 1799962,
        "names": {
            "en": "Nanjing",
            "ru": "Нанкин",
            "fr": "Nankin",
            "pt-BR": "Nanquim",
            "zh-CN": "南京",
            "es": "Nankín",
            "de": "Nanjing",
            "ja": "南京市"
        }
    },
    "subdivisions": [{
            "geoname_id": 1806260,
            "names": {
                "en": "Jiangsu",
                "fr": "Province de Jiangsu",
                "zh-CN": "江苏省"
            },
            "iso_code": "32"
        }
    ],
    "country": {
        "geoname_id": 1814991,
        "names": {
            "en": "China",
            "ru": "Китай",
            "fr": "Chine",
            "pt-BR": "China",
            "zh-CN": "中国",
            "es": "China",
            "de": "China",
            "ja": "中国"
        },
        "iso_code": "CN"
    },
    "registered_country": {
        "geoname_id": 1814991,
        "names": {
            "en": "China",
            "ru": "Китай",
            "fr": "Chine",
            "pt-BR": "China",
            "zh-CN": "中国",
            "es": "China",
            "de": "China",
            "ja": "中国"
        },
        "iso_code": "CN"
    },
    "location": {
        "time_zone": "Asia\/Shanghai",
        "longitude": 118.7778,
        "accuracy_radius": 50,
        "latitude": 32.0617
    },
    "continent": {
        "geoname_id": 6255147,
        "names": {
            "en": "Asia",
            "ru": "Азия",
            "fr": "Asie",
            "pt-BR": "Ásia",
            "zh-CN": "亚洲",
            "es": "Asia",
            "de": "Asien",
            "ja": "アジア"
        },
        "code": "AS"
    }
}
node name: city, value: {
    "geoname_id": 1799962,
    "names": {
        "en": "Nanjing",
        "ru": "Нанкин",
        "fr": "Nankin",
        "pt-BR": "Nanquim",
        "zh-CN": "南京",
        "es": "Nankín",
        "de": "Nanjing",
        "ja": "南京市"
    }
}
```

<a name="97caa17e"></a>
## 压测 & 性能
事先安装好 [wrk](https://github.com/wg/wrk/wiki)
```bash
sudo tee /tmp/wrk.lua <<-'EOF'
wrk.method = "GET";
wrk.body = "";

logfile = io.open("wrk.log", "w");

request = function()
ip = tostring(math.random(1, 255)).."."..tostring(math.random(1, 255)).."."..tostring(math.random(1, 255)).."."..tostring(math.random(1, 255))
path = "/?ip=" .. ip
return wrk.format(nil, path)
end

response = function(status,header,body)
logfile:write("\nbody:" .. body .. "\n-----------------");
end
EOF

sudo wrk -t50 -c200 -d120s -s /tmp/wrk.lua --latency http://127.0.0.1
```
![](https://cdn.nlark.com/yuque/0/2019/png/226273/1552578843226-04258e12-de95-4cad-89a2-5a4e8021de30.png#align=left&display=inline&height=157&originHeight=258&originWidth=1227&size=0&status=done&width=746)<br />![](https://cdn.nlark.com/yuque/0/2019/png/226273/1552578867709-755ce4c2-069f-4db9-b8ee-dfaa91331193.png#align=left&display=inline&height=209&originHeight=209&originWidth=635&size=0&status=done&width=635)

<a name="35808e79"></a>
## 参考资料

- [OpenResty® Linux Packages](https://openresty.org/en/linux-packages.html)
- [Automatic Updates for GeoIP2 and GeoIP Legacy Databases](https://dev.maxmind.com/geoip/geoipupdate/)
- [maxmind/libmaxminddb](https://github.com/maxmind/libmaxminddb)
- [maxmind/geoipupdate](https://github.com/maxmind/geoipupdate)
- [wg/wrk#wiki](https://github.com/wg/wrk/wiki)
- [MMDB_free_entry_data_list (entry_data_list=0x23) at maxminddb.c:1860 #9](https://github.com/anjia0532/lua-resty-maxminddb/issues/9)

<a name="fb674066"></a>
## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。

长期招聘，Java程序员，大数据工程师，运维工程师，前端工程师。


