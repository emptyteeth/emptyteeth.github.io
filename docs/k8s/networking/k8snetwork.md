# k8s network

- <https://kubernetes.io/docs/concepts/cluster-administration/networking>

## 网络模型

- 每个pod拥有独立的IP
- 集群内pod之间可直接路由,不需要NAT
- 集群内pod与任何node之间可直接路由,不需要NAT
- pod内的container共享network namespace,可以通过loopback通信,存在端口冲突

- 常见的一些实现
  - AWS VPC CNI <https://github.com/aws/amazon-vpc-cni-k8s>
  - Azure CNI <https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni>
  - flannel
  - cilium
  - calico
  - weave
