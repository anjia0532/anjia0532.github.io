---
title: nginx正则表达式快捷测试方法
date: 2017-06-29 16:03:19
tags: [nginx]
categories: [nginx]
---
之前在配置时都是本地起一个nginx服务，修改`location`规则,然后`nginx -s reload` 或则 `service nginx reload`不断尝试来判断是否符合预期。显而易见，效率极低。使用一些在线正则表达式测试(e.g. [在线工具](http://tool.oschina.net/regex/))又因为使用的库不同，多少存在差异。

<!-- more -->

正则表达式有不同的规则引擎，具体参见 wikipedia的 [Comparison of regular expression engines](https://en.wikipedia.org/wiki/Comparison_of_regular_expression_engines#Libraries)

nginx使用的是[PCRE](http://www.pcre.org/)

截取nginx官方文档 [Building nginx from Sources](http://nginx.org/en/docs/configure.html)
> --with-pcre=path — sets the path to the sources of the PCRE library. The library distribution (version 4.4 — 8.40) needs to be downloaded from the PCRE site and extracted. The rest is done by nginx’s ./configure and make. The library is required for regular expressions support in the location directive and for the ngx_http_rewrite_module module.

建议使用linux下的 `grep` 工具

windows可以使用[cygwin](http://www.cygwin.com/) 或者[git for windows](https://git-for-windows.github.io/)中的`git-bash.exe`

```bash
$ grep --help

# ...

Regexp selection and interpretation:
  -E, --extended-regexp     PATTERN is an extended regular expression (ERE)
  -F, --fixed-strings       PATTERN is a set of newline-separated strings
  -G, --basic-regexp        PATTERN is a basic regular expression (BRE)
  -P, --perl-regexp         PATTERN is a Perl regular expression
  -e, --regexp=PATTERN      use PATTERN for matching
  -f, --file=FILE           obtain PATTERN from FILE
  -i, --ignore-case         ignore case distinctions
  -w, --word-regexp         force PATTERN to match only whole words
  -x, --line-regexp         force PATTERN to match only whole lines
  -z, --null-data           a data line ends in 0 byte, not newline

# ...
```

使用 `grep -P`命令即可

```bash
$ echo 'a.gif' | grep -P '\.(jp?g|gif|bmp|png)'

#输出
a.gif

```

如果只想输出匹配部分，则加上`-o`参数

```bash
$ echo 'a.gif' | grep -P -o '\.(jp?g|gif|bmp|png)'

#输出
.gif
```

具体 perl 正则表达式语法，可参考

[Perl regular expressions man page](http://perldoc.perl.org/perlre.html)

[汤姆的猫-Perl入门（四）Perl的正则表达式](http://blog.csdn.net/sunshoupo211/article/details/31769837)


博客 [https://anjia.ml/2017/06/29/nginx-regex-test-way/][blog]
简书 [http://www.jianshu.com/p/17eb0ba22ff6][jianshu]
掘金 [https://juejin.im/post/5954ad1b5188250d8f602bca][juejin]

[jianshu]: http://www.jianshu.com/p/17eb0ba22ff6
[juejin]: https://juejin.im/post/5954ad1b5188250d8f602bca
[blog]: https://anjia.ml/2017/06/29/nginx-regex-test-way/