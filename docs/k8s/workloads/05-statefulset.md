# statefulset

><https://kubernetes.io/docs/concepts/workloads/controllers/statefulset>

- pod名称规则
  >pod-0,pod-1,pod-2 ... pod-(N-1)
- 每个pod都被分配一个独有的dns记录
  >podname.svcname.nsname.svc.cluster.local
- 每个pod都会被加上hash标签,和独有的pod-name标签
  >controller-revision-hash=gitea-5878c49d9b  
  >statefulset.kubernetes.io/pod-name=gitea-0
- 使用pvc时,pod被删除,对应的pvc被保留
- 使用pvc时,sts被删除,所有的pv被保留
- 如果更新失败需要回滚时,更新配置后需要手动删除失败的pod

## podManagementPolicy `<string>`

管理pod在被创建和scaling时的次序策略

- OrderedReady (默认)
  >pod被创建时按顺序逐个创建,必须等前一个ready  
  >pod被删除时按反向顺序逐个删除,必须等前一个删除完成,并且后面的pod必须ready
- Parallel
  >等不及模式,同时进行,只影响scaling,不影响update

## serviceName `<string> -required-`

sts需要指定一个存在的svc

## updateStrategy `<Object>`

- type `<string>`
  - RollingUpdate (默认)
    >按照反向顺序逐个更新,必须等ready后才会进行下一个
  - OnDelete
    >需要手动删除pod,controller才会创建更新的pod
- rollingUpdate `<Object>`
  - partition `<integer>`
    >设置一个序号,所有小于此序号的pod不会被更新

## volumeClaimTemplates `<[]Object>`

>设置pvc模版,会为每一个pod分配对应的pvc
