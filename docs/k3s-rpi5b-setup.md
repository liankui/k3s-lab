# k3s on Raspberry Pi 5B 安装与验证

适用范围：Raspberry Pi 5B, Debian 13, 单节点 k3s

## 核心内容

### 1. 启用 memory cgroup

编辑 `/boot/firmware/cmdline.txt`，在 `rootwait` 后加入：

```text
cgroup_enable=memory cgroup_memory=1
```

重启：

```bash
sudo reboot
```

验证：

```bash
cat /sys/fs/cgroup/cgroup.controllers
```

结果应包含 `memory`。

### 2. 安装 k3s

```bash
curl -sfL https://get.k3s.io | sudo sh -
```

### 3. 配置镜像源

创建 `/etc/rancher/k3s/config.yaml`：

```yaml
pause-image: registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.10
```

创建 `/etc/rancher/k3s/registries.yaml`：

```yaml
mirrors:
  docker.io:
    endpoint:
      - "https://docker.m.daocloud.io"
```

重启：

```bash
sudo systemctl restart k3s
```

### 4. 验证集群

```bash
sudo systemctl is-active k3s
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get pods -A
```

期望结果：

- 节点状态是 `Ready`
- `coredns`、`local-path-provisioner`、`metrics-server`、`traefik` 是 `Running`

## 常用命令

```bash
sudo journalctl -u k3s -n 100 --no-pager
sudo /usr/local/bin/k3s-uninstall.sh
```

## 备注

- 这台 Pi 需要先启用 memory cgroup，k3s 才能正常起来。
- Docker Hub 和 `registry.k8s.io` 在这条网络上不稳定，所以用了镜像源配置。
- 验证通过版本：`v1.35.5+k3s1`
- 首次部署看 [`k3s-rpi5b-first-deploy-checklist.md`](./k3s-rpi5b-first-deploy-checklist.md)。
- 如果后面想把部署流程改成可重复的 GitOps，看 [`k3s-rpi5b-argocd-stack.md`](./k3s-rpi5b-argocd-stack.md)。
- 日常排障和命名速查请看 [`k3s-rpi5b-commands.md`](./k3s-rpi5b-commands.md)。
