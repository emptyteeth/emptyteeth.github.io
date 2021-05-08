# job

- 一次性任务,因为要保留任务结果,所以用完需要手动删除
- 以contrainer进程返回值判断是否成功,多个container需要全部成功
- restartPolicy只能是Never或OnFailure,需要显式定义

## backoffLimit `<integer>`

任务失败时的自动重试次数,默认为6
backoff delay是10s, 20s, 40s … 360s

## completions `<integer>`

任务执行次数,默认为1
10次就会创建10个同样的pod来跑任务

## parallelism `<integer>`

并行任务数,就是分批运行,前一批运行成功,下一批才会开始,默认为1
比如执行10次,并行2,那就是分5批运行,一次运行2个

## ttlSecondsAfterFinished `<integer>`

任务完成后过多长时间自动删除任务
需要开TTLAfterFinished FG(alpha)

## selector `<Object>`

一般情况下不需要设置这个
比如说有个job正在运行第一批pod,这时你想保持这一批pod继续运行并加入到一个新的job中
用非级联方部删除前一个job `kubectl delete jobs/old --cascade=false`
使用新template新建一个job,selector使用正在运行的这批pod的label
因为template hash和为pod生成的label本应该是匹配的,但是手动指定了不同的label会导致请求被拒绝
这时需要把manualSelector设置为true,让系统忽略这个问题

## manualSelector `<boolean>`

# cronjob

就是在job外面再套层皮定期运行
时间和时区基于controller-manager的运行环境


## concurrencyPolicy `<string>`

套圈策略
Allow 默认值,充许不同周期的job同时运行
Forbid 前一周期没有完成的情况下,当前周期的job被跳过
Replace 前一周期没有完成的情况下,取消前一周期的job,运行当前周期的
在使用Forbid模式,job被连续跳过100次,计划任务被停止
如果设置了startingDeadlineSeconds,那么这100次会变成只在这个时间内计数

## failedJobsHistoryLimit `<integer>`

保留失败的job的个数,默认1

## successfulJobsHistoryLimit `<integer>`

保留成功的job的个数,默认3

## jobTemplate `<Object> -required-`

直接套job

## schedule `<string> -required-`

使用crontab格式的运行计划

suspend `<boolean>`

挂机计划任务,默认false