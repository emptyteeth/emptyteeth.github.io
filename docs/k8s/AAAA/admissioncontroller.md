# admission controller

><https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/>

```shell
kube-apiserver --enable-admission-plugins=
kube-apiserver --disable-admission-plugins=
```

- 是编译进apiserver的
- 分为两种操作类型 验证和修改
- 一个controller至少有一种,可以有两种
- 修改操作总在验证操作之前执行
- 未通过任一操作的请求立即被拒绝

## 默认启用的controller

```shell
kube-apiserver -h | grep enable-admission-plugins
```

1.18默认

- NamespaceLifecycle
- LimitRanger
- ServiceAccount
- TaintNodesByCondition
- Priority
- DefaultTolerationSeconds
- DefaultStorageClass
- StorageObjectInUseProtection
- PersistentVolumeClaimResize
- RuntimeClass
- CertificateApproval
- CertificateSigning
- CertificateSubjectRestriction
- DefaultIngressClass
- MutatingAdmissionWebhook
- ValidatingAdmissionWebhook
- ResourceQuota

## AlwaysPullImages

- 强制修改所有的pod imagePullPolicy为Always
- 在多租户,使用私有镜像的场景,防止在知道镜像名称的前提下使用不应被授权使用的镜像

## CertificateApproval

><https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/>

用于在证书签名请求被批准时,验证批准者的权限

## CertificateSigning

用于在证书签名请求被签发时,验证签发者的权限

## CertificateSubjectRestrictions

在请求CSR时,拒绝所有organization为system:masters的请求

## DefaultStorageClass

- 在pvc被创建时,自动为未指定sc的pvc设置默认的sc
- 如果没有默认sc,那它什么都不干
- 如果默认sc多于一个,那它就拒绝
- 只管create不管update

## DefaultTolerationSeconds

为pod对notready:NoExecute, unreachable:NoExecute 设置默认的容忍时间5分钟

## EventRateLimit

alpha, 设置event rate

## ExtendedResourceToleration

当有拥有扩展资源(gpu等)的专用node时,自动为请求这些扩散资源的pod添加相应的Toleration

## ImagePolicyWebhook

使用外包的image policy服务
将pod所要使用的image发给外部服务review

## LimitPodHardAntiAffinityTopology

This admission controller denies any pod that defines AntiAffinity topology key other than kubernetes.io/hostname in requiredDuringSchedulingRequiredDuringExecution

## LimitRanger

## NamespaceAutoProvision

在请求的ns不存在时自动创建

## NamespaceExists

如果请求的ns不存在就拒绝

## NamespaceLifecycle

- 确保default, kube-system, kube-public不能被删除
- 确保请求的ns不存在就拒绝
- 确保ns在被删除过程中,不能在其中创建新对象

## NodeRestriction

- 限制kubelet的权限

- 前提是kubelet运行身份为
- group: `system:nodes`
- user: `system:node:<nodeName>`

- 限制只能修改自己运行的pod对象
- 限制只能修改自己的node对象
- 不能修改或删除node对象的taints
- 不能删除node对象
- 不能添加删除修改`node-restriction.kubernetes.io/`开头的node label
- 除了以下之外禁止使用其它的`kubernetes.io`或`k8s.io`开头的node label

  - kubernetes.io/hostname
  - kubernetes.io/arch
  - kubernetes.io/os
  - beta.kubernetes.io/instance-type
  - node.kubernetes.io/instance-type
  - failure-domain.beta.kubernetes.io/region
  - failure-domain.beta.kubernetes.io/zone
  - topology.kubernetes.io/region
  - topology.kubernetes.io/zone
  - kubelet.kubernetes.io/-prefixed labels
  - node.kubernetes.io/-prefixed labels

## OwnerReferencesPermissionEnforcement

## PodNodeSelector

- 为ns设置annotation限制pod可以使用的NodeSelector
- 另外设置全局默认可用的NodeSelector

## PodPreset

## PodSecurityPolicy

## PodTolerationRestriction

- 为ns设置annotation限制pod可以使用的Toleration白名单
- 为ns设置annotation设置pod默认Toleration

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: apps-that-need-nodes-exclusively
  annotations:
    scheduler.alpha.kubernetes.io/defaultTolerations: '{"operator": "Exists", "effect": "NoSchedule", "key": "dedicated-node"}'
    scheduler.alpha.kubernetes.io/tolerationsWhitelist: '{"operator": "Exists", "effect": "NoSchedule", "key": "dedicated-node"}'
```

## Priority

## ResourceQuota

## RuntimeClass

## SecurityContextDeny

## ServiceAccount

## StorageObjectInUseProtection

## TaintNodesByCondition

自动为新node添加NotReady, NoSchedule taint

## ValidatingAdmissionWebhook

## MutatingAdmissionWebhook
