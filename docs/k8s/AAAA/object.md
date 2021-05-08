# ojbect

- object是对目标状态的定义,k8s会持久存储object,并试图使系统运行状态符合object的定义
- object的基本信息
  - apiVersin 即所属的api组
  - kind 所属的类型
  - metadata
    - namespace 所属的ns,集群级ojbect不需要这个
    - name 在同类型中唯一,ns级的则是在同ns内同类型中唯一
    - uid 系统为每个object生成的uuid,在集群范围内的所有object中唯一
    - selfLink 对象所对应的api路径
  - spec object的具体规格,大部分obect使用spec字段,有一些使用其它字段
  - status 系统对于实际运行状态与object之间差异的描述

## 操作object

><https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config>

- 对于object操作本质上就对相应api url进行post get put patch delete操作,附上相应的json格式的object定义
- kubectl封装的一些常用操作
  - create 用来创建部分常用的对象
  - expose 专门用于创建svc
  - run 用于创建pod
  - set 用于对一部分对象设置特定的属性
  - get 获取一个或多个对象的信息
  - edit 编辑对象信息
  - delete 删除对象
  - rollout scale autosecale 针对deploy的一些操作
  - label 更新对像的label
  - annotaion更新对象的annotion
- kubectl使用yaml/json
  - get 获取文件中object对应的live信息
  - diff 对比文件与live
  - apply 应用文件中的对象到live
    - 如果live不存在,那么相当于create
    - 每次apply,文件中的信息会被存入last
    - apply背后的逻辑是进行patch操作,根据文件 live last-applied-configuration来决定如何patch
      - 字段不存在于文件,存在于last,不考虑live,字段从live中清除
      - 字段存在于文件,不存在于live中,字段在live中被创建
      - 字段存在于文件和live中,值不同,文件中的值被应用到live
        - 对于string integer boolean这种primitive类型字段直接replace
        - 对于map类型的字段以key为基准进行merge
        - 对于list类型的字段则根据元素字段的类型进行replace或merge
        - 如果list中元素全部为primitive,那么list被整体replace
        - 对于复杂的list,以patchMergeKey为基准进行merge,不同字段的patchMergeKey:<https://github.com/kubernetes/api/blob/master/core/v1/types.go>
  - create 创建文件中定义的对象,文件中未定义的默认值会被应用到live
  - patch 对字段操作
  - replace 用文件替换live,所有文件中没有定义的字段都会从live中剔除,如果文件没有定义具有默认值但是是required的字段时会出错
  - delete 从live删除文件中定义的对象
  - kustomize

```text
#primitive
file        live        last        Action
Yes         Yes         -           Set live to configuration file value.
Yes         No          -           Set live to local configuration.
No          -           Yes         Clear from live configuration.
No          -           No          Do nothing. Keep live value.
#map
file        live        last        Action
Yes         Yes         -           Compare sub fields values.
Yes         No          -           Set live to local configuration.
No          -           Yes         Delete from live configuration.
No          -           No          Do nothing. Keep live value.
```

## label

- label是一种对object进行逻辑分组的机制
- label本身的内容对系统没有意义,系统关心的是基于label进行分组的逻辑
- k:v形式,key支持一个可选的prefix,以/分隔,prefix必须是dns模式 abc.com/key1: value1
- kubernetes.io k8s.io这两个prefix为系统保留

## label selector

- 多条件逻辑
  >多条件间总是与逻辑,不存在或逻辑
- 基于等式查询
  >= == !=,前两种是一样的,!=表达不等于
  >abc.com/key1 = value1
- set-based查询
  >三种操作符 in notin exists
  >in 选择value存在于给定的一个列表中
  >notin 选择value不存在于给定的一个列表中
  >exists 选择key存在,不关心value

  ```text
    environment in (production, qa)
    tier notin (frontend, backend)
    partition
    !partition
  ```

  ```shell
  kubectl get pods -l 'environment in (production),tier in (frontend)'
  ```

  ```yaml
  selector:
    matchLabels:
     component: redis
    matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
  ```

## Field Selectors

通过字段来选择object

```shell
kubectl get services  --all-namespaces --field-selector metadata.namespace!=default
kubectl get statefulsets,services --all-namespaces --field-selector metadata.namespace!=default
kubectl get pods --field-selector=status.phase!=Running,spec.restartPolicy=Always
```
