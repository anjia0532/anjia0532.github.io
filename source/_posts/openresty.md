---
title: OpenResty编译安装以及安全加固(WAF)
date: 2017-07-19 16:04:57
tags: [nginx,openresty,waf,firewall]
---

## [Nginx][] 还是[Tengine][Tengine]

Tengine是阿里巴巴的深度定制的nginx，目前最新版本[Tengine-2.2.0.tar.gz][] , 继承了nginx 1.8.1的所有特性，并且兼容nginx的配置，但是最后一次更新是`2016-12-02`截止到目前，已经半年多没更新了。https://github.com/alibaba/tengine 上已经有137条未关闭的issus和39条pull request

下面是官网自述

>Tengine是由淘宝网发起的Web服务器项目。它在Nginx的基础上，针对大访问量网站的需求，添加了很多高级功能和特性。Tengine的性能和稳定性已经在大型的网站如淘宝网，天猫商城等得到了很好的检验。它的最终目标是打造一个高效、稳定、安全、易用的Web平台。

但是鉴于阿里有很多看似不错的项目最后都人走政息的传统(KPI驱动的项目),比如 微服务框架[dubbo][] 长期不维护，后来被坑的几家(当当，韩都衣舍)为了自身需要，又在他基础上搞了[dubbox][], 淘宝家的玉伯的[seajs][]

而且Tengine还不支持Windows,网上文档比nginx少很多，所以如无特殊必要，还是建议用nginx。

nginx 最新主线版本1.13.3，稳定版本1.12.1，基本保持1月一更甚至3更的频率，响应很快，堪称版本帝，可以参考[CHANGES][]和[security][]来考虑是否有必要升级

如果不差钱，可以考虑一下 `nginx plus` ,价格很感人，[Pricing - Application Delivery for the Modern Web | NGINX][pricing]


## nginx

**本文主要讲解openresty编译安装以及加固，对于nginx只做简单描述。**

nginx本身提供编译好的二进制文件，linux的参见 [prebuilt][] ，windows的从 [download][] 下载`nginx/Windows-VERSION`相关`zip`包，建议生产环境使用 `stable`(稳定版本)

如果是想从源码编译的话，从[download][] 下载`nginx-VERSION` 的`tar.gz`包，一般格式为`http://nginx.org/download/nginx-VERSION.tar.gz` 注意将`VERSION`替换成实际版本号，e.g. `1.12.1`


### nginx编译参数

ubuntu nginx 默认编译参数如下(为了便于阅读，将一行的编译参数展开成多行),`nginx -V`是查看构建参数，`nginx -v`是查看版本号
```bash
/usr/sbin/nginx -V

nginx version: nginx/1.13.0
built by gcc 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.2) 
built with OpenSSL 1.0.2g-fips  1 Mar 2016 (running with OpenSSL 1.0.2g  1 Mar 2016)
TLS SNI support enabled
configure arguments: 
--prefix=/etc/nginx  \
--sbin-path=/usr/sbin/nginx  \
--modules-path=/usr/lib/nginx/modules  \
--conf-path=/etc/nginx/nginx.conf  \
--error-log-path=/var/log/nginx/error.log  \
--http-log-path=/var/log/nginx/access.log  \
--pid-path=/var/run/nginx.pid  \
--lock-path=/var/run/nginx.lock  \
--http-client-body-temp-path=/var/cache/nginx/client_temp  \
--http-proxy-temp-path=/var/cache/nginx/proxy_temp  \
--http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp  \
--http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp  \
--http-scgi-temp-path=/var/cache/nginx/scgi_temp  \
--user=nginx  \
--group=nginx  \
--with-compat  \
--with-file-aio  \
--with-threads  \
--with-http_addition_module  \
--with-http_auth_request_module  \
--with-http_dav_module  \
--with-http_flv_module  \
--with-http_gunzip_module  \
--with-http_gzip_static_module  \
--with-http_mp4_module  \
--with-http_random_index_module  \
--with-http_realip_module  \
--with-http_secure_link_module  \
--with-http_slice_module  \
--with-http_ssl_module  \
--with-http_stub_status_module  \
--with-http_sub_module  \
--with-http_v2_module  \
--with-mail  \
--with-mail_ssl_module  \
--with-stream  \
--with-stream_realip_module  \
--with-stream_ssl_module  \
--with-stream_ssl_preread_module  \
--with-cc-opt='-g -O2 -fstack-protector-strong -Wformat -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -fPIC'  \
--with-ld-opt='-Wl,-Bsymbolic-functions -Wl,-z,relro -Wl,-z,now -Wl,--as-needed -pie'
```

具体参数含义可 参考官网 [Building nginx from Sources][linkBuildingNginxFromSources] 和 [梦想远航#nginx安装及编译参数详解][5225895], 如果要构建`nginx for windows`参见 [Building nginx on the Win32 platform with Visual C][linkBuildingNginxOnTheWin32Platform]

### 编译nginx
参考 官方文档 [INSTALLING NGINX OPEN SOURCE][linkInstallingNginxOpenSource]

```bash
# pcre 正则库 
$ wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.41.tar.gz
$ tar -zxf pcre-*.tar.gz
$ cd pcre-*
$ ./configure
$ make && sudo make install

# zlib gzip 库
$ wget http://zlib.net/zlib-1.2.11.tar.gz
$ tar -zxf zlib-1.2.11.tar.gz
$ cd zlib-1.2.11
$ ./configure
$ make && sudo make install

# openssl https库 注意官网代码是mac编译，建议如果失败，搜索一下openssl 编译 

$ wget https://www.openssl.org/source/openssl-1.0.2l.tar.gz
$ tar -zxf openssl-*.tar.gz
$ cd openssl-*
$ ./config --prefix=/usr/local/openssl/
$ make && sudo make install


#主线和稳定二选一
# 主线版本
$ wget http://nginx.org/download/nginx-1.13.3.tar.gz

#稳定版本
$ wget http://nginx.org/download/nginx-1.12.1.tar.gz

$ tar zxf nginx-*.tar.gz

$ cd nginx-*

$ ./configure --prefix=/etc/nginx  \
--sbin-path=/usr/sbin/nginx  \
--modules-path=/usr/lib/nginx/modules  \
--conf-path=/etc/nginx/nginx.conf  \
--error-log-path=/var/log/nginx/error.log  \
--http-log-path=/var/log/nginx/access.log  \
--pid-path=/var/run/nginx.pid  \
--lock-path=/var/run/nginx.lock  \

--with-http_gunzip_module  \
--with-http_gzip_static_module  \

--with-http_addition_module  \
--with-http_auth_request_module  \
--with-http_realip_module  \
--with-http_slice_module  \
--with-http_stub_status_module  \
--with-http_sub_module  \
--with-compat  \
--with-file-aio  \
--with-threads  \

--with-stream  \
--with-stream_realip_module  \
--with-stream_ssl_module  \
--with-stream_ssl_preread_module  \

--with-http_v2_module  \
--with-http_ssl_module  \

--with-pcre=../pcre-8.41  \
--with-zlib=../zlib-1.2.11 \

--without-http_autoindex_module \
--without-http_fastcgi_module \
--without-http_uwsgi_module \
--without-http_scgi_module \
--without-http_memcached_module \
--without-http_empty_gif_module

$ make && sudo make install

# 从官方标准参数中去除不用的模块，并新增了pcre和zlib模块
# 临时文件相关
#--http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp  \
#--http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp  \
#--http-scgi-temp-path=/var/cache/nginx/scgi_temp  \
#--http-client-body-temp-path=/var/cache/nginx/client_temp  \
#--http-proxy-temp-path=/var/cache/nginx/proxy_temp  \

# dav，媒体相关
#--with-http_dav_module  \
#--with-http_flv_module  \
#--with-http_mp4_module  \

#随机首页，安全连接相关
#--with-http_random_index_module  \
#--with-http_secure_link_module  \

#email相关
#--with-mail  \
#--with-mail_ssl_module  \

#gcc相关
#--with-cc-opt='-g -O2 -fstack-protector-strong -Wformat -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -fPIC'  \
#--with-ld-opt='-Wl,-Bsymbolic-functions -Wl,-z,relro -Wl,-z,now -Wl,--as-needed -pie'

#组，用户相关
#--user=nginx 
#--group=nginx 
#如果指定user和group 则通过此命令创建用户
#$ sudo adduser --system --no-create-home --shell /bin/false --group --disabled-login nginx

#如果用不到https，可以把ssl和http2模块也禁掉

#禁用未用模块，减少安全风险
#--without-http_autoindex_module \
#--without-http_fastcgi_module \
#--without-http_uwsgi_module \
#--without-http_scgi_module \
#--without-http_memcached_module \
#--without-http_empty_gif_module
$ nginx -t && nginx

nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

```

至此nginx编译完成。可以通过`curl localhost`或者浏览器打开`localhost` 查看nginx默认页面

### nginx服务脚本
```bash
$ vi /etc/init.d/nginx

#!/bin/sh
### BEGIN INIT INFO
# Provides:          nginx
# Required-Start:    $network $remote_fs $local_fs 
# Required-Stop:     $network $remote_fs $local_fs
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Stop/start nginx
### END INIT INFO

# Author: Sergey Budnevitch <sb@nginx.com>

PATH=/sbin:/usr/sbin:/bin:/usr/bin

if [ -L $0 ]; then
    SCRIPTNAME=`/bin/readlink -f $0`
else
    SCRIPTNAME=$0
fi

sysconfig=`/usr/bin/basename $SCRIPTNAME`

[ -r /etc/default/$sysconfig ] && . /etc/default/$sysconfig

DESC=${DESC:-nginx}
NAME=${NAME:-nginx}
CONFFILE=${CONFFILE:-/etc/nginx/nginx.conf}
DAEMON=${DAEMON:-/usr/sbin/nginx}
PIDFILE=${PIDFILE:-/var/run/nginx.pid}
SLEEPSEC=${SLEEPSEC:-1}
UPGRADEWAITLOOPS=${UPGRADEWAITLOOPS:-5}
CHECKSLEEP=${CHECKSLEEP:-3}

[ -x $DAEMON ] || exit 0

DAEMON_ARGS="-c $CONFFILE $DAEMON_ARGS"

. /lib/init/vars.sh

. /lib/lsb/init-functions

do_start()
{
    start-stop-daemon --start --quiet --pidfile $PIDFILE --exec $DAEMON -- \
        $DAEMON_ARGS
    RETVAL="$?"
    return "$RETVAL"
}

do_stop()
{
    # Return
    #   0 if daemon has been stopped
    #   1 if daemon was already stopped
    #   2 if daemon could not be stopped
    #   other if a failure occurred
    start-stop-daemon --stop --quiet --oknodo --retry=TERM/30/KILL/5 --pidfile $PIDFILE
    RETVAL="$?"
    rm -f $PIDFILE
    return "$RETVAL"
}

do_reload() {
    #
    start-stop-daemon --stop --signal HUP --quiet --pidfile $PIDFILE
    RETVAL="$?"
    return "$RETVAL"
}

do_configtest() {
    if [ "$#" -ne 0 ]; then
        case "$1" in
            -q)
                FLAG=$1
                ;;
            *)
                ;;
        esac
        shift
    fi
    $DAEMON -t $FLAG -c $CONFFILE
    RETVAL="$?"
    return $RETVAL
}

do_upgrade() {
    OLDBINPIDFILE=$PIDFILE.oldbin

    do_configtest -q || return 6
    start-stop-daemon --stop --signal USR2 --quiet --pidfile $PIDFILE
    RETVAL="$?"
    
    for i in `/usr/bin/seq  $UPGRADEWAITLOOPS`; do
        sleep $SLEEPSEC
        if [ -f $OLDBINPIDFILE -a -f $PIDFILE ]; then
            start-stop-daemon --stop --signal QUIT --quiet --pidfile $OLDBINPIDFILE
            RETVAL="$?"
            return
        fi
    done

    echo $"Upgrade failed!"
    RETVAL=1
    return $RETVAL
}

do_checkreload() {
    templog=`/bin/mktemp --tmpdir nginx-check-reload-XXXXXX.log`
    trap '/bin/rm -f $templog' 0
    /usr/bin/tail --pid=$$ -n 0 --follow=name /var/log/nginx/error.log > $templog &
    /bin/sleep 1
    start-stop-daemon --stop --signal HUP --quiet --pidfile $PIDFILE
    /bin/sleep $CHECKSLEEP
    /bin/grep -E "\[emerg\]|\[alert\]" $templog
}

case "$1" in
    start)
        [ "$VERBOSE" != no ] && log_daemon_msg "Starting $DESC " "$NAME"
        do_start
        case "$?" in
            0|1) [ "$VERBOSE" != no ] && log_end_msg 0 ;;
            2) [ "$VERBOSE" != no ] && log_end_msg 1 ;;
        esac
        ;;
    stop)
        [ "$VERBOSE" != no ] && log_daemon_msg "Stopping $DESC" "$NAME"
        do_stop
        case "$?" in
            0|1) [ "$VERBOSE" != no ] && log_end_msg 0 ;;
            2) [ "$VERBOSE" != no ] && log_end_msg 1 ;;
        esac
        ;;
  status)
        status_of_proc -p "$PIDFILE" "$DAEMON" "$NAME" && exit 0 || exit $?
        ;;
  configtest)
        do_configtest
        ;;
  upgrade)
        do_upgrade
        ;;
  reload|force-reload)
        log_daemon_msg "Reloading $DESC" "$NAME"
        do_reload
        log_end_msg $?
        ;;
  restart|force-reload)
        log_daemon_msg "Restarting $DESC" "$NAME"
        do_configtest -q || exit $RETVAL
        do_stop
        case "$?" in
            0|1)
                do_start
                case "$?" in
                    0) log_end_msg 0 ;;
                    1) log_end_msg 1 ;; # Old process is still running
                    *) log_end_msg 1 ;; # Failed to start
                esac
                ;;
            *)
                # Failed to stop
                log_end_msg 1
                ;;
        esac
        ;;
    check-reload)
        do_checkreload
        RETVAL=0
        ;;
    *)
        echo "Usage: $SCRIPTNAME {start|stop|status|restart|reload|force-reload|upgrade|configtest|check-reload}" >&2
        exit 3
        ;;
esac

exit $RETVAL

```

## openresty

学习资料 官网 [openresty][] ，开涛博客 [使用Nginx+Lua(OpenResty)开发高性能Web应用][] ，温铭的gitbook [OpenResty 最佳实践][linkOpenresty最佳实践] 温铭的stuq视频教程[OpenResty 系列课程][linkOpenresty系列课程]

### 安装预编译包

详见官方文档 [OpenResty® Linux 包][linkOpenresty®Linux包]
[linkOpenresty®Linux包]: https://openresty.org/cn/linux-packages.html


### openresty编译参数
ubuntu openresty 默认编译参数如下(为了便于阅读，将一行的编译参数展开成多行),`resty -V`是查看构建参数，`resty -v`是查看版本号

[openresty官方组件][] ,[nginx 模块][linkNginx模块]

```bash
nginx version: openresty/1.11.2.4
built with OpenSSL 1.0.2k  26 Jan 2017
TLS SNI support enabled
configure arguments: 
--prefix=/usr/local/openresty/nginx \
--with-cc-opt='-O2 -I/usr/local/openresty/zlib/include -I/usr/local/openresty/pcre/include -I/usr/local/openresty/openssl/include' \
--add-module=../ngx_devel_kit-0.3.0 \
--add-module=../echo-nginx-module-0.60 \
--add-module=../xss-nginx-module-0.05 \
--add-module=../ngx_coolkit-0.2rc3 \
--add-module=../set-misc-nginx-module-0.31 \
--add-module=../form-input-nginx-module-0.12 \
--add-module=../encrypted-session-nginx-module-0.06 \
--add-module=../srcache-nginx-module-0.31 \
--add-module=../ngx_lua-0.10.8 \
--add-module=../ngx_lua_upstream-0.06 \
--add-module=../headers-more-nginx-module-0.32 \
--add-module=../array-var-nginx-module-0.05 \
--add-module=../memc-nginx-module-0.18 \
--add-module=../redis2-nginx-module-0.14 \
--add-module=../redis-nginx-module-0.3.7 \
--with-ld-opt='-Wl,-rpath,/usr/local/openresty/luajit/lib -L/usr/local/openresty/zlib/lib -L/usr/local/openresty/pcre/lib -L/usr/local/openresty/openssl/lib -Wl,-rpath,/usr/local/openresty/zlib/lib:/usr/local/openresty/pcre/lib:/usr/local/openresty/openssl/lib' \
--with-pcre-jit \
--with-ipv6 \
--with-stream \
--with-stream_ssl_module \
--with-http_v2_module \
--without-mail_pop3_module \
--without-mail_imap_module \
--without-mail_smtp_module \
--with-http_stub_status_module \
--with-http_realip_module \
--with-http_addition_module \
--with-http_auth_request_module \
--with-http_secure_link_module \
--with-http_random_index_module \
--with-http_gzip_static_module \
--with-http_sub_module \
--with-http_dav_module \
--with-http_flv_module \
--with-http_mp4_module \
--with-http_gunzip_module \
--with-threads \
--with-file-aio \
--with-dtrace-probes \
--with-http_ssl_module
```

### 构建openresty

参见 [构建openresty][]

```bash
$ sudo apt-get install -y libreadline-dev libncurses5-dev libpcre3-dev libssl-dev perl make build-essential
$ wget https://openresty.org/download/openresty-1.11.2.4.tar.gz
$ tar zxf openresty-1.11.2.4.tar.gz
$ cd openresty-*

# 查看所有编译参数
$ ./configure --help 

#进行编译
./configure --prefix=/etc/openresty \
--user=nginx \
--group=nginx \
--with-cc-opt='-O2 -I/usr/local/openresty/zlib/include -I/usr/local/openresty/pcre/include -I/usr/local/openresty/openssl/include' \
--with-ld-opt='-Wl,-rpath,/usr/local/openresty/luajit/lib -L/usr/local/openresty/zlib/lib -L/usr/local/openresty/pcre/lib -L/usr/local/openresty/openssl/lib -Wl,-rpath,/usr/local/openresty/zlib/lib:/usr/local/openresty/pcre/lib:/usr/local/openresty/openssl/lib' \
--with-pcre-jit \
--with-stream \
--with-stream_ssl_module \
--with-http_v2_module \
--with-http_stub_status_module \
--with-http_realip_module \
--with-http_gzip_static_module \
--with-http_sub_module \
--with-http_gunzip_module \
--with-threads \
--with-file-aio \
--with-http_ssl_module \
--with-http_auth_request_module \
--without-mail_pop3_module \
--without-mail_imap_module \
--without-mail_smtp_module \
--without-http_fastcgi_module \
--without-http_uwsgi_module \
--without-http_scgi_module \
--without-http_autoindex_module \
--without-http_memcached_module \
--without-http_empty_gif_module \
--without-http_ssi_module \
--without-http_userid_module \
--without-http_browser_module \
--without-http_rds_json_module \
--without-http_rds_csv_module \
--without-http_memc_module \
--without-http_redis2_module \
--without-lua_resty_memcached \
--without-lua_resty_mysql \
-j4

#禁用memcached模块
#--without-http_memc_module \
#禁用redis模块(保留redis2模块)
#--without-http_redis_module \
#禁用email相关模块
#--without-mail_pop3_module \
#--without-mail_imap_module \
#--without-mail_smtp_module \
#禁用rds模块
#--without-http_rds_json_module \
#--without-http_rds_csv_module  \
#禁用cgi 
#--without-http_fastcgi_module \
#--without-http_uwsgi_module \
#--without-http_scgi_module \
#--without-http_autoindex_module \
#--without-http_memcached_module \
#--without-http_empty_gif_module \
$ make -j4 && sudo make install

#确保80端口没被占用
$ lsof -i:80

$ /opt/openresty/nginx/nginx/sbin/nginx -t && /opt/openresty/nginx/nginx/sbin/nginx

$ curl localhost
```

### openresty 服务脚本

```bash
$ vi /etc/init.d/openresty 
#!/bin/sh

### BEGIN INIT INFO
# Provides:   openresty
# Required-Start:    $local_fs $remote_fs $network $syslog $named
# Required-Stop:     $local_fs $remote_fs $network $syslog $named
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: starts openresty
# Description:       starts openresty using start-stop-daemon
### END INIT INFO

PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
DAEMON=/usr/local/openresty/nginx/sbin/nginx
NAME=nginx
DESC="full-fledged web platform"

if [ -r /etc/default/openresty ]; then
        . /etc/default/openresty
fi

STOP_SCHEDULE="${STOP_SCHEDULE:-QUIT/5/TERM/5/KILL/5}"

test -x $DAEMON || exit 0

. /lib/init/vars.sh
. /lib/lsb/init-functions

PID=$(cat /usr/local/openresty/nginx/conf/nginx.conf | grep -Ev '^\s*#' | awk 'BEGIN { RS="[;{}]" } { if ($1 == "pid") print $2 }' | head -n1)
if [ -z "$PID" ]; then
    PID=/usr/local/openresty/nginx/logs/nginx.pid
fi

start_nginx() {
    # Start the daemon/service
    #
    # Returns:
    #   0 if daemon has been started
    #   1 if daemon was already running
    #   2 if daemon could not be started
    start-stop-daemon --start --quiet --pidfile $PID --exec $DAEMON --test > /dev/null \
        || return 1
    start-stop-daemon --start --quiet --pidfile $PID --exec $DAEMON -- \
        $DAEMON_OPTS 2>/dev/null \
        || return 2
}

test_config() {
    $DAEMON -q -t $DAEMON_OPTS >/dev/null 2>&1
}

stop_nginx() {
    # Stops the daemon/service
    #
    # Return
    #   0 if daemon has been stopped
    #   1 if daemon was already stopped
    #   2 if daemon could not be stopped
    #   other if a failure occurred
    start-stop-daemon --stop --quiet --retry=$STOP_SCHEDULE --pidfile $PID --name $NAME
    RETVAL="$?"
    sleep 1
    return "$RETVAL"
}

reload_nginx() {
    # Function that sends a SIGHUP to the daemon/service
    start-stop-daemon --stop --signal HUP --quiet --pidfile $PID --name $NAME
    return 0
}

rotate_logs() {
    # Rotate log files
    start-stop-daemon --stop --signal USR1 --quiet --pidfile $PID --name $NAME
    return 0
}

upgrade_nginx() {
    # Online upgrade nginx executable
    # http://nginx.org/en/docs/control.html
    #
    # Return
    #   0 if nginx has been successfully upgraded
    #   1 if nginx is not running
    #   2 if the pid files were not created on time
    #   3 if the old master could not be killed
    if start-stop-daemon --stop --signal USR2 --quiet --pidfile $PID --name $NAME; then
        # Wait for both old and new master to write their pid file
        while [ ! -s "${PID}.oldbin" ] || [ ! -s "${PID}" ]; do
            cnt=`expr $cnt + 1`
            if [ $cnt -gt 10 ]; then
                return 2
            fi
            sleep 1
        done
        # Everything is ready, gracefully stop the old master
        if start-stop-daemon --stop --signal QUIT --quiet --pidfile "${PID}.oldbin" --name $NAME; then
            return 0
        else
            return 3
        fi
    else
        return 1
    fi
}

case "$1" in
    start)
        log_daemon_msg "Starting $DESC" "$NAME"
        start_nginx
        case "$?" in
            0|1) log_end_msg 0 ;;
            2)   log_end_msg 1 ;;
        esac
        ;;
    stop)
        log_daemon_msg "Stopping $DESC" "$NAME"
        stop_nginx
        case "$?" in
            0|1) log_end_msg 0 ;;
            2)   log_end_msg 1 ;;
        esac
        ;;
    restart)
        log_daemon_msg "Restarting $DESC" "$NAME"

        # Check configuration before stopping nginx
        if ! test_config; then
            log_end_msg 1 # Configuration error
            exit $?
        fi

        stop_nginx
        case "$?" in
            0|1)
                start_nginx
                case "$?" in
                    0) log_end_msg 0 ;;
                    1) log_end_msg 1 ;; # Old process is still running
                    *) log_end_msg 1 ;; # Failed to start
                esac
                ;;
            *)
                # Failed to stop
                log_end_msg 1
                ;;
        esac
        ;;
    reload|force-reload)
        log_daemon_msg "Reloading $DESC configuration" "$NAME"

        # Check configuration before stopping nginx
        #
        # This is not entirely correct since the on-disk nginx binary
        # may differ from the in-memory one, but that's not common.
        # We prefer to check the configuration and return an error
        # to the administrator.
        if ! test_config; then
            log_end_msg 1 # Configuration error
            exit $?
        fi

        reload_nginx
        log_end_msg $?
        ;;
    configtest|testconfig)
        log_daemon_msg "Testing $DESC configuration"
        test_config
        log_end_msg $?
        ;;
    status)
        status_of_proc -p $PID "$DAEMON" "$NAME" && exit 0 || exit $?
        ;;
    upgrade)
        log_daemon_msg "Upgrading binary" "$NAME"
        upgrade_nginx
        log_end_msg $?
        ;;
    rotate)
        log_daemon_msg "Re-opening $DESC log files" "$NAME"
        rotate_logs
        log_end_msg $?
        ;;
    *)
        echo "Usage: $NAME {start|stop|restart|reload|force-reload|status|configtest|rotate|upgrade}" >&2
        exit 3
        ;;
esac

```


## WAF 基于[ModSecurity][]

参考资料 [Ubuntu 15.04][linkUbuntu15.04]

```bash
$ git clone -b v3/master --single-branch https://github.com/SpiderLabs/ModSecurity.git --depth=1
$ cd ModSecurity/
$ git checkout -b v3/master origin/v3/master
$ sh build.sh
$ git submodule init
$ git submodule update #[for bindings/python, others/libinjection, test/test-cases/secrules-language-tests]
$ ./configure
$ make
$ sudo make install

#使用 ModSecurity-nginx 而不是网上流传的独立版 详见 https://github.com/SpiderLabs/ModSecurity-nginx

$ export MODSECURITY_INC="/home/anjia/openresty/ModSecurity/headers"
$ export MODSECURITY_LIB="/home/anjia/openresty/ModSecurity/src/.libs"
$ git clone https://github.com/SpiderLabs/ModSecurity-nginx --depth=1
$ git clone https://github.com/SpiderLabs/owasp-modsecurity-crs.git --depth=1
$ sudo cp -R owasp-modsecurity-crs/rules /opt/openresty/nginx/nginx/conf 
$ cp owasp-modsecurity-crs/crs-setup.conf.example /opt/openresty/nginx/nginx/conf/crs-setup.conf
$ sudo wget -P /opt/openresty/nginx/nginx/conf https://raw.githubusercontent.com/SpiderLabs/ModSecurity/master/modsecurity.conf-recommended h
ttps://raw.githubusercontent.com/SpiderLabs/ModSecurity/master/unicode.mapping
$ sudo mv /opt/openresty/nginx/nginx/conf/modsecurity.conf-recommended /opt/openresty/nginx/nginx/conf/modsecurity.conf
$ sudo mkdir /opt/openresty/nginx/nginx/conf/sites-enabled
#使用www-data用户
$ sudo sed -i '1s/^/user www-data;\n/' /opt/openresty/nginx/nginx/conf/nginx.conf
$ sudo vim /opt/openresty/nginx/nginx/conf/nginx.conf
#删除36-116行，即server{}段，可以在英文输入法状态按     :36,166d  然后 :wq
#如果确认行数没问题，也可以用sudo sed '35,116d' -i /opt/openresty/nginx/nginx/conf/nginx.conf
$ sudo sed '$i include /opt/openresty/nginx/nginx/conf/sites-enabled/*; ' -i /opt/openresty/nginx/nginx/conf/nginx.conf
#嫌费事，也可以直接用下面的配置文件
user www-data;
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    include /opt/openresty/nginx/nginx/conf/sites-enabled/*;
}

$ vi /opt/openresty/nginx/nginx/conf/modsecurity.conf

#Load OWASP Config 
Include crs-setup.conf 
#Load all other Rules 
Include rules/*.conf 
#Disable rule by ID from error message 
#SecRuleRemoveById 920350

$ sudo sed  s/"SecRuleEngine DetectionOnly"/"SecRuleEngine On"/g -i /opt/openresty/nginx/nginx/conf/modsecurity.conf
$ sudo /opt/openresty/nginx/nginx/sbin/nginx -t && sudo /opt/openresty/nginx/nginx/sbin/nginx -s reload
$ curl "http://localhost/wp-admin/admin.php?where1=%3Cscript%3Ealert(String.fromCharCode(88,+83,+83))%3C/script%3E&searchsubmit=Buscar&page=nsp_search"
# 返回403 Forbidden
```

[Nginx]: http://nginx.org/
[Tengine]: http://tengine.taobao.org/
[Tengine-2.2.0.tar.gz]: http://tengine.taobao.org/download/tengine-2.2.0.tar.gz
[dubbo]: http://dubbo.io/
[dubbox]: https://github.com/dangdangdotcom/dubbox
[seajs]: https://github.com/seajs/seajs
[change]: http://nginx.org/en/CHANGES
[security]: http://nginx.org/en/security_advisories.html
[prebuilt]: https://www.nginx.com/resources/admin-guide/installing-nginx-open-source/#prebuilt
[download]: http://nginx.org/en/download.html
[linkBuildingNginxFromSources]: http://nginx.org/en/docs/configure.html
[pricing]: https://www.nginx.com/products/pricing/
[5225895]: http://www.cnblogs.com/HKUI/p/5225895.html
[linkBuildingNginxOnTheWin32Platform]: http://nginx.org/en/docs/howto_build_on_win32.html
[linkInstallingNginxOpenSource]: https://www.nginx.com/resources/admin-guide/installing-nginx-open-source/
[openresty]: https://openresty.org/cn/
[使用Nginx+Lua(OpenResty)开发高性能Web应用]: http://jinnianshilongnian.iteye.com/blog/2280928
[linkOpenresty最佳实践]: https://www.gitbook.com/book/moonbingbing/openresty-best-practices/details
[构建openresty]: https://openresty.org/cn/installation.html#-openresty
[linkOpenresty系列课程]: http://www.stuq.org/course/1015
[ModSecurity]: https://github.com/SpiderLabs/ModSecurity
[linkUbuntu15.04]: https://github.com/SpiderLabs/ModSecurity/wiki/Compilation-recipes#ubuntu-1504
[openresty官方组件]: https://openresty.org/en/components.html

[linkNginx模块]: 
