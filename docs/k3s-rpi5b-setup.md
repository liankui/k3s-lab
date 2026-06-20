# k3s on Raspberry Pi 5B

Tested on: Debian 13 (trixie), ARM64, Raspberry Pi 5B

## Quick setup

### 1) Enable memory cgroup

Edit `/boot/firmware/cmdline.txt` and add:

```text
cgroup_enable=memory cgroup_memory=1
```

Reboot:

```bash
sudo reboot
```

Verify:

```bash
cat /sys/fs/cgroup/cgroup.controllers
```

`memory` should be listed.

### 2) Install k3s

```bash
curl -sfL https://get.k3s.io | sudo sh -
```

### 3) Set image sources

Create `/etc/rancher/k3s/config.yaml`:

```yaml
pause-image: registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.10
```

Create `/etc/rancher/k3s/registries.yaml`:

```yaml
mirrors:
  docker.io:
    endpoint:
      - "https://docker.m.daocloud.io"
```

Restart:

```bash
sudo systemctl restart k3s
```

### 4) Verify

```bash
sudo systemctl is-active k3s
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get pods -A
```

Expected:

- node status: `Ready`
- pods: `coredns`, `local-path-provisioner`, `metrics-server`, `traefik` are `Running`

## Useful commands

```bash
sudo journalctl -u k3s -n 100 --no-pager
sudo /usr/local/bin/k3s-uninstall.sh
```

## Notes

- This Pi needed the memory cgroup fix before k3s started cleanly.
- Docker Hub and `registry.k8s.io` were unreliable here, so the mirror setup above was required.
- Final working version: `v1.35.5+k3s1`
