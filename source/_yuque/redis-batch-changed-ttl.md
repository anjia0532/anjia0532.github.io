
---

title: 019-批量修改redis TTL和批量删除key

date: 2019-05-10 08:37:14 +0800

tags: [redis,python]

categories: redis

---

> 这是坚持技术写作计划（含翻译）的第19篇，定个小目标999，每周最少2篇。


如果因为历史原因，导致redis里存在无用且没有设置ttl的key，会造成浪费。本文主要讲如何在不阻塞redis的情况下批量修改redis的ttl和使用通配符删除key。

<!-- more -->

<a name="s9rs2"></a>
### 通配符删除key

```
redis-cli [-a password] [-h localhost] [-p 6379] --scan --pattern pattern* | xargs redis-cli [-a password] [-h localhost] [-p 6379] del
```
其中 `[]` 包裹的都是可选项

- -p 端口
- -h 是redis主机
- -a 是密码
- pattern* 是通配符
> SCAN,SSCAN,HSCAN,ZSCAN四个命令都支持增量式迭代， 它们每次执行都只会返回少量元素， 所以这些命令可以用于生产环境， 而不会出现像 KEYS 命令、 SMEMBERS 命令带来的问题 —— 当 KEYS 命令被用于处理一个大的数据库时， 又或者 SMEMBERS 命令被用于处理一个大的集合键时， 它们可能会阻塞服务器达数秒之久。

参考资料  [redis 命令 ](http://doc.redisfans.com/key/scan.html)[SCAN](http://doc.redisfans.com/key/scan.html) 

<a name="hG47Y"></a>
### 批量打印或者修改TTL
使用方式

```
$ pip install redis
$ python keys.py --help

usage: keys.py [-h] [-p PORT] [-d DB_LIST] [--host HOST] [--password PASSWORD]
               [--expire EXPIRE] [--random_upper RANDOM_UPPER]
               [--max_ttl MAX_TTL]

optional arguments:
  -h, --help            show this help message and exit
  -p PORT               port of redis
  -d DB_LIST            ex : -d all / -d 1,2,3,4
  --host HOST           ex : --host 127.0.0.1
  --password PASSWORD   ex : --password password
  --expire EXPIRE       unit: sec ,ex 1 days = 86400 sec: --expire 86400
  --random_upper RANDOM_UPPER
                        unit: sec ,ex 1 mins = 60 sec: --random_upper 60
  --max_ttl MAX_TTL     unit: sec ,ex 1 mins = 60 sec: --max_ttl 60

```

```python
# encoding: utf-8
"""
author: yangyi@youzan.com
time: 2018/4/26 下午4:34
func: 获取数据库中没有设置ttl的 key

author: anjia0532@gmail.com
time: 2019/05/10 上午8:19
desc: 增加cli选项，增加批量修改ttl功能
"""
import redis
import argparse
import time
import sys, os
import random

class ShowProcess:
    
    """
    显示处理进度的类
    调用该类相关函数即可实现处理进度的显示
    """
    i = 0 # 当前的处理进度
    max_steps = 0 # 总共需要处理的次数
    max_arrow = 50 # 进度条的长度

    # 初始化函数，需要知道总共的处理次数
    def __init__(self, max_steps):
        self.max_steps = max_steps
        self.i = 0

    # 显示函数，根据当前的处理进度i显示进度
    # 效果为[>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>]100.00%
    def show_process(self, i = None):
        if i is not None:
            self.i = i
        else:
            self.i += 1
        num_arrow = int(self.i * self.max_arrow / self.max_steps) # 计算显示多少个'>'
        num_line = self.max_arrow - num_arrow # 计算显示多少个'-'
        percent = self.i * 100.0 / self.max_steps # 计算完成进度，格式为xx.xx%
        process_bar = '[' + '>' * num_arrow + ' ' * num_line + ']'\
                      + '%.2f' % percent + '%' + '\r' # 带输出的字符串，'\r'表示不换行回到最左边
        sys.stdout.write(process_bar) # 这两句打印字符到终端
        sys.stdout.flush()

    def close(self, words='done'):
        print(words)
        self.i = 0


def check_ttl(redis_conn, dbindex,max_ttl,random_upper,expire):
    start_time = time.time()
    changed_ttl_num = 0
    keys_num = redis_conn.dbsize()
    print( "there are {num} keys in db {index} ".format(num=keys_num, index=dbindex))
    process_bar = ShowProcess(keys_num)
    for key in redis_conn.scan_iter(count=1000):
        process_bar.show_process()
        ttl = redis_conn.ttl(key)
        if ttl > max_ttl or ttl == -1:
            changed_ttl_num += 1
            redis_conn.expire(key, expire + random.randint(0, random_upper))
        else:
            continue

    process_bar.close()
    print("cost time(s):", time.time() - start_time)
    print("changed ttl keys number:", changed_ttl_num)


def main():
    parser = argparse.ArgumentParser()
    
    parser.add_argument('--port', type=int, dest='port', action='store', default=6379,help='port of redis ')
    parser.add_argument('--db_list', type=str, dest='db_list', action='store', default='0',
                        help='ex : -d all / -d 1,2,3,4 ')
    parser.add_argument('--host', type=str, dest='host', action='store', default='127.0.0.1',
                        help='ex : --host 127.0.0.1 ')
    parser.add_argument('--password', type=str, dest='password', action='store', default='',
                        help='ex : --password password ')
    parser.add_argument('--expire', type=int, dest='expire', action='store', default='0',
                        help='unit: sec ,ex 1 days = 86400 sec: --expire 86400 ')
    parser.add_argument('--random_upper', type=int, dest='random_upper', action='store', default='60',
                        help='unit: sec ,ex 1 mins = 60 sec: --random_upper 60 ')
    parser.add_argument('--max_ttl', type=int, dest='max_ttl', action='store', default='60',
                        help='unit: sec ,ex 1 mins = 60 sec: --max_ttl 60 ')
    
    args = parser.parse_args()
    port = args.port
    expire = args.expire
    random_upper = args.random_upper
    max_ttl = args.max_ttl
    host = args.host
    password = args.password
    
    if args.db_list == 'all':
        db_list = [i for i in range(0, 16)]
    else:
        db_list = [int(i) for i in args.db_list.split(',')]
    for index in db_list:
        try:
            pool = redis.ConnectionPool(host=host, port=port, db=index,password=password)
            r = redis.StrictRedis(connection_pool=pool)
        except redis.exceptions.ConnectionError as e:
            print(e)
        else:
            check_ttl(r, index,max_ttl,random_upper,expire)
if __name__ == '__main__':
    main()

```

参考资料  [【Redis】获取没有设置ttl的key脚本](http://blog.itpub.net/22664653/viewspace-2153419/)

<a name="fb674066"></a>
## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。<br />长期招聘，Java程序员，大数据工程师，运维工程师，前端工程师。

