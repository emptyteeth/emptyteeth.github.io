# ingress

>垃圾垃圾垃圾,太想当然了,这东西必须回炉或者直接砍掉,不学也罢!!!
>为了考过cka还是不要偷懒学一下吧

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress1
spec:
  #可选项,为不匹配任何规则的流量设置一个默认后端
  backend:
    #ref同ns的其它资源,与下面两个字段互斥,不知道干嘛用
    resource:
      apiGroup:
      kind:
      name:
    #后端服务名称
    serviceName: defaultsvc
    #后端服务端口
    servicePort: defaultport
  #设置要使用的ingressclass,如果未设置就使用默认ingreeclass
  ingressClassName: ic1
  #设置匹配规则,一组规则中host和paths是与的关系,首先判断host
  #多重匹配的优先原则是,最长匹配优先,再之后就是exact优先于prefix
  rules:
    #只接受hostname,可使用wildcard,*只能使用一次,且只能出现在开头
  - host: *.abc.com
    http:
      #不为同path设置后端 <[]Object> -required-
      paths:
        #匹配url路径,必须/开头
      - path: /foo
        #默认为ImplementationSpecific,就是由controller决定怎么匹配
        #Exact就是完全匹配,只匹配a.abc.com/foo这个url
        #Prefix,匹配/foo/xxx,但不匹配/fooo/xxx
        pathType: Exact
        #<[]Object> -required-
        backend:
          serviceName:
          servicePort:
        #不指定path则匹配所有url路径
      - backend:
          serviceName:
          servicePort:
    ##可以不指定host,只根据http的规则来匹配流量
  - http:
      paths:
      - backend:
          serviceName:
          servicePort:
```
