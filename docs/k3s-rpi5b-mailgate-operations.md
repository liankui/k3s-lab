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

### 5. 同步更新

如果 Git 里改了 `examples/mailgate-app/base/`：

```bash
git -C /tmp/k3s-lab-mirror.git fetch origin main
git -C /tmp/k3s-lab-mirror.git update-server-info
kubectl annotate application mailgate -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

### 6. 回滚

```bash
argocd app history mailgate
argocd app rollback mailgate <id>
```

### 7. 清理

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
