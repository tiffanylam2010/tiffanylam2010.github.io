关于Apache和mod_wsgi的部分笔记
======
最近在使用apache和flask的过程中，遇到一些疑惑，于是看了一些关于apache的mod_wsgi的介绍，并在虚拟机中做了些测试，以解决这些疑惑。没有记录详细的测试过程，只记录结论：

### 1. apache的MPM的两种模式: prefork & worker模式:

`prefork` 是多进程单线程, 每个进程处理一个客户端连接;

`worker` 是多进程多线程, 每个线程处理一个客户端连接;

在处理进程/线程满的情况下,其他请求,就会在等待队列等待空闲的进程/线程.由于看mod_wsgi的官方文档，里面只提到这2中模式都可以，所以也只考虑用这两种模式。

###  2. 如何查看当前用的是哪个模式:

```
root@vm:/etc/apache2/mods-enabled# apache2ctl -V
Server version: Apache/2.4.10 (Debian)
Server built:   Aug 28 2015 16:28:08
Server's Module Magic Number: 20120211:37
Server loaded:  APR 1.5.1, APR-UTIL 1.5.4
Compiled using: APR 1.5.1, APR-UTIL 1.5.4
Architecture:   64-bit
Server MPM:     worker
  threaded:     yes (fixed thread count)
    forked:     yes (variable process count)
```

###  3. 线程或进程的增长方式

无论用prefork还是worker模式, 进程增长方式一样. apache的进程增长方式: apache每秒检查一次, 发现需要增长时, 第一秒增加1个进程,第二秒增加2个进程,直到够; 增长的进程数=pow(2,n) n从0开始. 

`服务器启动时候` 启动配置中的StartServers个进程, 然后apache每秒检查一次, 如果当前进程数少于最小进程数, 则第一秒增加1个进程, 第二秒增加2个进程, 第三秒增加4个进程, 直到达到最小进程数.

`在并发比较大,当前进程数不够用时` apache也不是马上创建进程, 依然是每秒检查一次,第一次涨1个进程,第二次涨2个进程,直到涨满或符合需求. 所以有时候服务器很空闲，突然多几个请求，客户端等待的响应时间就比较长，正是因为apache处理进程增加需要时间。所以最小最大空闲进程数的设定很重要，在机器性能允许的情况下，可以稍微设置得大点。

`在并发高峰过去后` apache依然每秒检查一次, 发现空闲的进程/线程超过设定的最大空闲数量时,则开始清除多余的空闲进程/线程.

###  4. prefork 和 worker 的配置项意义:

```
<IfModule mpm_prefork_module>
    # apache启动时启动多少个进程
    StartServers      1

    # apache每秒检查一次,如果当前进程数少于MinSpareServers,
    # 则第一秒创建1个, 第二秒创建2个, 第三秒创建4个,..
    MinSpareServers   1

    # apache每秒检查一次,如果空闲进程数多于MaxSpareServers,
    # 也会逐渐kill掉空闲进程.
    MaxSpareServers  30

    # apache逐渐增长进程数时,不能超过此最大进程数,
    # 因此这是apache的并发处理数.
    MaxRequestWorkers     50

    # 一个进程处理多少个请求之后, 自动销毁,避免泄漏.
    # 设定为0则表示无限;
    # 建议设定一个相对大的数值, 因为进程的开销比较大,而且apache涨进程需要时间.
    MaxConnectionsPerChild 0

    # 最大进程数,MaxRequestWorkers不能超过此数值
    ServerLimit          100
</IfModule>
```

```
<IfModule mpm_worker_module>
    # apache启动时启动多少个进程
    StartServers           2

    # apache每秒检查一次,如果当前线程数少于MinSpareThreads,则创建进程
    # 进程数量遵循pow(2,n)规律;
    # 每个进程会创建ThreadsPerChild处理线程
    MinSpareThreads       25

    # apache每秒检查一次,
    # 如果空闲进程数多于MaxSpareServers,也会逐渐kill掉空闲线程和进程
    MaxSpareThreads       75

    # 每个进程将创建多少个线程
    ThreadsPerChild       25

    # 最多MaxRequestWorkers个线程,
    # MaxRequestWorkers/ThreadsPerChild=最大进程数,
    # MaxRequestWorkers则是apache处理的最大并发连接数
    MaxRequestWorkers    150

    # 一个进程处理多少个请求之后, 自动销毁,避免泄漏.
    # 设置为0则表示无限;
    # 建议设定一个相对大的数值, 因为进程的开销比较大,而且apache涨进程需要时间.
    MaxConnectionsPerChild 0

    # 最大的线程数限制,
    # MaxRequestWorkers不能超过此数值,
    # 修改此值需要重启apache,不能graceful
    ThreadLimit           64

    # 最大进程数,
    # MaxRequestWorkers/ThreadsPerChild=最大进程数, 不能超过此数值
    ServerLimit          100
</IfModule>
```

###  5. VirtualHost和apache处理进程或线程的关系

apache的处理进程和线程，是所有VirtualHost共享的。不能独自配置某个虚拟机分配几个处理进程或线程。（或者说，我还没找到配置的方法？）

###  6. 关于KeepAlive
KeepAlive会占用一个连接不释放,直到客户端关闭或服务端超时, 所以空闲的keepalive会降低服务端的并发数.一般情况下，不信任的客户端不直连apache，在前面加个nginx做代理。

###  7. 关于access log中的处理时间
apache的access log, 可以通过配置记录请求的处理时间. 该处理时间, 是apache分配到处理进程/线程之后到http请求全部发送回给客户端的时间, 不包括该请求中等待队列中的等待时间.

###  8. wsgi有2种模式: Embedded vs Daemon

在Embedded Mode下:
- wsgi application是在apache的进程或线程中运行;
- 进程进程数/线程数 是由apache的进程/线程扩展决定,数量比较不可控;

在Daemon Mode下:
- wsgi application是独立于apache的进程, apache的进程或线程只是做动态请求的转发代理; 静态文件依然由apache直接返回.
- wsgi application的进程和线程数是固定的;
- wsgi application进程/线程处理完请求后, 返回给apache进程/线程, 则此wsgi application进程/线程可以处理下一个请求; 而该apache的进程/线程,则需要等客户端收取完response才能释放出来处理下一个.
- wsgi application的线程是python的线程,受GIL影响, 所以Wsgi的Thread&Process数量配置:

> 如果是cpu型,则多process,少thread <br>
> 如果是io阻塞型,则多thread,少process

###  参考资料：

GrahamDumpleton 应该是mod_wsgi的作者吧, 他解释的关于wsgi的应该比较权威.

* http://www.slideshare.net/GrahamDumpleton/pycon-us-2013-making-apache-suck-less-for-hosting-python-web-applications
* http://stackoverflow.com/questions/7968840/apache-mod-wsgi-interaction
* http://blog.dscpl.com.au/

###  测试工具：

- 用monotor.sh 可监控apache的进程和线程数（会发现每个进程的线程数总是比配置的多2个，那2个非worker线程,他们具体做的事情,可以查看 GrahamDumpleton的blog ）

```bash
#!/bin/bash
cnt=1000
for ((i=1; i<=$cnt; i++))
do
    t=`date +'%Y-%m-%d %H:%M:%S'`
    processnum=`ps aux | grep "www-data.*apache2" | grep -v "grep" | wc -l`
    threadnum=`ps auxH | grep "www-data.*apache2" | grep -v "grep" | wc -l`
    echo $t $processnum $threadnum >> log.txt
    echo $t $processnum $threadnum
    sleep 0.5
done

```

- 用keepalive.py 可发起keepalive的请求；

```python
# coding: utf8
import requests
import time
import sys
url = sys.argv[1]
s = requests.Session()
print "get:", url
r = s.get(url)
print(r.text)
time.sleep(1000)
```

- 用startmany.sh 可发起很多并发请求；

```bash
#!/bin/bash
cnt=200
for ((i=1; i<=$cnt; i++))
do
    wget -b  http://192.168.232.128:4000/sleep
    # python keepalive.py http://192.168.232.128:4000/hello  &
done

```

- wsgi是测试用的wsgi

```python
#! /usr/bin/env python
# -*- coding: utf-8 -*-
import time
from flask import Flask
app = Flask(__name__)
application = app
@app.route("/hello")
def hello():# 模拟快速的响应
    return "Hello World!"

@app.route("/sleep")
def sleep(): # 模拟慢的响应
    time.sleep(20)
    return "Hello World!"

@app.route("/download")
def download(): # 返回一个大文件方便测试客户端网络差的情况
    n open("/home/ltt/fastdfs/zipfiles/fastdfs-master.zip").read()

if __name__ == "__main__":
    run()

```

- 配置2个apache的VirtualHost，配置相同，只是端口不同(记得修改apache的access log的格式，增加处理时间的记录)

```
Listen 4000
<VirtualHost *:4000>
    DocumentRoot /home/ltt/test/cgi
    <Directory />
        Options FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>
    ErrorLog "/home/ltt/test/cgi/logs/apache-error.log"
    CustomLog "/home/ltt/test/cgi/logs/apache-access.log" common
    WSGIDaemonProcess test-domain processes=1 threads=3 python-eggs=/tmp/.python-eggs display-name=%{GROUP}
    WSGIProcessGroup test-domain
    WSGIApplicationGroup test-domain
    WSGIScriptAlias / /home/ltt/test/cgi/wsgi
</VirtualHost>

```

- 用chrome浏览器的开发者工具，模拟客户端限制速度访问

有了这些就可以模拟各种情况以确认上面的结论了。

