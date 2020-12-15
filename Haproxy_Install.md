Haproxy 安装及配置

HAProxy提供高可用性、负载均衡以及基于TCP和HTTP的应用代理，支持虚拟主机，它是免费、快速并且可靠的一种负载均衡解决方案。适合处理高负载  站点的七层数据请求。类似的代理服务可以屏蔽内部真实服务器，防止内部服务器遭受攻击。
HAProxy特点和优点：
   1.支持原声SSL,同时支持客户端和服务器的SSL.
   2.支持IPv6和UNIX套字节（sockets）
   3.支持HTTP Keep-Alive
   4.支持HTTP/1.1压缩，节省宽带
   5.支持优化健康检测机制（SSL、scripted TCP、check agent...）
   6.支持7层负载均衡。
   7.可靠性和稳定性非常好。
   8.并发连接40000-50000个，单位时间处理最大请求20000个，最大数据处理10Gbps.
   9.支持8种负载均衡算法，同时支持session保持。
   10.支持虚拟主机。
   11.支持连接拒绝、全透明代理。
   12.拥有服务器状态监控页面。
   13.支持ACL.
 
 HAProxy为了让同一客户端访问服务器可以保持会话。有三种解决方法：客户端IP、Cookie以及Session
   1.HAProxy通过客户端IP进行Hash计算并保存，以此确保当相同IP访问代理服务器可以转发给固定的真实服务器。
   2.HAProxy依靠真实服务器发送客户端的Cookie信息进行会话保持。
   3.HAProxy将保存真实服务器的Session以及服务器标识，实现会话保持。
   （HAProxy只要求后端服务器能够在网络联通，也没有像LVS那样繁琐的ARP配置）

 HAProxy的balance8种负载均衡算法：
   1.roundrobin : 基于权重轮循。
   2.static-rr : 基于权重轮循。静态算法，运行时改变无法生效
   3.source : 基于请求源IP的算法。对请求的源IP进行hash运算，然后将结果与后端服务器的权重总数想除后转发至某台匹配服务器。使同一IP客户端请求始终被转发到某特定的后端服务器。
   4.leastconn : 最小连接。（适合数据库负载均衡，不适合会话短的环境） 
   5.uri : 对部分或整体URI进行hash运算，再与服务器的总权重想除，最后转发到匹配后端。
   6.uri_param : 根据URL路径中参数进行转发，保证在后端服务器数量不变的情况下，同一用户请求分发到同一机器。
   7.hdr(<name>) : 根据http头转发，如果不存在http头。则使用简单轮循。

 HAProxy主要工作模式
   1.tcp模式:该模式下，在客户端和服务器之间将建立一个全双工的连接，且不会对7层的报文做任何处理的简单模式。此模式默认，通常用于SSL、SSH、SMTP应用。
   2.http模式（一般使用）：该模式下，客户端请求在转发给后端服务器之前会被深度分析，所有不与RFC格式兼容的请求都会被拒绝。

docker部署HAproxy：
  创建安装目录：
  mkdir -p /opt/haproxy
  导出镜像：
  docker save -o haproxy.tar haproxy:2.3
  #导入镜像：
  docker load -i haproxy.tar

net:bridge
docker run -d --restart=always -p 28443:28443 -p 9999:9999 --name haproxy -v /opt/haproxy:/usr/local/etc/haproxy:ro haproxy:2.3
net:host
docker run -d --restart=always --net=host --name haproxy -v /etc/localtime:/etc/localtime -v /opt/haproxy:/usr/local/etc/haproxy:ro haproxy:2.3

cat << EOF > /opt/haproxy/haproxy.cfg
global
    log         127.0.0.1 local2
    chroot      /usr/local/etc/haproxy    #锁定运行目录
    pidfile     /var/run/haproxy.pid
    maxconn     4000                      #每个haproxy进程的最大并发连接数,要考虑到ulimit -n的大小限制
    user        root                      #运行haproxy的用户
    group       root                      #运行haproxy的用户组
    daemon
    nbproc 2                  #开启的haproxy进程数，与CPU保持一致
    #nbthread  4              #指定每个haproxy进程开启的线程数，默认为每个进程一个线程
    #cpu-map 1 0              #绑定haproxy 进程至指定CPU
    #cpu-map 2 1
    #cpu-map 3 2
    #cpu-map 4 3
    #maxsslconn  100000        #SSL每个haproxy进程ssl最大连接数
    maxconnrate  100000        #每个进程每秒最大连接数
    spread-checks  3           #后端server状态check随机提前或延迟百分比时间，建议2-5(20%-50%)之间

defaults
    mode                    tcp     #默认的模式mode { tcp|http|health }，tcp是4层，http是7层，health只会返回OK
    log                     global
    retries                 3
    timeout connect         10s    #连接超时
    timeout client          1m     #客户端超时
    timeout server          1m     #服务器超时

########统计页面配置########  
listen admin_status  
    bind 0.0.0.0:9999                #监听端口  
    mode http                        #http的7层模式  
    option httplog                   #采用http日志格式  
    #log 127.0.0.1 local0 err  
    maxconn 10  
    stats refresh 30s                #统计页面自动刷新时间  
    stats uri /status                #统计页面url  
    stats realm Haproxy \ statistic  #统计页面密码框上提示文本  
    stats auth admin:HA@2020         #统计页面用户名和密码设置  
    stats hide-version               #隐藏统计页面上HAProxy的版本信息  

frontend app-server
    bind *:28443 # 指定前端端口
    mode tcp
    default_backend master

backend master # 指定后端机器及端口，负载方式为轮询
    balance roundrobin
    mode tcp
    server app-server01  50.104.1.181:28443 check weight 1 maxconn 2000 check inter 2000 rise 2 fall 3
   #server app-server02  50.104.1.181:28443 check weight 1 maxconn 2000 check inter 2000 rise 2 fall 3
   #server app-server02  50.104.1.181:28443 check weight 1 maxconn 2000 check inter 2000 rise 2 fall 3
EOF