# authentication

><https://kubernetes.io/docs/reference/access-authn-authz/authentication/>
><https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/>

## x509 client certs

- x509客户证书的生命周期与集群ca同步,就是说只要ca没有变,那么客户证书就一直有效,一般用于初始管理员和集群组件向apiserver验证身份.
- certificate-authority-data的内容是base64编码后的x509格式集群ca证书,用于客户端验证apiserver身份
- client-certificate-data和client-key-data的内容是base64编码后的x509格式用户证书和密钥,由集群ca签发,用于apiserver验证客户端身份

>集群ca
--client-ca-file=/etc/kubernetes/pki/ca.crt

```yaml
#kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ca
    server: https://apiserver-entrypoint
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: cert
    client-key-data: key
```

- client-certificate-data证书中`Organization`代表用户在集群中的组成员身份,这里的`system:masters`就是集群内置的超级管理员组

```text
echo client-certificate-data |base64 -d |openssl x509 -text -noout

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 280792628507468656 (0x3e593686dbc6370)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = kubernetes
        Validity
            Not Before: May  4 11:13:18 2020 GMT
            Not After : May  4 11:13:28 2021 GMT
        Subject: O = system:masters, CN = kubernetes-admin
```

## Service Account Tokens

- 当创建一个sa时,会自动创建对应的secret,其中包括了ca.crt和token,token由controller-manager以service-account-private-key签发.
  - ca.crt是集群ca,用于客户端验证apiserver身份
  - token就是客户端向apiserver证明身份的凭证
- 当客户端使用token向apiserver验证身份时,apiserver使用service-account-key进行签名验证,同时也要验证该token在集群内是否存在.
- token的生命周期与sa,sa对应的secret,service account key相关
  - sa被删除,对应的secret也会被删除,token失效
  - sa对应的secret被删除,token失效,新的secret和token自动生成,类似于重置密码
  - service account key改变,token失效
- sa token的方式相对于x509客户证书更灵活,结合role rolebinding适用于集群用户的日常管理

>#controller-manager
>--service-account-private-key-file=/etc/kubernetes/pki/sa.key
>>Filename containing a PEM-encoded private RSA or ECDSA key used to sign service account tokens

>#apiserver
>--service-account-key-file=/etc/kubernetes/pki/sa.pub
>>File containing PEM-encoded x509 RSA or ECDSA private or public keys, used to verify ServiceAccount tokens. The specified file can contain multiple keys, and the flag can be specified multiple times with different files. If unspecified, --tls-private-key-file is used. Must be specified when --service-account-signing-key is provided

>--service-account-lookup     Default: true
>>If true, validate ServiceAccount tokens exist in etcd as part of authentication.

```yaml
apiVersion: v1
data:
  ca.crt: ca
  namespace: ZGVmYXVsdA==
  token: token
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: test
```

## OpenID Connect Tokens

tbd

## Webhook Token Authentication

tbd

## Authenticating Proxy

tbd
