# how it works

- <https://kubernetes.io/docs/reference/setup-tools/kubeadm/implementation-details/>
- <https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/>
- <https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/>
- <https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2>
- <https://godoc.org/k8s.io/kubernetes/pkg/kubelet/apis/config#KubeletConfiguration>

## init

### 条件检查

```bash
kubeadm init phase preflight
```

- 内核版本
- cgroup
- cri
- root privilege
- kubelet
- ports
- swap

### 生成证书

><https://kubernetes.io/docs/setup/best-practices/certificates/>

>如果使用外部ca,则需要手动生成这些证书放到相应的位置

```bash
kubeadm init phase certs all
```

- 集群ca
- apiserver服务器证书(ca签发)
  - servicesCIDR中的首个地址,也就是apiserver的clusterip
  - apiserver内部dns名
    >基于以下设置:
    >clusterName: kubernetes
    >dnsDomain: cluster.local
    - kubernetes.default.svc.cluster.local
    - kubernetes.default.svc
    - kubernetes.default
    - kubernetes
  - node name
  - apiserver advertiseAddress
  - controlPlaneEndpoint
  - certSANs参数中自定义的条目
- apiserver to kubelet的客户端证书(ca签发)
  >证书organization为:system:masters
- 用于签发sa的密钥sa.key sa.pub
  >目测和ssh-keygen一个路数
- front-proxy ca
- front-proxy client cert(front-proxy ca签发)

### 生成kubeconfig

```bash
kubeadm init phase kubeconfig all
```

>kubeconfig都由群集ca签发,关键内容为:
>群集ca证书,用于验证服务器身份
>群集地址
>客户端证书/token

- kubelet
  >Subject: O = system:nodes, CN = system:node:nodename
  >这里是kubeadm直接使用ca签发,因为init时不能用bootstrap
  ><https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/>
  ><https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/#limits>
- controller-manager
  >证书cn: system:kube-controller-manager
- scheduler
  >证书cn: system:kube-scheduler
- admin
  >证书organization: system:masters

### 生成control-plane manifest

```bash
kubeadm init phase control-plane all
```

>命名空间: kube-system
>标签: tier:control-plane component:{component-name}
>hostNetwork: true
>controller-manager和scheduler使用127.0.0.1连接apiserver
>apiserver使用127.0.0.1连接local etcd

- apiserver
- controller-manager
- scheduler

### 生成etcd manifest

```bash
kubeadm init phase etcd local
```

>只在未使用外部etcd时生成
>服务和metrics监听地址为127.0.0.1
>集群监听地址为hostip
>hostNetwork: true
>挂载hostPath做为数据存储

### 等待control-plane启动

>这个过程是启动kubelet服务,kubelet再启动所有静态pod

- kubelet启动
  >这里有些细节官方文档中没找到,前面提到kubelet和apiserver互相访问的客户端证书,和apiserver的服务器证书都是群集ca签发的,但是kubelet在启动时生成了自签名的服务器证书,和群集ca没有一毛钱关系,那apiserver怎么访问的kubelet呢,发现是kubelet使用了叫certificate_authorities的tls机制来使用群集ca验证
  ><https://tools.ietf.org/html/rfc5246#section-7.4.4>

  ```shell
  #inspect kubelet server cert
  openssl s_client -showcerts -connect 127.0.0.1:10250

  Acceptable client certificate CA names
  CN = kubernetes
  ```

- 检测kubelet liveness readiness
  >检测localhost:6443/healthz和localhost:10255/healthz/syncloop,分别在40和60秒后判定失败
- 检测apiserver liveness
  >检测localhost:6443/healthz,在4m00m后判定失败

### 上传kubeadm init配置

```bash
 kubeadm init phase upload-config
```

>在kube-system创建名为kubeadm-config的configmap,保存当前init所使用的配置信息,以供之后其它操作参考,比如kubeadm upgrade

### 上传集群证书

>添加`--upload-certs`参数,kubeadm会把集群ca证书密钥加密后上传到集群中(2h过期),然后把解密密码在init完成时打印出来,这样在加入新的master节点时就可以通过这个密码得到集群ca证书密钥
><https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/>

```shell
kubeadm init `--upload-certs`
```

### 标记master节点

```bash
 kubeadm init phase mark-control-plane
```

- 添加标签 `node-role.kubernetes.io/master=""`
- 添加taint `node-role.kubernetes.io/master:NoSchedule`

### config tls-bootstrapping

#### 创建bootstrap token

```yaml
#data内的value全部为base64编码,这里显示的是解码后的内容
kind: Secret
apiVersion: v1
metadata:
  name: bootstrap-token-jwh29g
  namespace: kube-system

data:
  auth-extra-groups: system:bootstrappers:kubeadm:default-node-token
  expiration: 2020-05-18T20:42:39+08:00
  token-id: jwh29g
  token-secret: cvjjdkqw6zq84ah6
  usage-bootstrap-authentication: true
  usage-bootstrap-signing: true

type: bootstrap.kubernetes.io/token
```

>kubeadm使用init参数token(未指定就自动生成)在kube-system中创建名为`bootstrap-token-<token-id>`的secret,这个名字的格式是固定的
>type为`bootstrap.kubernetes.io/token`
>添加组成员身份`system:bootstrappers:kubeadm:default-node-token`
>默认有效期为24小时,时长可自行设置,使用完后可手动删除,或者过期后会被自动删除
>还会在cluster-info中写入token的jwt签名???

#### 授权token访问CSR API

>CSR api用于向apiserver发起证书签名请求,并在请求被批准后获取证书
>kubeadm 创建名为`kubeadm:kubelet-bootstrap`的CRB
>授予`system:bootstrappers:kubeadm:default-node-token`组
>`system:node-bootstrapper`角色权限

```yaml
rules:
  - verbs:
      - create
      - get
      - list
      - watch
    apiGroups:
      - certificates.k8s.io
    resources:
      - certificatesigningrequests
```

#### 设置CSR自动批准

>kubeadm 创建名为`kubeadm:node-autoapprove-bootstrap`的CRB
>授予`system:bootstrappers:kubeadm:default-node-token`组
>`system:certificates.k8s.io:certificatesigningrequests:nodeclient`角色权限

>kubelet在bootstrap后拿到的证书
>O为`system:nodes`
>CN为`system:node:<节点名>`

```yaml
rules:
  - verbs:
      - create
    apiGroups:
      - certificates.k8s.io
    resources:
      - certificatesigningrequests/nodeclient
```

```text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            09:b2:e5:51:f8:41:2d:18:e0:49:a4:2d:89:20:b0:aa
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = kubernetes
        Validity
            Not Before: May  4 11:09:29 2020 GMT
            Not After : May  4 11:09:29 2021 GMT
        Subject: O = system:nodes, CN = system:node:master02
```

#### 设置node证书续期自动批准

>kubeadm 创建名为`kubeadm:node-autoapprove-certificate-rotation`的CRB
>授予`system:nodes`组
>`system:certificates.k8s.io:certificatesigningrequests:selfnodeclient`角色权限

```yaml
rules:
  - verbs:
      - create
    apiGroups:
      - certificates.k8s.io
    resources:
      - certificatesigningrequests/selfnodeclient
```

#### 创建cluster-info

>kubeadm 在kube-public内
>创建名为`cluster-info`的configmap
>包括一个kubeconfig文件,内容只包括集群ca和集群入口
>用于kubeadm join时的集群发现和验证
>创建名为`kubeadm:bootstrap-signer-clusterinfo`的角色,及同名RB授予`system:anonymous`组,该角色只有`get` `cluster-info`权限
>`cluster-info`可被匿名访问,内容并不敏感,但这个入口本身存在ddos风险,公网环境下需要妥善处理

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: 'kubeadm:bootstrap-signer-clusterinfo'
  namespace: kube-public
rules:
  - verbs:
      - get
    apiGroups:
      - ''
    resources:
      - configmaps
    resourceNames:
      - cluster-info
```

```bash
curl -k https://cluster.entrypoint:6443/api/v1/namespaces/kube-public/configmaps/cluster-info
```

### addons
>kube-system

#### kube-proxy

- 创建sa,CRB到`system:node-proxier`
- 创建daemonset

#### core-dns

- 创建sa,CRB到`system:kube-dns`
- 创建deployment
- 创建svc

## join

><https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/>

```bash
kubeadm join k8s.xxx.fun:6444 --token jwh29g.cvjjdkqw6zq84ah6 --discovery-token-ca-cert-hash sha256:886741caac02bcd6418581b18a6173aef2259ce8097a2f579c345283625739ac
```

### join条件检查

>和init差不多

### 发现cluster-info

- kubeadm以insecure方式访问cluster-info
- 使用token验证cluster-info中的jwt签名
- 使用ca-sert-hash验证cluster-info中的ca
- 获取ca并重新建立安全连接,并再次验证与cluster-info中ca的一致性

>搞这么复杂是为了防止中间人攻击

## tls-bootstrap

><https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/>
><https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/>

- kubeadm创建`bootstrap-kubelet.conf`

  ```yaml
  apiVersion: v1
  kind: Config
  clusters:
  - cluster:
      certificate-authority: /etc/kubernetes/pki/ca.pem
      server: https://my.server.example.com:6443
    name: bootstrap
  contexts:
  - context:
      cluster: bootstrap
      user: kubelet-bootstrap
    name: bootstrap
  current-context: bootstrap
  preferences: {}
  users:
  - name: kubelet-bootstrap
    user:
      token: 07401b.f395accd246ae52d
  ```

- kubelet通过bootstrap token向apiserver发送CSR签名请求

  >kubelet启动时如果未找到kubelet.conf,就会使用bootstrap-kubelet.conf

  ```shell
   --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf
  ```

  >此时kubelet的身份是`system:bootstrappers:kubeadm:default-node-token`

- kubelet拿到签名后的证书,生成新的kubelet.conf,bootstrap-kubelet.conf被kubeadm删除
- kubeadm重新启动kubelet,通过新的kubelet.conf与apiserver建立连接

  >此时kubelet的身份是`system:nodes`
