# prometheus

## terms

- target对应一个被scrape的目标进程
- job是对一组相同类型/用途(一般是同一个应用的多个副本)的target的定义,包括如何发现/定义target,如何操作标签,target的连接和验证方式等
- instance和target的概念差不多...吧,同时也是在job中唯一标识一个target的标签,默认取值自endpoint中的host:port

```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['10.1.1.1:9090']
      - targets: ['10.1.1.2:9090']
```

- endpoint是访问target metric的http url地址,由3部分组合而成,scheme+__address__+metrics_path,如果__address__不包含端口信息(port-free),则使用scheme默认端口
- metric name是从target中scrape出的各种指标的名称,是由client library预先定义的,被作为__name__标签的值使用

```text
##这两种表达方式的意义是相同的
<metric name>{<label name>=<label value>, ...}
{__name__=<metric name>, <label name>=<label value>, ...}
```

- label
  - server-side label,系统自动为target添加的label
  - meta label,不同的SD方法会为target生成相应的meta label
  - target relabel,使用relabel方法根据前两种label确定最终的target label
  - 原始label,scrape来的原始数据中存在的label,由client library生成
    >node_network_up{device="eth0"}
  - metric relabel,对已经包含target label和原始label的ts进行入库前的最终处理

- timeseries是系统存储数据的形式,系统会自动为ts添加__name__, job, instance标签,也就是说每一个ts对应一组特定的label组合,说人话就是一个target中有几个metric,就有几个ts.另外系统会为查询生成临时ts

```text
#一个target中存在两个metric,对应两个ts
#__name__标签值来自metric name
#job标签值来自job_name
#instance标签值来自endpoint中的host:port
[ * * * * * * * * * * * * ] {__name__="m1", job="j1", instance="10.11.1.1:9000"}
[* * * * * * * * * * * * *] {__name__="m2", job="j1", instance="10.11.1.1:9000"}

[ * * * * * * * * * * * * ] {__name__="m1", job="j1", instance="10.11.1.2:9000"}
[* * * * * * * * * * * * *] {__name__="m2", job="j1", instance="10.11.1.2:9000"}
<---------samples---------> <---------------------labels----------------------->
```

- sample是ts中某一时刻的数据,由一个float64的数值和一个毫秒级的时间戳组成

- service-discovery就是在一个job中,通过第三方来动态发现和维护一组可用的target,不同的SD方法会自动为target生成相应的meta label

## default behavior

- 以双下划线__开头的label为系统保留label名称
- label value为空表示该label不存在
- 系统会基于job状态生成一些ts

```text
up{job="<job-name>", instance="<instance-id>"}
值为1则表示正常,0表示target不可用

scrape_duration_seconds{job="<job-name>", instance="<instance-id>"}
scrape耗时

scrape_samples_post_metric_relabeling{job="<job-name>", instance="<instance-id>"}
the number of samples remaining after metric relabeling was applied.

scrape_samples_scraped{job="<job-name>", instance="<instance-id>"}
target的sample数量

scrape_series_added{job="<job-name>", instance="<instance-id>"}
target在当前scrape周期中新增ts数量
```

## metric type

prometheus server并不关心metric type,这个概念只是为了区别metrics的表述形式

- counter类型为记数型,从target的角度来看该数据只增不减,target本身被reset,该数据也会被reset为0,比如linux uptime
- gauge类型为测量型,这种类型的数量会上下浮动,比如内存使用或温度
- Histogram 统计一些数据分布的情况，用于计算在一定范围内的分布情况，同时还提供了度量指标值的总和
- Summary 计算在一定时间窗口范围内度量指标对象的总数以及所有对量指标值的总和


## configuration

- 使用flag配置本地存储和api相关设置
- 使用yaml格式配置文件配置scrape相关设置

```yaml
#全局设置及全局scrape默认值
global:
  # How frequently to scrape targets by default.
  [ scrape_interval: <duration> | default = 1m ]

  # How long until a scrape request times out.
  [ scrape_timeout: <duration> | default = 10s ]

  # How frequently to evaluate rules.
  [ evaluation_interval: <duration> | default = 1m ]

  # The labels to add to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    [ <labelname>: <labelvalue> ... ]

  # File to which PromQL queries are logged.
  # Reloading the configuration will reopen the file.
  [ query_log_file: <string> ]

# Rule files specifies a list of globs. Rules and alerts are read from
# all matching files.
rule_files:
  [ - <filepath_glob> ... ]

# A list of scrape configurations.
scrape_configs:
  [ - <scrape_config> ... ]

# Alerting specifies settings related to the Alertmanager.
alerting:
  alert_relabel_configs:
    [ - <relabel_config> ... ]
  alertmanagers:
    [ - <alertmanager_config> ... ]

# Settings related to the remote write feature.
remote_write:
  [ - <remote_write> ... ]

# Settings related to the remote read feature.
remote_read:
  [ - <remote_read> ... ]
```

### scrape_config

```yaml
# 设置job name
job_name: <job_name>

# 覆盖全局设置
[ scrape_interval: <duration> | default = <global_config.scrape_interval> ]
[ scrape_timeout: <duration> | default = <global_config.scrape_timeout> ]
[ metrics_path: <path> | default = /metrics ]

# 当原始label与server-side label, SD label, relabel冲突时的处理策略
# 默认为false,当冲突时原始label会被加上exported_前缀
# 当为true时,则保留原始label,其它来源被忽略
[ honor_labels: <boolean> | default = false ]

# 是否使用原始数据中的时间戳,默认为true,false时则忽略,然后呢?
[ honor_timestamps: <boolean> | default = true ]

# 使用http/https协议
[ scheme: <scheme> | default = http ]

# 可选http url参数
params:
  [ <string>: [<string>, ...] ]

# 使用basic auth验证,password和password_file二选一
basic_auth:
  [ username: <string> ]
  [ password: <secret> ]
  [ password_file: <string> ]

# 使用bearer token验证
[ bearer_token: <secret> ]

# 从文件读取bearer token,与上面的选项二选一
[ bearer_token_file: <filename> ]

# 使用代理
[ proxy_url: <string> ]

# 设置最终sample数限制,如果有metric relabel则是算apply之后的数
# 超过限制数的话当前scrape为fail,数据被丢掉,设为0表示不限制
[ sample_limit: <int> | default = 0 ]
```

### static_config

```yaml
# 手动配置target
targets:
  [ - '10.1.1.1:9000' ]

# 手动为上面的target添加标签
labels:
  [ <labelname>: <labelvalue> ... ]
```

### tls_config

```yaml
# CA certificate to validate API server certificate with.
[ ca_file: <filename> ]

# Certificate and key files for client cert authentication to the server.
[ cert_file: <filename> ]
[ key_file: <filename> ]

# ServerName extension to indicate the name of the server.
[ server_name: <string> ]

# Disable validation of the server certificate.
[ insecure_skip_verify: <boolean> ]
```

### kubernetes_sd_config

- 通过apiserver获取target信息
- 通过5种role发现target
  - node 将集群中每一个node做为一个target,__address__为NodeInternalIP/NodeExternalIP/NodeHostName中最先出现的地址加上kubelet的http端口,instance标签值为node name.

    ```text
    __meta_kubernetes_node_name: node name
    __meta_kubernetes_node_label_<labelname>: 所有的node label
    __meta_kubernetes_node_labelpresent_<labelname>: 存在的node label值为true
    __meta_kubernetes_node_annotation_<annotationname>: 所有的node annotation
    __meta_kubernetes_node_annotationpresent_<annotationname>: 存在的annotation值为true
    __meta_kubernetes_node_address_<address_type>: 如果地址存在,值为该地址类型中的首个地址
    ```

  - service 将每一个service中的每一个port做为一个target,__address__为service的dns name加上相应的端口

    ```text
    __meta_kubernetes_namespace: service所在的ns
    __meta_kubernetes_service_annotation_<annotationname>: 所有的service annotation
    __meta_kubernetes_service_annotationpresent_<annotationname>: 所有的service annotation, 值为true
    __meta_kubernetes_service_cluster_ip: service的clusterIP,不应用于ExternalName类型的service
    __meta_kubernetes_service_external_name: ExternalName类型service的dns名称,只应用于ExternalName类型的service
    __meta_kubernetes_service_label_<labelname>: 所有的service label
    __meta_kubernetes_service_labelpresent_<labelname>: 所有的service label,值为true
    __meta_kubernetes_service_name: service名称
    __meta_kubernetes_service_port_name: 端口名称
    __meta_kubernetes_service_port_protocol: 端口协议
    __meta_kubernetes_service_type: service类型
    ```

  - pod 将每个pod中每个container的每个声明的端口做为一个target,__address__为podIP加上相应的端口,如果container没有声明端口,那么container被当做一个port-free target,__address__中只有podIP

    ```text
    __meta_kubernetes_namespace: pod所在的ns
    __meta_kubernetes_pod_name: pod名称
    __meta_kubernetes_pod_ip: pod IP
    __meta_kubernetes_pod_label_<labelname>: 所有pod label
    __meta_kubernetes_pod_labelpresent_<labelname>: 所有pod label,值为true
    __meta_kubernetes_pod_annotation_<annotationname>: 所有pod annotation
    __meta_kubernetes_pod_annotationpresent_<annotationname>: 所有pod annotation,值为true
    __meta_kubernetes_pod_container_init: 如果是InitContainer值为true
    __meta_kubernetes_pod_container_name: container名称
    __meta_kubernetes_pod_container_port_name: 端口名称
    __meta_kubernetes_pod_container_port_number: 端口号
    __meta_kubernetes_pod_container_port_protocol: 端口协议
    __meta_kubernetes_pod_ready: Set to true or false for the pod's ready state.
    __meta_kubernetes_pod_phase: Set to Pending, Running, Succeeded, Failed or Unknown in the lifecycle.
    __meta_kubernetes_pod_node_name: 所在的node
    __meta_kubernetes_pod_host_ip: 所在node的ip
    __meta_kubernetes_pod_uid: The UID of the pod object.
    __meta_kubernetes_pod_controller_kind: Object kind of the pod controller.
    __meta_kubernetes_pod_controller_name: Name of the pod controller.
    ```

  - endpoint 将endpoint中每个地址对应的每个端口做为一个target(target数=地址数x端口数),同时对ep背后的pod做pod发现(排除ep中的端口),__address__为ep地址:ep端口

    ```text
    __meta_kubernetes_namespace: ep所在ns
    __meta_kubernetes_endpoints_name: ep名称
    # 对于直接由ep发现的target:
        __meta_kubernetes_endpoint_hostname: Hostname of the endpoint.
        __meta_kubernetes_endpoint_node_name: Name of the node hosting the endpoint.
        __meta_kubernetes_endpoint_ready: Set to true or false for the endpoint's ready state.
        __meta_kubernetes_endpoint_port_name: Name of the endpoint port.
        __meta_kubernetes_endpoint_port_protocol: Protocol of the endpoint port.
        __meta_kubernetes_endpoint_address_target_kind: Kind of the endpoint address target.
        __meta_kubernetes_endpoint_address_target_name: Name of the endpoint address target.
    # 如果ep属于一个service,那么会附上service发现的meta label
    # 如果ep背后是一个pod,那么会附上pod发现的meta label
    ```

  - ingress 将ingress中每个path做为一个target,__address__为ingress中的host

    ```text
    __meta_kubernetes_namespace: The namespace of the ingress object.
    __meta_kubernetes_ingress_name: The name of the ingress object.
    __meta_kubernetes_ingress_label_<labelname>: Each label from the ingress object.
    __meta_kubernetes_ingress_labelpresent_<labelname>: true for each label from the ingress object.
    __meta_kubernetes_ingress_annotation_<annotationname>: Each annotation from the ingress object.
    __meta_kubernetes_ingress_annotationpresent_<annotationname>: true for each annotation from the ingress object.
    __meta_kubernetes_ingress_scheme: Protocol scheme of ingress, https if TLS config is set. Defaults to http.
    __meta_kubernetes_ingress_path: Path from ingress spec. Defaults to /.
    ```

```yaml
# The API server addresses. If left empty, Prometheus is assumed to run inside
# of the cluster and will discover API servers automatically and use the pod's
# CA certificate and bearer token file at /var/run/secrets/kubernetes.io/serviceaccount/.
[ api_server: <host> ]

# The Kubernetes role of entities that should be discovered.
# One of endpoints, service, pod, node, or ingress.
role: <string>

# Optional namespace discovery. If omitted, all namespaces are used.
namespaces:
  names:
    [ - <string> ]
```

### (target)relabel_configs

- relabel主要目的是定义那些target是需要被scrape的,其它的则被忽略
- 另一个主要目的是确定这些需要被scrape的target应该具有那些label
- 在scrape之后,这些label会自动传播到target中包含的所有ts上的
- 所以这里的操作和原始label是没有一毛钱关系的
- 一个relabel配置中,可以有多个action,它们按顺序执行
- source_labels 在action中定义要操作的源标签列表
- target_label 在action中定义要操作的目标标签
- regex 正则表达式,默认为(.*),表示全部匹配并全部捕获.可使用多个表达多匹配对应的多个source_labels,以separator分隔
- separator 定义regex中多个表达式的分隔符,默认为;
- replacement 定义如何引用regex的捕获组,默认为$1,即直接引用regex中第一个捕获组,多个regex的捕获组序号向后累加,比如有两个regex各有两个捕获组,那么可以使用$1 $2 $3 $4来引用
- modulus hashmod系数
- action 默认为replace
  - replace 使用regex匹配source_labels的值,将匹配到的replacement写入target_label
  - keep 丢弃regex不能匹配source_labels值的target
  - drop 丢弃regex匹配source_labels值的target
  - hashmod 根据modulus系数计算source_labels的hash值,写入target_label
  - labelmap 使用regex匹配所有的label name,匹配到的label的值被复制到以replacement命名的label中
  - labeldrop 使用regex匹配所有的label name,为target丢弃所有匹配到的label
  - labelkeep 使用regex匹配所有的label name,为target丢弃所有不能匹配的label

### metric_relabel_configs

- 系统在在scrape之后,入库之前,可以来一波metric relabel操作
- 这里匹配和操作的都是待入库的ts,比如丢掉某些ts,或者去掉某些标签等.
- 用法relabel一样

## arch

## promQL

- 数据数型
  - 瞬时向量 Instant vector 一组相同时间戳的ts的sample 可以用于图形化
    - 比如班级所有学生(metrics)在一次考试中(timestamp)的成绩(sample)
  - 区间向量 Range vector 一组ts,包含它们在相同的特定时间段内的所有sample 不能用于图形化
    - 比如班级所有学生(metrics)在一个学期内(timestamp范围)的考试成绩(sample)
  - 标量 一个浮点数值
  - 字符串 一个字符串,目前没有用

- ts selector
  - 瞬时向量selector,查询结果为瞬时向量的表达式
  - 区间向量selector,查询结果为区间向量的表达式
    - 就是瞬时向量selector后面跟上时间范围 exp [5m] 可使用的单位为s m h d w y

- 表达式中的字符转义
  - 在单引号和双引号内,使用反斜杠\作为转定字符
  - 在反引号``内,一切保持原样,不存在转义

- 条件查询逻辑符号
  - = 查询标签值与条件完全相等的ts,metric1{label1='value1'}
  - != 取反
  - =~ 使用regex来匹配标签值,metric1{label1=~'value[0-9]{1,3}'}
    >不能将会匹配空值的regex做为唯一条件来查询
  - !~ 取反

- offset 偏移时间
  - 使用ts selector查询的结果默认是基于当前时间的
  - 在ts selector后跟上offset 5m 即可改变时间基准
  - offset必须紧跟在ts selector后面,不能放在函数后
  - 可使用的单位为s m h d w y

- 关于查询结果的时间戳
  - 原始数据中不同ts的时间戳并不总是对齐的
  - 查询和聚合时使用的是统一的基准时间
  - 系统会取基准时间前最新的值来做为结果
  - 所以查询结果与原始数据几乎总是有偏差的

- 消逝的电波???
  先翻页吧,实在看不下去 !!reveiw
  ><https://prometheus.io/docs/prometheus/latest/querying/basics/#gotchas>

- 数学运算
  - `+ - * / % ^` 加减乘除 取模 求余
  - 标量与瞬时向量运算相当于将运算应用到每个ts的值
    - node_memory_free_bytes_total / (1024 * 1024)
  - 瞬时向量间的运算是以左侧每条ts的标签搜索右侧与它完全一至的ts,不匹配的数据被丢弃,结果不包含metric name
    - 相当于两家公司合并,左公司有ABC部门,右公司有BCD部门,那么BC部门员工合并,AD部门员工layoff
    >node_disk_bytes_written + node_disk_bytes_read
    >{device="sdb",instance="localhost:9100",job="node_exporter"}
    >{device="sdc",instance="localhost:9100",job="node_exporter"}

- 布尔运算
  - `== != > < >= <=` 过滤器
  - 标量与瞬时向量运算,将每个ts的值与标量进行运算,丢掉false的ts,然后输出结果
    >(node_memory_MemTotal_bytes /1024 /1024) >2048
  - 瞬时向量间的运算是以左侧每条ts的标签搜索右侧与它完全一至的ts,标签不匹配和运算结果false的数据被丢弃
    - 首先像数学运算一样丢弃标签不匹配的ts
    - 然后进行布尔运算,保留为true的ts做为输出
    - 使用布尔修饰符不对ts进行过滤,将布尔运算结果作为ts的值输出
      >http_requests_total > bool 1000

- 逻辑集合运算
  - and or unless 匹配的是两个瞬时向量的标签
  - and 相当于过滤器,保留所有左侧中与右侧匹配的ts做为结果输出
  - or 相当于匹配去重,保留所有左侧ts,及右侧中不与左侧匹配的ts
  - unless 相当于and取反,保留左侧中不与右侧匹配的ts

- 操作符优先级
  1. ^
  2. *, /, %
  3. +, -
  4. ==, !=, <=, <, >=, >
  5. and, unless
  6. or

- 匹配模式
  - 一对一 是指在匹配中,有且只有一次匹配的ts
  - on 使用on(label list)来限定用来匹配的labels,其它label被忽略
  - ignoring 使用ignoring(label list)来从匹配条件中排除某些label
  - on 和ignoring放在运算符后面
  - 一对多/多对一 是指在一侧的ts中,有另一侧有多于一次匹配的ts
    - 需要使用group_left或group_right来指定"多"的一侧
    - group只用在数学运算和布尔运算中

  ```sql
  #样本
  method_code:http_errors:rate5m{method="get", code="500"}  24
  method_code:http_errors:rate5m{method="get", code="404"}  30
  method_code:http_errors:rate5m{method="put", code="501"}  3
  method_code:http_errors:rate5m{method="post", code="500"} 6
  method_code:http_errors:rate5m{method="post", code="404"} 21
  method:http_requests:rate5m{method="get"}  600
  method:http_requests:rate5m{method="del"}  34
  method:http_requests:rate5m{method="post"} 120

  #一对一查询
  method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m
  #一对一查询结果
  {method="get"}  0.04            //  24 / 600
  {method="post"} 0.05            //   6 / 120

  #多对一查询
  method_code:http_errors:rate5m / ignoring(code) group_left method:http_requests:rate5m
  #多对一查询结果
  {method="get", code="500"}  0.04            //  24 / 600
  {method="get", code="404"}  0.05            //  30 / 600
  {method="post", code="500"} 0.05            //   6 / 120
  {method="post", code="404"} 0.175           //  21 / 120
  ```

- 聚合操作符
  - sum 求和
  - min 选择最小值
  - max 选择最大值
  - avg 计算平均值
  - group 将所有返回值置1,需要配合by,without使用,一般用于分组计数
  - stddev 计算标准差 就是平均值与样本值的最大距离 比如170±10cm
  - stdvar 计算方差 就是标准差的2次方
  - count 计数
  - count_values
  - bottomk 最小值排行
  - topk 最大值排行
  - quantile 分位数计算

  ```sql

  #忽略所有标签将所有ts求和,结果只有一个没有标签的ts
  sum (node_memory_MemTotal_bytes)
  #按条件分组求和,条件标签有多少种不同的值,结果就有多少个ts,结果的标签只有条件标签
  #相当于忽略其它所有标签
  sum by (job) (node_memory_MemTotal_bytes)
  #可以使用多个条件来分组,ts数<=条件数的2次方
  #根据不同node上的不同service来求和
  sum by (service,kubernetes_node) (traefik_service_requests_total)
  #忽略条件标签求和,就是先从原数据去掉所有条件标签然后求和,结果包括除条件标签外的所有原始标签
  #如果原数据有标签abcd,那么sum by (a)和sum without (b,c,d)的效果是一样的
  sum without (code) (traefik_service_requests_total)
  #一样也可以多条件
  sum without (code,method) (traefik_service_requests_total)

  #取ts中最大/最小的值,结果只有一个值,没有标签
  #先按不同service求和,然后取最大/最小值
  min (sum by (service) (traefik_service_requests_total))
  min (sum by (service) (traefik_service_requests_total))

  #取ts的平均值,结果只有一个值,没有标签
  #先按不同service求和,然后取平均值
  avg (sum by (service)(traefik_service_requests_total))

  #取ts值的标准差/方差,只有一个结果,没有标签
  #先按entrypoint求和,然后取标准差/方差
  stddev (sum by (entrypoint)(traefik_entrypoint_open_connections))
  stdvar (sum by (entrypoint)(traefik_entrypoint_open_connections))

  #计算ts数量,只有一个结果,没有标签
  count (kube_deployment_created)

  #根据ts值,统计各个不同值的数量,使用一个自定义的字值串参数做为标签名
  count_values ("cores",kube_node_status_capacity_cpu_cores)
  #从结果可以看出一共有6个node,2个4核的,4个8核的
  cores    value
  4           2
  8           4

  #根据ts的值,返回最大/最小排名,参数为自定义的整型排名数
  #结果为按最大/最小排列的3个ts,包含全部原标签
  topk (3,etcd_object_counts)
  bottomk (3,etcd_object_counts)

  #计算ts值的分位数,参数为0-1之间的浮点数,结果只有一个值,没有标签
  quantile (0.5,etcd_object_counts)
  ```

- 函数
  - abc(instant) 将ts的值转换为绝对值
  - absent(instant)
  - absent_over_time(range)
  - ceil(instant) 将ts的值向上取整
  - floor(instant) 将ts的值向下取整
  - round(instant,scaler) 四舍五入取整,参数默认为1,代表取整到该参数的倍数,可以是小数
  - changes(range) 返回一个瞬时向量,标签保留,值为区间向量中值的变化次数
  - clamp_max(instant,scalar) 将大于scalar的值替换为该值
  - clamp_min(instant,scalar) 将小于scalar的值替换为该值
  - minute(instant(time)) 返回0-59的分钟数
  - hour(instant(time)) 返回0-23的小时数
  - day_of_month(instant(time)) 返回1-31的日期
  - day_of_week(instant(time)) 返回周几,值为0-6,0是周日
  - days_in_month(instant(time)) 返回这个月有几天 28-31
  - month(instant(time)) 返回1-12的月份数
  - year(instant(time)) 返回年份数
  - delta(range) 返回一个瞬时向量,取值为区间向量中第一个和最后一个值的差,用于测量型metric
    >delta(cpu_temp_celsius{host="zeus"}[2h])返回当前CPU温度与两小时前的差别
  - idelta(range) 与delta的区别是计算的是区间内最后两个值的差
  - rate(range) 计算计数类型区间向量的平均每秒增长量,区间增长量/秒数
  - irate(range) 与rate不同的是irate根据区间内最后两个值计算增长量
  - increase(range) 与delta类似,计算的是计数型metric在区间内的增长量
    >increase(a[1m])/60 与 rate(a[1m]) 是一样的
  - deriv(range) 计算区间向量的导数
  - exp(instant) 计算瞬时向量的自然指数
  - histogram_quantile(scalar,instant) 计算bucket分位数
  - holt_winters(range,scalar,scalar) 计算区间向量值的平滑度
  - label_join(instant,dst_lable,separator,src_label+) 将多个目标标签的值组合成一个新标签
    >label_join(url{host='1.1.1.1',port='80'},joined,":","host","port")
    >结果是url{host='1.1.1.1',port='80',joined='1.1.1.1:80'}
  - label_replace(instant,dst_label,replacement,src_label,regex)
    >用regex匹配目标标签的值,将捕捉组内容写入新建的目标标签中,如果regex不匹配则原样返回
    >label_replace(node{instance="1.1.1.1:80"},port,"$1","instance",".*:(.*)")
    >结果是node{instance="1.1.1.1:80",port="80"}
  - ln(instant) 计算所有值的自然对数
  - log2(instant) 计算所有值的二进制自然对数
  - log10(instant) 计算所有值的十进制自然对数
  - predict_linear(range,scalar) 基于简单线性回归方式,预测指定秒数后的值
    >predict_linear(node_filesystem_free_bytes[12h],3600*24)
  - resets(range) 返回计数器类型metric在区间范围内的计算器重围次数
  - scalar(instant) 将单个ts的瞬时向量转换为标量,ts数不为1时返回none
  - sort(instant) 升序排序
  - sort_desc(instant) 降序排序
  - sqrt(instant) 计算平方根
  - time() ???
  - timestamp(instant) 返回原始时间戳
  - vector(scalar) 将标量转换为向量
  - `aggregation`_over_time(range) 将区间内的值进行聚合,返回一个瞬时向量
    - avg_over_time(range) 取平均值
    - min_over_time(range) 取最小值
    - max_over_time(range) 取最大值
    - sum_over_time(range) 求和
    - count_over_time(range) 计数
    - quantile_over_time(range,scalar) 取分位数
    - stddev_over_time(range) 取标准差
    - stdvar_over_time(range) 取方差

```sql

#查询所有名为http_requests_total的ts,也就是__name__标签值为http_requests_total的ts
http_requests_total

#查询所有名为http_requests_total,job为apiserver,handler为/api/comments的ts
http_requests_total{job="apiserver", handler="/api/comments"}

#查询出所有符合条件的ts在[5m]时间范围内所有的sample
#属于range vector,不能被直接图形化
http_requests_total{job="apiserver", handler="/api/comments"}[5m]

#查询中可以使用正则表达式,查询值以server结尾的job
#方法是key=~"exp"
http_requests_total{job=~".*server"}

#查询status的值不等于4xx,这里使用了!取反
http_requests_total{status!~"4.."}

#子表达式
#以1分钟为分辨率,返回30分钟内的(http_requests_total在5分钟内的速率)
rate(http_requests_total[5m])[30m:1m]

#暂时不明白
max_over_time(deriv(rate(distance_covered_total[5s])[30s:5s])[10m:])

#rate函数,计算每秒速率,目标需要是range vector
rate(http_requests_total[5m])

#sum函数,以指定的标签汇总
sum by (kubernetes_node) (
  traefik_entrypoint_requests_total
)
返回:
{kubernetes_node="master02"}  1197
{kubernetes_node="master03"}  1253

#CPU使用率
100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)
#内存使用率
(node_memory_MemFree_bytes+node_memory_Cached_bytes+node_memory_Buffers_bytes) / node_memory_MemTotal_bytes * 100
#空闲内存剩余率
100 - (node_memory_MemFree_bytes+node_memory_Cached_bytes+node_memory_Buffers_bytes) / node_memory_MemTotal_bytes * 100
#磁盘使用率
100 - (node_filesystem_free_bytes{mountpoint="/",fstype=~"ext4|xfs"} / node_filesystem_size_bytes{mountpoint="/",fstype=~"ext4|xfs"} * 100)

sum(traefik_entrypoint_requests_total{entrypoint="websecure"}) without (instance,kubernetes_node)
sum(traefik_entrypoint_requests_total{entrypoint="websecure"}) by (code,method)

rate(
  traefik_entrypoint_requests_total{entrypoint="websecure"}
  [5m]
)

sum(
  traefik_entrypoint_requests_total{entrypoint="websecure"}
  )
  by (code)

sum(
  rate(
    traefik_entrypoint_requests_total{entrypoint="websecure"}
    [30m]
      )
  )
  by (code,method)

```

## alerting
