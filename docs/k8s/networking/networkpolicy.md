# network policy

- ns资源
- 需要cni支持
- 作用目标为pod (podSelector)
- 控制谁可以访问目标pod (ingress)
- 控制目标pod可以访问谁 (egress)
- 白名单机制,目标pod被策略匹配以外的流量都会被阻止
- pod被应用多个策略时,效果是`allowed`的叠加

## podSelector `<Object> -required-`

- 匹配目标pod,留空匹配ns内全部pod
- 可以使用matchLabels和matchExpressions

## policyTypes `<[]string>`

- 包括Ingress,Egress两个选项,默认Ingress
- 当你只设置ingress或同时设置ingress和egress时,可以忽略这个选项,它会自动判断你有没有设置egress
- 当你只设置egress时需要显式设置,不然的话它会默认开启Ingress

## ingress `<[]Object>`

>空值匹配任意traffic,也就是放行所有流量
>policyTypes包含Ingress的情况下,未设置这个选项,表示阻止所有流量

- from `<[]Object>`
  >匹配traffic source,可设置多组规则
  >规则组之间是OR关系,组内规则是AND关系
  - ipBlock `<Object>`
    >设置IP范围,不能和其它规则同组
    - cidr `<string> -required-`
      >"192.168.1.1/24" or "2001:db9::/64"
    - except <[]string>
      >排除一些IP,cidr格式
  - namespaceSelector `<Object>`
    >匹配ns,留空则匹配全部ns
    >单独使用表示匹配指定ns中的所有pod
    >与podSelector搭配使用表示匹指定ns中的指定pod
    - matchLabels
    - matchExpressions
  - podSelector `<Object>`
    >匹配pod,留空则匹配所有pod的
    >单独使用表示匹配当然ns内符合条件的pod
    >与namespaceSelector搭配使用表示匹指定ns中的指定pod
    - matchLabels
    - matchExpressions
- ports `<[]Object>`
  >匹配目标pod的端口和协议,默认为所有tcp端口
  - port `<string>`
    >可以使用pod中定义的端口名,或直接使用端口号
  - protocol `<string>`
    >TCP, UDP, or SCTP 默认TCP

## egress `<[]Object>`

>空值匹配任意traffic,也就是放行所有流量
>policyTypes包含Egress的情况下,未设置这个选项,表示阻止所有流量

- to `<[]Object>`
  >匹配traffic dest,其它和ingress一样

```yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
spec:
  podSelector: {}
  ingress:
  - {}
  policyTypes:
  - Ingress

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
spec:
  podSelector: {}
  egress:
  - {}
  policyTypes:
  - Egress

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-allow-all
spec:
  podSelector: {}
  egress:
  - {}
  ingress:
  - {}
  policyTypes:
  - Ingress
  - Egress
```
