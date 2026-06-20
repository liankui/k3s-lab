# k3s on Raspberry Pi 5B Mailgate GitOps 部署

适用范围：Raspberry Pi 5B, Debian 13, 单节点 k3s

## 核心内容

### 1. 适用场景

这份文档用于把 `mailgate/mailgate-server:latest-arm64` 和 `mailgate/mailgate-frontend:latest-arm64` 这两个镜像，作为第二个 GitOps 应用部署到当前单节点 k3s。

这个应用的访问路径是：

- Web 界面：`http://mailgate.rpi5b.local`
- API：同源访问，路径前缀为 `/mailgate/v1`

### 2. 前置条件

- Argo CD 已安装并可用
- 这份仓库已经推送到 GitHub
- 树莓派上的本地 Git mirror 已经同步到最新提交
- `mailgate.rpi5b.local` 能解析到树莓派地址

如果你是在树莓派本机上操作，把 `kubectl` 换成 `sudo k3s kubectl`。

如果你的访问端没有 DNS 记录，可以先加一条：

```text
<pi-ip> mailgate.rpi5b.local
```

### 3. 镜像与运行方式

这个 GitOps 应用对应的运行约定来自 `chaos-io/mailgate/release/README.md`：

- 后端镜像：`mailgate/mailgate-server:latest-arm64`
- 前端镜像：`mailgate/mailgate-frontend:latest-arm64`
- 后端监听：`31101` HTTP、`31102` gRPC、`31103` debug
- 前端监听：`80`，对外入口在 Ingress 上暴露
- 前端通过 Caddy 把 `/mailgate/v1` 反代到后端
- 后端使用 SQLite，数据写入 `/app/data/mailgate.db`

### 4. 安装这个应用

先把仓库里的 `mailgate` 示例提交推到 GitHub，然后刷新树莓派上的本地 mirror：

```bash
git -C /tmp/k3s-lab-mirror.git fetch origin main
git -C /tmp/k3s-lab-mirror.git update-server-info
```

然后把 bootstrap `Application` 应用到集群：

```bash
kubectl apply -f examples/mailgate-app/bootstrap/application.yaml
```

如果你是在树莓派本机上操作，也可以先把这个文件拷过去再应用：

```bash
sudo k3s kubectl apply -f /path/to/examples/mailgate-app/bootstrap/application.yaml
```

### 5. 验证安装

```bash
kubectl get application mailgate -n argocd
kubectl get pods -n mailgate -o wide
kubectl get svc -n mailgate
kubectl get ingress -n mailgate
kubectl get pvc -n mailgate
```

期望结果：

- `mailgate-server` 和 `mailgate-frontend` 都是 `Running`
- `mailgate-data` PVC 已绑定
- `mailgate.rpi5b.local` 的 ingress 已创建

### 6. 访问服务

浏览器访问：

```text
http://mailgate.rpi5b.local
```

如果你想在命令行里先确认入口：

```bash
curl -I http://mailgate.rpi5b.local
```

API 同源路径是：

```text
/mailgate/v1
```

### 7. 数据持久化

后端使用 SQLite，数据文件在：

```text
/app/data/mailgate.db
```

这个路径由 `mailgate-data` PVC 持久化，所以 Pod 重建后数据不会直接丢失。

### 8. 变更与同步

后续如果你改了 `examples/mailgate-app/base/` 下的任意 YAML，按这个顺序复用：

1. 提交并推送仓库
2. 刷新树莓派上的本地 mirror
3. 让 Argo CD 自动同步

如果需要手动加速同步：

```bash
kubectl annotate application mailgate -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

### 9. 清理

```bash
kubectl delete application mailgate -n argocd
kubectl delete namespace mailgate
```

## 常用命令

```bash
kubectl get application mailgate -n argocd
kubectl get pods -n mailgate
kubectl get ingress -n mailgate
kubectl get pvc -n mailgate
kubectl annotate application mailgate -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

## 备注

- 这份示例把 `mailgate` 作为第二个 GitOps 应用，和 `hello-web` demo 分开管理。
- 当前使用的是 `latest-arm64`，适合先把流程跑通；如果后面要更严格的可重复性，建议再把镜像 pin 到固定版本或 digest。
- 如果后面要继续扩展，可以再把数据库、缓存和对象存储补成可选子系统，但这份 GitOps 示例先保持最小可用。
- 操作命令和其他 k3s 速查仍然看 [`k3s-rpi5b-commands.md`](./k3s-rpi5b-commands.md)。
