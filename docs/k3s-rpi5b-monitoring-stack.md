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
