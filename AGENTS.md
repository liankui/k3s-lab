# AGENTS.md

## Scope

- This repository is a documentation-only k3s lab for Raspberry Pi 5B on Debian 13.
- Prefer the commands, file paths, and values already documented under `docs/`.
- Keep changes minimal and grounded in the existing docs; if something is uncertain, add a short `TODO` instead of guessing.

## Primary References

- `docs/k3s-rpi5b-setup.md`
- `docs/k3s-rpi5b-first-deploy-checklist.md`
- `docs/k3s-rpi5b-commands.md`
- `docs/k3s-rpi5b-monitoring-stack.md`
- `docs/k3s-rpi5b-monitoring-values.yaml`
- `docs/k3s-rpi5b-monitoring-todo.md`

## Common Workflow

1. Enable memory cgroup in `/boot/firmware/cmdline.txt` by adding `cgroup_enable=memory cgroup_memory=1` after `rootwait`, then reboot.
2. Install k3s with `curl -sfL https://get.k3s.io | sudo sh -`.
3. Configure k3s at `/etc/rancher/k3s/config.yaml` and `/etc/rancher/k3s/registries.yaml` when mirror settings are needed.
4. Restart k3s with `sudo systemctl restart k3s`.
5. Verify cluster health with `sudo k3s kubectl get nodes -o wide` and `sudo k3s kubectl get pods -A`.

## Common Commands

### Cluster status

```bash
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get pods -A
sudo k3s kubectl get all -A
sudo systemctl status k3s --no-pager -l
```

### Troubleshooting

```bash
sudo journalctl -u k3s -n 100 --no-pager
sudo k3s kubectl get events -A --sort-by=.lastTimestamp
sudo k3s kubectl describe pod <pod-name> -n <namespace>
sudo k3s kubectl logs -f <pod-name> -n <namespace>
sudo k3s kubectl exec -it <pod-name> -n <namespace> -- sh
```

### App operations

```bash
sudo k3s kubectl apply -f app.yaml
sudo k3s kubectl delete -f app.yaml
sudo k3s kubectl rollout status deploy/<deployment-name> -n <namespace>
sudo k3s kubectl port-forward svc/<service-name> 8080:80 -n <namespace>
```

### Service maintenance

```bash
sudo systemctl restart k3s
sudo /usr/local/bin/k3s-uninstall.sh
```

### First deploy

```bash
sudo k3s kubectl create namespace demo
sudo k3s kubectl apply -f demo.yaml
sudo k3s kubectl -n demo get pods,svc
sudo k3s kubectl -n demo port-forward svc/hello 8080:80
curl http://127.0.0.1:8080
```

### Monitoring stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --version 86.3.2 \
  --values docs/k3s-rpi5b-monitoring-values.yaml
kubectl get pods -n monitoring
kubectl get ingress -n monitoring
kubectl get pvc -n monitoring
kubectl get secret -n monitoring monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

## Notes

- Use `sudo k3s kubectl` for commands run directly on the Pi.
- Use plain `kubectl` only when the current shell already points at `/etc/rancher/k3s/k3s.yaml`.
- The monitoring stack docs assume `grafana.rpi5b.local` and `prometheus.rpi5b.local`.
- `kube-state-metrics` is intentionally not enabled in the current single-node monitoring values.
- TODO: add any new app-specific deployment steps only after they are documented in `docs/`.
