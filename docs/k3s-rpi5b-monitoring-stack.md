# k3s on Raspberry Pi 5B Monitoring Stack

适用范围：Raspberry Pi 5B, Debian 13, 单节点 k3s

## 核心内容

### 1. 前置条件

- k3s 已安装并且节点已 `Ready`
- `helm` 已安装在执行部署命令的机器上
- `grafana.rpi5b.local` 和 `prometheus.rpi5b.local` 能解析到 Pi 的地址

### 2. 安装监控栈

在具备集群访问权限的机器上执行：

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --version 86.3.2 \
  --values docs/k3s-rpi5b-monitoring-values.yaml
```

如果你是在树莓派本机部署，可用：

```bash
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --version 86.3.2 \
  --values /path/to/docs/k3s-rpi5b-monitoring-values.yaml
```

### 3. 验证安装

```bash
kubectl get pods -n monitoring
kubectl get ingress -n monitoring
kubectl get pvc -n monitoring
kubectl get svc -n monitoring
```

期望结果：

- `grafana`、`prometheus`、`alertmanager`、`node-exporter` 都是 `Running`
- `grafana` 和 `prometheus` 的 ingress 已创建
- `grafana` 和 `prometheus` 的 PVC 已绑定
- 当前这份单节点配置默认不启用 `kube-state-metrics`

### 4. 访问服务

- Grafana: `http://grafana.rpi5b.local`
- Prometheus: `http://prometheus.rpi5b.local`

如果当前电脑没有 DNS 解析，先在本机 `/etc/hosts` 加入地址映射：

```text
100.109.40.53 grafana.rpi5b.local prometheus.rpi5b.local
```

如果你是在同一局域网里访问，也可以把这里替换成树莓派的内网 IP。

### 5. 获取 Grafana 密码

```bash
kubectl get secret -n monitoring monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

用户名默认是 `admin`。

### 6. 卸载

```bash
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```

### 7. 查看业务指标

`mailgate-server` 现在已经通过 `ServiceMonitor` 接入 Prometheus。

在 Prometheus 里最直接的查看方式是打开查询页，然后输入：

```text
up{namespace="mailgate",service="mailgate-server"}
```

如果要看请求量和延迟：

```text
request_count
request_latency_seconds
```

如果要看 95 分位延迟：

```text
histogram_quantile(0.95, sum by (le, method) (rate(request_latency_seconds_bucket[5m])))
```

如果你更想先看原始端点，也可以在树莓派上跑：

```bash
sudo k3s kubectl -n mailgate port-forward svc/mailgate-server 31103:31103
curl http://127.0.0.1:31103/metrics
```

如果你习惯在 Grafana 里看，打开 `http://grafana.rpi5b.local`，进 `Explore`，把上面的 PromQL 直接贴进去就行。

现在仓库已经给 `mailgate` 预置了固定 dashboard，名字是 `Mailgate Overview`，在 Grafana 里打开 `Dashboards` 后直接搜索这个名字即可。

这个页面的第一屏是运维总览，顺序是：

- `Request Rate`
- `Error Rate`
- `p95 Latency`
- `p99 Latency`
- `Request Rate Trend`
- `Latency Trend`
- `Request Rate by Method`
- `Error Rate by Method`

如果你想继续排障，`Explore` 还是最灵活的方式。

如果 dashboard 为空，不要先怀疑 `ServiceMonitor`，按下面顺序排：

1. 先看 `up{namespace="mailgate",service="mailgate-server"}`
2. 再看 `request_count`
3. 如果 `up=1` 但没有时序，通常是最近没有业务请求，或者时间范围太短
4. 如果 `up` 都没有，再回头查 `ServiceMonitor`、`Service` 标签和 `debug` 端口

## 常用命令

```bash
kubectl get pods -n monitoring
kubectl get ingress -n monitoring
kubectl get pvc -n monitoring
kubectl get secret -n monitoring monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

## 备注

- 这套配置使用 `kube-prometheus-stack` 的默认告警和采集组件。
- Grafana 和 Prometheus 都启用了本地持久化，适合单节点 Pi。
- 这台 Pi 目前不启用 `kube-state-metrics`，因为可达镜像源无法稳定提供该镜像。
- 如果以后重装 k3s，直接复用这份 values 文件和本页命令即可。
