# netfilter-iprouting

- <https://baturin.org/docs/iproute2/>
- <http://linux-ip.net/linux-ip/>
- <https://github.com/tLDP/LDP/tree/master/LDP/guide/docbook/linux-ip>
- <http://www.embeddedlinux.org.cn/linux_net/0596002556/understandlni-chp-31-sect-2.html>
- <http://www.policyrouting.org/iproute2.doc.html>
- <https://opengers.github.io/openstack/openstack-base-netfilter-framework-overview/>
- <https://opengers.github.io/openstack/openstack-base-virtual-network-devices-tuntap-veth/>
- <https://opengers.github.io/openstack/openstack-base-virtual-network-devices-bridge-and-vlan/>

- netfilter
  - 通过hooks在网络栈中特定位置对数据进行过滤,标记,修改,追踪
- routing policy database
  - 根据路由规则决定所要使用的routing table
- routing tables
  - 根据路由表条目做出路由决定

## netfilter

- <https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture>

### hooks

- NF_IP_PRE_ROUTING
  - 在数据进入网络栈时触发,在做出路由决定之前
- NF_IP_LOCAL_IN
  - 数据在路由决定要发往本地时触发
- NF_IP_FORWARD
  - 数据在路由决定要转发到非本地时触发
- NF_IP_LOCAL_OUT
  - 在由本地发出的数据刚进入网络栈时触发,在做出路由决定之前
- NF_IP_POST_ROUTING
  - 所有经过路由决定后需要向外发出或转发的数据,在将要出网络栈时触发

### 表和链

表和链都只是逻辑上的分类,实际对应的是在不同hooks上执行的各种操作
同一hook不同操作的执行次序 raw->mangle->nat->filter

- filter表 对数据进行过滤,决定是否丢弃数据
  - INPUT链 对应 NF_IP_LOCAL_IN
  - OUTPUT链 对应 NF_IP_LOCAL_OUT
  - FORWARD链 对应 NF_IP_FORWARD
- nat表 对数据src,dst地址进行修改和追踪
  - PREROUTING链 对应 NF_IP_PRE_ROUTING 例如修改dst地址dnat操作
  - POSTROUTING链 对应 NF_IP_POST_ROUTING 例如修改src地址snat操作
  - OUTPUT链 对应 NF_IP_LOCAL_OUT
- mangle表 修改数据的ip header,还可以mark数据(mark是内存中的操作,不修改原数据)
  - PRERONTING
  - OUTPUT
  - FORWARD
  - INPUT
  - POSTOUTING
- raw表 可以为数据包设置NOTRACK
  - PREROUTING
  - OUTPUT
- security表 SECMARK CONNSECMARK SELinux
  - INPUT
  - OUTPUT
  - FORWARD

### conntrack

- <https://blog.cloudflare.com/conntrack-tales-one-thousand-and-one-flows/>
- <https://opengers.github.io/openstack/openstack-base-netfilter-framework-overview/>
- <http://arthurchiao.art/blog/conntrack-design-and-implementation-zh/>
- <https://voipmagazine.wordpress.com/2015/02/27/tuning-the-linux-connection-tracking-system/>

#### 表

- conntrack 存放已跟踪连接
- expect 存放related连接
- dying 存放过期连接
- unconfirmed 存放首个packet已入栈还未出栈的连接

#### hooking

- NF_IP_PRE_ROUTING, NF_IP_LOCAL_OUT
  - 在这两个hook上的执行顺序是在raw和mangle之间
  - 首先跳过有NOTRACK标记的packet
  - 然后从跟踪表查找入栈packet的连接条目
  - 如果不存在
    - 添加新的条目到unconfirmed表
  - 如果存在
    - 重置连接条目的ttl
    - 更新连接条目的packet,byte计数
    - 如果是[UNREPLIED]条目的反向packet,那么将连接跟踪状态更新为[ASSURED]
      - 也就是iptable中的 --state ESTABLISHED
      - 就是说看到了一个连接的双向数据
      - 对于tcp要看到完整的3次握手才进入这个状态吗?

- NF_IP_POST_ROUTING, NF_IP_LOCAL_OUT
  - 在这两个hook上最后执行
  - 从unconfirmed表查找出栈packet的连接条目
  - 如果匹配
    - 说明这条新连接的packet没有被丢弃,正要被送往目的地
    - 那么可以开始跟踪了,将相应的连接条目转到conntrack表
    - 此时连接条目的跟踪状态被设置为[UNREPLIED]
    - 也就是iptable中的 --state NEW
    - 就是说目前只看到连接的单向packet

#### 支持跟踪的协议

TCP、UDP、ICMP、DCCP、SCTP、GRE
对于不同协议,使用的元数据hash方法,确认[ASSURED]的标准也不同

#### 跟踪表的二元结构

- 索引 根据packet的tuple(元数据)算出的hash值,遍历效率高
  - 这个是非唯一可重复的,当对应的bucket满了就会新建一个
- bucket 存储跟踪连接的链表,遍历效率相对低
  - 包含若干个连接条目,数量与bucket size有关

#### 跟踪表条目内容

```text
cat /proc/net/nf_conntrack
ipv4 2 tcp 6 6626 ESTABLISHED src=10.99.99.99 dst=123.123.123.123 sport=43906 dport=443 packets=11 bytes=1297 src=123.123.123.123 dst=234.234.234.234 sport=443 dport=43906 packets=9 bytes=1033 [ASSURED] mark=0 zone=0 use=1
```

- ipv4/v6协议号
- 4层协议号
- 连接跟踪ttl,到期条目被GC,可以通过内核参数为不同的协议单独设置默认值
- 协议状态,对于有连接协议会有协议状态
- tuple,从packet提取的元数据,对于tcp,udp为src,dst,sport,dport
- tuple方向的packet,byte计数
- 反向tuple,自动生成,便于匹配反向packet
- 反向tuple的packet,byte计数
- 跟踪状态,udp这种无连接协议,只有单向packet时是noreplied,有双向时会去掉noreplied,不会有assured
- mark
- zone
- use

#### packet与跟踪表的匹配过程

- 首先提取tuple,算出hash值
- 遍历表中索引值,得到所有匹配的bucket
- 遍历这些bucket中的连接条目进行匹配

#### 内核参数

- <https://www.kernel.org/doc/Documentation/networking/nf_conntrack-sysctl.txt>

net.netfilter.nf_conntrack_max #最大连接条目数
net.netfilter.nf_conntrack_buckets #bucket数
nf_conntrack_max / nf_conntrack_buckets = bucket size

#### nat的现实

<https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts>
<https://manpages.debian.org/testing/conntrack/conntrack.8.en.html>
<https://wiki.aalto.fi/download/attachments/69901948/netfilter-paper.pdf>
<https://blog.csdn.net/dog250/article/details/12275041>

netfilter实现有状态的nat完全依赖conntrack

- 第一个nat packet
  - 当packet入栈时触发conntrack,写入unconfirmed表,设置snat/dnat标志??有这个吗??
  - nat操作被触发,寻找对应的nat规则(没找到就跳出)
  - 确认tuple没有在已经confirmed的conntrack表中(否则丢弃packet)
  - 然后根据nat规则修改unconfirmed表中的连接的反向tuple
  - packet本身根据nat则规被修改
  - packet到达prerouting hook,连接被确认,转入conntrack表,跟踪状态被设置为[UNREPLIED]
- 第一个返回的nat packet
  - 当packet入栈时触发conntrack,连接跟踪状态被设置为[REPLIED]
  - 触发nat,根据nat规则和conntrack表中原始tuple信息修改packet
  - 没找到看得懂的文档,所以是我猜的,这个返回包的nat操作应该是nat的隐含行为,自动完成反向操作
- 后续的packet
  - 看到有几个文档说是不再触发nat了,需要的操作也都会自动完成,至于是怎么搞的也是没搞懂
  - 应该是有谁看到snat/dnat标志然后就看packet给改了吧???

### iptables syntax

```shell
iptables -t <TABLE> -<A|I|R|D> <CHAIN> <MATCH> -j <TARGET>
man iptables
man iptables-extensions

-t #选择表,省略时默认为filter表
-A chain #向链中添加规则
-I chain #向链中插入规则
-R chain rulenum #更新指定编号的规则
-D chain rule/rulenum #删除指定规则
-C chain rule/rulenum #检查规则是否存在
-L [chain] #列出所有或指定链中的规则,-v显示计数,-n不反向解析dns
-S [chain] #列出所有或指定链中的规则,以iptables命令的方式
-F [chain] #清空所有或指定链中的规则
-Z [chain[rulenum]] #重置packet和byte计数器
-N chain #新建自定义链
-X chain #删除自定义链
-E chain newname #重命名自定义链
-P chain target #设置链默认行为,只能是ACCEPT或DROP
-j TARGET
-g chain
```

- MATCH
  - ! 配合其它MATCH取反
  - -i -o 匹配出入接口
  - -s -d 匹配源地址和目标地址
  - -p 匹配协议 tcp udp icmp all
  - -p tcp --sport 匹配tcp源端口
  - -p tcp --dport 匹配tcp目标端口
  - -p tcp --tcp-flags 匹配tcp标记
  - -p tcp -syn 匹配tcp init连接请求
  - -p tcp --tcp-option 匹配发送方接受的最大packet size
  - -p udp --sport 匹配udp源端口
  - -p udp --dport 匹配udp目标端口
  - -p icmp --icmp-type 匹配icmp类型
  - -m 使用指定的extension

- MATCH extensions
  - set --match-set 匹配ipset
  - cgroup 匹配cgroup
  - comment --comment "drama" 为规则添加注释
  - multiport --sports/dports/ports 11,22,33:44 匹配来源/目标/来源和目标端口范围
  - iprange --src-range/dst-range from 1.1.1.1 -to 2.2.2.2 匹配来源/目标地址
  - string algo bm/kmp --string/hex-string 匹配字符串或十六进制字符串
  - time 匹配packet到达时间
  - tos 匹配ipv4 header tos字段
  - owner 匹配从本地发出的packet所属uid gid
  - mark --mark 匹配packet标记,配合MARK TARGET使用
  - connmark --mark 匹配连接标记,配合CONNMARK TARGET使用
  - conntrack
    - --ctstate 匹配connection tracking状态
      - NEW 发起新连接的packet
      - ESTABLISHED 属于已建立的连接的packet
      - RELATED 与已建立的连接相关的packet
      - UNTRACKED 明确标记了不追踪的packet,也就是在raw中使用了-j CT --notrack
      - INVALID 与现有连接无关的packet,packet校验失败,数据错误
      - SNAT A virtual state, matching if the original source address differs from the reply destination
      - DNAT A virtual state, matching if the original destination differs from the reply source
  - state --state 是conntrack --ctstate的子集,不包括SNAT和DNAT

- TAGET
  - ACCEPT 接受 filter表
  - DROP 丢弃 filter表
  - RETURN 跳出当前链

- TAGET extensions
  - REJECT --reject-with 丢弃并回应
    - icmp-port-unreachable 默认
    - icmp-net-unreachable
    - icmp-host-unreachable
    - icmp-proto-unreachable
    - icmp-net-prohibited
    - icmp-host-prohibited
    - icmp-admin-prohibited
    - tcp-reset 仅应用于tcp协议
  - MARK --set-mark 为packet打标记,可以用于mark匹配和ip rule fwmark匹配
  - CONNMARK --set-mark 为连接打标记,可以用于connmark匹配
    - --save-mark 将packet标记应用到连接标记
    - --restore-mark 将连接标记应用到packet标记 仅用于mangle表
  - DNAT --to-destination 修改packet的目标地址和端口,并应用到连接后续packet
    - 1.1.1.1 只修改目标地址
    - :80 只修改目标端口
    - 1.1.1.1:80 修改目标地址和端口
    - 1.1.1.1-1.1.1.10 修改目标地址范围
    - :80-90 修改目标端口范围
    - --random 随机端口映射
    - persistent 为连接固定地址 连接黏性的意思
  - SNAT --to-source 修改packet的源地址和端口,并应用到同一连接后续packet
    - 1.1.1.1 只修改目标地址
    - :80 只修改目标端口
    - 1.1.1.1:80 修改目标地址和端口
    - 1.1.1.1-1.1.1.10 修改目标地址范围
    - :80-90 修改目标端口范围
    - --random 随机端口映射 hash算法
    - --random-fully 随机端口映射 PRNG算法
    - persistent 为连接固定地址 连接黏性的意思
  - MASQUERADE 和SNAT一样,会自动将流出接口的地址设置为源地址
    - to-ports
    - random
    - random-full
  - AUDIT 为packets生成审计记录
  - LOG 为packets生成日志 使用dmesg或syslog读取
  - LED 触发led设备
  - REDIRECT --to-ports 8080[-8088] 将packet dnat到本地
    - 目标地址被修改为流入接口的primary address,和MASQUERADE的手法类似

## routing policy

- 系统按照路由规则和优级来查找路由表
- 首先匹配路由规则,然后按照优先级逐个查找相应的路由表
- 直到在路由表中找到对应的路由条目,找不到则数据被丢弃
- 优先级0最高,自定义的规则优先级默认从32765开始

- 默认的路由规则
  - from all lookup local
    >优先级0,匹配local表,看是不是到本机地址的
  - from all lookup main
    >优先级32766,匹配main表
  - from all lookup default
    >优先级32767,匹配default表

```shell
ip rule add [selector] [action]
#默认action是lookup table,可以省略,默认是main表

#selector:
#not 取反,结合其它selector使用
#from 匹配source ip
#to 匹配dst ip
#fwark 匹配标记
#iif 匹配入接口
#oif 匹配出接口,只对从local发出的包有效,对转发的无效
#ipproto 匹配ip协议
#sport 匹配source port
#dport 匹配dst port
#uidrange 匹配uid范围
#tun_id 匹配tunnel id

#action:
#pref 设置优先级,结合其它action使用
#lookup 使用指定的路由表
#protocol
#nat
#goto
#blackhole 默默丢掉
#unreachable 返回icmp host unreachable
#prohibit 返回icmp adminitratively prohibited
#throw 返回net unreachable,如果一张表没有默认路由那么就隐含throw default

ip rule add to 1.1.1.0/24 lookup table 200
ip rule add from 2.2.2.0/24 200
ip rule add iif eth1 200
ip rule add ofi eth0 200
ip rule add fwmark 1 200
ip rule add not fwmark 1 200
```

## routing table

- 内置的路由表
  - local表中包含所有到达本机地址的路由和直连网络的广播路由
  - main表包含所有到达非本机地址的路由
- 路由最长匹配原则
  - 就是以二进制ip地址去匹配,选择匹配长度最长的路由
  - 说白了就是在匹配的路由中选择范围最小的一个

```text
ip route add [TYPE] [PREFIX] ... [table TABLE_ID]
TYPE:
unicast 默认值,可省略
local 本地路由
broadcast 广播路由
multicast 组播路由
anycast
blackhole 默默丢掉
unreachable 返回icmp host unreachable
prohibit 返回icmp adminitratively prohibited
throw 返回net unreachable,如果一张表没有默认路由那么就隐含throw default

PREFIX:
目标网络,地址加掩码,如果省略掩码则默认32
default相当于0.0.0.0/0
在查询时可以在前面加match,exact来匹配

table:
指定要操作的路由表,省略时默认为main表,type为local,broadcast的路由默认为local表

preference 路由优先级,数字

dev 用于设置直连路由,就是声明这个接口直连的网段,为一个接口设置ip之后,kernel中会自动生成这种路由

src 接口有多个地址时,使用src指定这条路由对应的ip地址

via 对于非直连路由则要使用via方式
ip route add 1.1.1.0/24 via 1.1.2.1

nexthop 同一条路由有多个路径时使用nexthop,相当于lb,可以设置weight
ip route add default nexthop via 1.1.2.1 weight 1 nexthop dev eth1 weight 1

protocol 路由协议类型

onlink 强制添加直连路由,就算本地接口没有相应的地址

scope 默认为global,local,broadcast和直连路由使用link

metric 就是metric

tos Type Of Service
```

## layer2

- <https://opengers.github.io/openstack/openstack-base-virtual-network-devices-bridge-and-vlan/>
- <http://ebtables.netfilter.org/br_fw_ia/br_fw_ia.html>
