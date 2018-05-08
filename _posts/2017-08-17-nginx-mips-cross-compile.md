---
layout:     post
title:      "Nginx mips交叉编译总结"
date:       2017-08-17 10:27:52 +0800
categories: 经验总结
tag:        Nginx系列
---

*content
{: toc}

```sh
PREFIX=$PWD/install
CONF=$PWD/install/config/nginx.conf

mkdir -p $PREFIX

./configure --prefix=$PREFIX \
    --with-ipv6 \
    --with-debug \
    --without-http_rewrite_module \
    --without-http_gzip_module \
    --without-http_userid_module \
    --without-http_geo_module \
    --without-http_map_module \
    --without-http_split_clients_module \
    --without-http_referer_module \
    --without-http_fastcgi_module \
    --without-http_uwsgi_module \
    --without-http_scgi_module \
    --without-http_memcached_module \
    --without-http_limit_conn_module \
    --without-http_limit_req_module \
    --without-http_empty_gif_module \
    --without-http_browser_module \
    --without-http-cache \
    --without-http_upstream_hash_module \
    --without-http_upstream_ip_hash_module \
    --without-http_upstream_least_conn_module \
    --without-http_upstream_keepalive_module \
    --without-http_upstream_zone_module \
    --without-mail_pop3_module \
    --without-mail_imap_module \
    --without-mail_smtp_module \
    --without-stream_limit_conn_module \
    --without-stream_access_module \
    --without-stream_map_module \
    --without-stream_return_module \
    --without-stream_upstream_hash_module \
    --without-stream_upstream_least_conn_module \
    --without-stream_upstream_zone_module \
    --without-pcre \
    --without-http_rewrite_module \
    --with-cc=/opt/buildroot-gcc463/usr/bin/mipsel-linux-gcc \
    --with-cpp=/opt/buildroot-gcc463/usr/bin/mipsel-linux-g++ \
    --without-pcre \
    --with-cc-opt="-I /opt/buildroot-gcc463/usr/include -std=c99 -DNGX_HAVE_SYSVSHM=1 -DNGX_SYS_NERR=132" \
    --with-ld-opt="-L /opt/buildroot-gcc463/usr/lib"```

问题1
---
```
checking for C compiler ... found but is not working

./configure: error: C compiler /opt/buildroot-gcc463/usr/bin/mipsel-linux-gcc is not found
```
解决：编辑auto/cc/name文件，将21行`exit 1`注释掉


问题2
---
```
autotest : 4 : not found 
autotest:Syntax error: Unterminated quoted string bytes
./configure: error: can not detect int size

```
解决：编辑auto/types/sizeof文件，大概36行的位置（ $CC 改为 gcc ）
~~ngx_test="$CC $CC_TEST_FLAGS $CC_AUX_FLAGS~~
~~改为ngx_test="gcc $CC_TEST_FLAGS $CC_AUX_FLAGS~~

将eval "$ngx_test >> $NGX_AUTOCONF_ERR 2>&1"注释掉
（之前改gcc会有问题，在64bit的linux虚拟机中，会将long sizeof置为8，导致在32bit上的程序overflow，所以改为不执行ngx_test）


问题3
---
```
make: Nothing to be done for 'default'.
```
解决：主目录中不能有build的可执行文件或目录


问题4
---
```
src/core/ngx_string.c: In function 'ngx_atoi':
src/core/ngx_string.c:910:5: error: overflow in implicit constant conversion [-Werror=overflow]
```
解决：此问题是在64bit上虚拟机交叉编译导致，解决方法同问题2


问题5
---
```
src/os/unix/ngx_errno.c : In function'ngx_strerror':
src/os/unix/ngx_errno.c : 37 : 31 :error:'NGX_SYS_NERR' undeclared (first use in this function)
```
解决:`.configure --with-cc-opt=-DNGX_SYS_NERR=132`


问题6
---
```
`ngx_shm_free'函数未定义
```
解决：`.configure -DNGX_HAVE_SYSVSHM=1`


问题7
---
```
目标平台支持epoll，但是`epoll ... found but is not working`,autoconfig不会配置epoll
```
解决方法1:`.configure -DNGX_HAVE_EPOLLRDHUP=1` （改方法未经过我验证）

解决方法2:编辑./auto/os/linux 文件，找到epoll部分,改为`ngx_feature_run=no`,并且注释掉`ngx_feature_test=`后面的内容(不让其运行测试epoll)
