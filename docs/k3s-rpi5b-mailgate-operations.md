# k3s on Raspberry Pi 5B Mailgate GitOps 操作

适用范围：已经完成 Mailgate GitOps 部署的单节点 Raspberry Pi 5B

如果你是在树莓派本机上操作，把 `kubectl` 换成 `sudo k3s kubectl`。

## 核心内容

### 1. 查看应用

```bash
argocd app get mailgate
argocd app diff mailgate
argocd app sync mailgate
```

### 2. 查看运行状态

```bash
kubectl get pods -n mailgate
kubectl get svc -n mailgate
kubectl get ingress -n mailgate
kubectl get pvc -n mailgate
```

### 3. 看日志

```bash
kubectl logs -n mailgate deploy/mailgate-server
kubectl logs -n mailgate deploy/mailgate-frontend
```

### 4. 访问与验证

```bash
curl -I http://mailgate.rpi5b.local
curl http://mailgate.rpi5b.local
```

API 路径前缀是：

```text
/mailgate/v1
```

### 5. 查看指标

先直接看应用端点：

```bash
kubectl -n mailgate port-forward svc/mailgate-server 31103:31103
curl http://127.0.0.1:31103/metrics
```

如果你要确认 Prometheus 是否已经接到这个目标，可以查：

```text
up{namespace="mailgate",service="mailgate-server"}
```

如果要看业务指标，优先找这两个名字：

```text
request_count
request_latency_seconds
```

如果你想看固定面板，打开 Grafana 后搜索 `Mailgate Overview`。

这个页面的第一屏现在是运维总览，顺序是：

- `Request Rate`
- `Error Rate`
- `p95 Latency`
- `p99 Latency`
- `p50 Latency`
- `Availability`
- `CPU Usage (cores)`
- `Memory Working Set`
- `Request Rate Trend`
- `Latency Trend`
- `Request Rate by Method`
- `Error Rate by Method`

如果这里看起来没有数据，先确认两件事：

1. `up{namespace="mailgate",service="mailgate-server"}` 是否为 `1`
2. 最近 5 分钟内是否真的有请求量，也就是 `sum(rate(request_count[5m]))` 是否大于 `0`

### 6. 查看告警

如果你要看 `mailgate` 的告警规则是否已经加载，可以查：

```bash
kubectl get prometheusrule -n mailgate
kubectl describe prometheusrule mailgate-alerts -n mailgate
```

如果要看 Prometheus 端已经触发了什么，可以用：

```text
ALERTS{namespace="mailgate"}
```

### 7. 同步更新

如果 Git 里改了 `examples/mailgate-app/base/`：

```bash
git -C /tmp/k3s-lab-mirror.git fetch origin main
git -C /tmp/k3s-lab-mirror.git update-server-info
kubectl annotate application mailgate -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

### 8. 回滚

```bash
argocd app history mailgate
argocd app rollback mailgate <id>
```

### 9. 清理

```bash
kubectl delete application mailgate -n argocd
kubectl delete namespace mailgate
```

## 常用命令

```bash
kubectl get pods -n mailgate
kubectl get ingress -n mailgate
kubectl logs -n mailgate deploy/mailgate-server
kubectl logs -n mailgate deploy/mailgate-frontend
argocd app diff mailgate
argocd app sync mailgate
```

## 备注

- 这份文档只记录 Mailgate 的日常操作，不重复安装流程。
- 如果你要重做部署，看 [`k3s-rpi5b-mailgate-stack.md`](./k3s-rpi5b-mailgate-stack.md)。
- 如果你要查 Argo CD 的通用操作，看 [`k3s-rpi5b-argocd-operations.md`](./k3s-rpi5b-argocd-operations.md)。
