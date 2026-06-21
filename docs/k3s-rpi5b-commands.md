# k3s on Raspberry Pi 5B 常用命令与命名速查

适用范围：单节点 k3s 日常操作、排障、资源命名

## 核心内容

### 1. 集群状态

```bash
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get pods -A
sudo k3s kubectl get all -A
sudo systemctl status k3s --no-pager -l
```

### 2. 排障

```bash
sudo journalctl -u k3s -n 100 --no-pager
sudo k3s kubectl get events -A --sort-by=.lastTimestamp
sudo k3s kubectl describe pod <pod-name> -n <namespace>
sudo k3s kubectl logs -f <pod-name> -n <namespace>
sudo k3s kubectl exec -it <pod-name> -n <namespace> -- sh
```

### 3. 应用操作

```bash
sudo k3s kubectl apply -f app.yaml
sudo k3s kubectl delete -f app.yaml
sudo k3s kubectl rollout status deploy/<deployment-name> -n <namespace>
sudo k3s kubectl port-forward svc/<service-name> 8080:80 -n <namespace>
```

### 4. 服务维护

```bash
sudo systemctl restart k3s
sudo /usr/local/bin/k3s-uninstall.sh
```

### 5. 监控栈

```bash
kubectl get pods -n monitoring
kubectl get ingress -n monitoring
kubectl get pvc -n monitoring
kubectl get secret -n monitoring monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

### 6. Argo CD

```bash
kubectl get pods -n argocd
kubectl get ingress -n argocd
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
argocd app list
argocd app sync <app-name> --prune --self-heal
argocd app diff <app-name>
argocd app history <app-name>
```

### 7. Mailgate

```bash
kubectl get pods -n mailgate
kubectl get svc -n mailgate
kubectl get ingress -n mailgate
kubectl get pvc -n mailgate
kubectl logs -n mailgate deploy/mailgate-server
kubectl logs -n mailgate deploy/mailgate-frontend
argocd app diff mailgate
argocd app sync mailgate
```

### 8. Mailgate 指标

```bash
kubectl -n mailgate port-forward svc/mailgate-server 31103:31103
curl http://127.0.0.1:31103/metrics
```

Prometheus 里常用查询：

```text
up{namespace="mailgate",service="mailgate-server"}
request_count
request_latency_seconds
```

Grafana 里的固定 dashboard 名称是 `Mailgate Overview`。

## 命名速查

### 1. 常见 Kubernetes 资源简称

| 资源 | 常用简称 | 示例 |
| --- | --- | --- |
| Namespace | `ns` | `demo` |
| Pod | `po` | `hello-xxxx` |
| Deployment | `deploy` | `hello` |
| Service | `svc` | `hello` |
| Ingress | `ing` | `web` |
| ConfigMap | `cm` | `app-config` |
| Secret | `secret` | `db-secret` |
| PersistentVolumeClaim | `pvc` | `data` |
| PersistentVolume | `pv` | `local-pv` |
| Job | `job` | `migrate-db` |
| CronJob | `cronjob` | `backup-nightly` |

### 2. k3s 默认常见组件名

| 组件 | 常见名称 |
| --- | --- |
| DNS | `coredns` |
| 本地存储 | `local-path-provisioner` |
| 指标服务 | `metrics-server` |
| Ingress | `traefik` |
| ServiceLB | `svclb-traefik-*` |

### 3. 命名建议

- Namespace 用短而明确的业务名，比如 `demo`、`web`、`ops`
- Deployment 和 Service 尽量保持同名，比如都叫 `hello`
- ConfigMap / Secret 用用途命名，比如 `app-config`、`db-secret`
- PVC 用数据语义命名，比如 `data`、`uploads`、`cache`

## 备注

- 这份文档只收日常高频命令，不重复安装步骤。
- 如果要部署新应用，先看 [`k3s-rpi5b-first-deploy-checklist.md`](./k3s-rpi5b-first-deploy-checklist.md)。
- 如果要重装或修环境，先看 [`k3s-rpi5b-setup.md`](./k3s-rpi5b-setup.md)。
- 如果要装监控栈，先看 [`k3s-rpi5b-monitoring-stack.md`](./k3s-rpi5b-monitoring-stack.md)。
- 如果要做 GitOps 部署，先看 [`k3s-rpi5b-argocd-stack.md`](./k3s-rpi5b-argocd-stack.md) 和 [`k3s-rpi5b-argocd-operations.md`](./k3s-rpi5b-argocd-operations.md)。
- 如果要部署 Mailgate 第二个 GitOps 应用，先看 [`k3s-rpi5b-mailgate-stack.md`](./k3s-rpi5b-mailgate-stack.md) 和 [`k3s-rpi5b-mailgate-operations.md`](./k3s-rpi5b-mailgate-operations.md)。
