# api

- <https://kubernetes.io/docs/concepts/overview/kubernetes-api/>
- <https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/>
- <https://kubernetes.io/docs/tasks/administer-cluster/access-cluster-services/>

## api版本

>api版本特指api组的版本

- alpha v1alpha1
- beta v2beta2
- stable v1

## api组

- the core group
  >路径 `api/v1`
  >yaml文件 `apiVerison: v1`
- the named group
  >路径 `/apis/$GROUP_NAME/$VERSION`
  >yaml文件 `apiVersion: $GROUP_NAME/$VERSION`

>另外可以通过CRD来扩展api

## 启用/停用api组

>可以通过apiserver参数--runtime-config来设置

```text
--runtime-config=batch/v1=false,batch/v2alpha1
```

>单独设置其内部指定资源的启用,只有`extensions/v1beta1`组可以这么干

```text
--runtime-config=extensions/v1beta1/deployments=true,extensions/v1beta1/daemonsets=true
```

## kubectl proxy

><https://kubernetes.io/docs/tasks/administer-cluster/access-cluster-services/>

```text
http://kubernetes_master_address/api/v1/namespaces/namespace_name/services/[https:]service_name[:port_name]/proxy

curl http://127.0.0.1:8001/api/v1/namespaces/default/services/echo/proxy/
curl http://127.0.0.1:8001/api/v1/namespaces/default/pods/echo-7478dcd959-w7v2q/proxy/
```

## Extending the Kubernetes API

### CRD

https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/
https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/

- CRD就是向api-server注册自定义对象,之后就可以像内置对象一样通过api-server进行CURD操作
- 注册CRD,向apiextensions.k8s.io API提交CustomResourceDefinition对象
- CustomResourceDefinition定义了自定义对象的名称,api组名称,和对象的spec,status,subresources
- api-server根据CustomResourceDefinition生成相应的api endpoint,并和内置对象一样负责它们的访问和存储
- 如果CustomResourceDefinition被删除,api-server会删除所有相应的自定义对象及相应的api endpoint
- 在定义CustomResourceDefinition时,schema,status,subresources是可选的
- schema用于定义CRD spec中可用字段的名称和取值规范,如果不定义则api-server不对其进行验证
- status定义CRD的状态字段,subresources可以定义/status和/scale两种子资源

### aggregation layer

- <https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/>
- <https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/>
- <https://kubernetes.io/docs/tasks/extend-kubernetes/setup-extension-api-server/>

- aggregation layer就是用api-server充当代理,将特定api组的访问转发到自定义的extension-api-server或service catalog
- 注册APIService,向apiregistration.k8s.io API提交APIService对象
- APIService对象定义了需要转发的api组,和转发到的目标service
- api-server启动时通过以下参数来配置如何与extension-api-server通信

```shell
#api-server访问extension-api-server使用的证书,key,ca
#api-server访问不同的extension-api-server时均使用同一套证书
--proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
--proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
--requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
#告诉extension-api-server需要验证证书中的CN,如果为空则不验证
--requestheader-allowed-names=front-proxy-client
#向extension-api-server传递username,group,extra信息时使用的http header名
--requestheader-extra-headers-prefix=X-Remote-Extra-
--requestheader-group-headers=X-Remote-Group
--requestheader-username-headers=X-Remote-User
#如果api-server没有运行在有kube-proxy的node上,无法直接访问service ip
#使用这个参数让api-server使用endpoint ip来访问extension-api-server
--enable-aggregator-routing=true
```

- api-server在kube-system中创建名为extension-apiserver-authentication的configmap
- api-server在kube-system中创建名为extension-apiserver-authentication-reader的role授权访问上面的configmap
- extension-api-server需要通过sa绑定这个role,来读取configmap中的信息
- configmap中的信息包括:

```shell
requestheader-extra-headers-prefix
requestheader-group-headers
requestheader-username-headers
requestheader-allowed-names
#这个是base64编码的群集CA
client-ca-file
#这个是base64编码的front-proxy-ca
requestheader-client-ca-file
```

- apiserver在收到请求后对请求的api路径进行auth,authz验证
- 在转发api时,使用front-proxy-client证书访问extension-apiserver
- 并在http header中加入user,group,extra信息

- extension-apiserver在收到转发来的请求时,会使用从configmap中获取的信息
- 来验证请求的证书,CN,以及从相应的http header中得到user,group,extra信息
- 在执行api请求前,extension-apiserver还需要对请求的资源进行authz验证
- 这个授权验证需要发送SubjectAccessReview到api-server来代为完成
- 发送SubjectAccessReview这个操作需要绑定system:auth-delegator这个clusterrole
- 在授权验证完成后,extension-apiserver执行api请求并反回结果

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v2beta2.acb.com
spec:
  #api组名
  group: abc.com
  #api版本
  version: v2beta2
  #api组优先级,数字越大优先级越高
  groupPriorityMinimum: 100
  #同组的不同版本的api优先级,数字越大优先级越高
  #如果同组不同版本的api优先级相同,那么优先级按ga/beta/alpha,数字,字母排序
  #v10, v2, v1, v11beta2, v10beta3, v3beta1, v12alpha1, v11alpha2, foo1, foo10
  versionPriority: 100
  service:
    #目标service所在命名空间
    namespace: abc
    #目标service名称
    name: cba
    #目标端口,默认443
    port: 443
  #验证目标服务器证书的CA
  caBundle: <pem encoded ca cert that signs the server cert used by the webhook>
  #跳过验证目标服务器证书
  insecureSkipTLSVerify: true
```

## service catalog

https://kubernetes.io/docs/concepts/extend-kubernetes/service-catalog/

## tbd

```shell
k api-versions

k api-resources
--namespaced=true
- o wide


curl http://127.0.0.1:8001/api/v1/ |jq '.resources[].name'

"bindings"
"componentstatuses"
"configmaps"
"endpoints"
"events"
"limitranges"
"namespaces"
"namespaces/finalize"
"namespaces/status"
"nodes"
"nodes/proxy"
"nodes/status"
"persistentvolumeclaims"
"persistentvolumeclaims/status"
"persistentvolumes"
"persistentvolumes/status"
"pods"
"pods/attach"
"pods/binding"
"pods/eviction"
"pods/exec"
"pods/log"
"pods/portforward"
"pods/proxy"
"pods/status"
"podtemplates"
"replicationcontrollers"
"replicationcontrollers/scale"
"replicationcontrollers/status"
"resourcequotas"
"resourcequotas/status"
"secrets"
"serviceaccounts"
"services"
"services/proxy"
"services/status"

curl http://127.0.0.1:8001/apis/apps/v1/ |jq '.resources[].name'

"controllerrevisions"
"daemonsets"
"daemonsets/status"
"deployments"
"deployments/scale"
"deployments/status"
"replicasets"
"replicasets/scale"
"replicasets/status"
"statefulsets"
"statefulsets/scale"
"statefulsets/status"
```
