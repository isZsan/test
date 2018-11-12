title:LVS

##### LVS NAT模式

！！！都先关闭防火墙和selinux设置！

```shell
1、安装ipvsadm工具
# yum -y install ipvsadm
2、设置所有的RS服务器的网关为调度器的IP地址
[root@localhost ntpstats]# vim /etc/sysconfig/network-scripts/ifcfg-eth0
#修改网关为调度器的ip地址
GATEWAY=192.168.40.200
#三个rs服务器都需要设
确保在各个RS服务的路由信息的正确性
# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.40.0    *               255.255.255.0   U     0      0        0 eth0
link-local      *               255.255.0.0     U     1002   0        0 eth0
default         192.168.40.200  0.0.0.0         UG    0      0        0 eth
3、给每个RS服务器安装http服务并测试
# yum install httpd              #每个rs服务器都安装httpd服务
# service httpd restart          #每个rs服务器都重新启动
# echo rs1> /var/www/html/index.html  #给rs1服务器创建一个主页，只在rs1上执行
# echo rs2> /var/www/html/index.html  #给rs2服务器创建一个主页，只在rs2上执行
# echo rs2> /var/www/html/index.html  #给rs3服务器创建一个主页，只在rs3上执行
#接下来使用在调度器lvs服务器先测试下各个页面是否可以正常访问
# elinks -source 192.168.40.201
rs1
# elinks -source 192.168.40.202
rs2
# elinks -source 192.168.40.203
rs3
4、创建LVS
#ipvsadm -A -t 192.168.168.200:80 -s rr              #创建一个集群服务，调度算法rr
# ipvsadm -a -t 192.168.168.200:80 -r 192.168.40.201 -m    #添加rs1 nat方式
# ipvsadm -a -t 192.168.168.200:80 -r 192.168.40.201 -m    #添加rs1 nat方式
# ipvsadm -a -t 192.168.168.200:80 -r 192.168.40.203 -m    #添加rs3 nat方式
5、打开IP转发功能
调度器要能访问168,40两个网段的，需要打开ipforward功能。
# echo 1 >/proc/sys/net/ipv4/ip_forward #打开ipv的ip转发机制，当前生效
# vim /etc/sysctl.conf 
#打开ipv的ip转发机制，永久生效
net.ipv4.ip_forward = 1
#sysctl -p
6、确保集群的时间不能太大

这里建议在调度器lvs上安装ntp服务，其他RS服务器来同步时间
我们使用宿主机进行测试

浏览器输入http://192.168.168.200，不断刷新，就可以看到如下的三个图片信息。测试成功后，就可以把三个RS服务器页面做成一样的啦。


```

![](/img2/1.jpg)



##### TUN模式

LVS-TUN模式是如何工作的？

用户请求负载均衡服务器，当IP数据包到达负载均衡服务器后，根据算法选择一台真实的服务器，然后通过IP隧道技术将数据包原封不动再次封装，并发送给真实服务器，当这个数据包到达真实服务器以后，真实服务器进行拆包（拆掉第一层的IP包）拿到里面的IP数据包进行处理，然后将结果直接返回给客户端。

![tun工作机制](/img2/tun工作机制.jpg)

![tun报文](/img2/tunip报文.jpg)

作者：温故而知新666 
来源：CSDN 
原文：https://blog.csdn.net/nimasike/article/details/53893609 

###### 配置



```shell

```













#### 问题

1、LVS多端口的问题

```shell
1. 可以直接IP到IP的映射（那就是要开启-p选项，不指定端口即可），当访问LVS的5000端口，会自动转发到RS服务器的5000端口。
2. 但是开启-p选项，表示要启用会话保持，它会在-p指定的一段时间里将同一IP的请求全部转发到同一台RS服务器上，从某种程度上来说负载均衡的能力会减弱。所以要慎重考虑此功能。（当然咯，你将-p后的时间值设置得相当小，也是可以的，但是就失去了会话保持的原本意义）
------------------------------------------------------------------------
你需要指定-p选项，此时不指定端口，就可以做到IP到IP的映射。
[root@localhost zhangzq]# ipvsadm -A -t 192.168.199.199 -s rr -p 900
[root@localhost zhangzq]# ipvsadm -a -t 192.168.199.199 -r 1.1.1.1
[root@localhost zhangzq]# ipvsadm -a -t 192.168.199.199 -r 2.2.2.2
[root@localhost zhangzq]# ipvsadm -l
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
-> RemoteAddress:Port Forward Weight ActiveConn InActConn
TCP 192.168.199.199:0 rr persistent 900
-> 2.2.2.2:0 Route 1 0 0
-> 1.1.1.1:0 Route 1 0 0
```

原文：http://zh.linuxvirtualserver.org/node/2891

![多端口问题](/img2/LVS多端口的问题.png)

