LVS
===

A distribution of Linux Virtual Server with some advanced features.

FullNAT: A new packet forwarding method for IPVS, other than DR/NAT/TUNNEL
The main principle is as follows: the module introduces local ip address (IDC internal ip address, lip), IPVS translates cip-vip to/from lip-rip, in which lip and rip both are IDC internal ip address, so that LVS load balancer and real servers can be in different vlans, and real servers only need to access internal network. See Virtual Server via Full NAT for more information. 

SYNPROXY: Defence module against synflooding attack
The main principle: based on tcp syncookies, please refer to http://en.wikipedia.org/wiki/SYN_cookies; 

This FullNAT and SYNPROXY code for IPVS in Linux kernel 2.6.32 was written by Jiaming Wu,Jiajun Chen,Ziang Chen,Shunmin Zhu at taobao.com, Jian Chen at 360.cn, with some advising from Wensong Zhang at taobao.com. The code was affected by ideas of the source NAT and SYNPROXY version that was hard coded to IPVS in Linux kernel 2.6.9 by Wen Li, Yan Tian, Jian Chen, Yang Yi, Yaoguang Sun, Fang Han, Ying liu and Jiaming Wu at baidu.com in 2009.

The FullNAT and SYNPROXY support were added to keepalived/ipvsadm by Jiajun Chen and Ziang Chen at taobao.com.



1 SLB功能介绍
 
SLB（Server Load Balance）服务通过设置虚拟服务地址（IP），将位于同一地域（Region）的多台云服务器（Elastic Compute Service，简称ECS）资源虚拟成一个高性能、高可用的应用服务池；再根据应用指定的方式，将来自客户端的网络请求分发到云服务器池中。
 
SLB服务会检查云服务器池中ECS的健康状态，自动隔离异常状态的ECS，从而解决了单台ECS的单点问题，同时提高了应用的整体服务能力。在标准的负载均衡功能之外，SLB服务还具备TCP与HTTP抗DDoS攻击的特性，增强了应用服务器的防护能力。
 
SLB服务是ECS面向多机方案的一个配套服务，需要同ESC结合使用。
2 SLB技术架构
 
整个SLB系统由3部分构成：四层负载均衡，七层负载均衡 和 控制系统，如下图所示；
    - 四层负载均衡，采用开源软件LVS（linux virtual server），并根据云计算需求对其进行了定制化；该技术已经在阿里巴巴内部业务全面上线应用2年多详见第3节；
    - 七层负载均衡，采用开源软件Tengine；该技术已经在阿里巴巴内部业务全面上线应用3年多；参见第4节；
    - 控制系统，用于 配置和监控 负载均衡系统；
 

 
3 LVS技术特点
 
LVS是全球最流行的四层负载均衡开源软件，由章文嵩博士（当前阿里云产品技术负责人）在1998年5月创立，可以实现LINUX平台下的负载均衡。
 
LVS是 基于linux netfilter框架实现（同iptables）的一个内核模块，名称为ipvs；其钩子函数分别HOOK在LOCAL_IN和FORWARD两个HOOK点，如下图所示；
 

 
在云计算大规模网络环境下，官方LVS存在如下问题；
    - 问题1：LVS支持NAT/DR/TUNNEL三种转发模式，上述模式在多vlan网络环境下部署时，存在网络拓扑复杂，运维成本高的问题；
    - 问题2：和商用负载均衡设备（如，F5）相比，LVS缺少DDOS攻击防御功能；
    - 问题3：LVS采用PC服务器，常用keepalived软件的VRRP心跳协议进行主备部署，其性能无法扩展；
    - 问题4：LVS常用管理软件keepalived的配置和健康检查性能不足；
 
为了解决上述问题，我们在官方LVS基础上进行了定制化；
    - 解决1：新增转发模式FULLNAT，实现LVS-RealServer间跨vlan通讯；
    - 解决2：新增synproxy等攻击TCP标志位DDOS攻击防御功能，；
    - 解决3：采用LVS集群部署方式；
    - 解决4：优化keepalived性能；
 
注1：ali-LVS开源地址https://github.com/alibaba/LVS；
 
3.1 FULLNAT技术
 
FULLNAT实现主要思想：引入local address（内网ip地址），cip-vip转换为lip->rip，而 lip和rip均为IDC内网ip，可以跨vlan通讯；
 
IN/OUT的数据流全部经过LVS，为了保证带宽，采用万兆（10G）网卡；
 
FULLNAT转发模式，当前仅支持TCP协议；
 

 
3.2 SYNPROXY技术
 
LVS针对TCP标志位DDOS攻击，采取如下策略；
    1. Synflood攻击，利用synproxy模块进行防御，如下图所示；实现主要思想：参照linux tcp协议栈中syncookies的思想，LVS代理TCP三次握手；代理过程：client发送syn包给LVS，LVS构造特殊seq的synack包给client，client回复ack给LVS，LVS验证ack包中ack_seq是否合法；如果合法，则LVS再和Realserver建立3次握手；
 

 
    1. Ack/fin/rstflood攻击，查找连接表，如果不存在，则直接丢弃；
 
3.3 集群部署方式
 
LVS集群部署方式实现的主要思想：LVS和上联交换机间运行OSPF协议，上联交换机通过ECMP等价路由，将数据流分发给LVS集群，LVS集群再转发给业务服务器；
 
健壮性：lvs和交换机间运行ospf心跳，1个vip配置在集群的所有LVS上，当一台LVS down，交换机会自动发现并将其从ECMP等价路由中剔除；
 
可扩展：如果当前LVS集群无法支撑某个vip的流量，LVS集群可以进行水平扩容；
 
集群部署方式极大的保证了异常情况下，负载均衡服务的稳定性；
 

 
3.4 keepalived优化
 
对LVS管理软件keepalived进行了全面优化；
    1. 优化了网络异步模型，select改为epoll方式；
    2. 优化了reload过程；
 
综上所述，四层负载均衡产品有如下特点；
    1. 高可用，LVS集群保证了冗余性，无单点；
    2. 安全，LVS自生攻击防御+云盾，提供了近实时防御能力；
    3. 健康检查：对后端ECS进行健康检查，自动屏蔽异常状态的ECS，待该ECS恢复正常后自动解除屏蔽；
 
4 Tengine技术特点
 
Tengine是阿里巴巴发起的web服务器项目，其在Nginx的基础上，针对大访问量网站的需求，添加了很多高级功能和特性；Nginx是当前最流行的7层负载均衡开源软件之一；
 
注：Tengine开源地址http://tengine.taobao.org/；
 
针对云计算场景，tengine定制的主要特性如下；
    1. 继承Nginx-1.4.6的所有特性，100%兼容Nginx的配置；
    2. 动态模块加载（DSO）支持。加入一个模块不再需要重新编译整个Tengine；
    3. 更加强大的负载均衡能力，包括一致性hash模块、会话保持模块，还可以对后端的服务器进行主动健康检查，根据服务器状态自动上线下线；
    4. 监控系统的负载和资源占用从而对系统进行保护；
    5. 显示对运维人员更友好的出错信息，便于定位出错机器；
    6. 更强大的防攻击（访问速度限制）模块；
 
采用Tengine作为SLB的基础模块，阿里七层负载均衡产品有如下特点；
    1. 高可用，Tengine集群保证了冗余性，无单点；
    2. 安全，多维度的CC攻击防御能力；；
    3. 健康检查，对后端ECS进行健康检查，自动屏蔽异常状态的ECS，待该ECS恢复正常后自动解除屏蔽；
    4. 支持7层会话保持功能；
    5. 支持一致性hash调度；
 
5 技术展望
 
SLB作为负载均衡设备，其最重要的指标是 稳定性，在进一步提高稳定性方面，主要工作有2点;
    1. 支持集群内部 session同步；
    2. 采用anycast技术实现同城双A；
 
同时，在功能方面有更多支持；
    1. 白名单访问控制。从SLB层面实现访问控制，用户可以在SLB系统上配置白名单，便于用户灵活限定外部访问请求；
    2. 更多服务协议的支持。如：HTTPS、UDP等；
