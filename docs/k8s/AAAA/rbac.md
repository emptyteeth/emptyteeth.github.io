# rabc authorization

- <https://kubernetes.io/docs/reference/access-authn-authz/rbac/>
- <https://kubernetes.io/docs/reference/access-authn-authz/controlling-access/>
- <https://kubernetes.io/docs/reference/access-authn-authz/authorization/>
- api组 `rbac.authorization.k8s.io`
- 启用参数 `--authorization-mode=RBAC`

```shell
curl http://127.0.0.1:8001/apis/rbac.authorization.k8s.io/v1
```

## role

- role是对单一命名空间内api权限集合的定义
- 不存在拒绝的逻辑
- role的做用范围仅限命名空间内
- 可以定义多组rule
- rule明确定义了对于何种api组的何种资源进行何种操作

- apiGroups
  - 列表,指定api组
  - ["", "apps", "events.k8s.io"]
  - *代表所有
  - ""代表核心组
- nonResourceURLs
  - 列表,指定非资源类url
  - 非命名空间资源,只能用于clusterrole

    ```yaml
    rules:
    - nonResourceURLs: ["/healthz", "/healthz/*"] # '*' in a nonResourceURL is a suffix glob match
      verbs: ["get", "post"]
    ```

- resourceNames
  - 列表,可选项,指定资源名称
  - 指定资源名称进行过滤,不定义则相当于*
  - 不适用于create操作
- resources
  - 列表,指定资源类型
  - 与api中的object name对应
  - *代表所有
- verbs
  - 列表,定义操作类型
  - ["get", "list", "watch", "create", "update", "replace", "patch", "delete"]
  - *代表全部

```shell
#使用kubectl create时只能定义一组rule
#resource可以用.指定api组,默认是核心组
kubectl create role dev01 -n dev01
--verb="*"
--resource="*"

kubectl create role dev01 -n dev01
--verb=get,list
--verb=watch
--verb=create
--resource=pods,services
--resource-name=test

kubectl create role dev01 -n dev01
--verb='*'
--resource=deployments.apps
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups:
  - app
  resourceNames:
  - abc
  resources:
  - deployments
  - statefulsets/status
  verbs:
  - '*'
- apiGroups: [""] # ""指核心 API 组
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

## clusterrole

- clusterrole是对集群内api权限集合的定义
- clusterrole的做用范围是整个集群
- 可以设置nonResourceURLs
- 可以设置非命名空间的资源
- 其它和role一样
- 另外可以当做通用role模版用,在不同命名空间使用rolebinding引用,这样权限可以统一设置,但还是会被限制在相应的命名空间内

  ```shell
  kubectl create clusterrole "foo" --verb=get --non-resource-url=/logs/*

  kubectl create clusterrole monitoring --aggregation-rule="rbac.example.com/aggregate-to-monitoring=true"
  ```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: secret-reader
rules:
- apiGroups: [""]
  #
  # at the HTTP level, the name of the resource for accessing Secret
  # objects is "secrets"
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

## rolebinding

- rolebinding是把role的权限授权给subjects
- 命名空间资源,定义的内容只在命名空间内有效
- 创建后不能更改roleRef

- subjects
  - 对授权对象的引用,可以定义多个
  - kind: User, Group, ServiceAccount

- roleRef
  - 对role的引用,只能定义1个
  - kind: Role, ClusterRole

```shell
kubectl create rolebinding bob-admin-binding --clusterrole=admin --user=bob --namespace=acme

kubectl create rolebinding myapp-view-binding --clusterrole=view --serviceaccount=acme:myapp --namespace=acme

kubectl create rolebinding myappnamespace-myapp-view-binding --clusterrole=view --serviceaccount=myappnamespace:myapp --namespace=acme
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "jane" to read pods in the "default" namespace.
# You need to already have a Role named "pod-reader" in that namespace.
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# You can specify more than one "subject"
- kind: User
  name: jane # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
  #对于kind为sa,该值默认为"",也就是核心组,对于user和group,默认为rbac.authorization.k8s.io
  #意思就是说这个是全自动的,不需要设置
roleRef:
  # "roleRef" specifies the binding to a Role / ClusterRole
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "dave" to read secrets in the "development" namespace.
# You need to already have a ClusterRole named "secret-reader".
kind: RoleBinding
metadata:
  name: read-secrets
  #
  # The namespace of the RoleBinding determines where the permissions are granted.
  # This only grants permissions within the "development" namespace.
  namespace: development
subjects:
- kind: User
  name: dave # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

## clusterrolebinding

- clusterrolebinding是把clusterrole的权限授权给subjects
- 集群资源,在整个集群内有效
- 创建后不能更改roleRef
- roleRef只能引用ClusterRole
- 其它和rolebindding一样

```shell
kubectl create clusterrolebinding root-cluster-admin-binding --clusterrole=cluster-admin --user=root

kubectl create clusterrolebinding kube-proxy-binding --clusterrole=system:node-proxier --user=system:kube-proxy

kubectl create clusterrolebinding myapp-view-binding --clusterrole=view --serviceaccount=acme:myapp
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# This cluster role binding allows anyone in the "manager" group to read secrets in any namespace.
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

## clusterrole权限聚合

可以为clusterrole设置权限自动聚合其它clusterrole的权限

```yaml
#使用aggregationRule设置要聚合的ClusterRole匹配规则
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: [] # The control plane automatically fills in the rules
---
#在要聚合的ClusterRole设置相应的label
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
# When you create the "monitoring-endpoints" ClusterRole,
# the rules below will be added to the "monitoring" ClusterRole.
rules:
- apiGroups: [""]
  resources: ["services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
```

## 集群默认的权限聚合

admin, edit, view

```shell
kubectl get clusterrole -o json |jq '.items[].aggregationRule' |grep -v null
```

## 用户和组

- 自定义用户和组应避免使用system:开头

- sa用户
  - system:serviceaccount:dev:view 代表dev命名空间内名为view的sa
- 系统组
  - system:serviceaccounts组 包括集群内所有的sa
  - system:serviceaccounts:dev组 包括dev命名空间内的所有sa
  - system:authenticated组 包括所有通过认证的用户
  - system:unauthenticated组 包括所有未认证的用户

## 默认clusterrole, clusterrolebinding

- 系统会默认创建基本的clusterrole, clusterrolebinding
- 它们大部分都以`system:`开头
- 它们都有`kubernetes.io/bootstrapping=rbac-defaults`标签
- 它们都有`rbac.authorization.kubernetes.io/autoupdate=true`标签
  - apiserver在每次启动时都会检查和修复它们缺少的设置,除非把相应的这个标签设为false

## API discovery roles

## 预设用户role

- cluster-admin
  - 超级管理员角色,预设绑定system:master组,full access

- admin
  - 管理员角色,不能修改ns和quota,适合使用rolebinding做为ns管理员

- edit
  - 比admin少了role和rolebinding读写权限,但可以读secret,privilege escalation

- view
  - 比edit少了写权限,但可以读secret,privilege escalation

## 预设核心组件role

- system:kube-scheduler
  - 绑定`system:kube-scheduler`用户
- system:volume-scheduler
  - 绑定`system:kube-scheduler`用户
- system:kube-controller-manager
  - 绑定`system:kube-controller-manager`用户
- system:node
  - 绑定为`空`,只是为了兼容
  - 这个功能是可以修改所有node相关设置
  - 被Node authorizer和NodeRestriction admission取代
- system:node-proxier
  - 绑定`system:kube-proxy`用户

## 其它预设组件role

- system:auth-delegator
  - 绑定为`空`,用于使用外部认证授权
- system:heapster
  - 绑定为`空`,deprecated
- system:kube-aggregator
  - 绑定为`空`,用于kube-aggregator
- system:kube-dns
  - 绑定为`空`,用于kube-dns,用coredns的话这个就为空
- system:kubelet-api-admin
  - 绑定为`空`,允许full access kubelet api
- system:node-bootstrapper
  - 用于node bootstrap时申请证书
  - 使用kubeadm的话会绑定给`system:bootstrappers:kubeadm:default-node-token`组
  - 也就是kubeadm生成的join token所属的组
- system:node-problem-detector
  - 绑定为`空`,node-problem-detector
- system:persistent-volume-provisioner
  - 绑定为`空`,用于dynamic volume provisioners

## Roles for built-in controllers

- controller-manager启动时如果使用`--use-service-account-credentials`参数
- 那么每个controller都会使用单独的sa,对应不同的role和binding,这些role都以`system:controller:`开头

- system:controller:attachdetach-controller
- system:controller:certificate-controller
- system:controller:clusterrole-aggregation-controller
- system:controller:cronjob-controller
- system:controller:daemon-set-controller
- system:controller:deployment-controller
- system:controller:disruption-controller
- system:controller:endpoint-controller
- system:controller:expand-controller
- system:controller:generic-garbage-collector
- system:controller:horizontal-pod-autoscaler
- system:controller:job-controller
- system:controller:namespace-controller
- system:controller:node-controller
- system:controller:persistent-volume-binder
- system:controller:pod-garbage-collector
- system:controller:pv-protection-controller
- system:controller:pvc-protection-controller
- system:controller:replicaset-controller
- system:controller:replication-controller
- system:controller:resourcequota-controller
- system:controller:root-ca-cert-publisher
- system:controller:route-controller
- system:controller:service-account-controller
- system:controller:service-controller
- system:controller:statefulset-controller
- system:controller:ttl-controller

## 安全

- 用户不能创建权限超出自身拥有权限的role或clusterrole

  - 除非用户本身拥有对roles或clusterroles的escalate操作权限

  ```yaml
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["clusterroles"]
    verbs: ["escalate"]
  ```

- 用户不能创建权限引用超出自身拥有权限的rolebinding或clusterrolebinding

  - 除非用户本身拥有对roles或clusterroles的bind操作权限

  ```yaml
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["roles"]
    verbs: ["bind"]
  ```

## kubectl auth reconcile

## verbs

- `get` 是对某个对象的读取权限
- `list` 是对某一类对象的读取权限,在没有get权限的情况下,相当于取整个列表,列表中元素信息还是完整的,只是不能单独取列表中的元素
- `watch` 是动态获取对象更新状态的权限
- `create` 创建
- `update` 更新
- `delete` 删除
- `use` verb on podsecuritypolicies resources in the policy API group
- `bind` and `escalate` verbs on roles and clusterroles resources in the rbac.authorization.k8s.io API group.
- `impersonate` verb on users, groups, and serviceaccounts in the core API group, and the userextras in the authentication.k8s.io API group
