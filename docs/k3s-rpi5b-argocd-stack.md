# k3s on Raspberry Pi 5B Argo CD GitOps 部署

适用范围：Raspberry Pi 5B, Debian 13, 单节点 k3s

## 核心内容

### 1. 适用场景

这份文档用于在当前单节点 k3s 上安装 Argo CD，并通过现有的 `traefik` 暴露 Web 入口。

推荐用途：

- 后续把应用部署流程改成 GitOps
- 让部署动作尽量只保留在 Git 里
- 后面新增应用时，重复使用同一套 Argo CD 流程

### 2. 前置条件

- k3s 已安装并且节点已 `Ready`
- `helm` 已安装在执行部署命令的机器上
- `argocd.rpi5b.local` 能解析到这台 Pi 的地址
- 当前集群已经在运行 `traefik`

如果你是在树莓派本机上操作，把 `kubectl` 换成 `sudo k3s kubectl`，把 `helm` 前面加上 `sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml`。

如果当前电脑没有 DNS 记录，可以先在本机 `/etc/hosts` 加一条映射：

```text
<pi-ip> argocd.rpi5b.local
```

### 3. 安装 Argo CD

先添加 Helm 仓库：

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

然后安装：

```bash
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 9.5.22 \
  --values docs/k3s-rpi5b-argocd-values.yaml
```

如果你是在树莓派本机执行，也可以显式指定：

```bash
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 9.5.22 \
  --values /path/to/docs/k3s-rpi5b-argocd-values.yaml
```

### 4. 验证安装

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
kubectl get ingress -n argocd
```

期望结果：

- `argocd-server`、`argocd-repo-server`、`argocd-application-controller`、`argocd-dex-server`、`argocd-notifications-controller` 都已启动
- `argocd-server` 已经通过 `traefik` 暴露

### 5. 打开 Web 界面

浏览器访问：

```text
http://argocd.rpi5b.local
```

Argo CD 初始管理员用户名是 `admin`。

初始密码可以从 Secret 里读取：

```bash
kubectl get secret -n argocd argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

如果你更习惯走本地端口转发，也可以：

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

然后在另一个终端登录：

```bash
argocd login 127.0.0.1:8080 --username admin --password <password> --plaintext
```

### 6. 初始化第一个 GitOps 应用

后续建议把应用定义写成 Argo CD `Application`，然后直接 `apply` 到集群里。

典型结构如下：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hello
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-org>/<your-repo>.git
    targetRevision: main
    path: apps/hello
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

这就是后面重复使用的关键：以后每个应用都沿用同样的 `Application` 模式，只改 `repoURL`、`path` 和 `namespace`。

### 7. 卸载

```bash
helm uninstall argocd -n argocd
kubectl delete namespace argocd
```

## 常用命令

```bash
kubectl get pods -n argocd
kubectl get ingress -n argocd
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

## 备注

- 这套部署默认用现有 `traefik` 暴露 Web 入口，不额外引入新的入口控制器。
- 当前已验证的 Helm chart 版本是 `argo-cd-9.5.22`，应用版本是 `v3.4.4`。
- 当前入口通过 `http://argocd.rpi5b.local` 提供，Traefik 已返回 `200 OK`。
- 如果后面要做更正式的访问控制，再考虑给 `argocd.rpi5b.local` 加 TLS。
- 如果后面要把所有应用都纳入 GitOps，先从这份文档里的 `Application` 模板开始复用。
- 第一个可直接跑的示例看 [`k3s-rpi5b-argocd-first-app.md`](./k3s-rpi5b-argocd-first-app.md)。
- 日常操作和重复部署命令看 [`k3s-rpi5b-argocd-operations.md`](./k3s-rpi5b-argocd-operations.md)。
