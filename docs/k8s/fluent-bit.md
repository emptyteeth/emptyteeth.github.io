# fluent-bit

- <https://docs.fluentbit.io/manual/installation/kubernetes>
- https://docs.fluentbit.io/manual/output/elasticsearch.html>
- https://github.com/fluent/fluent-bit-kubernetes-logging>

## parser

- <https://github.com/fluent/fluent-bit/blob/master/conf/parsers.conf>

### cri (cri-o / containerd)

>2020-05-11T15:17:15.896201993+08:00 stdout F 127.0.0.1 - - [11/May/2020:07:17:15 +0000] "GET / HTTP/1.1"4; rv:76.0) Gecko/20100101 Firefox/76.0" "-"

- cri日志解析,在INPUT中引用,分离出cri附加的`time stream logtag`字段,以及原始日志`log`字段
- 之后在pod注解中指定的解析器将基于原始日志`log`字段再次解析
- 关于`time_key`,如果所有解析器都没有指定或解析不出来,那么入库的时间戳就是入库时间,如果有指定并正确解析,那么原始日志时间将做为时间戳入库
- 如果多个解析器都指定了`time_key`,那么以最后的有效解析为准(猜测)

  >入库时间 就是数据写入数据库的时间,这个可能因为性能或网络的原因,与实际日志时间有较大出入,应该避免使用
  >cri附加时间 这个和实际日志时间几乎相同,在原始日志没有时间时可以使用
  >原始日志时间 用这个最准确

```ini
[PARSER]
    Name        cri
    Format      regex
    Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<log>.*)$
    #Regex       ^[^ ]+ (stdout|stderr) [^ ]* (?<log>.*)$
    Time_Key    time
    #https://apidock.com/ruby/DateTime/strftime
    Time_Format %Y-%m-%dT%H:%M:%S.%L%z
```

### nginx

```ini
[PARSER]
    Name   nginx
    Format regex
    Regex ^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")? "(?<xforward>[^\"]*)"?$
    Time_Key time
    Time_Format %d/%b/%Y:%H:%M:%S %z
```

## input

- input的主要作用是设置要收集的日志的位置和范围,标记修改,设置解析器
- 可设置多个input来设置不同位置或范围的日志,及设置不同的标记和解析器
- 每一个日志文件都会以`文件路径`做为标记,比如`var.log.container.xxxx.log` 其中/以.代替
- `Tag`命令可以对标记进能扩展,以*代表原标记,扩展后为`kube.var.log.container.xxxx.log`
- `Key`命令指定非结构化数据的字段名,也就是分离出的`原始日志`,该字段的数据会被交给pod指定的解析器,默认值为`log`.
  - 如果解析器没有成功解析,那么`全部数据`都会被放进该字段.
  - 在使用docker做为cri时,docker的json格式日志中原始日志的字段名就是log
  - 在引用的parser中,解析原始日志的`字段名`要与`Key`的值一致.

```ini
[INPUT]
    Name              tail
    Tag               kube.*
    Path              /var/log/containers/*.log
    Parser            cri
    #Key              log
    DB                /var/log/flb_kube.db
    Mem_Buf_Limit     5MB
    Skip_Long_Lines   On
    Refresh_Interval  10
```

## filter

### kubernetes

- 内置的kubernetes过滤器的工作机制是首先通过`Match`命令来匹配所有input中标记为`kube.`开头的数据
- 通过`Kube_Tag_Prefix`命令来去除路径得到文件名(不会修改标记),再通过文件名来得到`podname namespace containername`等信息
- 内置的文件名解析表达式是固定的,对应目前统一的cri日志文件名规范,特殊情况下可以使用`Regex_Parser`来自定义表达式
- 通过这些信息到apiserver查询相应的metadata,默认情况下会把所有`annotation`和`label`添加为字段
- `Merge_Log` 自动探测原始日志是否为json格式,如果是就自动解析
- `Merge_Log_Key` 如果探测出json格式日志,指定json日志父字段名,不指定则为解析器中设置的字段名log

```ini
[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
    Kube_Tag_Prefix     kube.var.log.containers.
    Merge_Log           On
    #Merge_Log_Key      log_processed
    K8S-Logging.Parser  On
    K8S-Logging.Exclude On
```

#### pod annotation

```yaml
annotations:
    #标记不处理pod日志
    fluentbit.io/exclude: true
    #标记pod日志解析器
    fluentbit.io/parser: nginx
```

### tbd

```ini
[FILTER]
    Name record_modifier
    Match kube.*
    Remove_key something
```

## ouput

### elasticsearch

- `Replace_Dots`是为了避免es数据类型冲突,比如k8s中同时存在这两种label:
  1. `app=gitea`  在es中表现为`kubernetes.labels.app:gitea`
        >app的类型为text
  2. `app.kubernetes.io/name=traefik` 在es中表现为`kubernetes.labels.app.kubernetes.io/name:traefik`
        >app的类型为object
- 启用`Replace_Dots`之后, `kubernetes.labels.`之后的`.`会被替换为`_`来避免类型冲突:
    >kubernetes.labels.`app_kubernetes_io/name`:traefik

- 这样做理论上会丢失掉一些结构,另一种办法就是规划好label和annotation,来避免这种情况.

```ini
[OUTPUT]
    Name            es
    Match           *
    HTTP_User       ${FLUENT_ELASTICSEARCH_USER}
    HTTP_Passwd     ${FLUENT_ELASTICSEARCH_PASS}
    Host            ${FLUENT_ELASTICSEARCH_HOST}
    Port            ${FLUENT_ELASTICSEARCH_PORT}
    Index           fluentbit
    #Logstash_Format On
    #Logstash_Prefix fluentbit
    #Logstash_DateFormat %Y.%m
    Retry_Limit     False
    Replace_Dots    On
    tls             On
    #tls.vhost
```
