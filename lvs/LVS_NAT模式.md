# LVS_NAT模式

[TOC]

## LVS种类

- NAT模式

- TUN模式

- DR模式
- FULL_NAT（有淘宝研发开源）



名词约定

VIP: 	虚拟IP地址		VIP为了Director用于向客户端计算机提供服务的IP地址

RIP：	真实IP地址		在群集下面节点上使用的IP地址

DIP：	Director的IP地址	Director用于链接内外网络的IP地址，物理网卡上的IP

CIP：	客户端IP			客户端用户计算机请求群集服务的IP地址，该地址用作发送给群集的请求的源IP地址

## LVS_NAT原理

​	通过网络地址转换，调度器LB重写请求报文的目的地址，根据预设的调度算法，将请求分派给后端的真实服务器；真实服务器的响应报文通过调度器时，报文的源地址被重写，再返回给客户端，完成整个负载调度的过程。

​	**整个过程所有的流量都要经过调度器！所以所有的RIP的网关都要指向负载均衡器**。

**流程：**

​	客户端CIP通过网咯访问到调度器，请求的报文到达调度器后查看LVS规则，若在规则内，根据算法做出一下几步：

1. CIP请求的目标地址改为RIP的IP和端口，发送给目标RIP
2. 调度器在连接的Hash表中记录这个链接，当这个链接的下一个报文到达时，从链接的Hash表中可以得到原选定服务器的IP和端口，进行同样的改写操作。
3. 当RIP服务器的响应报文返回给调度器时，调度器将返回的源IP和源端口改为自己的VIP的对应的端口，再发送给CIP。

![流程图](/img/原理流程图.jpg)

## 步骤

信息

| 调度器     | IP                    | 外网MAC           |
| ---------- | --------------------- | ----------------- |
| Centos     | 192.168.1.253（外网） | 00:0c:29:f2:36:ba |
|            | 10.10.10.1 （内网）   | 00:0c:29:f2:36:c4 |
| **服务器** |                       |                   |
| ip-tun-one | 10.10.10.2            | 00:0c:29:6c:67:7d |
| ip-tun-two | 10.10.10.3            | 00:0c:29:4b:72:45 |
| 测试PC     | 192.168.1.2           | B4-6D-83-EA-2B-BC |

**目的：**

PC机访问centos：80web页面，进行负载均衡（轮询）到ip-tun-one、ip-tun-two

![Nattp.jpg](/img/NAT_拓扑.jpg)

#####调度器（Centos）配置

在配置调度器前首先检查一下几条件

- [x] 关闭SeLinux 、firewalld或iptables以免影响实验

然后开启一下功能：

- [x] 开启内核路由转发		//因为需要做NAT转发
- [x] **加载LVS内核模块** 
- [x] 安装管理软件ipvsadm //用来配置和查询lvs
- [x] 编辑调度服务器脚本 

```shell
前期准备：
检查selinux和防火墙,开启内核路由转发，
[root@Centos ~]# sestatus 	//检查Selinux
SELinux status:                 disabled
[root@Centos ~]# vim /etc/sysctl.conf  //添加一下一条命令开启路由转发
net.ipv4.ip_forward = 1
[root@Centos ~]# sysctl -p			//生效内核命令
[root@Centos ~]# modprobe ip_vs		//加载LVS模块，重启后消失
[root@Centos ~]# lsmod |grep ip_vs	
ip_vs                 141432  0 
nf_conntrack          133053  8 ip_vs,nf_nat,nf_nat_ipv4,nf_nat_ipv6,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_ipv4,nf_conntrack_ipv6
libcrc32c              12644  4 xfs,ip_vs,nf_nat,nf_conntrack
```

```shell
Ipvsadm配置
[root@Centos ~]# yum install ipvsadm		//安装管理软件ipvsadm，不需要启动
[root@Centos ~]# ipvsadm -C
[root@Centos ~]# ipvsadm -A -t 192.168.1.253:80 -s rr
[root@Centos ~]# ipvsadm -a -t 192.168.1.253:80 -r 10.10.10.2:80 -m
[root@Centos ~]# ipvsadm -a -t 192.168.1.253:80 -r 10.10.10.3:80 -m   
[root@Centos ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.1.251:80 rr
  -> 192.168.1.225:80             Masq    1      0          0         
  -> 192.168.1.252:80             Masq    1      0          0         
```

| 配置说明                                           |                                                              |
| -------------------------------------------------- | :----------------------------------------------------------- |
| ipvsadm -C                                         | C是清除所有的LVS配置                                         |
| ipvsadm -A -t 192.168.1.251:80 -s rr               | "-A"表示添加虚拟服务器， "-a"表示添加真实服务器              |
|                                                    | "-s"用来指定负载调度算法rr（轮询）wrr（加权轮询）lc（最少连接）wlc（加权最少连接） |
| ipvsadm -a -t 192.168.1.253:80 -r 10.10.10.3:80 -m | "-t"用来指定VIP地址及TCP端口 "-r"用来指定RIP地址及TCP端口    |
|                                                    | "-m"表示使用NAT群集模式（"-g"是DR模式，"-i"是TUN模式）       |
|                                                    | m: masquerade 伪装       g:gateway网关         i :ip_tunner  |



##### 配置Web服务器 

- [x] 安装http，配置页面
- [x] **将网关地址改成调度器的IP地址**

BLOG和BLOG2配置一样：

```shell
#安装http服务,编辑主页页面，重启服务（略）。
#修改默认网关
[root@Mini-Centos ~]# curl 10.10.10.3
<h2>hahaha,IP-tun-two</h2>
22222222222222222222222222
[root@Mini-Centos ~]# curl 10.10.10.2
111111111111111111111111111
<h1>IP-tun-one<h1>
################################################################################
[root@ip-tun-one ~]# ip r s
default via 10.10.10.1 dev eth1 
10.10.10.0/24 dev eth1 proto kernel scope link src 10.10.10.2 metric 101 
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.241 metric 100 
################################################################################
[root@ip-tun-two ~]# ip r s
default via 10.10.10.1 dev eth1 
10.10.10.0/24 dev eth1 proto kernel scope link src 10.10.10.3 metric 101 
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.242 metric 100 

```

##遇到的问题

##### 1、页面无响应

按照以上配置后访问192.168.1.253页面无响应：

![超时](/img/页面超时.jpg)

解决：

```shell
Centos上抓包：
[root@Mini-Centos ~]# tcpdump -enn -i any -p tcp  port 80
....
15:34:49.430077  In b4:6d:83:ea:2b:bc ethertype IPv4 (0x0800), length 68: 192.168.1.2.55936 > 192.168.1.253.80: Flags [S], seq 4271078537, win 17520, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
15:34:49.430195 Out 00:0c:29:f2:36:c4 ethertype IPv4 (0x0800), length 68: 192.168.1.2.55936 > 10.10.10.3.80: Flags [S], seq 4271078537, win 17520, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
15:34:49.688149  In b4:6d:83:ea:2b:bc ethertype IPv4 (0x0800), length 68: 192.168.1.2.55937 > 192.168.1.253.80: Flags [S], seq 3532130825, win 17520, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
15:34:49.688410 Out 00:0c:29:f2:36:c4 ethertype IPv4 (0x0800), length 68: 192.168.1.2.55937 > 10.10.10.2.80: Flags [S], seq 3532130825, win 17520, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
....
```

 In b4:6d:83:ea:2b:bc(测试PC)  192.168.1.2.55936 > 192.168.1.253.80

 Out 00:0c:29:f2:36:c4（eth1）  192.168.1.2.55936 > 10.10.10.3.80

```shell
RIP上抓包：
[root@ip-tun-one ~]# tcpdump -enn -i any -p tcp  port 80
....
23:34:13.558803  In 00:0c:29:f2:36:c4 ethertype IPv4 (0x0800), length 68: 192.168.1.2.55937 > 10.10.10.2.80: Flags [S], seq 3532130825, win 17520, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
23:34:16.550536  In 00:0c:29:f2:36:c4 ethertype IPv4 (0x0800), length 68: 192.168.1.2.55937 > 10.10.10.2.80: Flags [S], seq 3532130825, win 17520, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
23:34:22.534487  In 00:0c:29:f2:36:c4 ethertype IPv4 (0x0800), length 68: 192.168.1.2.55937 > 10.10.10.2.80: Flags [S], seq 3532130825, win 17520, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
23:36:01.393137  In 00:0c:29:f2:36:c4 ethertype IPv4 (0x0800), length 68: 192.168.1.2.55961 > 10.10.10.2.80: Flags [S], seq 1808335528, win 17520, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
23:36:04.385814  In 00:0c:29:f2:36:c4 ethertype IPv4 (0x0800), length 68: 192.168.1.2.55961 > 10.10.10.2.80: Flags [S], seq 1808335528, win 17520, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
23:36:10.370256  In 00:0c:29:f2:36:c4 ethertype IPv4 (0x0800), length 68: 192.168.1.2.55961 > 10.10.10.2.80: Flags [S], seq 1808335528, win 17520, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
....
```

抓包发现调度器（Centos）上LVS已经有做过NAT数据转发了，而RIP处却只收不发，想到RIP的eth0为了方便测试配置了192.168.1.x段的地址 是否是因为路由问题导致的？

那么我们先删除掉RIP上的192.168.1.x的路由

```shell
[root@IP_tunnel_one ~]# ip r s
default via 10.10.10.1 dev eth1 
10.10.10.0/24 dev eth1 proto kernel scope link src 10.10.10.2 metric 101
##################################################################
[root@IP_tunnel_two ~]# ip r del 192.168.1.0/24 dev eth0 
[root@IP_tunnel_two ~]# ip r s
default via 10.10.10.1 dev eth1 
10.10.10.0/24 dev eth1 proto kernel scope link src 10.10.10.3 metric 101 
```

再次刷新网页，这时果然出现了

![效果图](/img/finally.jpg)



在centos上查询负载情况

![超时](/img/lvs_nat负载情况.jpg)

------

##### 2.貌似负载不均

在打开centos地址时WEB页面时显示的时two页面：

​	使劲刷新但是页面却不会跳转到one页面，一直保持在two页面上！！很费解，是不是这个LVS的算法有问题？

------

其实不是的！经过仔细研究后发现原来时TCP链接问题。

![Nattp.jpg](/img/问题2.jpg)



我理解的原因：当一个页面打开时， TCP链接会进入ESTABLISHED（ESTABLISHED的意思是**建立连接。表示两台机器正在通信** ）状态，如果此时一直刷新WEB页面就会一直保持此状态，而不会进入CLOSE_WAIT 或TIME_WAIT，这对于LVS来说其实就是一个链接，所以才不会进行负载。

仔细看下图中**ActiveConn  InActConn**的值，当ActiveConn活动链接变为0时LVS才会进行负载！

-  ActiveConn是活动连接数 ，也就是tcp连接状态的ESTABLISHED 
- InActConn是指除了ESTABLISHED以外的,所有的其它状态的tcp连接 

![Natview.gif](/img/Natview.gif)



更详细的资料：[lvs中ipvsadm的ActiveConn和InActConn的深入理解](http://blog.51cto.com/tonychiu/950822)



## NAT的优缺点

NAT模式技术是将请求重写（IN修改目的，OUT修改源），所有的请求和响应报文都必须要经过调度器。

NAT的主要缺点：

-  所有的流量都需要经过调度器服务器，占用出口带宽，调度器容器形成瓶颈。
- 所有的RIP服务器的网关都必须指向调度服务器（我认为这个才是更主要的原因）

NAT的优点：

- 节省IP资源
- LVS模式中**唯一支持端口转换**的模式





