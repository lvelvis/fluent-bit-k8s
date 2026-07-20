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
