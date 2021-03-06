[摘抄](https://mp.weixin.qq.com/s/hDmkqSXyDs0VUe6pg6TLuA)  

<a href="http://www.youtube.com/watch?feature=player_embedded&v=o2zGUnL3tP8
" target="_blank"><img src="http://img.youtube.com/vi/o2zGUnL3tP8/1.jpg" 
alt="IMAGE ALT TEXT HERE" width="240" height="180" border="10" /></a>

## 配置sysctl
```bash
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 1024 65023
net.ipv4.tcp_max_syn_backlog = 10240
net.ipv4.tcp_max_tw_buckets = 400000
net.ipv4.tcp_max_orphans = 60000
net.ipv4.tcp_synack_retries = 3
net.core.somaxconn = 10000
```

## 安装haproxy

```bash
wget http://www.haproxy.org/download/1.7/src/haproxy-1.7.2.tar.gz
tar -xzf haproxy-1.7.2.tar.gz

## 编译并安装
make PREFIX=/home/ha/haproxy TARGET=linux2628
make install PREFIX=/home/ha/haproxy

```

## 创建HAProxy配置文件
```bash
mkdir -p /home/ha/haproxy/conf
vi /home/ha/haproxy/conf/haproxy.cfg
global #全局属性
    daemon  #以daemon方式在后台运行
    maxconn 256  #最大同时256连接
    pidfile /home/ha/haproxy/conf/haproxy.pid  #指定保存HAProxy进程号的文件

defaults #默认参数
    mode http  #http模式
    timeout connect 5000ms  #连接server端超时5s
    timeout client 50000ms  #客户端响应超时50s
    timeout server 50000ms  #server端响应超时50s

frontend http-in #前端服务http-in
    bind *:8080  #监听8080端口
    default_backend servers  #请求转发至名为"servers"的后端服务

backend servers #后端服务servers
    server server1 127.0.0.1:8000 maxconn 32  #backend servers中只有一个后端服务，名字叫server1，起在本机的8000端口，HAProxy同时最多向这个服务发起32个连接

#注意：HAProxy要求系统的ulimit -n参数大于[maxconn*2+18]，在设置较大的maxconn时，注意检查并修改ulimit -n参数
```

将HAProxy注册为系统服务
```
在/etc/init.d目录下添加HAProxy服务的启停脚本：

vi /etc/init.d/haproxy
```
```bash
#! /bin/sh
set -e

PATH=/sbin:/bin:/usr/sbin:/usr/bin:/home/ha/haproxy/sbin
PROGDIR=/home/ha/haproxy
PROGNAME=haproxy
DAEMON=$PROGDIR/sbin/$PROGNAME
CONFIG=$PROGDIR/conf/$PROGNAME.cfg
PIDFILE=$PROGDIR/conf/$PROGNAME.pid
DESC="HAProxy daemon"
SCRIPTNAME=/etc/init.d/$PROGNAME

# Gracefully exit if the package has been removed.
test -x $DAEMON || exit 0

start()
{
       echo -e "Starting $DESC: $PROGNAME\n"
       $DAEMON -f $CONFIG
       echo "."
}

stop()
{
       echo -e "Stopping $DESC: $PROGNAME\n"
       haproxy_pid="$(cat $PIDFILE)"
       kill $haproxy_pid
       echo "."
}

restart()
{
       echo -e "Restarting $DESC: $PROGNAME\n"
       $DAEMON -f $CONFIG -p $PIDFILE -sf $(cat $PIDFILE)
       echo "."
}

case "$1" in
 start)
       start
       ;;
 stop)
       stop
       ;;
 restart)
       restart
       ;;
 *)
       echo "Usage: $SCRIPTNAME {start|stop|restart}" >&2
       exit 1
       ;;
esac

exit 0

service haproxy start
service haproxy stop
service haproxy restart
```

添加日志

HAProxy不会直接输出文件日志，所以我们要借助Linux的rsyslog来让HAProxy输出日志

修改haproxy.cfg

在global域和defaults域中添加：
```bsh
global
    ...
    log 127.0.0.1 local0 info
    log 127.0.0.1 local1 warning
    ...

defaults
    ...
    log global
    ...
    
```
意思是将info级（及以上）的日志推送到rsyslog的local0接口，将warn级（及以上）的日志推送到rsyslog的local1接口，并且所有frontend都默认使用global中的日志配置。

注：info级的日志会打印HAProxy处理的每一条请求，会占用很大的磁盘空间，在生产环境中，建议将日志级别调整为notice

为rsyslog添加haproxy日志的配置
```bash
vi /etc/rsyslog.d/haproxy.conf
$ModLoad imudp
$UDPServerRun 514
$FileCreateMode 0644  #日志文件的权限
$FileOwner ha  #日志文件的owner
local0.*     /var/log/haproxy.log  #local0接口对应的日志输出文件
local1.*     /var/log/haproxy_warn.log  #local1接口对应的日志输出文件
```
修改rsyslog的启动参数
```
vi /etc/sysconfig/rsyslog
# Options for rsyslogd
# Syslogd options are deprecated since rsyslog v3.
# If you want to use them, switch to compatibility mode 2 by "-c 2"
# See rsyslogd(8) for more details
SYSLOGD_OPTIONS="-c 2 -r -m 0"
```
重启rsyslog和HAProxy

service rsyslog restart
service haproxy restart
此时就应该能在/var/log目录下看到haproxy的日志文件了

用logrotate进行日志切分

通过rsyslog输出的日志是不会进行切分的，所以需要依靠Linux提供的logrotate(Linux系统Logrotate服务介绍)来进行切分工作

使用root用户，创建haproxy日志切分配置文件：
```
mkdir /root/logrotate
vi /root/logrotate/haproxy
/var/log/haproxy.log /var/log/haproxy_warn.log {  #切分的两个文件名
    daily        #按天切分
    rotate 7     #保留7份
    create 0644 ha ha  #创建新文件的权限、用户、用户组
    compress     #压缩旧日志
    delaycompress  #延迟一天压缩
    missingok    #忽略文件不存在的错误
    dateext      #旧日志加上日志后缀
    sharedscripts  #切分后的重启脚本只运行一次
    postrotate   #切分后运行脚本重载rsyslog，让rsyslog向新的日志文件中输出日志
      /bin/kill -HUP $(/bin/cat /var/run/syslogd.pid 2>/dev/null) &>/dev/null
    endscript
}
```
并配置在crontab中运行：

0 0 * * * /usr/sbin/logrotate /root/logrotate/haproxy
HAProxy搭建L7负载均衡器
总体方案

本节中，我们将使用HAProxy搭建一个L7负载均衡器，应用如下功能

负载均衡
会话保持
健康检查
根据URI前缀向不同的后端集群转发
监控页面
架构如下：

架构中共有6个后端服务，划分为3组，每组中2个服务：

ms1：服务URI前缀为ms1/的请求
ms2：服务URI前缀为ms2/的请求
def：服务其他请求
搭建后端服务

部署6个后端服务，可以使用任意的Web服务，如Nginx、Apache HTTPD、Tomcat、Jetty等，具体Web服务的安装过程省略。

此例中，我们在192.168.8.111和192.168.8.112两台主机上分别安装了3个Nginx：

ms1.srv1 - 192.168.8.111:8080
ms1.srv2 - 192.168.8.112:8080
ms2.srv1 - 192.168.8.111:8081
ms2.srv2 - 192.168.8.112:8081
def.srv1 - 192.168.8.111:8082
def.srv2 - 192.168.8.112:8082
在这6个Nginx服务分别部署健康检查页面healthCheck.html，页面内容任意。确保通过http://ip:port/healthCheck.html可以访问到这个页面

接下来在6个Nginx服务中部署服务页面：

在第一组中部署ms1/demo.html
在第二组中部署ms2/demo.html
在第三组中部署def/demo.html
demo.html的内容，以部署在192.168.8.111:8080上的为例：

Hello! This is ms1.srv1!
部署在192.168.8.112:8080上的就应该是

Hello! This is ms1.srv2!
以此类推

搭建HAProxy

在192.168.8.110主机安装HAProxy,HAProxy的安装和配置步骤如上一章中描述，此处略去。

HAProxy配置文件：
```
global
    daemon
    maxconn 30000   #ulimit -n至少为60018
    user ha
    pidfile /home/ha/haproxy/conf/haproxy.pid
    log 127.0.0.1 local0 info
    log 127.0.0.1 local1 warning

defaults
    mode http
    log global
    option http-keep-alive   #使用keepAlive连接
    option forwardfor        #记录客户端IP在X-Forwarded-For头域中
    option httplog           #开启httplog，HAProxy会记录更丰富的请求信息
    timeout connect 5000ms
    timeout client 10000ms
    timeout server 50000ms
    timeout http-request 20000ms    #从连接创建开始到从客户端读取完整HTTP请求的超时时间，用于避免类DoS攻击
    option httpchk GET /healthCheck.html    #定义默认的健康检查策略

frontend http-in
    bind *:9001
    maxconn 30000                    #定义此端口上的maxconn
    acl url_ms1 path_beg -i /ms1/    #定义ACL，当uri以/ms1/开头时，ACL[url_ms1]为true
    acl url_ms2 path_beg -i /ms2/    #同上，url_ms2
    use_backend ms1 if url_ms1       #当[url_ms1]为true时，定向到后端服务群ms1中
    use_backend ms2 if url_ms2       #当[url_ms2]为true时，定向到后端服务群ms2中
    default_backend default_servers  #其他情况时，定向到后端服务群default_servers中

backend ms1    #定义后端服务群ms1
    balance roundrobin    #使用RR负载均衡算法
    cookie HA_STICKY_ms1 insert indirect nocache    #会话保持策略，insert名为"HA_STICKY_ms1"的cookie
    #定义后端server[ms1.srv1]，请求定向到该server时会在响应中写入cookie值[ms1.srv1]
    #针对此server的maxconn设置为300
    #应用默认健康检查策略，健康检查间隔和超时时间为2000ms，两次成功视为节点UP，三次失败视为节点DOWN
    server ms1.srv1 192.168.8.111:8080 cookie ms1.srv1 maxconn 300 check inter 2000ms rise 2 fall 3
    #同上，inter 2000ms rise 2 fall 3是默认值，可以省略
    server ms1.srv2 192.168.8.112:8080 cookie ms1.srv2 maxconn 300 check

backend ms2    #定义后端服务群ms2
    balance roundrobin
    cookie HA_STICKY_ms2 insert indirect nocache
    server ms2.srv1 192.168.8.111:8081 cookie ms2.srv1 maxconn 300 check
    server ms2.srv2 192.168.8.112:8081 cookie ms2.srv2 maxconn 300 check

backend default_servers    #定义后端服务群default_servers
    balance roundrobin
    cookie HA_STICKY_def insert indirect nocache
    server def.srv1 192.168.8.111:8082 cookie def.srv1 maxconn 300 check
    server def.srv2 192.168.8.112:8082 cookie def.srv2 maxconn 300 check

listen stats    #定义监控页面
    bind *:1080                   #绑定端口1080
    stats refresh 30s             #每30秒更新监控数据
    stats uri /stats              #访问监控页面的uri
    stats realm HAProxy\ Stats    #监控页面的认证提示
    stats auth admin:admin        #监控页面的用户名和密码
```
修改完成后，启动HAProxy

service haproxy start
测试

首先，访问一下监控页面http://192.168.8.110:1080/stats 并按提示输入用户名密码

接下来就能看到监控页面：

图片
监控页面中列出了我们配置的所有frontend和backend服务，以及它们的详细指标。如连接数，队列情况，session rate，流量，后端服务的健康状态等等

接下来，我们一一测试在HAProxy中配置的功能

健康检查
从监控页面中就可以直接看出健康检查配置的是否正确，上图中可以看到，backend ms1、ms2、default_servers下属的6个后端服务的Status都是20h28m UP，代表健康状态已持续了20小时28分钟，而LastChk显示L7OK/200 in 1ms则代表在1ms前进行了L7的健康检查（即HTTP请求方式的健康检查），返回码为200

此时我们将ms1.srv1中的healthCheck.html改名

mv healthCheck.html healthCheck.html.bak
然后再去看监控页面：

图片
ms1.srv1的状态变成了2s DOWN，LastChk则是L7STS/404 in 2ms，代表上次健康检查返回了404，再恢复healthCheck.html，很快就能看到ms1.srv1重新恢复到UP状态。

通过URI前缀转发请求：访问http://192.168.8.110:9001/ms1/demo.html 

图片
可以看到成功定向到了ms1.srv1上

访问http://192.168.8.110:9001/ms2/demo.html :

图片
访问http://192.168.8.110:9001/def/demo.html :

图片
负载均衡和会话保持策略

在分别访问过ms1/demo.html, ms2/demo.html, m3/demo.html后，查看一下浏览器的Cookie

图片
可以看到HAProxy已经回写了三个用于会话保持的cookie，此时反复刷新这三个页面，会发现总是被定向到*.srv1上

接下来我们删除HA_STICKY_ms1这条cookie，然后再访问ms1/demo.html，会看到

图片
同时也被新写入了一条Cookie

图片
如果发现仍然被定位到ms1.srv1，同时也没有写入新的HA_STICKY_ms1 Cookie，那么可能是浏览器缓存了ms1/demo.html页面，请求并没有到达HAProxy。F5刷新一下应该就可以了。

HAProxy搭建L4负载均衡器
HAProxy作为L4负载均衡器工作时，不会去解析任何与HTTP协议相关的内容，只在传输层对数据包进行处理。也就是说，以L4模式运行的HAProxy，无法实现根据URL向不同后端转发、通过cookie实现会话保持等功能。

同时，在L4模式下工作的HAProxy也无法提供监控页面。

但作为L4负载均衡器的HAProxy能够提供更高的性能，适合于基于套接字的服务（如数据库、消息队列、RPC、邮件服务、Redis等），或不需要逻辑规则判断，并已实现了会话共享的HTTP服务。

总体方案

本例中，我们使用HAProxy以L4方式来代理两个HTTP服务，不提供会话保持。

global
    daemon
    maxconn 30000   #ulimit -n至少为60018
    user ha
    pidfile /home/ha/haproxy/conf/haproxy.pid
    log 127.0.0.1 local0 info
    log 127.0.0.1 local1 warning

defaults
    mode tcp
    log global
    option tcplog            #开启tcplog
    timeout connect 5000ms
    timeout client 10000ms
    timeout server 10000ms   #TCP模式下，应将timeout client和timeout server设置为一样的值，以防止出现问题
    option httpchk GET /healthCheck.html    #定义默认的健康检查策略

frontend http-in
    bind *:9002
    maxconn 30000                    #定义此端口上的maxconn
    default_backend default_servers  #请求定向至后端服务群default_servers

backend default_servers    #定义后端服务群default_servers
    balance roundrobin
    server def.srv1 192.168.8.111:8082 maxconn 300 check
    server def.srv2 192.168.8.112:8082 maxconn 300 check
L4模式下的会话保持

虽然TCP模式下的HAProxy无法通过HTTP Cookie实现会话保持，但可以很方便的实现基于客户端IP的会话保持。只需将

    balance roundrobin
改为
    balance source
此外，HAProxy提供了强大的stick-table功能，HAProxy可以从传输层的数据包中采样出大量的属性，并将这些属性作为会话保持的策略写入stick-table中。

HAProxy关键配置详解
总览

HAProxy的配置文件共有5个域

global：用于配置全局参数
default：用于配置所有frontend和backend的默认属性
frontend：用于配置前端服务（即HAProxy自身提供的服务）实例
backend：用于配置后端服务（即HAProxy后面接的服务）实例组
listen：frontend+backend的组合配置，可以理解成更简洁的配置方法
global域的关键配置

daemon：指定HAProxy以后台模式运行，通常情况下都应该使用这一配置
user [username] ：指定HAProxy进程所属的用户
group [groupname] ：指定HAProxy进程所属的用户组
log [address] [device] [maxlevel] [minlevel]：日志输出配置，如log 127.0.0.1 local0 info warning，即向本机rsyslog或syslog的local0输出info到warning级别的日志。其中[minlevel]可以省略。HAProxy的日志共有8个级别，从高到低为emerg/alert/crit/err/warning/notice/info/debug
pidfile ：指定记录HAProxy进程号的文件绝对路径。主要用于HAProxy进程的停止和重启动作。
maxconn ：HAProxy进程同时处理的连接数，当连接数达到这一数值时，HAProxy将停止接收连接请求
frontend域的关键配置

acl [name] [criterion] [flags] [operator] [value]：定义一条ACL，ACL是根据数据包的指定属性以指定表达式计算出的true/false值。如"acl url_ms1 path_beg -i /ms1/"定义了名为url_ms1的ACL，该ACL在请求uri以/ms1/开头（忽略大小写）时为true
bind [ip]:[port]：frontend服务监听的端口
default_backend [name]：frontend对应的默认backend
disabled：禁用此frontend
http-request [operation] [condition]：对所有到达此frontend的HTTP请求应用的策略，例如可以拒绝、要求认证、添加header、替换header、定义ACL等等。
http-response [operation] [condition]：对所有从此frontend返回的HTTP响应应用的策略，大体同上
log：同global域的log配置，仅应用于此frontend。如果要沿用global域的log配置，则此处配置为log global
maxconn：同global域的maxconn，仅应用于此frontend
mode：此frontend的工作模式，主要有http和tcp两种，对应L7和L4两种负载均衡模式
option forwardfor：在请求中添加X-Forwarded-For Header，记录客户端ip
option http-keep-alive：以KeepAlive模式提供服务
option httpclose：与http-keep-alive对应，关闭KeepAlive模式，如果HAProxy主要提供的是接口类型的服务，可以考虑采用httpclose模式，以节省连接数资源。但如果这样做了，接口的调用端将不能使用HTTP连接池
option httplog：开启httplog，HAProxy将会以类似Apache HTTP或Nginx的格式来记录请求日志
option tcplog：开启tcplog，HAProxy将会在日志中记录数据包在传输层的更多属性
stats uri [uri]：在此frontend上开启监控页面，通过[uri]访问
stats refresh [time]：监控数据刷新周期
stats auth [user]:[password]：监控页面的认证用户名密码
timeout client [time]：指连接创建后，客户端持续不发送数据的超时时间
timeout http-request [time]：指连接创建后，客户端没能发送完整HTTP请求的超时时间，主要用于防止DoS类攻击，即创建连接后，以非常缓慢的速度发送请求包，导致HAProxy连接被长时间占用
use_backend [backend] if|unless [acl]：与ACL搭配使用，在满足/不满足ACL时转发至指定的backend
backend域的关键配置

acl：同frontend域
balance [algorithm]：在此backend下所有server间的负载均衡算法，常用的有roundrobin和source，完整的算法说明见官方文档configuration.html#4.2-balance
cookie：在backend server间启用基于cookie的会话保持策略，最常用的是insert方式，如cookie HA_STICKY_ms1 insert indirect nocache，指HAProxy将在响应中插入名为HA_STICKY_ms1的cookie，其值为对应的server定义中指定的值，并根据请求中此cookie的值决定转发至哪个server。indirect代表如果请求中已经带有合法的HA_STICK_ms1 cookie，则HAProxy不会在响应中再次插入此cookie，nocache则代表禁止链路上的所有网关和缓存服务器缓存带有Set-Cookie头的响应。
default-server：用于指定此backend下所有server的默认设置。具体见下面的server配置。
disabled：禁用此backend
http-request/http-response：同frontend域
log：同frontend域
mode：同frontend域
option forwardfor：同frontend域
option http-keep-alive：同frontend域
option httpclose：同frontend域
option httpchk [METHOD] [URL] [VERSION]：定义以http方式进行的健康检查策略。如option httpchk GET /healthCheck.html HTTP/1.1
option httplog：同frontend域
option tcplog：同frontend域
server [name] [ip]:[port] [params]：定义backend中的一个后端server，[params]用于指定这个server的参数，常用的包括有：
check：指定此参数时，HAProxy将会对此server执行健康检查，检查方法在option httpchk中配置。同时还可以在check后指定inter, rise, fall三个参数，分别代表健康检查的周期、连续几次成功认为server UP，连续几次失败认为server DOWN，默认值是inter 2000ms rise 2 fall 3
cookie [value]：用于配合基于cookie的会话保持，如cookie ms1.srv1代表交由此server处理的请求会在响应中写入值为ms1.srv1的cookie（具体的cookie名则在backend域中的cookie设置中指定）
maxconn：指HAProxy最多同时向此server发起的连接数，当连接数到达maxconn后，向此server发起的新连接会进入等待队列。默认为0，即无限
maxqueue：等待队列的长度，当队列已满后，后续请求将会发至此backend下的其他server，默认为0，即无限
weight：server的权重，0-256，权重越大，分给这个server的请求就越多。weight为0的server将不会被分配任何新的连接。所有server默认weight为1

timeout connect [time]：指HAProxy尝试与backend server创建连接的超时时间
timeout check [time]：默认情况下，健康检查的连接+响应超时时间为server命令中指定的inter值，如果配置了timeout check，HAProxy会以inter作为健康检查请求的连接超时时间，并以timeout check的值作为健康检查请求的响应超时时间
timeout server [time]：指backend server响应HAProxy请求的超时时间
default域

上文所属的frontend和backend域关键配置中，除acl、bind、http-request、http-response、use_backend外，其余的均可以配置在default域中。default域中配置了的项目，如果在frontend或backend域中没有配置，将会使用default域中的配置。
listen域

listen域是frontend域和backend域的组合，frontend域和backend域中所有的配置都可以配置在listen域下
使用Keepalived实现HAProxy高可用
尽管HAProxy非常稳定，但仍然无法规避操作系统故障、主机硬件故障、网络故障甚至断电带来的风险。所以必须对HAProxy实施高可用方案。

下文将介绍利用Keepalived实现的HAProxy热备方案。即两台主机上的两个HAProxy实例同时在线，其中权重较高的实例为MASTER，MASTER出现问题时，另一台实例自动接管所有流量。

原理

在两台HAProxy的主机上分别运行着一个Keepalived实例，这两个Keepalived争抢同一个虚IP地址，两个HAProxy也尝试去绑定这同一个虚IP地址上的端口。显然，同时只能有一个Keepalived抢到这个虚IP，抢到了这个虚IP的Keepalived主机上的HAProxy便是当前的MASTER。Keepalived内部维护一个权重值，权重值最高的Keepalived实例能够抢到虚IP。同时Keepalived会定期check本主机上的HAProxy状态，状态OK时权重值增加。

搭建HAProxy主备集群
环境准备

在两台物理机上安装并配置HAProxy，本例中，将在192.168.8.110和192.168.8.111两台主机上上安装两套完全一样的HAProxy，具体步骤省略，请参考“使用HAProxy搭建L7负载均衡器”一节。

安装Keepalived

下载，解压，编译，安装：
```
wget http://www.keepalived.org/software/keepalived-1.2.19.tar.gz
tar -xzf keepalived-1.2.19.tar.gz
./configure --prefix=/usr/local/keepalived
make
make install
```
注册为系统服务：
```
cp /usr/local/keepalived/sbin/keepalived /usr/sbin/
cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/init.d/
chmod +x /etc/init.d/keepalived
```
注意：Keepalived需要使用root用户进行安装和配置

配置Keepalived

创建并编辑配置文件
```
mkdir -p /etc/keepalived/
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
vi /etc/keepalived/keepalived.conf
```
配置文件内容：
```
global_defs {
    router_id LVS_DEVEL  #虚拟路由名称
}

#HAProxy健康检查配置
vrrp_script chk_haproxy {
    script "killall -0 haproxy"  #使用killall -0检查haproxy实例是否存在，性能高于ps命令
    interval 2   #脚本运行周期
    weight 2   #每次检查的加权权重值
}

#虚拟路由配置
vrrp_instance VI_1 {
    state MASTER           #本机实例状态，MASTER/BACKUP，备机配置文件中请写BACKUP
    interface enp0s25      #本机网卡名称，使用ifconfig命令查看
    virtual_router_id 51   #虚拟路由编号，主备机保持一致
    priority 101           #本机初始权重，备机请填写小于主机的值（例如100）
    advert_int 1           #争抢虚地址的周期，秒
    virtual_ipaddress {
        192.168.8.201      #虚地址IP，主备机保持一致
    }
    track_script {
        chk_haproxy        #对应的健康检查配置
    }
}
```
如果主机没有killall命令，则需要安装psmisc包：

yum intall psmisc
分别启动两个Keepalived

service keepalived start
验证

启动后，先分别在两台主机查看虚IP 192.168.8.201由谁持有，执行命令：

ip addr sh enp0s25   （将enp0s25替换成主机的网卡名）
持有虚IP的主机输出会是这样的：

图片
另一台主机输出则是这样的：

图片
如果你先启动备机的Keepalived，那么很有可能虚IP会被备机抢到，因为备机的权重配置只比主机低1，只要执行一次健康检查就能把权重提高到102，高于主机的101。

此时访问http://192.168.8.201:9001/ms1/demo.html ，可以看到我们先前部署的网页。

此时，检查/var/log/haproxy.log，能看到此请求落在了抢到了虚IP的主机上。

接下来，我们停掉当前MASTER主机的HAProxy实例（或者Keepalive实例，效果一样）

service haproxy stop
再次访问http://192.168.8.201:9001/ms1/demo.html ，并查看备机的/var/log/haproxy.log，会看到此请求落在了备机上，主备自动切换成功。

也可以再次执行ip addr sh enp0s25命令，会看到虚IP被备机抢去了。

在/var/log/message中，也能够看到keepalived输出的切换日志：

图片
