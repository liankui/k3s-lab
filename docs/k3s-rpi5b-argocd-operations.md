# k3s on Raspberry Pi 5B Argo CD GitOps 操作

适用范围：已经完成 Argo CD 部署的单节点 Raspberry Pi 5B

如果你是在树莓派本机上操作，把 `kubectl` 换成 `sudo k3s kubectl`，把 `argocd` CLI 和 `kubectl` 统一指向同一套 kubeconfig。

## 核心内容

### 1. 登录

如果你是通过端口转发访问：

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

另一个终端里登录：

```bash
argocd login 127.0.0.1:8080 --username admin --password <password> --plaintext
```

如果你是通过 `http://argocd.rpi5b.local` 访问，先把域名解析好，再按浏览器和 CLI 的实际网络路径登录。

### 2. 查看应用

```bash
argocd app list
argocd app get <app-name>
argocd app diff <app-name>
```

### 3. 同步应用

手动同步：

```bash
argocd app sync <app-name>
```

建议长期使用的自动同步模式：

- `prune: true`：Git 里删掉的资源会同步删除
- `selfHeal: true`：集群里被手动改动的资源会自动拉回到 Git 状态

如果你希望每次同步都尽量回到声明式状态，可以使用：

```bash
argocd app sync <app-name> --prune --self-heal
```

### 4. 新增应用

后续每个新应用都建议沿用同一个模式：

1. 在 Git 仓库里新增一个应用目录
2. 写好 Kubernetes manifests 或 Helm/Kustomize 入口
3. 新建一个 `Application`
4. 让 Argo CD 负责持续同步

示例：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-org>/<your-repo>.git
    targetRevision: main
    path: apps/demo
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 5. 回滚

```bash
argocd app history <app-name>
argocd app rollback <app-name> <id>
```

### 6. 常见维护

查看集群里的 Argo CD 组件：

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
kubectl get ingress -n argocd
```

查看初始管理员密码：

```bash
kubectl get secret -n argocd argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

如果需要重置管理登录，最稳妥的做法是删掉 `argocd-initial-admin-secret` 之后重新从 Secret 里取一次密码。

## 常用命令

```bash
argocd app list
argocd app sync <app-name> --prune --self-heal
argocd app diff <app-name>
argocd app history <app-name>
kubectl get pods -n argocd
kubectl get ingress -n argocd
```

## 备注

- 这份文档只记录 Argo CD 的日常操作，不重复安装流程。
- 第一个可直接跑的 GitOps 应用示例看 [`k3s-rpi5b-argocd-first-app.md`](./k3s-rpi5b-argocd-first-app.md)。
- 如果你要新建第一批 GitOps 应用，先看 [`k3s-rpi5b-argocd-stack.md`](./k3s-rpi5b-argocd-stack.md)。
- 如果你要查 k3s 和通用命令，再看 [`k3s-rpi5b-commands.md`](./k3s-rpi5b-commands.md)。
