# arch

## node

node包括3个组件:kubelet cri kube-proxy

- 添加node到集群的方式有两种
  - kubelet自动向集群注册node对象
    >kubelet默认参数`--register-node`,会在启动时会向集群注册node
    >如删除了node对象,那么kubelet在重启后会重新注册
  - 手工创建node对象
    - 使用`--register-node=false`参数启动kubelet
    - 创建node对象

    ```yaml
    apiVersion: v1
    kind: Node
    metadata:
      name: master03
      #需要为node写好基本的label,手动注册的话这些不会自动生成
      #不然的话node虽然显示ready,但scheduler无法把任何pod调度到node上
      labels:
        beta.kubernetes.io/arch: arm64
        beta.kubernetes.io/os: linux
        kubernetes.io/arch: arm64
        kubernetes.io/hostname: master03
        kubernetes.io/os: linux
    ```

    - kubelet会获取node controller分配的podCIDR,写入node.spec

    ```yaml
    spec:
    podCIDR: 10.244.1.0/24
    podCIDRs:
    - 10.244.1.0/24
    ```

    - kubelet会自动上报一些node信息

    ```yaml
    status:
      daemonEndpoints:
        kubeletEndpoint:
          Port: 10250
      addresses:
      - address: 10.11.1.23
        type: InternalIP
      - address: master03
        type: Hostname
      capacity:
        cpu: "4"
        ephemeral-storage: 61098136Ki
        memory: 3884368Ki
        pods: "110"
      nodeInfo:
        architecture: arm64
        bootID: df544e2e-6300-4f16-b535-e3c98d5ffb08
        containerRuntimeVersion: containerd://1.3.3-0ubuntu2
        kernelVersion: 5.4.0-1012-raspi
        kubeProxyVersion: v1.18.3
        kubeletVersion: v1.18.3
        machineID: 77b80a8eb56d4c3e9d9d65bcecc80845
        operatingSystem: linux
        osImage: Ubuntu 20.04 LTS
        systemUUID: 77b80a8eb56d4c3e9d9d65bcecc80845

- Node status
  - Addresses
    >kubelet默认会自动探测ip地址和hostname
    >也可以使用参数或配置文件来指定
  - Conditions
    >node controller负责维护这些信息
    >node lifecycle controller根据这些状态自动为node标记taint
    - Ready
      >true为健康可调度
      >false为不健康不可调度
      >unknown表示node controller在`node-monitor-grace-period`(controller-manager)这个时间内无法联系到node 默认40秒
      >如果ready处于false或unknown超过`pod-eviction-timeout`(controller-manager)这个时间,默认5分钟,node上的pod会被标记删除,并处于等待node确认删除的状态,如果node无法再回来,这些pod需要手动删除
    - DiskPressure  磁盘空间不足
    - MemoryPressure  内存不足
    - PIDPressure PID资源不足
    - NetworkUnavailable  网络故障
    - heartbeats
      >Heartbeats包括两种形式
      - lease
        >kubelet在kube-node-lease命名空间内创建并更新一个与node对应的lease对象
        >这是一个轻量级的heartbeat,只是确认存活,默认10秒更新
      - NodeStatus
        >通过node对象,默认5分钟一次

- Node controller
  - 负责为node分配podCIDR
  - 负责监控node健康状态,维护node Conditions
  - 负责与cloud provider同步node信息

## 集群组件通信

- etcd做为集群状态的持久存储与apiserver单线联系
- apiserver做为集群api的入口
  >连接多节点etcd集群可以使用etcd集群lb入口,或是设置所有节点入口
  >--etcd-servers=
- 集群组件和其它客户端使用同一套api与apiserver通信
  >对于多实例apiserver,一般是访问lb入口,具体访问那一个实例没有影响
- 多实例的controller-manager和scheduler会自动进行leader elect
- apiserver访问kubelet
  >apiserver只会主动与两个组件通信,一个是etcd,另一个是kubelet
  >与kubelet通信是为了实现3个功能:

    1. 提取pod log
    2. attatch到pod
    3. 实现port-forward

  - 不验证kubelet ca
    >在kubeadm的部署中,为了便于证书管理,kubelet使用自签名ca,同时设置信任集群ca
    >这时apiserver使用集群ca签发的客户端证书访问kubelet,不验证kubelet ca
    >这种情况下就有可能出现中间人攻击
  - 验证kubelet ca
    >使用`--kubelet-certificate-authority`参数
  - 使用安全隧道
    >在apiserver kubelet之间先建立安全隧道再进行连接
    >适用于不验证kubelet ca和复杂网络架构时使用
    >方式有两种,ssh tunnel(deprecated)和Konnectivity

## controller

- controller-manager内置controller
  - attachdetach, bootstrapsigner, cloud-node-lifecycle, clusterrole-aggregation, cronjob, csrapproving, csrcleaner, csrsigning, daemonset, deployment, disruption, endpoint, endpointslice, garbagecollector, horizontalpodautoscaling, job, namespace, nodeipam, nodelifecycle, persistentvolume-binder, persistentvolume-expander, podgc, pv-protection, pvc-protection, replicaset, replicationcontroller, resourcequota, root-ca-cert-publisher, route, service, serviceaccount, serviceaccount-token, statefulset, tokencleaner, ttl, ttl-after-finished
- cloud-controller-manager内置controller
  - cloud-node, cloud-node-lifecycle, route, service
- controller就是一个不断重复的控制循环,它们监听特定的object,根据不同的控制逻辑做出反应以达到控制逻辑设定的目标
