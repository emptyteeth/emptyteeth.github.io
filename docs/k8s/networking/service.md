# service

典型的svc实现

- service api object
  >定义spec
- service-controller
  >监听svc对象,负责loadbalancer类型的svc与外部cloud lb之间的配置同步
- endpoint api object
  >定义svc后端入口
- endpoint-controller
  >监听svc对象,维护相应的endpoint
  >当svc被创建时,自动创建与svc同名的endpoint对象,根据svc的selector将相应的pod加入到endpoint中
  >当svc被删除时,自动删除同名的endpoint,即使是手动创建的也会被删除
  >如果svc没有使用selector,则不会自动创建endpoint
  >根据svc的selector动态维护endpoint中的目标pod
  >根据目标pod的ready状态,将目标pod放入addresses或notReadyAddresses池中
  >dns与kube-proxy引用addresses池中的信息
  >endpoint中的ip地址只能是单播地址.不能是loopback地址.不能是其它svc的clusterip,kube-proxy不支持把虚拟ip做为目标
- dns
  - 监听svc和endpoint,为每个svc建立A/AAAA记录,指向clusterip,对于headless则是指向endpoint中的pod
  - 为每个定义了name的port建立srv记录,headless也有
    >_portname._protoname.svcname.nsname.svc.cluster.local
  - 对于ExternalName类型的svc,建立相应的cname记录

- kube-proxy
  - 运行在每个node上,通过apiserver监测所有svc和endpoint
  - 在node上为svc建立clusterip nodeip与endpoint的转发规则
  - 建立转发规则的模式有3种
    1. user space模式下kube-proxy本身充当一个代理服务器,通过iptable将相应的流量重定向给自己,这样所有流量都需要经过kube-proxy进程转发,如果kube-proxy进程出现问题,那么所有svc在node范围内不可用,这个模式是比较早期的实现方式,现在已经不用了
    1. iptable模式下kube-proxy通过iptable为svc和endpoint维护相应的重定向规则,这样所有流量都只在内核空间内流转,这种模式比user space模式性能和安全性更高,但是在svc数量庞大的集群中,由于node上创建了数量过多的iptable规则,会造成性能瓶颈.默认模式
    1. ipvs模式下kube-proxy通过ipvs为svc和endpoint维护相应的重定向规则,这种模式比iptable模式性能更高,支持更多种均衡模式.如果node内核不支持ipvs,kube-proxy会fallback到iptable模式

## type

- clusterip
  - 只能在集群内被访问,默认类型
- nodeport
  - 在clusterip的基础上,kube-proxy会在所有node上设置额外的规则将外部流量转发到clusterip上
  - service-node-port-range
    >可使用的nodeport范围,以下是默认值
    >kube-apiserver --service-node-port-range=30000-32767

  - nodeport-addresses
    >指定nodeport的地址范围,相当于一个过滤器,kube-proxy只会在这范围内的地址上开启nodeport,默认为空,表示在所有地址开启
    >kube-proxy --nodeport-addresses=10.0.0.0/8,192.0.2.0/25

- loadbalancer
  - 在nodeport的基础上,通过cloud-controller自动配置外部的lb服务

- ExternalName
  - 将svc指向一个域名,dns返回一个cname
  - 不需要selector

## headless

- 将clusterip设置为None,svc不会被分配clusterip
- dns会把记录解析到所有的pod ip
- kube-proxy不参与

## Services without selectors

- 在不使用seletcor时,endpoint不会被自动创建
- 需要手动创建同名endpoint来自定义svc
- 同样适用于headless,相当于使用ip地址的ExternalName类型

## Multi-Port Services

对于有多个端口svc,每个端口必须设置name属性

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc1
spec:
  selector:
    k8s-app: abc
  #指定clusterip地址,一般不需要设置自动分配即可
  #不能被update,除非svc被删除,否则不会改变
  #如果是headless则设置为None
  #type为ExternalName时被忽略
  clusterIP: 1.1.1.1
  #默认类型为ClusterIP
  type: ClusterIP
  #type为ExternalName时,通过该字段设置相应的cname
  externalName: g.cn
  #影响kube-proxy在流量由外部进入集群时的转发行为
  #默认为Cluster,该模式下kube-proxy使用集群范围内的目标pod进行转发
  #此时收到数据的node对数据包进行snat和dnat,源地址转换为自己的nodeip,目标地址转换为某个目标pod
  #之后数据被路由到目标pod所在的node上,然后目标pod的响应数据又被路由回之前的node上,最后该node将响应数据被发送到外部
  #这种情况下由于进行了snat,对于目标pod来说真正的源地址彻底丢失,对于http应用可以使用proxyprotocol来避免丢失源地址信息
  #另一个模式为Local,该模式下kube-proxy只使用node本地的目标pod进行转发,这种情况下因为不需要进行snat,所以真实的源地址信息被保留
  externalTrafficPolicy: Cluster
  #只适用于type为loadbalancer同时externalTrafficPolicy为Local的情况
  #默认为nodeport的值,用来设置外部lb进行healthcheck时所使用的端口
  healthCheckNodePort: 34344
  #设置是否将notReadyAddresses池中的目标pod发布到dns记录中,默认为false
  publishNotReadyAddresses: false
  #设置会话粘性,默认为None,表示不关心
  #可以设置为ClientIP,这样同一个会话总是会被转发到相同的目标pod上
  sessionAffinity: None
  #设置会话粘性时效,默认为3小时
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
  #这个没搞懂 !review
  externalIPs:
  ports:
    #多个port必须设置name
  - name: web
    port: 80
    #如果不设置则默认与port相同,可以使用container port name
    #headless因为不需要转发,所以该字段会被忽略
    targetPort: 80
    #type为NodePort时可使用该字段手动指定node端口,默认为自动分配
    nodePort: 32080
    #可使用的协议为TCP UDP SCTP,默认TCP
    protocol: TCP
  - name: websecure
    port: 443
    targetPort: 443
```

```yaml
apiVersion: v1
Kind: Endpoints
metadata:
  name: svc1
subsets:
- addresses:
  - ip:
  ports:
  - port:
- notReadyAddresses:
  - ip:
```
