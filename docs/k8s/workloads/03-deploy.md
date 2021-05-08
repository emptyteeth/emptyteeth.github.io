# deployment

><https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>

deploy创建和管理rs,rs创建和管理pod
deploy向rs传递template, replicas, selector
传递给rs的selector中会额外加入pod-template-hash
deploy创建的rs name为`deploy name`-`hash值`

## replicaset

><https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/>

- rs创建的pod name为`rs name`-`随机字符`
- rs通过selector配匹labels(包括hash)来接收pod

  ```yaml
  selector:
    matchLabels:
      k8s-app: echo
      pod-template-hash: 64477b95fb
  ```

## selector

在apps/v1版本中selector在创建后不能更改
除了matchLabels之外, 还可以使用多个matchExpressions,需要同时满足

## rollout

### strategy `<Object>`

- type `<string>`
  >更新策略,默认为RollingUpdate,另一个选项是Recreate  
  >Recreate就是先排空当前版本的rs,再创建新的rs  
  >也就是说原有pod全部被干掉,之后新的pod才会被创建  
  >rollingUpdate是根据滚动更新策略,交替操作新旧两个版本rs,达到滚动更新的目的  
  >也就是说更新过程中可用pod的数量会根据滚动更新策略保持在相应的范围内
- rollingUpdate `<Object>`
  >滚动更新策略,这里设置的是更新过程中,可用pod的总数(包含新旧两个版本)  
  >与目标pod数量之间,可接受的差别范围,可使用百分比或数字,两个值不能同时为0
  - maxSurge
    >超出目标数量,默认25%
  - maxUnavailable
    >少于目标数量,默认25%

### Rollover

>当更新过程中,又有来了一波更新,controller会直接创建新的rs,前两次的rs会被一起scale down

### Rolling Back

>滚回到之前的rs版本  
>controller根据pod-template-hash创建不同的rs,只有pod template改变才会有相应的rs版本给你滚
>默认保留10个版本

```yaml
spec:
    revisionHistoryLimit: 3
```

```shell
#查看rs版本
kubectl rollout history deployment.v1.apps/nginx-deployment
#滚回上一个版本
kubectl rollout undo deployment.v1.apps/nginx-deployment
#滚加指定版本
kubectl rollout undo deployment.v1.apps/nginx-deployment --to-revision=2
```

### Pausing and Resuming

rollout的操作可以被暂停和恢复,在这之间所有的rollout操作不会立即生效,直到恢复

```shell
kubectl rollout pause deployment.v1.apps/nginx-deployment
kubectl rollout resume deployment.v1.apps/nginx-deployment
```

```yaml
spec:
    pause: true
```

## Scaling

```shell
kubectl scale deployment.v1.apps/nginx-deployment --replicas=10
kubectl autoscale deployment.v1.apps/nginx-deployment --min=10 --max=15 --cpu-percent=80
```

### Proportional scaling

当在rollout过程中进行了scale up,那么默认多出来的pod都会计入新的rs里
如果使用Proportional scaling,那么新旧rs会按所拥有的pod比例同时scale up

## zzzz

```yaml
spec:
    #pod ready之后多少被计入available,默认为0,就是马上
    minReadySeconds:
    #被标示为ProgressDeadlineExceeded状态的时间,默认600
    progressDeadlineSeconds:
```
