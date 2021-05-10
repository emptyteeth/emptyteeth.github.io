# wireguard route based site2site vpn

```shell
wg genkey |tee -a keypair |wg pubkey >>keypair
#手动
ip link add wg0 type wireguard
wg setconf wg0 /etc/wireguard/wg0.conf
ip address add 10.100.0.1/24 dev wg0
ip link set wg0 up
#自动
wg-quick up wg0
#开机启动
systemctl enable wg-quick@wg0
man wg-quick
```

```conf
[Interface]
#接口地址,可以使用多次
Address =
MTU =
DNS =
#设置路由表,默认auto为自动注入路由到默认路由表,可以设置为其它的表号,或者设置为off关闭注入
Table =
PostUp =
PostDown =
ListenPort =
PrivateKey =
SaveConfig =
```

## route based site2site

### server(vps)

wan public ip

```conf
#wg0.conf
[Interface]
Address = 10.1.100.254/24
MTU = 1280
PostUp =
PostDown =
ListenPort =
PrivateKey =

[Peer]
PublicKey =
AllowedIPs = 10.1.100.1,10.11.1.0/24

[Peer]
PublicKey =
AllowedIPs = 10.1.100.2,10.11.2.0/24
```

### node1(home router)

wan behind nat
lan 10.11.1.0/24

```conf
#wg0.conf
[Interface]
Address = 10.1.100.1/24
MTU = 1280
PostUp = iptables -A FORWARD -i wg0 -o lan -j ACCEPT; iptables -A FORWARD -i lan -o wg0 -j ACCEPT;
PostDown = iptables -D FORWARD -i wg0 -o lan -j ACCEPT; iptables -D FORWARD -i lan -o wg0 -j ACCEPT;
ListenPort =
PrivateKey =
[Peer]
PublicKey =
AllowedIPs = 10.1.100.0/24,10.1.2.0/24
Endpoint =
PersistentKeepalive = 25
```

### node2(home router)

  wan behind nat
  lan 10.11.2.0/24

```conf
#wg0.conf
[Interface]
Address = 10.1.100.2/24
MTU = 1280
PostUp = iptables -A FORWARD -i wg0 -o lan -j ACCEPT; iptables -A FORWARD -i lan -o wg0 -j ACCEPT;
PostDown = iptables -D FORWARD -i wg0 -o lan -j ACCEPT; iptables -D FORWARD -i lan -o wg0 -j ACCEPT;
ListenPort =
PrivateKey =
[Peer]
PublicKey =
AllowedIPs = 10.1.100.0/24,10.1.1.0/24
Endpoint =
PersistentKeepalive = 25
```

## openwrt theotherside

```shell
#server端开启转发和SNAT
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -s 10.11.1.0/24 -o eth0 -j MASQUERADE

#openwrt中allow ips设置为0.0.0.0/0
#这个是wireguard自身的ip规则,不匹配的数据是过不去的,因为后面要做为自定义路由表的默认网关,所以设为0.0.0.0/0
#还要并关闭自动路由设置,相当于Table=off,不然的话就变成了main表的默认网关
#因为没有自动设置路由,所以要手动设置到server端的路由:
ip route add 10.1.100.254/32 dev wg0
#如果使用site2site则额外设置其它端的路由
#
#防火墙区域中开启lan的source转发
#相当于 iptables -A FORWARD -i lan -o wg0 -j ACCEPT
#如果使用site2site则同时开启lan的destination转发
#相当于 iptables -A FORWARD -o lan -i wg0 -j ACCEPT
#因为系统默认有下面这条conntrack规则,所以不用处理回程数据
#iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
#
#安全起见可以将区域的input设为drop
#相当于 iptables -A INPUT -i wg0 -j DROP
#因为系统默认有下面这条conntrack规则,所以不用处理回程数据
#iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
#
#然后是针对theotherside的路由规则:
iptables -t mangle -N fwmark
iptables -t mangle -A PREROUTING -j fwmark
iptables -t mangle -A fwmark -i br-lan -m set --match-set theotherside dst -j MARK --set-mark 1
ip route add default via 10.1.100.254 dev wg0 table 200
ip rule add fwmark 1 table 200
```

```shell
#!/bin/sh
#/etc/hotplug.d/iface/99-ifup-wg0
[ "$ACTION" = "ifup" -a "$INTERFACE" = "wg0" ] && {
logger "iface $INTERFACE up detected..."
ipset -N theotherside hash:net
iptables -t mangle -N toschain
iptables -t mangle -A PREROUTING -i br-lan -j toschain
iptables -t mangle -A OUTPUT -j toschain
iptables -t mangle -A toschain -m set --match-set theotherside dst -j MARK --set-mark 1
ip route add 10.1.100.254/32 dev wg0
ip route add default via 10.1.100.254 dev wg0 table 200
ip rule add fwmark 1 table 200
}
exit 0
```

```shell
#!/bin/sh
#/etc/hotplug.d/iface/99-ifdown-wg0
[ "$ACTION" = "ifdown" -a "$INTERFACE" = "wg0" ] && {
logger "iface $INTERFACE down detected..."
ip rule del fwmark 1 table 200
ip route del default table 200
ip route del 10.1.100.254/32 dev wg0
iptables -t mangle -F toschain
iptables -t mangle -D PREROUTING -i br-lan -j toschain
iptables -t mangle -D OUTPUT -j toschain
iptables -t mangle -X toschain
ipset destroy theotherside
}
exit 0
```
