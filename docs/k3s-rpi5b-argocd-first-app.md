# k3s on Raspberry Pi 5B 第一个 GitOps 应用示例

适用范围：已经安装 Argo CD 的单节点 Raspberry Pi 5B

## 目标

这份示例把 Argo CD 的完整闭环跑通：

1. 用一个 `Application` 把 Git 仓库里的清单接入 Argo CD
2. 让 Argo CD 自动创建命名空间、Deployment、Service 和 Ingress
3. 通过修改 Git 文件观察 Argo CD 自动同步

## 目录结构

示例文件放在仓库的 `examples/argocd-first-app/` 下：

- `base/`：真正由 Argo CD 持续同步的应用清单
- `bootstrap/`：一次性引导 Argo CD 的 `Application`

## 1. 前置条件

- Argo CD 已安装并且 `argocd-server` 可访问
- 这份仓库已经推送到 GitHub
- `hello.rpi5b.local` 可以解析到树莓派地址，或者你愿意用 `curl -H 'Host: hello.rpi5b.local'`

如果你没有本机 DNS，可以先在访问端 `/etc/hosts` 加一条：

```text
<pi-ip> hello.rpi5b.local
```

## 2. 先看示例应用做了什么

这个示例包含：

- 一个 `demo-gitops` 命名空间
- 一个带有静态 HTML 页面的 `ConfigMap`
- 一个 `nginx:1.27-alpine` Deployment
- 一个 `Service`
- 一个 `Traefik Ingress`

页面内容现在是：

```text
Hello from GitOps v1
```

## 3. 用 Argo CD 接管这个应用

先把 bootstrap `Application` 应用到集群：

```bash
kubectl apply -f https://raw.githubusercontent.com/liankui/k3s-lab/main/examples/argocd-first-app/bootstrap/application.yaml
```

如果你是在树莓派本机上操作，也可以先把文件下载下来再应用：

```bash
curl -fsSL -o /tmp/hello-web-application.yaml \
  https://raw.githubusercontent.com/liankui/k3s-lab/main/examples/argocd-first-app/bootstrap/application.yaml
sudo k3s kubectl apply -f /tmp/hello-web-application.yaml
```

然后查看 Argo CD 是否已经接管：

```bash
kubectl get applications.argoproj.io -n argocd
kubectl describe application hello-web -n argocd
```

期望结果：

- `hello-web` 这个 Application 已创建
- `demo-gitops` 命名空间会被自动生成
- `hello-web` 的资源开始同步到集群

## 4. 验证页面

查看 Pod 和 Ingress：

```bash
kubectl get pods -n demo-gitops
kubectl get ingress -n demo-gitops
```

访问页面：

```bash
curl http://hello.rpi5b.local
```

如果你没有 DNS，也可以在树莓派上本地测：

```bash
curl -H "Host: hello.rpi5b.local" http://127.0.0.1
```

期望页面里能看到：

```text
Hello from GitOps v1
```

## 5. 做一次 Git 变更

打开 `examples/argocd-first-app/base/configmap.yaml`，把标题改成例如：

```text
Hello from GitOps v2
```

然后提交并推送到 GitHub。

Argo CD 会自动发现这个变更，并把集群里的页面内容同步到新版本。

同步后刷新页面，你应该能看到新的文案。如果浏览器缓存比较顽固，等几十秒再试一次。

## 6. 回滚

如果要回到上一个版本，有两种方式：

直接回退 Git 提交并推送，Argo CD 会把集群状态拉回 Git 中的版本。  
如果你想强制让 Nginx 重新加载当前内容，也可以补一次：

```bash
kubectl rollout restart deploy/hello-web -n demo-gitops
```

## 7. 清理

删除 Application 后，Argo CD 会根据 `prune: true` 清掉大部分资源：

```bash
kubectl delete application hello-web -n argocd
kubectl delete namespace demo-gitops
```

## 备注

- 这个示例刻意保持很小，方便你复制成第二个、第三个应用。
- 如果你想把“一个仓库管理多个应用”做成固定套路，下一步可以继续加 `apps/` 目录和 app-of-apps 结构。
- 这份示例和现有 Argo CD 部署文档配套使用：先看 [`k3s-rpi5b-argocd-stack.md`](./k3s-rpi5b-argocd-stack.md)，再看这份文档。
