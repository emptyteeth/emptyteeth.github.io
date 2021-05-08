# containers

## image

- 使用的容器镜像,不指定标签默认为latest
- 不指定仓库的话就要看cri的搜索次序了

## imagePullPolicy

- 不设置策略 默认为IfNotPresent
- 不设置策略 使用:latest标签 隐含Always
- 不设置策略 不设置标签 隐含Always

## env

### 默认env

kubelet会自动为pod注入集群信息和pod name

```text
  HOSTNAME=echo-7478dcd959-w7v2q
  KUBERNETES_PORT=tcp://10.96.0.1:443
  KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
  KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
  KUBERNETES_PORT_443_TCP_PORT=443
  KUBERNETES_PORT_443_TCP_PROTO=tcp
  KUBERNETES_SERVICE_HOST=10.96.0.1
  KUBERNETES_SERVICE_PORT=443
  KUBERNETES_SERVICE_PORT_HTTPS=443
```

### serveice env

><https://kubernetes.io/docs/concepts/services-networking/service/#discovering-services>

kubelet会自动为pod注入与之关联的svc信息,前提是svc要在pod之前创建

```text
  ECHO_PORT=tcp://10.109.53.168:8080
  ECHO_PORT_8080_TCP=tcp://10.109.53.168:8080
  ECHO_PORT_8080_TCP_ADDR=10.109.53.168
  ECHO_PORT_8080_TCP_PORT=8080
  ECHO_PORT_8080_TCP_PROTO=tcp
  ECHO_SERVICE_HOST=10.109.53.168
  ECHO_SERVICE_PORT=8080
```

### 直接设置env

```yaml
spec:
  containers:
    env:
    - name: ENV_A
      value: aaa
```

### 从configMapKey引用

><https://kubernetes.io/docs/concepts/configuration/configmap/#configmaps-and-pods>

```yaml
env:
  - name: ENV_B
    valueFrom:
      configMapKeyRef:
        name: game-demo           # The ConfigMap this value comes from.
        key: player_initial_lives #configmap key name
```

### 从secret引用

><https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-environment-variables>

```yaml
env:
  - name: ENV_C
    valueFrom:
      secretKeyRef:
        name: mysecret  #secret name
        key: username   #secret key name
```

### downward api

><https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/#the-downward-api>

```text
   fieldRef     <Object>
     Selects a field of the pod: supports metadata.name, metadata.namespace,
     metadata.labels, metadata.annotations, spec.nodeName,
     spec.serviceAccountName, status.hostIP, status.podIP, status.podIPs.

   resourceFieldRef     <Object>
     Selects a resource of the container: only resources limits and requests
     (limits.cpu, limits.memory, limits.ephemeral-storage, requests.cpu,
     requests.memory and requests.ephemeral-storage) are currently supported.
```

```yaml
env:
  - name: MY_NODE_NAME
    valueFrom:
      fieldRef:
        fieldPath: spec.nodeName
  - name: MY_POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
  - name: MY_POD_NAMESPACE
    valueFrom:
      fieldRef:
        fieldPath: metadata.namespace
  - name: MY_POD_IP
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
  - name: MY_POD_SERVICE_ACCOUNT
    valueFrom:
      fieldRef:
        fieldPath: spec.serviceAccountName
---
env:
  - name: MY_CPU_REQUEST
    valueFrom:
      resourceFieldRef:
        containerName: test-container
        resource: requests.cpu
  - name: MY_CPU_LIMIT
    valueFrom:
      resourceFieldRef:
        containerName: test-container
        resource: limits.cpu
  - name: MY_MEM_REQUEST
    valueFrom:
      resourceFieldRef:
        containerName: test-container
        resource: requests.memory
  - name: MY_MEM_LIMIT
    valueFrom:
      resourceFieldRef:
        containerName: test-container
        resource: limits.memory
```

## envFrom

- 和env类似,支持configMapRef, secretRef
- 把key当变量名,value当做变量值,变量名与env冲突时,优先级低于env
- 另外可以使用prefix给configmap或secret中的key加上前缀
- 如果key名称不符合变量命名规范,那么会被跳过,并记录event

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-configmap
data:
  APP_NAME: Mans Not Hot
  APP_ENV: production
---
envFrom:
- configMapRef:
    name: env-configmap
  prefix: APP
- secretRef:
    name: env-secrets
  prefix: sec
```

## command & args

><https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/>

- 设置command, args则使用command, args
- 不设置command, args则使用ENTRYPOINT, CMD
- 设置command则使用command
- 设置args则使用ENTRYPOINT, args

```yaml
#使用env $(ENVNAME)
command: ["/bin/echo"]
args: ["$(MESSAGE)"]
---
#运行shell
command: ["/bin/sh"]
args: ["-c", "while true; do echo hello; sleep 10;done"]
```

## livenessProbe & readinessProbe & startupProbe

- livenessProbe,定义一个存活指标的探测方法和条件,让kubelet来判断容器进程是否运行正常
- 如果kubelet判断不正常就会干掉pod,不定义的话默认为Success
- readinessProbe,只有探测成功,与之关联的svc才会把pod加入endpoint,失败就会去掉,不定义的话默认为Success
- startupProbe,只有探测成功,livenessProbe和readinessProbe才会开始工作

```yaml
#startupProbe:
#readinessProbe:
livenessProbe:
  #定义连续探测失败次数
  failureThreshold: 3
  #定义连续探测成功数次
  successThreshold: 1
  #定义开始探测前的延时
  initialDelaySeconds: 10
  #定义探测周期
  periodSeconds: 10
  #定义探测超时时间
  timeoutSeconds: 2

  #httpget探测方法,返回200至399算成功
  httpGet:
    #可选项,默认是podip
    host:
    #可选项,定义headers
    httpHeaders:
      - name: Host
        value: abc.com
    #http路径
    path: /ping
    #http端口,可以使用定义的端口名称
    port: 9000
    #可选项,默认是HTTP,可以是HTTPS
    scheme: HTTP

  #exec方法,在容器内运行shell命令,返回0算成功
  exec:
    command:
    - cat
    - /tmp/healthy

  #TCPSocket方法,判断tcp端口是否打开
  TCPSocket:
    #默认为podip
    host:
    #指定探测端口,可以使用定义的端口名称
    port:

```

## resources

## lifecycle

可以定义一个容器运行前和结束后的任务,方法和livenessProbe一样,tcp方法好像还不能用

```yaml
containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        httpGet:
          port: 80
          host: abc.com
          path: /done
```

## ports

可选项,定义容器端口,方便在其它地方使用端口名称引用

```yaml
ports:
    #定义端口名称
  - name: myport
    #定义容器端口 -required-
    containerPort: 8080
    #定义协议,默认是TCP,可以是UDP, SCTP
    protocol: TCP
    #bind地址,一般用不上吧
    hostIp:
    #如果使用主机网络必须和containerPort一致,一般用不上
    hostPort:
```

## name

容器名称,对应kubectl -c 选项

## workingDir

定义工作目录,应该是可以覆盖镜像中的设置,一般用不上

## volumeMounts

挂载pod中定义的volume

```yaml
volumeMounts:
    #对应pod中定义的volume name -required-
  - name: abc
    #挂载到容器内的路径
    mountPath: /mnt/abc
    #是否只读挂载,默认false
    readOnly: true
    #指定挂载volume中的路径,默认为"",也就是整个挂载
    subPath: subdir1
    #和subPath一样,但可以使用环境变量,与subPath互斥
    subPathExpr: $(VAR_NAME)
    #https://kubernetes.io/docs/concepts/storage/volumes/#mount-propagation
    mountPropagation: None
```

## volumeDevices

挂载block device

```yaml
volumeDevices:
    #对应pvc名称
  - name: pvc1
    #本地设备路径
    devicePath:
```

## stdin & stdinOnce & tty

```yaml
#stdin+tty 相当于kubectl run -it
#保持stdin打开
stdin: true
#stdin在attach过一次后关闭
stdinOnce: true
#为容器分配一个tty
tty: true
```

## terminationMessagePath

```txt
string
Optional: Path at which the file to which the container's termination
message will be written is mounted into the container's filesystem. Message
written is intended to be brief final status, such as an assertion failure
message. Will be truncated by the node if greater than 4096 bytes. The
total message length across all containers will be limited to 12kb.
Defaults to /dev/termination-log. Cannot be updated.
```

## terminationMessagePolicy

```txt
string
Indicate how the termination message should be populated. File will use the
contents of terminationMessagePath to populate the container status message
on both success and failure. FallbackToLogsOnError will use the last chunk
of container log output if the termination message file is empty and the
container exited with an error. The log output is limited to 2048 bytes or
80 lines, whichever is smaller. Defaults to File. Cannot be updated.
```
