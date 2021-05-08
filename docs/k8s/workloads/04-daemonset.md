# daemonset

- pod命名
  >dsname-随机字符

- 默认toleration

    >controller默认为pod设置toleration

    ```yaml
    tolerations:
    - effect: NoExecute
        key: node.kubernetes.io/not-ready
        operator: Exists
    - effect: NoExecute
        key: node.kubernetes.io/unreachable
        operator: Exists
    - effect: NoSchedule
        key: node.kubernetes.io/disk-pressure
        operator: Exists
    - effect: NoSchedule
        key: node.kubernetes.io/memory-pressure
        operator: Exists
    - effect: NoSchedule
        key: node.kubernetes.io/pid-pressure
        operator: Exists
    - effect: NoSchedule
        key: node.kubernetes.io/unschedulable
        operator: Exists
    ```

- node affinity

    >controller为每个pod设置相应的node affinity  
    >隐含使用默认scheduler调度

    ```yaml
    affinity:
        nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchFields:
              - key: metadata.name
                  operator: In
                  values:
                  - master03
    ```

- pod label

    >controller为pod添加额外label

    ```yaml
    labels:
        controller-revision-hash: 57df69cf47
        pod-template-generation: "1"
    ```

## updateStrategy `<Object>`

- type `<string>`
  - RollingUpdate (默认)
    >滚动更新
  - OnDelete
    >需要手动删除pod,controller才会创建更新的pod
- rollingUpdate `<Object>`
  - maxUnavailable `<integer>`
    >update时可接受的不可用pod数量
    >默认为1,不可为0,可使用百分比
