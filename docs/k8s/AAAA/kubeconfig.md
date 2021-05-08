# kubeconfig

主要包括3种信息

- 集群信息
定义集群入口和CA

- 用户信息
定义用户凭证,常用的是证书和token

- context信息
定义用户与集群的关联,也就是用什么用户访问那个集群

```yaml
apiVersion: v1
kind: Config
preferences: {}

#可定义多个集群
clusters:
- cluster:
    #base64编码后的集群ca
    certificate-authority-data:
    #也可以使用集群ca文件路径
    certificate-authority:
    #集群入口
    server: https://my.cluster:6443
  #cluster name,在context中引用
  name: kubernetes

#context就是cluster和user的组合,可定义多个
contexts:
#当前使用的context name
current-context: default
- context:
    #引用cluster name
    cluster: kubernetes
    #引用user name
    user: dev01
  #定义context name
  name: dev01@kubernetes

#可定义多个user
users:
- name: dev01
  user:
    #嵌入token,这里不使用base64编码
    token:
    #使用token文件路径
    tokenFile:
    #嵌入base64编码后的客户端证书
    client-certificate-data:
    #嵌入base64编码后的客户端证书密钥
    client-key-data:
    #使用客户端证书文件路径
    client-certificate:
    #使用客户端证书密钥文件路径
    client-key:
```

## kubectl config

### 设置集群

```shell
#创建或修改集群
kubectl config set-cluster NAME --server=https://k8s:6443 --certificate-authority=path/to/ca
#不验证群集ca
--insecure-skip-tls-verify=true
#指定tls hostname
--tls-server-name=example.com
#嵌入ca证书
--embed-certs=true

#删除集群
kubectl config delete-cluster NAME
```

### 设置用户

```shell
#创建或修改user
kubectl config set-credentials NAME
#客户端cert/key
--client-certificate=path/to/certfile
--client-key=path/to/keyfile
#是否嵌入cert/key
--embed-certs=true
#使用token
--token=bearer_token
```

### 设置context

```shell
#查看当前context
kubectl config current-context

#查看指定的context
kubectl config get-context NAME

#切换当前context
kubectl config use NAME
kubectl config use-context NAME
kubectl config set current-context NAME

#重命名context
kubectl config rename-context NAME new-NAME

#创建或修改context
kubectl config set-context [NAME | --current]
--cluster=cluster_nickname
--user=user_nickname

#删除context
kubectl config delete-context NAME
```
