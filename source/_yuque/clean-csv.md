---
title: 032-csv文件容错处理
urlname: clean-csv
date: '2019-07-16 21:30:00 +0800'
tags:
  - python
  - csv
  - 数据分析
  - 数据处理
categories: 大数据
---

> 这是坚持技术写作计划（含翻译）的第 32 篇，定个小目标 999，每周最少 2 篇。

如果数据库有特殊字符(换行符，转义符),会导致生成的 csv 无法正常导入。

```
val1,val2,val3
aa,bb,cc
a
a,bb,cc
aa
,bb,cc
aa,
bb,cc
a\a,bb,cc
```

> 第一行 header 和第二行数据正常。
> 第三行第一个列有换行符，此时导致第四行看着正常(3 列),但是数据又是错误的。
> 第五行跟第三行类似
> 第七行实际是第二个单元格首字符换行，导致第八行缺失一列。
> 第九行有转义符

处理成

```
val1,val2,val3
aa,bb,cc
aa,bb,cc
aa,bb,cc
aa,bb,cc
aa,bb,cc
```

<!-- more -->

利用空闲时间，用 python 写了个修补工具,原理是利用，csv 是从上往下读的，如果前一行列数不够，一定可以从后一列补上。但是可能存在补完后超过指定列(比如列内包含分隔符，导致数据库 3 列，变成 4 列)，所以需要对其切片，只保留指定列数。

clean_csv.py

```python
# -*- coding: utf-8 -*-
# Author AnJia(anjia0532@gmail.com https://anjia0532.github.io)
import argparse
import sys, os
import io

reload(sys)
sys.setdefaultencoding('utf8')
black_dict={"\\":"","\"":""}
def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--cols', type=int, dest='cols', action='store', default=-1,help="count of columns,default first line's cells")
    parser.add_argument('--src', type=str, dest='src', action='store', default='',
                        help='path to source csv file')
    parser.add_argument('--dest', type=str, dest='dest', action='store', default='',
                        help='path to dest csv file')
    parser.add_argument('--encoding', type=str, dest='encoding', action='store', default='utf-8',
                        help='file encoding,default utf-8')
    parser.add_argument('--chunksize', type=int, dest='chunksize', action='store', default='10000',
                        help='batch lines to write dest file,default 10000')
    parser.add_argument('--delimiter', type=str, dest='delimiter', action='store', default=',',
                        help='csv delimiter,default ,')

    args = parser.parse_args()
    cols = args.cols
    src = args.src
    dest = args.dest
    encoding = args.encoding
    chunksize = args.chunksize
    delimiter = args.delimiter

    if not (src and dest) or chunksize <= 0:
      print("invaild args!")
      sys.exit(-1)

    olds=[]
    lines=[]
    with io.open(src,encoding=encoding) as fp:
      for line in fp.readlines():
        line = line.strip()
        for k,v in black_dict.items():
          if k in line:
            line=line.replace(k,v)
        cells = line.split(delimiter)
        if cols == -1:
          cols=len(cells)

        if(len(cells) < cols or (len(olds)>0 and len(olds) < cols)):
          if not olds:
            olds = cells
          else:
            cells[0]=olds[-1]+cells[0]
            olds.pop()
            olds.extend(cells)

        if len(olds) >= cols:
          cells=olds
          olds=[]
        if not olds:
          lines.append(delimiter.join(cells[0:cols])+"\n")

        if len(lines) % chunksize == 0:
          write_to_file(dest=dest,lines=lines)
          lines=[]

      write_to_file(dest=dest,lines=lines)

def write_to_file(dest,lines=[],encoding='utf-8'):
  p = os.path.split(dest)[0]
  if not os.path.exists(p):
    os.makedirs(p)
  with io.open(file=dest,mode="a+",encoding=encoding) as fp:
    fp.writelines(lines)

if __name__ == '__main__':
    main()
```

使用方式
`python clean_csv.py --src=src.csv --dest=dest.csv --chunksize=50000 --cols --encoding=utf-8 --delimiter=,`

## 参考资料

- [我的博客](https://anjia0532.github.io/2019/07/16/clean-csv/)
- [我的掘金](https://juejin.im/post/5d2ff38b51882526ae230186)
- [anjia0532/clean_csv.py](https://gist.github.com/anjia0532/6db48b0886d91d9a663e5a9fd19f2aaa)
