# kubeadm k8s集群

使用kubeadm引导和升级k8s集群

## 安装CRI

><https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/>

### docker

```bash
mkdir /etc/docker

cat > /etc/docker/daemon.json <<EOF
{
"registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"],
"exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

apt install docker.io
```

### containerd

```bash
apt install containerd.io

cat > /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# 设置必需的sysctl参数，这些参数在重新启动后仍然存在。
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system

mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# systemd cgroup
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        SystemdCgroup = true

systemctl restart containerd

cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
EOF
```

### cri-o

>目前1.17版本arm64架构使用会有问题,在拉取镜像时会匹配主机的 OS architecture variant,但gcr的镜像没有variant标记,然后就报错了
>>no image found in manifest list for architecture arm64, variant v8, OS linux

```bash
modprobe overlay
modprobe br_netfilter

# Set up required sysctl params, these persist across reboots.
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system

# Configure package repository
. /etc/os-release
sudo sh -c "echo 'deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/x${NAME}_${VERSION_ID}/ /' > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list"
wget -nv https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/x${NAME}_${VERSION_ID}/Release.key -O- | sudo apt-key add -
sudo apt-get update

sudo apt-get install cri-o-1.17
```

## kubeadm kubelet kubectl

### 包安装

><https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/>
><https://developer.aliyun.com/mirror/kubernetes>

### 二进制安装

> <https://kubernetes.io/docs/setup/release/notes/>
> <https://github.com/kubernetes-sigs/cri-tools/releases>
> <https://github.com/containernetworking/plugins/releases>
> <https://mirrors.aliyun.com/kubernetes/apt/pool/>

- 需要用到kubeadm包中的kubelet.service.d和kubelet包中的kubelet.service
- cri-tools路径/usr/bin
- kubenetes-cni路径/opt/cni/bin
- kubelet deps: iptables (>= 1.4.21), kubernetes-cni (>= 0.7.5), iproute2, socat, util-linux, mount, ebtables, ethtool, conntrack
- kubeadm deps: kubelet (>= 1.13.0), kubectl (>= 1.13.0), kubernetes-cni (>= 0.7.5), cri-tools (>= 1.13.0)

## haproxy配置

```text
listen k8s
    bind *:6444
    mode tcp
    timeout client  4h
    timeout server  4h
    server master01 10.11.1.21:6443
```

## 集群引导

### 命令行方式

```bash
kubeadm init --upload-certs --node-name master01 --control-plane-endpoint "k8s.xxx.fun:6444" --pod-network-cidr=10.244.0.0/16
```

### 配置文件方式

> <https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2>

- 生成默认配置

```bash
kubeadm config print init-defaults --component-configs KubeletConfiguration,KubeProxyConfiguration >kubeadm.yaml
```

- 定制配置文件

```yaml
#InitConfiguration
advertiseAddress: 10.11.1.22

#ClusterConfiguration
controlPlaneEndpoint: "k8s.xxx.fun:6444"
networking:
    podSubnet: 10.244.0.0/16

#KubeProxyConfiguration
mode: "ipvs"

#KubeletConfiguration
cgroupDriver: systemd
```

- 引导集群

```bash
kubeadm init --config kubeadm.yaml --upload-certs
```

### 可选镜像参数

```text
--image-repository gcr.azk8s.cn
--image-repository registry.aliyuncs.com/google_containers
```

## 安装CNI

### flannel

>需要在kubeadm init时设置了`pod-network-cidr`

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

## 生成node加入集群token

```bash
kubeadm token create --print-join-command
```

## 生成master加入集群token

><https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/high-availability/>

```bash
kubeadm init phase upload-certs --upload-certs
```

## 输出结果

```text
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join k8s.xxx.fun:6444 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:a6b93a5ff4633e1fb97a86c1f1bde42dfee969c2771e483d7427e56af2bc6c91 \
    --control-plane --certificate-key c4e45baa07697764320711faeab1eeea3a860878714e10afdc9a1f723e985a9d

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8s.xxx.fun:6444 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:a6b93a5ff4633e1fb97a86c1f1bde42dfee969c2771e483d7427e56af2bc6c91
```

## 重置集群

```bash
kubeadm reset
rm /etc/cni/net.d/*
ipvsadm --clear
ip l del dev {cni0,flannel.1,kube-ipvs0}
```

## upgrade cluster

### 第一个master node

```shell
kubectl drain --ignore-daemonsets node-name
#hold住kubelet,先升级kubeadm kubectl
apt-mark hold kubelet
apt upgrade

kubeadm upgrade plan
kubeadm upgrade apply v1.xx.x

kubectl uncordon node-name

#升级kubelet
apt-mark unhold kubelet
apt upgrade
apt-mark hold kubelet
```

### 其它master node

```shell
kubectl drain node-name --ignore-daemonsets

apt-mark hold kubelet
apt upgrade

kubeadm upgrade node

kubectl uncordon node-name

apt-mark unhold kubelet
apt upgrade
```

## worker node

```shell
kubectl drain --ignore-daemonsets node-name

apt-mark hold kubelet
apt upgrade

kubeadm upgrade node

apt-mark unhold kubelet
apt upgrade
apt-mark hold kubelet

kubectl uncordon node-name
```
