# Garbage Collection

- metadata.ownerReferences
  >k8s会使用ownerReferences来表示controller,pod之间owner与dependency的关系  
  >比如deploy创建rs,rs创建pod

    ```yaml
    #rs
        ownerReferences:
        - apiVersion: apps/v1
            blockOwnerDeletion: true
            controller: true
            kind: Deployment
            name: echo
            uid: db619adf-6af8-4432-be5b-80433f37163d
        ---
    #pod
        ownerReferences:
        - apiVersion: apps/v1
            blockOwnerDeletion: true
            controller: true
            kind: ReplicaSet
            name: traefik-ingress-7d6bcd7dc7
            uid: ce60e706-dfd9-4d40-9fb5-996e3f9f8cc3
    ```

- 删除方部
  - cascading deletion
    >就是owner被删除,对应的dependency也同时删除
    - Foreground
      >owner先被标记删除,状态为deletion in progress
      >gc介入开始删除dependency
      >最后删除owner
    - Background
      >owner直接被删除,dependency在后台删除
  - orphaned
    >只删除owner,保留dependency

  ```shell
  #默认cascading deletion
  kubectl delete replicaset my-repset
  #保留dependency
  kubectl delete replicaset my-repset --cascade=false
  ```
  