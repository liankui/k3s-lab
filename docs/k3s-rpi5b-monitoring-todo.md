# k3s on Raspberry Pi 5B 监控指标 TODO

适用范围：单节点 k3s 监控补强规划

## 目标

这份文档只记录后续要补的监控能力，暂不执行安装。

## 最小建议补强顺序

### 1. 先加回 `kube-state-metrics`

- 目的：补齐 Kubernetes 对象状态指标
- 重点关注：Deployment、Pod、Service、Ingress、PVC 的状态变化
- 完成标准：Grafana 能看到对象状态面板，Prometheus 能抓到 `kube-state-metrics` 指标

### 2. 再加 `blackbox-exporter`

- 目的：做站点和接口探活
- 重点关注：Grafana 访问页、Prometheus 访问页、业务 HTTP 接口是否可达
- 完成标准：Prometheus 能定时抓取探活结果，失败时能触发告警

### 3. 再补 `Loki + Promtail`

- 目的：把日志纳入统一查看
- 重点关注：k3s 系统日志、监控栈日志、后续业务容器日志
- 完成标准：Grafana 可以按 Pod、Namespace、时间范围检索日志

### 4. 最后接入告警通知渠道

- 目的：把告警真正送到日常使用的地方
- 重点关注：邮件、Telegram、企业微信、飞书或其他常用渠道
- 完成标准：关键告警能稳定送达，且能区分严重级别

## 建议补充顺序

1. 先补可观测对象状态，再补可用性探测
2. 再补日志，最后再做通知通道
3. 每加一层，都先确保当前层面板和告警都正常

## 备注

- 当前这套基础监控已经能看节点、Prometheus、Grafana 和基础告警框架
- 这份 TODO 的作用是把后续增强拆开，避免一次性扩太多
- 如果以后开始实施，可以按本文件顺序逐项推进
