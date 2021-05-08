# k8s cheaty way

## control plane

### apiserver lb

```text
listen k8sapi
    bind :8888
    mode tcp
    server master01 10.11.11.250:6443 check
   #server masterN
```

### pki

```shell
~~openssl genrsa -out ca.key~~
~~openssl req -new -nodes -key ca.key -subj "/CN=kubernetes" -x509 -out ca.crt -days 365~~
#ahhhhhhhh!! fuck this shit! let's kubeadm
kubeadm init phase certs all --control-plane-endpoint 10.11.11.250:8888
#--apiserver-cert-extra-sans
```

### etcd

```shell
#kubeadm init phase etcd local
mkdir /var/lib/etcd
etcd --advertise-client-urls=https://10.11.11.250:2379 \
--cert-file=/etc/kubernetes/pki/etcd/server.crt \
--client-cert-auth=true \
--data-dir=/var/lib/etcd \
--initial-advertise-peer-urls=https://10.11.11.250:2380 \
--initial-cluster=ubuntu=https://10.11.11.250:2380 \
--key-file=/etc/kubernetes/pki/etcd/server.key \
--listen-client-urls=https://127.0.0.1:2379,https://10.11.11.250:2379 \
--listen-metrics-urls=http://127.0.0.1:2381 \
--listen-peer-urls=https://10.11.11.250:2380 \
--name=ubuntu \
--peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt \
--peer-client-cert-auth=true \
--peer-key-file=/etc/kubernetes/pki/etcd/peer.key \
--peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt \
--snapshot-count=10000 \
--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

### apiserver

```shell
#kubeadm init phase control-plane apiserver
kube-apiserver \
--advertise-address=10.11.11.250 \
--allow-privileged=true \
--authorization-mode=Node,RBAC \
--client-ca-file=/etc/kubernetes/pki/ca.crt \
--enable-admission-plugins=NodeRestriction \
--enable-bootstrap-token-auth=true \
--etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt \
--etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt \
--etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key \
--etcd-servers=https://10.11.11.250:2379 \
--insecure-port=0 \
--kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt \
--kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key \
--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname \
--proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt \
--proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key \
--requestheader-allowed-names=front-proxy-client \
--requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt \
--requestheader-extra-headers-prefix=X-Remote-Extra- \
--requestheader-group-headers=X-Remote-Group \
--requestheader-username-headers=X-Remote-User \
--secure-port=6443 \
--service-account-key-file=/etc/kubernetes/pki/sa.pub \
--service-cluster-ip-range=10.96.0.0/12 \
--tls-cert-file=/etc/kubernetes/pki/apiserver.crt \
--tls-private-key-file=/etc/kubernetes/pki/apiserver.key \
--etcd-prefix=/cluster01
```

### kubeconfig

```shell
kubeadm init phase kubeconfig all --control-plane-endpoint 10.11.11.250:8888
```

### controller-manager

```shell
#kubeadm init phase control-plane controller-manager
kube-controller-manager \
--allocate-node-cidrs=true \
--authentication-kubeconfig=/etc/kubernetes/controller-manager.conf \
--authorization-kubeconfig=/etc/kubernetes/controller-manager.conf \
--bind-address=127.0.0.1 \
--client-ca-file=/etc/kubernetes/pki/ca.crt \
--cluster-cidr=10.244.0.0/16 \
--cluster-name=kubernetes \
--cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt \
--cluster-signing-key-file=/etc/kubernetes/pki/ca.key \
--controllers=*,bootstrapsigner,tokencleaner \
--kubeconfig=/etc/kubernetes/controller-manager.conf \
--leader-elect=true \
--node-cidr-mask-size=24 \
--requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt \
--root-ca-file=/etc/kubernetes/pki/ca.crt \
--service-account-private-key-file=/etc/kubernetes/pki/sa.key \
--service-cluster-ip-range=10.96.0.0/12 \
--use-service-account-credentials=true
```

### scheduler

```shell
#kubeadm init phase control-plane scheduler
kube-scheduler \
--authentication-kubeconfig=/etc/kubernetes/scheduler.conf \
--authorization-kubeconfig=/etc/kubernetes/scheduler.conf \
--bind-address=127.0.0.1 \
--kubeconfig=/etc/kubernetes/scheduler.conf \
--leader-elect=true
```

### masterN

```shell
#copy: ca.* sa.* front-proxy-ca.* /etcd/ca.*
kubeadm init phase certs apiserver --control-plane-endpoint 10.11.11.250:8888
kubeadm init phase certs apiserver-kubelet-client
kubeadm init phase certs apiserver-etcd-client
kubeadm init phase certs etcd-healthcheck-client
kubeadm init phase certs front-proxy-client

kubeadm init phase kubeconfig controller-manager --control-plane-endpoint 10.11.11.250:8888
kubeadm init phase kubeconfig scheduler --control-plane-endpoint 10.11.11.250:8888

```

## cni

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

## coredns

### kubeadm安装

```shell
kubeadm init phase addon coredns
```

### 手动安装

<details>

```yaml
#https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/dns/coredns/coredns.yaml.sed
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
  labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: Reconcile
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: EnsureExists
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
  labels:
      addonmanager.kubernetes.io/mode: EnsureExists
data:
  ####!!!
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  # replicas: not specified here:
  # 1. In order to make Addon Manager do not reconcile this replicas parameter.
  # 2. Default is 1.
  # 3. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'runtime/default'
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: coredns
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - name: coredns
        image: k8s.gcr.io/coredns:1.6.7
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            ####!!!
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  ####!!!
  clusterIP: 10.96.0.10
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
```

</details>

## node

### kubelet

- bootstrap rbac

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubelet-bootstrap
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:node-bootstrapper
subjects:
  - kind: Group
    apiGroup: rbac.authorization.k8s.io
    name: system:bootstrappers:kubeadm:default-node-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubelet-bootstrap-autoapprove
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
subjects:
  - kind: Group
    apiGroup: rbac.authorization.k8s.io
    name: system:bootstrappers:kubeadm:default-node-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubelet-clientcert-autorenew
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
subjects:
  - kind: Group
    apiGroup: rbac.authorization.k8s.io
    name: system:nodes
```

- bootstrap token

```shell
kubeadm token create
#1sa76r.hvkj1prq8ushc3pp
```

- bootstrap-kubelet.conf

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/pki/ca.crt
    server: https://10.11.11.250:8888
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:node:ubuntu
  name: system:node:ubuntu@kubernetes
current-context: system:node:ubuntu@kubernetes
kind: Config
preferences: {}
users:
- name: system:node:ubuntu
  user:
    token: 1sa76r.hvkj1prq8ushc3pp
```

- kubeletconfig.yaml

```yaml
#kubeadm config print init-defaults --component-configs KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
cpuManagerReconcilePeriod: 0s
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageMinimumGCAge: 0s
kind: KubeletConfiguration
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
rotateCertificates: true
runtimeRequestTimeout: 0s
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
cgroupDriver: systemd
resolvConf: /etc/resolv.conf
```

- start

```shell
rm /etc/kubernetes/kubelet.conf
kubelet \
--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf \
--kubeconfig=/etc/kubernetes/kubelet.conf \
--config=/etc/kubernetes/kubeletconfig.yaml \
--container-runtime=remote \
--container-runtime-endpoint=unix:///run/containerd/containerd.sock
#检查 /var/lib/kubelet/pki/ 中证书是否生成
#没问题的话重启一下kubelet就行了
```

### kubeproxy

#### kubeadm 安装

```shell
kubeadm init phase addon kube-proxy --control-plane-endpoint=10.11.11.250:8888 --pod-network-cidr=10.244.0.0/16
```

#### 手动本地安装

<details>

- rabc

```yaml
#https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/kube-proxy/kube-proxy-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-proxy
  namespace: kube-system
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: system:kube-proxy
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
subjects:
  - kind: ServiceAccount
    name: kube-proxy
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:node-proxier
  apiGroup: rbac.authorization.k8s.io
```

- kubeproxy.conf

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/pki/ca.crt
    server: https://10.11.11.250:8888
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubeproxy
  name: kubeproxy
current-context: kubeproxy
kind: Config
preferences: {}
users:
- name: kubeproxy
  user:
    token: #从上面的sa中得到token
```

- kubeproxyconfig.yaml

```yaml
#二选一
#kubeadm config print init-defaults --component-configs KubeProxyConfiguration
#kube-proxy --write-config-to kubeproxyconfig.yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: "/etc/kubernetes/kubeproxy.conf"
  qps: 5
clusterCIDR: "10.244.0.0/16"
configSyncPeriod: 15m0s
conntrack:
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
detectLocalMode: ""
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
  excludeCIDRs: null
  minSyncPeriod: 0s
  scheduler: ""
  strictARP: false
  syncPeriod: 30s
  tcpFinTimeout: 0s
  tcpTimeout: 0s
  udpTimeout: 0s
kind: KubeProxyConfiguration
metricsBindAddress: 127.0.0.1:10249
mode: "ipvs"
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
showHiddenMetricsForVersion: ""
udpIdleTimeout: 250ms
winkernel:
  enableDSR: false
  networkName: ""
  sourceVip: ""
```

- start

```shell
kube-proxy --config kubeproxyconfig.yaml
```

</details>
