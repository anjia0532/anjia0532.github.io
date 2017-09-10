---
title: xshell 使用 Oh My ZSH home键 end键 小键盘区无效解决办法
date: 2017-09-10 16:38:41
tags: [zsh,xshell,oh-my-zsh]
---

zsh是一款超赞的shell工具，但是配置复杂，有个闲着没事的程序员，弄了一个开源项目 [robbyrussell/oh-my-zsh][] 截止目前，58.8k+ star就知道有多火了。比如 Spring boot在github才 15.7K+ spring framework 也16.1k+,最近火到炸天的tensorflow 69.4k

同样很优秀的有 [fisherman/fisherman][]

此文不讲如何安装，如何配置 。有此需求的同学，出门左转，找度娘解决。

此文主要解决，xshell 远程连接时，host将zsh设置成默认shell时，<kbd>Home</kbd>,<kbd>End</kbd>,小键盘区诸键无效问题

<!--more-->

参考连接 [Cannot using home/end key after install oh-my-zsh][linkCannotUsingHome/endKeyAfterInstall]

第一种方法也是最简单的办法是，修改xshell连接此host的终端类型，改成`linux`
![](http://ww1.sinaimg.cn/large/afaffa71ly1fjekq3ap0wj20fu0d3jrt.jpg)

但是注意 [@linlinlinlin ][] 所说，改用`linux`可能会导致shell的颜色比较奇怪。

`linux`

![](http://ww1.sinaimg.cn/large/afaffa71ly1fjel9lc8faj205706rglk.jpg)

`xterm`

![](http://ww1.sinaimg.cn/large/afaffa71ly1fjel9lchsfj207e09gaa4.jpg)


结合 [@candrew34][] 和 [@linlinlinlin ][]的回复，得出第二种方案

第二种方法，稍微复杂点

```bash
cat <<ENDOF >> ~/.zshrc
# Home
bindkey '\e[1~' beginning-of-line
# End
bindkey '\e[4~' end-of-line

# Keypad
# 0 . Enter
bindkey -s "^[Op" "0"
bindkey -s "^[Ol" "."
bindkey -s "^[OM" "^M"
# 1 2 3
bindkey -s "^[Oq" "1"
bindkey -s "^[Or" "2"
bindkey -s "^[Os" "3"
# 4 5 6
bindkey -s "^[Ot" "4"
bindkey -s "^[Ou" "5"
bindkey -s "^[Ov" "6"
# 7 8 9
bindkey -s "^[Ow" "7"
bindkey -s "^[Ox" "8"
bindkey -s "^[Oy" "9"
# + -  * /
bindkey -s "^[Ok" "+"
bindkey -s "^[Om" "-"
bindkey -s "^[Oj" "*"
bindkey -s "^[Oo" "/"
ENDOF

source ~/.zshrc
```


另附 [客户端putty, xshell连接linux中vim的小键盘问题][link客户端putty,Xshell连接linux中vim的小键盘问题]

[robbyrussell/oh-my-zsh]: https://github.com/robbyrussell/oh-my-zsh
[fisherman/fisherman]: https://github.com/fisherman/fisherman
[@linlinlinlin ]: https://github.com/linlinlinlin
[@candrew34]: https://github.com/candrew34
[linkCannotUsingHome/endKeyAfterInstall]: https://github.com/robbyrussell/oh-my-zsh/issues/3061#issuecomment-93494905
[link客户端putty,Xshell连接linux中vim的小键盘问题]: http://blog.csdn.net/jiedushi/article/details/6266944
