[TOC]
> fpm 全名是FastCGI进程管理器，可以参考关于FastCGI的说明：CGI、FastCGI、PHP-CGI和PHP-FPM 概念区分
> fpm启动后会先读php.ini，然后再读取相应的conf配置文件，conf配置可以覆盖php.ini的配置。启动fpm之后，会创建一个master进程，监听9000端口（可配置），master进程又会根据fpm.conf/www.conf去创建若干子进程，子进程用于处理实际的业务。当有客户端（比如nginx）来连接9000端口时，空闲子进程会自己去accept，如果子进程全部处于忙碌状态，新进的待accept的连接会被master放进队列里，等待fpm子进程空闲；这个存放待accept的半连接的队列有多长，由 listen.backlog 配置。

[REFE]: https://www.kancloud.cn/digest/php-src/136260

# php-fpm的启动参数

## 测试php-fpm配置
```
/usr/local/php/sbin/php-fpm -t
/usr/local/php/sbin/php-fpm -c /usr/local/php/etc/php.ini -y /usr/local/php/etc/php-fpm.conf -t
```
## 启动php-fpm
```
/usr/local/php/sbin/php-fpm
/usr/local/php/sbin/php-fpm -c /usr/local/php/etc/php.ini -y /usr/local/php/etc/php-fpm.conf
```
## 关闭php-fpm
```
kill -INT `cat /usr/local/php/var/run/php-fpm.pid`
```
## 重启php-fpm
```
kill -USR2 `cat /usr/local/php/var/run/php-fpm.pid`
```


# php-fpm.conf重要参数详解

> pid设置，默认在安装目录中的var/run/php-fpm.pid，建议开启

pid = run/php-fpm.pid

> 错误日志，默认在安装目录中的var/log/php-fpm.log 如果设置为syslog，log就会发送给syslogd服务而不会写进文件里。

error_log = log/php-fpm.log

> 把日志写进系统log，linux还不够熟悉，暂时不用理会。

syslog.facility = daemon

> 系统日志标示，如果跑了多个fpm进程，需要用这个来区分日志是谁的。

syslog.ident = php-fpm

> 错误级别. 可用级别为: alert（必须立即处理）, error（错误情况）, warning（警告情况）, notice（一般重要信息）, debug（调试信息）. 默认: notice.

log_level = notice

> 表示在emergency_restart_interval所设值内出现SIGSEGV或者SIGBUS错误的php-cgi进程数如果超过 emergency_restart_threshold个，php-fpm就会优雅重启。这两个选项一般保持默认值。

emergency_restart_threshold = 60
emergency_restart_interval = 60s

> 设置子进程接受主进程复用信号的超时时间. 可用单位: s(秒), m(分), h(小时), 或者 d(天) 默认单位: s(秒). 默认值: 0.

process_control_timeout = 0

> 后台执行fpm,默认值为yes，如果为了调试可以改为no。在FPM中，可以使用不同的设置来运行多个进程池。 这些设置可以针对每个进程池单独设置。

daemonize = yes

> fpm监听端口，即nginx中php处理的地址，一般默认值即可。可用格式为: 'ip:port', 'port', '/path/to/unix/socket'. 每个进程池都需要设置.

listen = 127.0.0.1:9000

> 未accept处理的socket队列大小，-1 on FreeBSD and OpenBSD，其他平台默认65535，高并发时重要，合理设置会及时处理排队的请求；太大会积压太多，处理完后nginx在前面都等超时断开这个和fpm的socket连接了，就杯具了。不要用-1，建议1024以上，最好是2的幂值。一个池共用一个backlog队列，所有的池进程都去这个队列里accept连接。最大数量受限于系统配置 cat /proc/sys/net/core/somaxconn，系统配置修改：vim /etc/sysctl.conf，增加 net.core.somaxconn = 2000 则最大为2000，然后php最大的backlog可以到2000。

listen.backlog = -1

> 允许访问FastCGI进程的IP，设置any为不限制IP，如果要设置其他主机的nginx也能访问这台FPM进程，listen处要设置成本地可被访问的IP。默认值是any。每个地址是用逗号分隔. 如果没有设置或者为空，则允许任何服务器请求连接

listen.allowed_clients = 127.0.0.1

> unix socket设置选项，如果使用tcp方式访问，这里注释即可。

listen.owner = www
listen.group = www
listen.mode = 0666

> 启动进程的帐户和组

user = www
group = www


> 对于专用服务器，pm可以设置为static。如何控制子进程，选项有static和dynamic。如果选择static，则由pm.max_children指定固定的子进程数。如果选择dynamic，则由下开参数决定：
pm.max_children

pm = dynamic 

> 子进程最大数

pm.max_children

> 启动时的进程数

pm.start_servers

> 保证空闲进程数最小值，如果空闲进程小于此值，则创建新的子进程

pm.min_spare_servers 

> 保证空闲进程数最大值，如果空闲进程大于此值，此进行清理

pm.max_spare_servers

> 设置每个子进程重生之前服务的请求数. 对于可能存在内存泄漏的第三方模块来说是非常有用的. 如果设置为 '0' 则一直接受请求. 等同于 PHP_FCGI_MAX_REQUESTS 环境变量. 默认值: 0.

pm.max_requests = 1000

> FPM状态页面的网址. 如果没有设置, 则无法访问状态页面. 默认值: none. munin监控会使用到

pm.status_path = /status

> FPM监控页面的ping网址. 如果没有设置, 则无法访问ping页面. 该页面用于外部检测FPM是否存活并且可以响应请求. 请注意必须以斜线开头 (/)。

ping.path = /ping

> 用于定义ping请求的返回相应. 返回为 HTTP 200 的 text/plain 格式文本. 默认值: pong.

ping.response = pong

> 设置单个请求的超时中止时间. 该选项可能会对php.ini设置中的'max_execution_time'因为某些特殊原因没有中止运行的脚本有用. 设置为 '0' 表示 'Off'.当经常出现502错误时可以尝试更改此选项。

request_terminate_timeout = 0

> 当一个请求该设置的超时时间后，就会将对应的PHP调用堆栈信息完整写入到慢日志中. 设置为 '0' 表示 'Off'

request_slowlog_timeout = 10s


> 慢请求的记录日志,配合request_slowlog_timeout使用

slowlog = log/$pool.log.slow


> 设置文件打开描述符的rlimit限制. 默认值: 系统定义值默认可打开句柄是1024，可使用 ulimit -n查看，ulimit -n 2048修改。

rlimit_files = 1024

> 设置核心rlimit最大限制值. 可用值: 'unlimited' 、0或者正整数. 默认值: 系统定义值.

rlimit_core = 0

> 启动时的Chroot目录. 所定义的目录需要是绝对路径. 如果没有设置, 则chroot不被使用.

chroot =


> 设置启动目录，启动时会自动Chdir到该目录. 所定义的目录需要是绝对路径. 默认值: 当前目录，或者/目录（chroot时）

chdir =


> 重定向运行过程中的stdout和stderr到主要的错误日志文件中. 如果没有设置, stdout 和 stderr 将会根据FastCGI的规则被重定向到 /dev/null . 默认值: 空

catch_workers_output = yes



# fpm进程状态监控

```
1、nginx配置：遇到 status 的请求，直接转发给php
2、fpm配置：pm.status_path = /status
3、然后重新fpm和nginx，在浏览器里访问就能看到了：

默认以 text/plain 展示结果，可以传参数 ?json/html/xml 分别得到json等格式的结果；参数full可以查看每个子进程的明细

pool 进程池名称
process manager 进程管理方式
start time 进程什么时候启动的
start since 进程已经运行了多少秒
accepted conn 该池总共accept了多少连接
listen queue 等待accept的连接的数量
max listen queue fpm启动后，历史最高等待accept的连接的数量
listen queue len 配置的监听队列最大长度 受限于listen.backlog和系统cat /proc/sys/net/core/somaxconn，两者中取最小值
idle processes 闲置的进程数
active process 正在工作的进程数（加上限制的，就是总的子进程数）
total processes 总的子进程数量
max active processes fpm启动后，历史最多同时工作的进程数
max children reached 进程管理模式为 'dynamic'和 'ondemand'时，此数值是当子进程不够用时，master创建更多子进程的次数
slow requests 慢请求个数

# full参数下
pid 子进程ID;
state 子进程状态(Idle, Running, ...);
start time 子进程启动的时间;
start since 子进程启动后运行了多少秒;
requests 当前子进程一共处理了多少个请求;
request duration 请求耗费的纳秒数;
request method 请求方法 (GET, POST, ...);
request URI 请求参数;
content length POST请求时，请求的内容长度;
user - the user (PHP_AUTH_USER) (or '-' if not set);
script 请求的哪个php文件;
last request cpu 上次请求耗费的cpu资源
last request memory 上次请求耗费的内存峰值
如果进程是闲置状态，那这些信息记录的就是上次请求的相关数据，否则就是当前本次请求的相关数据。
参考：http://www.cnblogs.com/52php/p/6278265.html
```
