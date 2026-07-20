# kube-fluentd-operator (Fluent Bit Edition)
> 基于 [kube-fluentd-operator](https://github.com/vmware/kube-fluentd-operator) 改造,将底层采集引擎从 **Fluentd** 替换为 **Fluent Bit**,同时保留原有的 `config-reloader` 动态发现机制。
[![Build](https://img.shields.io/badge/build-passing-brightgreen)]()
[![License](https://img.shields.io/badge/license-Apache--2.0-blue)]()
## 特性
- **命名空间级日志采集**:通过给命名空间打标签即可开启/关闭采集。
- **动态发现,无需重启**:新增命名空间、新增 Pod、修改配置均通过 Fluent Bit 热重载(`/api/v2/reload`)自动生效。
- **兼容旧语法**:保留 `mounted-file` 的 XML 风格语法,自动转换为 Fluent Bit `tail` input。
- **时间格式自动归一化**:自动适配 Fluentd/Ruby strptime 与 Fluent Bit `flb_strptime` 的差异。
- **日志上下文字段增强**:自动补充 `log_offset`、`kafka_key`、`log_context_id`,便于在 Kafka/ES/Kibana 中还原上下文。
- **内置监控**:Fluent Bit 与 reloader 均暴露 Prometheus 指标。
---
## 一、快速开始:开启命名空间日志采集
### 1. 开启采集(打标签)
系统默认只采集带有 `logging=enabled` 标签的命名空间:
```bash
kubectl label namespace <your-namespace> logging=enabled
# 取消采集
kubectl label namespace <your-namespace> logging-
打上标签后,系统会自动采集该命名空间下所有容器的 stdout/stderr 日志。

2. 自定义配置(ConfigMap)
如需自定义输出目的地,或采集容器内的文件日志,需在该命名空间下创建 ConfigMap:

默认 ConfigMap 名称:fluentd-config,配置键名为 fluent.conf。
自定义 ConfigMap 名称:在命名空间上添加注解指定:
logging.csp.vmware.com/fluent-bit-configmap: fluent-bit-config-<your-namespace>
注意:配置文件内容必须使用 Fluent Bit 语法(仅 mounted-file 块保留旧有 XML 语法以便兼容)。

3. 自动发现(无需重启)
config-reloader 会监听以下变化并自动重新生成配置、通过 Fluent Bit 热重载生效,新增命名空间或新增 Pod 都无需 rollout restart:

命名空间的 logging=enabled 标签变化;
命名空间上 logging.csp.vmware.com/fluent-bit-configmap 注解变化;
目标 ConfigMap(fluent.conf)内容变化;
匹配 mounted-file 标签的 Pod 创建 / 调度 / 重启。
热重载已在基础模板 [SERVICE] 段通过 Hot_Reload On 启用。若该选项被关闭,/api/v2/reload 将变为空操作,新增命名空间只能靠重启生效。

推荐操作顺序:先创建 ConfigMap,再一次性给命名空间打标签 + 注解:

# 1. 创建该命名空间的采集配置
kubectl apply -f fluent-bit-config-<your-namespace>.yaml
# 2. 打标签 + 指定 ConfigMap 名称
kubectl label namespace <your-namespace> logging=enabled
kubectl annotate namespace <your-namespace> \
  logging.csp.vmware.com/fluent-bit-configmap=fluent-bit-config-<your-namespace>
二、Helm 安装
推荐使用 values 文件进行安装配置:

# values-fluent-bit.yaml
namespaceSelector: logging=enabled
prometheusEnabled: true
fluentdLogLevel: info
image:
  repository: <your-registry>/kube-fluentd-operator
  tag: fluent-bit
  pullPolicy: IfNotPresent
fluentBit:
  httpPort: 2020
  resources:
    requests:
      cpu: 10m
      memory: 25Mi
    limits:
      cpu: 100m
      memory: 512Mi
生成的 Fluent Bit 配置默认使用 filesystem 存储、30MB 容器日志 Mem_Buf_Limit、5MB mounted-file Mem_Buf_Limit,并在磁盘 chunk 超过 storage.max_chunks_up 时暂停输入以约束内存。调高内存前请先检查输出背压与 storage backlog。

执行安装:

helm upgrade --install log-router ./charts/log-router \
  -n fluent-bit --create-namespace \
  -f values-fluent-bit.yaml
基础模板 fluent-bit.conf 和 fluent-bit-parsers.conf 由 chart 生成 ConfigMap 并挂载。修改基础模板只需 helm upgrade,无需重新构建镜像。

三、配置示例
1. 基础输出(Elasticsearch)
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: <your-namespace>
data:
  fluent.conf: |
    [OUTPUT]
        Name            es
        Match           kube.*
        Host            <elasticsearch-host>
        Port            9200
        Logstash_Format On
        Logstash_Prefix <your-log-prefix>
2. 采集容器内文件日志(无 Sidecar)
保留了原有的 <source @type mounted-file> 语法,config-reloader 会自动将其转换为 Fluent Bit 的 tail input。

apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: <your-namespace>
data:
  fluent.conf: |
    # 采集容器内 /app/logs/test.log (需挂载 emptyDir)
    <source>
      @type mounted-file
      path /app/logs/test.log
      labels app=file-log-demo, _container=app
    </source>
    [OUTPUT]
        Name  stdout
        Match kube.*
    # 文件日志的默认 tag 格式为 podfile.<namespace>.*
    [OUTPUT]
        Name  stdout
        Match podfile.<your-namespace>.*
3. 时间格式(time_format)自动转换
mounted-file 的 <parse> 块沿用 Fluentd/Ruby 的 time_format 写法(strptime 风格),但 Fluent Bit 内置的 flb_strptime 与 Ruby strptime 并不完全兼容。config-reloader 在转换时会自动做以下归一化(见 config-reloader/generator/generator.go 的 normalizeFluentBitTimeFormat):

源里写法(Fluentd 风格)	转换后(Fluent Bit 实际生效)	说明
%N、%1N ~ %9N
%L
Fluent Bit 没有纳秒指令 %N,统一用 %L 匹配小数秒,可消费 9 位纳秒。
%:z、%::z、%:::z
%z
Fluent Bit 不支持 GNU 扩展的冒号时区指令,其 %z 本身就能解析带冒号的偏移(如 +08:00)。
例如源里写 time_format %Y-%m-%dT%H:%M:%S.%N%:z,最终生成的 parser 会是 Time_Format %Y-%m-%dT%H:%M:%S.%L%z。

编写建议:直接按服务真实日志的时间格式书写即可,%N / %:z 都会被自动适配。

常见报错与排查
4. Kafka 输出与日志上下文字段
基础采集模板会为容器 stdout/stderr 日志额外补充以下字段:

log_offset:日志在文件中的 offset,可用于同一文件内稳定排序。
kafka_key:由 namespace/pod/container 组成,适合作为 Kafka message key。
log_context_id:10 秒时间窗口对应的 epoch 秒字符串,用于快速过滤近邻上下文。
使用 Kafka output 时,建议指定 message_key_field kafka_key,让同一日志源尽量进入同一 partition:

apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: <your-namespace>
data:
  fluent.conf: |
    [OUTPUT]
        Name              kafka
        Match             kube.*
        Brokers           <kafka-broker-1>:9092,<kafka-broker-2>:9092
        Topics            <your-topic>
        Format            json
        message_key_field kafka_key
        rdkafka.request.required.acks 1
Elasticsearch index template 建议
四、监控与排障
Fluent Bit Metrics:暴露在 2020 端口(/api/v2/metrics/prometheus)。
Reloader Metrics:暴露在 9000 端口。
状态与缓存目录:默认挂载于宿主机的 <host-state-dir>。
查看日志:

# Fluent Bit 运行日志
kubectl -n fluent-bit logs daemonset/log-router-log-router -c fluent-bit --tail=100
# 配置发现与热重载日志(命名空间发现逻辑在 reloader 容器)
kubectl -n fluent-bit logs daemonset/log-router-log-router -c reloader --tail=100
排查"新增命名空间不生效"
五、回退到旧 Fluentd runtime
当前版本已移除旧 Fluentd runtime 分支,config-reloader 只生成并 reload Fluent Bit 配置。若要完整回退,需使用迁移前的旧镜像和旧 Helm chart。

License
Apache License 2.0
