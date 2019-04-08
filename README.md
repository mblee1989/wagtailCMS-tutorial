OSPF原理及配置
OSPF（Open Shortest Pass First,开放最短路径优先协议），是一个最常用的内部网管协议，是一个链路状态协议。

OSPF的特点
OSPF是一种无类路由协议，支持VLSM可变长子网掩码。支持IPV4和IPV6.
组播地址：224.0.0.5 224.0.0.6。
OSPF度量：从源到目的所有出接口的度量值，和接口带宽反比（10^8/带宽）。
收敛速度极快，但大型网络配置很复杂。
IP封装，协议号89
OSPF运行原理
OSPF组播的方式在所有开启OSPF的接口发送Hello包，用来确定是否有OSPF邻居，若发现了，则建立OSPF邻居关系，形成邻居表，之后互相发送LSA（链路状态通告）相互通告路由，形成LSDB（链路状态数据库）。再通过SPF算法，计算最佳路径（cost最小）后放入路由表。 
设计要求：1.必须配置骨干区域0 
                  2.其他区域连接到骨干区域 
好处:  1.减小路由表（通过域间汇总） 
          2.本地拓扑变化值影响一个区域（也是通过汇总） 
          3.某些LSA之子本地泛红，不泛洪到其他区域 
注：OSPF区域划分基于接口而不是设备

OSPF区域及路由器身份
OSPF区域
骨干区域（区域0）：骨干区域必须连接所有的非骨干区域，而且骨干区域不可分割，有且只有一个，一般情况下，骨干区域内没有终端用户。 
非骨干区域（非0区域）：非骨干区域一般根据实际情况而划分，必须连接到骨干区域（不规则区域也需通过tunnel或virtual-link连接到骨干区域）。一般情况下，费骨干区域主要连接终端用户和资源。

OSPF身份
DR（Designated Router）：指定路由器，OSPF协议启动后开始选举而来 
BDR（Back-up Designated Router）：备份指定路由器，同样是由OSPF启动后选举而来 
DRothers：其他路由器，非DR非BDR的路由器都是DRothers。

ABR（Area Border Routers）：区域边界路由器，连接不同OSPF区域。 
ASBR（Autonomous System Boundary Router）：自治系统边界路由器，位于OSPF和非OSPF网络之间。 
骨干路由器：至少有一个借口连接到骨干区域（区域0）。

OSPF邻居建立
邻居的两个状态 
Neighbors：邻居 
Adjacency：邻接

邻居不一定是邻接，邻接一定是邻居，只有交互了LSA的OSPF邻居才成为OSPF的邻接，之交互Hello包的支撑位邻居，
在点对点网络中，所有邻居都能成为邻接。
MA（广播多路访问网络，比如以太网）网络类型中，DR，BDR，DRothers三者关系为： 
DR、BDR与所有的邻居形成邻接，DRothers之间只是邻居而不交换LSA
影响OSPF邻居建立的原因：

Hello与Dead Time时间不一致（改Hello的话Dead自动*4，单改Dead的话Hello不变）
区域ID必须一致
认证（password一致）
Stub标识一致（与特殊区域有关，之后介绍）
MTU-携带在DBD报文中，两端口必须一致
掩码，如12.1.1.1/30——12.1.1.2/24 这种情况是可以ping通的，但邻居关系起不来 
（OSPF对环回口，无论掩码多少位，都按32位处理，所以建议环回口直接/32，或者在环回口下还原真实掩码）
ACL（是否放行OSPF）
OSPF更新
OSPF是一种触发更新的机制。一旦拓扑发生变化便会更新。
OSPF也有周期性更新（30分钟一次）
当收到一条LSA之后： 
首先查看是否在LSDB中，若没有则假如LSDB，回复LSACK。继续泛洪出去，并且通过SPF算法计算最佳路径并加入路由表。若存在，则比较谁的更“新”（看序号），序号大者新，若本地不如收到的信更新本地LSDB并泛洪，且通过SPF算法计算最佳路径并加入路由表，若比收到的新，则将本地的泛洪出去。
注：LSA序列号，4字节，16进制 
0x80000001-0x7FFFFFFF

OSPF数据包类型
Hello：10秒发送一次，死亡时间40s，4倍关系，可以修改。
DBD：Database Description 仅仅是一个对本地数据库的概念性叙述，供路由器核对数据库是否同步
LSR：Link-State Request 请求链路状态，在数据库同步过程中使用，请求其他角色发送自己失去的LSA最新版本。
LSU：Link-State Update 链路状态更新，LSU包括几种类型的LSA，LSU负责泛洪LSA，和相应LSR。LSA只会发送给之前以LSR请求的LSA的直连邻居，进行泛洪的时候，邻居路由负责把收到的LSA信息重新封装在新的LSU中。
LSACK：链路状态确认，路由器必须对每个收到的LSA进行LSACK确认，但可以用一个LSACK确认多个LSA。
DR、BDR的选举
DR、BDR的选举规则：比较router-id，router-id有以下获得方式：

由工程师指定
这台设备最大的环回口ip
没有环回口的话，物理接口ip地址最大的。
选举规则：

最高优先级值的路由器被选为DR（默认优先级相同：1），次高优先级的为BDR
若优先级相同，则比较router-id，拥有最高router-id的成为DR，次高的成为BDR
优先级被设置为0的不参与选举
OSPF系统启动后，若40s内没有新设备接入就会开始选举，所以为保证DR与BDR的选举不发生意外，建议优先配置想成为DR与BDR的设备。
DR与BDR不可以抢占
当DR小时之后，BDR直升DR，重新选BDR
所有DR，BDR，DRothers说的都是接口，而不是设备
不同网段间选DR，BDR，而不是以OSPF区域为单位
OSPF状态
Down State
Init State：发送了Hello包（还没收到）
Two-way State：收到了一个Hello包且Hello包中包括自己的router-id（对方回复的）
Exstart State：First DBD确认主从关系，router-id大的为主，先发包
Exchange State：交互DBD 相互学习
Loading State：LSR与LSU的交互过程
Full State：所有交互已经完成
注：DBD只是一个目录的性质，并且第一个DBD只是用来协商之后的DBD由谁先发送。



