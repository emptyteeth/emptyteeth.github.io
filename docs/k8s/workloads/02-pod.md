# pod

## activeDeadlineSeconds

可选项,设置pod的活动时间,时间到了会被kill

```text
Status:       Failed
Reason:       DeadlineExceeded
Message:      Pod was active on the node longer than the specified deadline
```

## affinity

><https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity>
><https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/>
><https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/>
><https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/>

比nodeSelector更灵活,有required和preferred两种策略,匹配条件可以是node,或者是运行在node上的pod

### nodeAffinity

如果与nodeSelector同时使用,则需要同时满足条件

- requiredDuringSchedulingIgnoredDuringExecution
  >必须满足条件pod才会被调度
  >只影响调度时的选择,运行中的pod不会因条件变化被影响
  >可使用多个nodeSelectorTerms,有一个满足即可
  - nodeSelectorTerms  `<[]Object> -required-`
    >可使用多个match,必须同时满足
    - matchExpressions `<[]Object>`
      >匹配label
    - matchFields `<[]Object>`
      >匹配field
- preferredDuringSchedulingIgnoredDuringExecution
  >与required一样,只是非强制性的
  >scheduler会以node与preference的匹配度打分
  >匹配的条件越多分数越高,再加上weight得出最终分数
  - preference `<Object> -required-`
    >可定义多个match,每一个都会以是否匹配来打分
    - matchExpressions `<[]Object>`
      >匹配label
    - matchFields `<[]Object>`
      >匹配field
  - weight `<integer> -required-`
    >1-100整数

### podAffinity

1. 使用labelSelector匹配目标pod
2. 获得目标pod所在node的topologyKey值
3. 把新pod调度到具有相同topologyKey值的node上

- podAffinity
  - requiredDuringSchedulingIgnoredDuringExecution
    >必须满足条件pod才会被调度
    >只影响调度时的选择,运行中的pod不会因条件变化被影响
    - labelSelector `<Object>`
      >与逻辑,多个match必须同时满足
      - matchExpressions `<[]Object>`
      - matchLabels `<map[string]string>`
    - namespeces `<[]string>`
      >可选指定的命名空间
    - topologyKey `<string> -required-`
      >这里填的是node标签,作为判断亲和性的范围标准
      >scheduler会选择具有相同标签值的node进行调度
      >比如想要调度到同一个node,使用`kubernetes.io/hostname`
      >比如想要调度到同一个zone,使用`failure-domain.beta.kubernetes.io/zone`

  - preferredDuringSchedulingIgnoredDuringExecution
    >尽量满足调度条件
    - podAffinityTerm `<Object> -required-`
      - labelSelector `<Object>`
        - matchExpressions
        - matchLabels
      - namespeces `<[]string>`
      - topologyKey `<string> -required-`
    - weight `<integer> 1-100 -required-`

### podAntiAffinity

1. 使用labelSelector匹配目标pod
2. 获得目标pod所在node的topologyKey值
3. 避免把新pod调度到具有相同topologyKey值的node上

- podAntiAffinity
  - requiredDuringSchedulingIgnoredDuringExecution
    >坚决避免
    - labelSelector `<Object>`
      - matchExpressions `<[]Object>`
      - matchLabels `<map[string]string>`
    - namespeces `<[]string>`
    - topologyKey `<string> -required-`

  - preferredDuringSchedulingIgnoredDuringExecution
    >尽量避免
    - podAffinityTerm `<Object> -required-`
      - labelSelector `<Object>`
        - matchExpressions `<[]Object>`
        - matchLabels `<map[string]string>`
      - namespeces `<[]string>`
      - topologyKey `<string> -required-`
    - weight `<integer> 1-100 -required-`

## automountServiceAccountToken

是否自动挂载sa token到容器
优先于svc中的automountServiceAccountToken设置

## containers

## dnsPolicy

- Default
  >直接拿node的resolv.conf来用
  >虽然叫Default,但不是默认选项
- ClusterFirst
  >使用集群dns搜索域,集群范围外的域名被转发到node的dns上游服务器
  >老子才是默认值,没想到吧hhhhhh

  ```shell
  cat /etc/resolv.conf
  search default.svc.cluster.local svc.cluster.local cluster.local
  nameserver 10.96.0.10
  options ndots:5
  ```

- ClusterFirstWithHostNet
  >对于使用hostNetwork的Pod,默认是直接用node的resolv.conf
  >使用这个选项来达到ClusterFirst的效果
- None
  >使用自定义设置,必须配合dnsConfig使用

## dnsConfig

```yaml
  dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster-domain.example
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

## enableServiceLinks

是否自动把svc信息注入pod ENV,默认true

## ephemeralContainers

使用临时容器,alpha

## hostAliases `<[]Object>`

添加额外的条目到/etc/hosts,不适用于hostNetwork

## hostIPC

使用node ipc命名空间,默认false

## hostNetwork

使用node network命名空间,默认false

## hostPID

使用node pid命名空间,默认false

## hostname

- pod中container的hostname默认是podname,就是说每一个pod副本的hostname是不同的
- 可以用这个指定hostname,这样所有pod副本的hostname都一样
- 默认情况下系统是不会为pod分配dns记录的
- 有一种方法可以为pod分配dns记录
- 需要pod的hostname和subdomain都有设置
- 并且需要一个与设置的subdomain同名且能match到pod的svc
- 此时pod的dns记录为hostname.subdomain.ns.svc.cluster.local
- 此dns记录返回的是拥有此hostname的pod所有副本的IP地址,与headless svc效果一样
- 与headless svc不同的是,headless svc返回所有匹配到的pod的IP
- 如果headless svc匹配多种hostname的pod,那么可以用这个记录加以区分
what's the point???

## subdomain `<string>`

在hostname后添加一个子域名

## imagePullSecrets

><https://kubernetes.io/docs/concepts/containers/images/>

使用hosted集群不同厂商有相应的方法,参考上面链接,自建集群有3种方法

### 手工设置secret

```bash
#创建secret
kubectl create secret docker-registry <name> --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
```

```yaml
#引用secret
spec:
  imagePullSecrets:
    - name: myregistrykey
  containers:
    - name: foo
      image: janedoe/awesomeapp:v1
```

### 使用serviceaccount

><https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#add-imagepullsecrets-to-a-service-account>

```bash
#创建secret
kubectl create secret docker-registry <name> --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
```

```yaml
#添加到sa中
apiVersion: v1
kind: ServiceAccount
imagePullSecrets:
- name: myregistrykey
```

使用该sa的pod将被自动注入imagePullSecrets

### 在node上配置cri

https://github.com/containerd/cri/blob/master/docs/registry.md

## initContainers <[]Object>

><https://kubernetes.io/docs/concepts/workloads/pods/init-containers/>

- 每个initContainer必须成功才会运行下一个initContainer
- 多个initContainer的运行顺序与它他在spec中的出场顺序相对应
- 所有initContainer必须成功才会运行container
- 任何失败都导致pod被kill
- 不支持lifecycle和各种probe

## nodeName

><https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodename>
指定某个node运行,相当于自助调度,kubelet直接读取这个信息来运行pod不需要scheduler参与
优先级高于nodeselector和nodeaffinity

## nodeSelector `<map[string]string>`

按标签选择node,和matchLabel用法一样
多个标签需同时满足

```yaml
  nodeSelector:
    disktype: ssd
```

## preemptionPolicy `<string>` `alpha`

This field is alpha-level and is only honored by servers that enable the NonPreemptingPriority feature.
设置此pod的抢占策略
默认为PreemptLowerPriority,就是资源可以被高优先级的pod抢占
可以设置为Never,拥有此优先级的pod不会被更高优先级的pod抢占资源

## priority `<integer>`

pod优先级,默认启用的priority Admission Controller会根据`priorityClassName`来设置这个值,并阻止用户设置或修改它.

## priorityClassName `<string>`

><https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/>
><https://kubernetes.io/docs/concepts/policy/resource-quotas/#limit-priority-class-consumption-by-default>

指定使用的优先级名称

有一种叫priorityclass的非ns资源
系统默认有两个

```text
system-cluster-critical        2000000000
system-node-critical           2000001000
```

自定义priorityclass
!review

```yaml
#如果被删除,具有此优先级的pod的priorityClassName保持不变
apiVersion: v1
description: some info
kind: PriorityClass
#设置为全局默认优先级,应用到新建的未指定优先级的pod,不影响已有的pod
#如果有多个全局默认优先级,value小的生效
globalDefault: true
#设置此优先级的抢占策略 alpha
#默认为PreemptLowerPriority,就是资源可以被高优先级的pod抢占
#可以设置为Never,拥有此优先级的pod不会被更高优先级的pod抢占资源
preemptionPolicy: PreemptLowerPriority
metadata:
  #不能以系统保留的`system-`开头
  name: classA
#数值越大优先级越高
value: 10000
```

## readinessGates `<[]Object>`

><https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/0007-pod-ready++.md>

- 为pod添加readiness,和container的readiness不同
- container的readiness是从内部确定readiness状态,所有container的readiness都成功时,ContainersReady为true
- pod的readiness是由外部注入到podstatus里的,通过PATCH操作注入到pod conditions中
- 比如在readinessGates的conditionType中设置了ab.cd/ef
- kubelet就会根据conditions中的ab.cd/ef的状态来判断pod是否ready
- 只有ContainersReady为true,加上ab.cd/ef为true,Ready才会true
- 如果ContainersReady为true,ab.cd/ef为false,这时Ready为false
- 这种情况下不会被加入到endpoints,rollingupdate也不计入成功数量,就这么一直运行

## restartPolicy `<string>`

>应到到所有container
当container停止,kubectl会重启它,默认是Always
back-off delay为10s 20s 40s ... 300s,成功运行10分钟后重置

- Always
  >总是重启
- OnFailure
  >只有非0退出才重启
- Never
  >不重启

## schedulerName `<string>`

指定scheduler,不指定就使用默认的

## securityContext `<Object>`

## serviceAccount `<string>`

Deprecated: Use serviceAccountName instead.

## serviceAccountName `<string>`

为pod设置sa

## shareProcessNamespace `<boolean>`

设置pod内container共享pid namespace
不能和hostpid一起使用,默认为false

## terminationGracePeriodSeconds `<integer>`

对container进程发送SIGTERM后多久发送SIGKILL,默认30秒

## topologySpreadConstraints `<[]Object>`

><https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints>

- 通过labelselector选择目标pod
- 获得目标pod所在node的topologykey值
- 以不同的topologykey值划分出不同的目标域
- 确保新pod被均匀分散调度到这些不同的目标域中
- maxskew用来设置不同目标域中最大pod数量差
- whenUnsatisfiable用来设置以上条件不能满足时如果调度新pod
- 比如topologykey设置为kubernetes.io/hostname,那么每一个node就是一个目标域
- 可以定义多组规则,它们需要被同时满足

```yaml
# <[]Object>
topologySpreadConstraints:
    #<Object>
  - labelSelector:
      matchLabels:
        foo: bar
        bar: foo
    #<integer> -required-
    maxSkew: 1
    #<string> -required-
    topologyKey: ab.cd/ef
    #<string> -required-
    #默认为DoNotSchedule,可以使用ScheduleAnyway,相当于尽量满足这个条件
    whenUnsatisfiable: DoNotSchedule
```

## tolerations `<[]Object>`

><https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/>
!review

### taint node

```shell
# k=v:effect
#指定key value effect
kubectl taint nodes node1 key=value:NoSchedule
#忽略value
kubectl taint nodes node1 key:NoSchedule
#去除taint
kubectl taint nodes node1 key:NoSchedule-
```

- NoSchedule
  >禁止调度
- PreferNoSchedule
  >尽量不要调度
- NoExecute
  >这个会影响正在运行的pod,不能容忍的pod被立即驱逐

```yaml
  taints:
  - effect: NoSchedule
    key: key
    value: value
```

### pod tolerations

```yaml
tolerations:
- key: "key"
  #默认值是Equal,判断value
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"

- key: "key"
  #使用Exists只判断key存在,忽略value
  operator: "Exists"
  effect: "NoSchedule"

  #空key+Exists+空effect容忍任何taint
- key: ""
  operator: "Exists"
  effect: ""
  #可选项,定义容忍有效时间,过期后容忍失效
  tolerationSeconds: 3600
```

## volumes `<[]Object>`

><https://kubernetes.io/docs/concepts/storage/volumes>

!reveiw

### configmap

- 挂载configmap是会自动更新的,kubelet默认有一分钟的缓存,和一分钟的轮询间隔,理论上最多有两分钟延迟
- 如果不希望自动更新,可以改用env,或者是使用subpath挂载
- 或者开启ImmutableEmphemeralVolumes feature gate 然后在cm中加入immutable: true

```yaml
volumes:
  - name: cm1
    configMap:
      name: cm1
  - name: cm2
    configMap:
      #挂载文件的权限,默认为0644
      defaultMode: 0644
      #默认如果cm或指定的key不存在就会报错,设置为true为忽略
      optional: true
      name: cm2
      #挂载cm中指定的kv
      items:
      - key: key1
        #可以指定挂载后的相对路径
        path: my/key1
      - key: key2
        #可以指定挂载后的文件名
        path: keytwo
```

### secret

>和configmap一样

```yaml
volumes:
  - name: sec1
    secret:
      secretName: sec1
  - name: sec2
    secret:
      #挂载文件的权限,默认为0644
      defaultMode: 0644
      #默认如果secret或指定的key不存在就会报错,设置为true为忽略
      optional: true
      secretName: sec2
      #挂载secret中指定的kv
      items:
      - key: key1
        #可以指定挂载后的相对路径
        path: my/key1
      - key: key2
        #可以指定挂载后的文件名
        path: keytwo
```

### downwardAPI

- 支持的字段
  - fieldRef
    - metadata.name
    - metadata.namespace
    - metadata.uid
    - metadata.labels
    - metadata.annotations
    - metadata.labels['mylabel']
    - metadata.annotations['myannotation']
  - resourceFieldRef
    - limits.cpu
    - requests.cpu
    - limits.memory
    - requests.memory

```yaml
volumes:
  - name: podinfo
    downwardAPI:
      items:
        - path: "labels"
          fieldRef:
            fieldPath: metadata.labels
        - path: "annotations"
          fieldRef:
            fieldPath: metadata.annotations
        - path: "cpu_limit"
          resourceFieldRef:
            containerName: client-container
            resource: limits.cpu
            divisor: 1m
        - path: "cpu_request"
          resourceFieldRef:
            containerName: client-container
            resource: requests.cpu
            divisor: 1m
        - path: "mem_limit"
          resourceFieldRef:
            containerName: client-container
            resource: limits.memory
            divisor: 1Mi
        - path: "mem_request"
          resourceFieldRef:
            containerName: client-container
            resource: requests.memory
            divisor: 1Mi
```

### emptyDir

- emptydir就是pod运行的node上的一个临时目录
- 当pod被从node删除,emptydir会被清空,如果是crash不会被清空
- 可以设置emptyDir.medium字段为Memory,相当于使用node上的一个tmpfs,会被计入内存使用

```yaml
volumes:
- name: cache-volume
  emptyDir:
    #默认为空,代表使用普通磁盘
    medium: Memory
    #默认为不限制,对普通磁盘或Memory都有效
    sizeLimit: 32Mi
```

### hostPath

- hostpath就是挂载node主机上的目录或文件
- 系统不会把hostpath计入资源占用
- 如果挂载的文件或目录不存在,那么kubelet会先创建它们,uid:gid对应kubelet的运行身份,权限为0644
- type
  - 默认为空,挂载目录,如果目标目录不存在,kubelet会先创建再挂载
  - Directory 目标目录必须存在
  - DirectoryOrCreate 和默认空值一样
  - File 挂载文件,文件必须存在
  - FileOrCreate 挂载文件,如果不存在则先touch一个
  - Socket 挂载unix socket,必须存在
  - CharDevice 挂载character device,必须存在
  - BlockDevice 挂载block device,必须存在

```yaml
volumes:
- name: test-volume
  hostPath:
    # directory location on host
    path: /data
    # this field is optional
    type: Directory
```

### persistentVolumeClaim

```yaml
volumes:
  - name: mypd
    persistentVolumeClaim:
      claimName: myclaim
      readOnly: true
```

### projected

- projected可以把4种类型的对象聚合挂载到同一个volume
- 包括 secret configMap downwardAPI serviceAccountToken
- 只能引用和pod相同ns的对象
- 感觉真tm好用

```yaml
volumes:
- name: all-in-one
  projected:
    sources:
    - secret:
        name: mysecret
        items:
          - key: username
            path: my-group/my-username
    - downwardAPI:
        items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "cpu_limit"
            resourceFieldRef:
              containerName: container-test
              resource: limits.cpu
    - configMap:
        name: myconfigmap
        items:
          - key: config
            path: my-group/my-config
    - serviceAccountToken:
        audience: api
        expirationSeconds: 3600
        path: token
```


## tbd

!review

```text
   overhead <map[string]string>
     Overhead represents the resource overhead associated with running a pod for
     a given RuntimeClass. This field will be autopopulated at admission time by
     the RuntimeClass admission controller. If the RuntimeClass admission
     controller is enabled, overhead must not be set in Pod create requests. The
     RuntimeClass admission controller will reject Pod create requests which
     have the overhead already set. If RuntimeClass is configured and selected
     in the PodSpec, Overhead will be set to the value defined in the
     corresponding RuntimeClass, otherwise it will remain unset and treated as
     zero. More info:
     https://git.k8s.io/enhancements/keps/sig-node/20190226-pod-overhead.md This
     field is alpha-level as of Kubernetes v1.16, and is only honored by servers
     that enable the PodOverhead feature.




   runtimeClassName <string>
     RuntimeClassName refers to a RuntimeClass object in the node.k8s.io group,
     which should be used to run this pod. If no RuntimeClass resource matches
     the named class, the pod will not be run. If unset or empty, the "legacy"
     RuntimeClass will be used, which is an implicit class with an empty
     definition that uses the default runtime handler. More info:
     https://git.k8s.io/enhancements/keps/sig-node/runtime-class.md This is a
     beta feature as of Kubernetes v1.14.
```
