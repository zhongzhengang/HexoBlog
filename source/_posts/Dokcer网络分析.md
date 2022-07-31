---
title: Docker网络分析
categories:
    - 编程
        - docker
tags:
    - docker网络
---


在进行探索之前，我们先来看一张Docker网络整体结构图

![image](images/Docker网络整体结构图.png)

在这个图中eth0和veth是虚拟网卡对，docker0是网桥，ens33是物理网卡。所有的网络数据报文最终都是要经过物理网卡ens33发送出去的，那么Container之间是怎么通信的？Container与外部互联网主机之间又是怎么通信的呢？接下来的内容就来探索这些问题。

<!-- more -->

## 虚拟网桥Docker0和主机物理网卡是如何传递数据的？

我们先来看看刚建好的容器里面是否可以和外部互联网通信？经过验证，**Docker中的容器可以ping通www.baidu.com**

```bash
root@b18022430c6d:/# ping www.baidu.com
PING www.a.shifen.com (220.181.38.149) 56(84) bytes of data.
64 bytes from 220.181.38.149 (220.181.38.149): icmp_seq=1 ttl=127 time=36.9 ms
64 bytes from 220.181.38.149 (220.181.38.149): icmp_seq=2 ttl=127 time=38.6 ms
64 bytes from 220.181.38.149 (220.181.38.149): icmp_seq=3 ttl=127 time=35.4 ms
64 bytes from 220.181.38.149 (220.181.38.149): icmp_seq=4 ttl=127 time=35.3 ms
64 bytes from 220.181.38.149 (220.181.38.149): icmp_seq=5 ttl=127 time=35.4 ms
^C
--- www.a.shifen.com ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4008ms
rtt min/avg/max/mdev = 35.335/36.325/38.631/1.290 ms
```

Docker中的容器是直接连到虚拟网桥docker0上面的，我们查看一下**主机上的网卡信息**：

```bash
mucao@mucao-vm:~$ ifconfig
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:22ff:fedc:b681  prefixlen 64  scopeid 0x20<link>
        ether 02:42:22:dc:b6:81  txqueuelen 0  (Ethernet)
        RX packets 10  bytes 667 (667.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 43  bytes 5565 (5.5 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.42.101  netmask 255.255.255.0  broadcast 192.168.42.255
        inet6 fe80::1bde:af17:bc0b:11fb  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:d9:79:39  txqueuelen 1000  (Ethernet)
        RX packets 947  bytes 97617 (97.6 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 603  bytes 66960 (66.9 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 204  bytes 17150 (17.1 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 204  bytes 17150 (17.1 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

vethae4af8f: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::944f:87ff:fe8a:1761  prefixlen 64  scopeid 0x20<link>
        ether 96:4f:87:8a:17:61  txqueuelen 0  (Ethernet)
        RX packets 10  bytes 807 (807.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 73  bytes 9019 (9.0 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

可以看到主机上面的网卡有虚拟网桥docker0、物理网卡ens33、虚拟网卡vethae4af8f。不管主机和docker的网络到底是怎么样的，一个TCP/IP数据包最后能够发送出去，肯定是通过主机上唯一的物理网卡ens33的。

容器里面发送的网络数据包肯定是会到达docker0，然后肯定是从物理网卡ens33发送到互联网。那么docker0和ens33之间的数据报文是怎么传递的呢？经过查证，答案就是iptables。

我们先来了解点儿iptables的预备知识。

1、iptables的工作流程

① 防火墙是一层一层过滤的。实际是按照配置规则的顺序从上到下，从前到后进行过滤的；
② 如果匹配上规则，即明确表明阻止还是通过，此时数据包就不再向下匹配新规则了；
③ 如果所有规则中没有明确表明是阻止还是通过这个数据包，也就是没有匹配上规则，则按照默认策略进行处理；
④ 防火墙的默认规则是对应的链的所有的规则执行完成后才会执行的；

2、iptables四表五链

iptables实际上是由四张表和五条链组成的。

规则链包括如下五个：

| 链名       | 作用                   |
| ---------- | ---------------------- |
| INPUT      | 处理输入数据包         |
| OUTPUT     | 处理输出数据包         |
| FORWARD    | 处理转发数据包         |
| PREROUTING | 用于目标地址转换(DNAT) |
| POSTOUTING | 用于源地址转换(SNAT)   |

五张表里面最重要的是filter和nat表。下面逐一介绍：

filter表主要是和主机自身相关，真正负责主机防火墙功能（过滤流入流出主机的数据包）, filter表是iptables默认使用的表，这个表定义了三个链（chains）。

| 链名    | filter表中作用           |
| ------- | ------------------------ |
| INPUT   | 过滤进入主机的数据包     |
| FORWARD | 负责转发流经主机的数据包 |
| OUTPUT  | 处理从主机发出去的数据包 |

nat表负责网络地址转换，即来源与目的IP地址和port的转换。和主机本身无关，一般用于局域网共享上网或者特殊的端口转换服务相关。比如，用于企业路由或网关，共享上网。这个表定义了3个链，nat功能相当于网络的acl控制，和网络交换机acl类似。

| 链名        | nat表中作用                                                  |
| ----------- | ------------------------------------------------------------ |
| OUTPUT      | 改变主机发出数据包的目标地址                                 |
| PREROUTING  | 在数据包到达防火墙时进行路由判断之前执行的规则；<br/>改写数据包的目的地址、目的端口 |
| POSTROUTING | 在数据包离开防火墙时进行路由判断之后执行的规则；<br/>改变数据包的源地址、源端口 |

下面我们来看一下iptables是如何使用这四表五链工作的？

![20200328205338](images/iptables工作原理.jpeg)

下面我们来查看下主机上iptables的信息：

```bash
# filter表信息
mucao@mucao-vm:~$ sudo iptables -t filter -L -n -v --line-numbers
Chain INPUT (policy ACCEPT 2508 packets, 199K bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1       16  1466 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0
2       16  1466 DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0
3        8   855 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
4        0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
5        8   611 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
6        0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0

Chain OUTPUT (policy ACCEPT 1860 packets, 252K bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain DOCKER (1 references)
num   pkts bytes target     prot opt in     out     source               destination

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1        8   611 DOCKER-ISOLATION-STAGE-2  all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
2       16  1466 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain DOCKER-ISOLATION-STAGE-2 (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 DROP       all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
2        8   611 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain DOCKER-USER (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1       16  1466 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0
```

```bash
# nat表信息
mucao@mucao-vm:~$ sudo iptables -t nat -L -n -v --line-numbers
Chain PREROUTING (policy ACCEPT 22 packets, 1821 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        2   104 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 19 packets, 1605 bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 235 packets, 18840 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 235 packets, 18840 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        3   216 MASQUER ADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0

Chain DOCKER (2 references)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0
```

关键字 “Chain”也就是链的意思，大概就是一个组的意思，Chain下面就是规则，规则的模版都是一样。接下来介绍iptables信息的每一列是什么含义：

| 规则列名    | 含义                                                         |
| ----------- | ------------------------------------------------------------ |
| pkts        | 对应规则匹配到的报文的个数                                   |
| bytes       | 对应匹配到的报文包的大小总和                                 |
| target      | 触发操作。意思是规则生效之后，进行怎样的处理                 |
| prot        | 表示规则对应的协议，是否只针对某些协议应用此规则             |
| opt         | 表示规则对应的选项                                           |
| in          | 表示数据包由哪个接口(网卡)流入，我们可以设置通过哪块网卡流入的报文需要匹配当前规则 |
| out         | 表示数据包由哪个接口(网卡)流出，我们可以设置通过哪块网卡流出的报文需要匹配当前规则 |
| source      | 表示规则对应的源头地址，可以是一个IP，也可以是一个网段       |
| destination | 表示规则对应的目标地址。可以是一个IP，也可以是一个网段       |

target代表匹配到规则后的操作，有下面这些常用的操作类型：

| 操作类型   | 作用                                                         |
| ---------- | ------------------------------------------------------------ |
| ACCEPT     | 允许数据包通过                                               |
| DROP       | 直接丢弃数据包，不给任何回应信息                             |
| REJECT     | 拒绝数据包通过，必要时会给数据发送端一个响应的信息，客户端刚请求就会收到拒绝的信息 |
| SNAT       | 源地址转换，解决内网用户用同一个公网地址上网的问题。         |
| MASQUERADE | 是SNAT的一种特殊形式，适用于动态的、临时会变的ip上           |
| DNAT       | 目标地址转换                                                 |
| REDIRECT   | 在本机做端口映射                                             |
| LOG        | 在/var/log/messages文件中记录日志信息，然后将数据包传递给下一条规则 |
| RETURN     | 停止匹配后续策略，如果是主链则跳到默认策略，如果是子链则跳到主链的下一条规则 |

下面我就可以愉快的分析查出来的iptables信息了。先看filter表中关键的FORWARD链信息：

```bash
...... # 省略INPIUT
Chain FORWARD (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1       16  1466 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0
2       16  1466 DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0
3        8   855 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
4        0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
5        8   611 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
6        0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0
...... # 省略OUTPUT和自定义链
```

规则2是作用是禁止docker0发送给docker0的数据。 docker -> docker0  禁止

规则3表明其他网卡发送docker0的数据允许通过。   * -> docker0  允许

队则5表明docker0发送给其他网卡的数据允许通过。docker0 -> * 允许

规则6表明docker0发送给docker0的数据允许通过。  docker0 -> docker0 允许

由于规则2是在规则6之前，所有可以看到规则6没有匹配到任何流量。

在nat表格中我们重点看下POSTROUTING链：

```bash
Chain POSTROUTING (policy ACCEPT 235 packets, 18840 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        3   216 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0
```

规则1的含义是只要不是传输给docker0网卡的数据包，并且源地址网段是172.17.0.0/16，那么在完成过滤规则后都会进行源地址转换，然后才会将数据包发送出去。

可以看到在POSTROUTING链中有一个规则，它的target是MASQUERADE（伪装），是动态SNAT（source networkaddress translation的缩写，即源地址目标转换），大概意思就是从这个主机上往外发出的网络数据报文源地址会被替换为ens33的IP地址，动态性体现在无论ens33通过DHCP获取到了任何IP地址都会被作为替换后的报文源地址。从外部互联网来看的话，从这台主机上出去的所有报文源地址都是ens33的IP地址，是看不见docker0以及容器IP地址的。

接下来看看SNAT在iptables工作流程中生效的位置：

<img src="images/SNAT在iptables工作流程中生效位置.png" alt="PIC" style="zoom:50%;" />

从这张图上我们可以清晰的看到，所有从主机发出的数据报文，不管是主机应用数据报文，还是转发的数据报文，都在POSTROUTING环境执行源地址替换操作（SNAT）。

从数据报文的视角我们来理解一下masquerade的工作过程：

![docker网络_源地址转换](images/docker网络_源地址转换.png)

从Container1中出发的报文在经过ens33时，源地址被替换为ens33的ip地址。相应地，响应报文在经过ens33时，目的地址被替换为Container1中eth0的ip地址。

总结一下，我们现在基本知道了docker0与物理网卡ens33之间的数据传输机制。

（1）容器之间通信： container1 -> docker0 -> container2

（2）容器发送数据到宿主机： container1 -> docker0 -> ens33

（3）宿主机发送数据到容器：ens33 -> docker0 -> container1

（4）容器发送数据到互联网主机：container1 -> docker0 -> ens33（源地址替换） -> network-host

（5）互联网主机发送数据到容器：失败。容器IP地址对于互联网主机是不可见的。



## 实际的网络报文在这些网卡之间是如何流转的？

上面通过网卡和iptables信息基本知道了容器网卡、docker0、物理网卡、虚拟网卡veth之间的关系。下面我们进行抓包来验证一下上面得出的关系，使用的抓包工具为tcpdump + wireshark。

从容器中ping百度：

```bash
root@b18022430c6d:/# ping www.baidu.com
PING www.a.shifen.com (14.215.177.39) 56(84) bytes of data.
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=1 ttl=127 time=49.3 ms
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=2 ttl=127 time=49.3 ms
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=3 ttl=127 time=45.4 ms
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=4 ttl=127 time=49.2 ms
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=5 ttl=127 time=63.8 ms
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=6 ttl=127 time=51.9 ms
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=7 ttl=127 time=45.2 ms
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=8 ttl=127 time=45.6 ms
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=9 ttl=127 time=69.0 ms
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=10 ttl=127 time=47.3 ms
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=11 ttl=127 time=46.3 ms
^C
--- www.a.shifen.com ping statistics ---
11 packets transmitted, 11 received, 0% packet loss, time 11043ms
rtt min/avg/max/mdev = 45.179/51.119/68.982/7.549 ms
```

下面是宿主机上物理网卡ens33的抓包数据：

![image-20220714192738693](images/image-20220714192738693.png)

从上面物理网卡ens33的报文数据得出，在www.baidu.com的互联网主机看来和它通信的其实是宿主机（192.168.42.101）。

下面是网桥docker0上面的报文数据：

![image-20220714192930043](images/image-20220714192930043.png)

上面的数据第一条是从DNS服务器解析到百度网址对应的IP地址。下面的报问就是容器中执行ping命令产生的。

我们来仔细看这些数据，发现报文是容器eth0的地址和百度地址在交互，也就是说在docker0这儿是能看到容器地址，并且还没有发生地址替换。

但是在物理网卡ens33上看到的报文都是宿主机(192.168.42.101)和百度地址的交互。和百度网址进行交互的地址发生了变化，肯定是iptables中规则完成的地址转换。

下面我们再来看看虚拟网卡veth。

![image-20220714193031530](images/image-20220714193031530.png)

从报文数据来看，和在docker0上的交互地址是一样的。

接下来我们在看看容器eth0的报文数据：

![image-20220714193229132](images/image-20220714193229132.png)

从容器的eth0网卡报文数据的地址交互来看，也是和docker0上的一样。



## Container中eth0和veth之间是怎么通信的？

我们先来看下虚拟网卡对veth之间的关联关系。首先看下这两个网卡的信息：

```bash
# 宿主机上
mucao@mucao-vm:~$ ifconfig
...... 省略其他网卡信息
vethae4af8f: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::944f:87ff:fe8a:1761  prefixlen 64  scopeid 0x20<link>
        ether 96:4f:87:8a:17:61  txqueuelen 0  (Ethernet)
        RX packets 28  bytes 2347 (2.3 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 124  bytes 13597 (13.5 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

```bash
# 容器中
root@b18022430c6d:/# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:02  txqueuelen 0  (Ethernet)
        RX packets 124  bytes 13597 (13.5 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 28  bytes 2347 (2.3 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
...... 省略本地环回
```

介绍一下网卡信息中重要的两项：

RX packets ：网卡接收的数据包数量

TX packets ：网卡发送的数据包数量

现在我们站在容器里面eth0网卡的角度来看数据包关系。

（1）eth0 发送了28个数据包，共2347字节，刚好vethae4af8f接收了28个数据包，共2347字节。

（2）eth0 接收了124个数据包，共13597字节，刚好vethae4af8发送了124个数据包，共13597字节。

容器eth0和虚拟网卡vethae4af8f之间数据是怎么传输的呢？我们来看看这个虚拟网卡对相关的Docker代码：

```go
// bridge.go文件
func (d *driver) CreateEndpoint(nid, eid string, ifInfo driverapi.InterfaceInfo, epOptions map[string]interface{}) error
{
    ...... // 省略
// Generate a name for what will be the host side pipe interface
	hostIfName, err := netutils.GenerateIfaceName(d.nlh, vethPrefix, vethLen)
	if err != nil {
		return err
	}

	// Generate a name for what will be the sandbox side pipe interface
	containerIfName, err := netutils.GenerateIfaceName(d.nlh, vethPrefix, vethLen)
	if err != nil {
		return err
	}

	// Generate and add the interface pipe host <-> sandbox
	veth := &netlink.Veth{
		LinkAttrs: netlink.LinkAttrs{Name: hostIfName, TxQLen: 0},
		PeerName:  containerIfName}
	if err = d.nlh.LinkAdd(veth); err != nil {
		return types.InternalErrorf("failed to add the host (%s) <=> sandbox (%s) pair interfaces: %v", hostIfName, containerIfName, err)
	}
     ...... // 省略
}
```

从上述代码中可以知道容器eth0和虚拟网卡vethae4af8f其实是调用Linux内核创建出来的虚拟网卡对，用来完成不同网络命名空间通信的问题。

我们先来对veth pair有个大概认识：

![img](images/Linux_veth_pair_结构.webp)

veth结构体通过peer字段来连接另外一段，然后看看它的作用：

![img](images/Linux_veth_pair_作用.png)

veth pair可以用来连通两个相互隔离的网络命名空间。接下来我们再看看它的工作流程：

![img](images/Linux_veth_pair_工作流程.png)

veth pair 之间传递数据通过内核协议栈完成的，这样就不会破坏网络命名空间之间的隔离性。

下面是虚拟网卡对之间发送数据的Linux源代码：

```c
// drivers/net/veth.c文件
static netdev_tx_t veth_xmit(struct sk_buff *skb, struct net_device *dev)
{
	...... // 省略
	rcu_read_lock();
	rcv = rcu_dereference(priv->peer); // 对端就是接收者
	...... // 省略
	skb_tx_timestamp(skb);
	if (likely(veth_forward_skb(rcv, skb, rq, use_napi) == NET_RX_SUCCESS)) {
		if (queue)
			txq_trans_cond_update(queue);
		if (!use_napi)
			dev_lstats_add(dev, length);
	} else {
drop:
		atomic64_inc(&priv->dropped);
	}
    	...... // 省略
	return NETDEV_TX_OK;
}
```



## 虚拟网卡vethae4af8f与docker0之间如何传递数据的？

最后我们再看看虚拟网卡vethae4af8f与docker0之间如何传递数据的？

先想一个场景，多个网络命名空间都要相互通信该怎么做呢？当然可以创建很多的veth pair来完成，但是很麻烦、非常浪费资源。Linux为了解决这个问题出现了网桥。网桥的作用如下：

![Linux网桥](images/Linux网桥.png)

由于网桥的存在，每个网络命名空间只需要一个veth就可以相互通信了。Docker0其实就是Linux网桥了，相当于现实世界的网络协议二层路交换机，完成数据报文转发工作。

网卡加入到网桥后，它的收发数据就会由网桥管理了。发送数据的时候，首先网桥会依据mac地址在网桥内广播，如果网桥内没找到目标设备，才会发送到下一跳；接收数据的时候，其实网桥是使用的加入的网卡地址，因此**一个网卡加入到网桥后就不会有自己的IP地址了**，如下所示：

```bash
mucao@mucao-vm:~$ ifconfig
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:22ff:fedc:b681  prefixlen 64  scopeid 0x20<link>
        ether 02:42:22:dc:b6:81  txqueuelen 0  (Ethernet)
        RX packets 10  bytes 667 (667.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 43  bytes 5565 (5.5 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
...... // 省略其他网卡信息
vethae4af8f: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::944f:87ff:fe8a:1761  prefixlen 64  scopeid 0x20<link>
        ether 96:4f:87:8a:17:61  txqueuelen 0  (Ethernet)
        RX packets 10  bytes 807 (807.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 73  bytes 9019 (9.0 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```



Linux网桥会将网络接口设备 MAC地址与网桥端口进行映射。实现网桥的数据结构：

<img src="images/Linux网桥数据结构.png" alt="image" style="zoom: 50%;" />

Linux网桥工作过程：

<img src="images/Linux网桥工作流程.png" alt="11、网络--Linux Bridge（网桥基础）_linux_02"  />

我们来瞄一眼网桥中最重要的函数br_handle_frame：

```c
// Linux：net/bridge/br_input.c
/*
 * Return NULL if skb is handled
 * note: already called with rcu_read_lock
 */
static rx_handler_result_t br_handle_frame(struct sk_buff **pskb)
{
	struct net_bridge_port *p;
	struct sk_buff *skb = *pskb;
	const unsigned char *dest = eth_hdr(skb)->h_dest;  // 获取目的地址

	if (unlikely(skb->pkt_type == PACKET_LOOPBACK))    // 本地环回
		return RX_HANDLER_PASS;

	if (!is_valid_ether_addr(eth_hdr(skb)->h_source))  // 检查无效来源地址
		goto drop;

	skb = skb_share_check(skb, GFP_ATOMIC); // 检查数据报文
	if (!skb)
		return RX_HANDLER_CONSUMED;

	memset(skb->cb, 0, sizeof(struct br_input_skb_cb));

	p = br_port_get_rcu(skb->dev);
	if (p->flags & BR_VLAN_TUNNEL)
		br_handle_ingress_vlan_tunnel(skb, p, nbp_vlan_group_rcu(p));

	if (unlikely(is_link_local_ether_addr(dest))) {    // 目的地址不是本地
		u16 fwd_mask = p->br->group_fwd_mask_required;

		/*
		 * See IEEE 802.1D Table 7-10 Reserved addresses
		 *
		 * Assignment		 		Value
		 * Bridge Group Address		01-80-C2-00-00-00
		 * (MAC Control) 802.3		01-80-C2-00-00-01
		 * (Link Aggregation) 802.3	01-80-C2-00-00-02
		 * 802.1X PAE address		01-80-C2-00-00-03
		 *
		 * 802.1AB LLDP 		01-80-C2-00-00-0E
		 *
		 * Others reserved for future standardization
		 */
		fwd_mask |= p->group_fwd_mask;
		switch (dest[5]) {                             // 依据地址类型进行数据报文转发
		case 0x00:	/* Bridge Group Address */
			/* If STP is turned off,
			   then must forward to keep loop detection */
			if (p->br->stp_enabled == BR_NO_STP ||
			    fwd_mask & (1u << dest[5]))
				goto forward;
			*pskb = skb;
			__br_handle_local_finish(skb);
			return RX_HANDLER_PASS;

		case 0x01:	/* IEEE MAC (Pause) */
			goto drop;

		case 0x0E:	/* 802.1AB LLDP */
			fwd_mask |= p->br->group_fwd_mask;
			if (fwd_mask & (1u << dest[5]))
				goto forward;
			*pskb = skb;
			__br_handle_local_finish(skb);
			return RX_HANDLER_PASS;

		default:
			/* Allow selective forwarding for most other protocols */
			fwd_mask |= p->br->group_fwd_mask;
			if (fwd_mask & (1u << dest[5]))
				goto forward;
		}

		/* The else clause should be hit when nf_hook():
		 *   - returns < 0 (drop/error)
		 *   - returns = 0 (stolen/nf_queue)
		 * Thus return 1 from the okfn() to signal the skb is ok to pass
		 */
		if (NF_HOOK(NFPROTO_BRIDGE, NF_BR_LOCAL_IN,
			    dev_net(skb->dev), NULL, skb, skb->dev, NULL,
			    br_handle_local_finish) == 1) {
			return RX_HANDLER_PASS;
		} else {
			return RX_HANDLER_CONSUMED;
		}
	}

	if (unlikely(br_process_frame_type(p, skb)))
		return RX_HANDLER_PASS;

forward:
	if (br_mst_is_enabled(p->br))
		goto defer_stp_filtering;

	switch (p->state) {
	case BR_STATE_FORWARDING:
	case BR_STATE_LEARNING:
defer_stp_filtering:
		if (ether_addr_equal(p->br->dev->dev_addr, dest))
			skb->pkt_type = PACKET_HOST;

		return nf_hook_bridge_pre(skb, pskb);
	default:
drop:
		kfree_skb(skb);
	}
	return RX_HANDLER_CONSUMED;
}
```



## 总结

我们最后来总结一下。Docker网络使用到的核心技术有虚拟网卡对、虚拟网桥、iptables。容器和主机网络通信是通过虚拟网卡对完成的，虚拟网卡对的两端是通过内核协议栈完成通信的，所以不会破坏网络命名空间的隔离性。在主机中的虚拟网卡都attach到虚拟网桥docker0上面，这样容器之间就可以通信了。虚拟网桥docker0再通过iptables的过滤规则和转发规则与主机物理网卡相连，这样docker0上的数据报文就可以通过主机物理网卡发送到互联网主机上了。由于iptables转发规则的地址替换机制，可以将容器地址隐藏，不会暴露到互联网中。收工！



参考资料：

1、[运维派](http://www.yunweipai.com/35061.html)

2、[使用wireshark分析tcpdump出来的pcap文件](https://blog.csdn.net/zeze_z/article/details/57919479)

3、[veth(4) — Linux manual page](https://man7.org/linux/man-pages/man4/veth.4.html)

4、[Linux虚拟网络设备 veth-pair详解](https://www.cnblogs.com/bakari/p/10613710.html)

5、[一文读懂Docker网络基础-虚拟网络设备对（veth）原理](https://www.bilibili.com/read/cv16733012/)

6、[linux bridge网桥的工作原理](https://blog.csdn.net/u014027051/article/details/53908878)

7、[IP Masquerading using iptables](http://billauer.co.il/ipmasq-html.html)

