# k3s Monitoring Stack Design

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy a reusable monitoring stack on the Raspberry Pi 5B k3s cluster using Prometheus and Grafana, exposed through Traefik and configured for persistent storage.

**Architecture:** Install `kube-prometheus-stack` as a single Helm release in a dedicated `monitoring` namespace. Use k3s `Traefik` for HTTP ingress so Grafana and Prometheus can be reached from a browser without port-forwarding. Keep the stack lightweight for a single-node Pi by using local-path persistent volumes, conservative resource requests, and the chart's default alerting components.

**Tech Stack:** `k3s`, `Helm`, `kube-prometheus-stack`, `Traefik`, `Prometheus`, `Grafana`, `local-path-provisioner`

---

## Scope

This design covers:

- Installation of a reusable monitoring stack
- Persistent storage for Grafana and Prometheus
- Browser access through Traefik
- A deployment document that can be reused later
- A concise values file that can be applied again on reinstall

This design does not cover:

- Multi-node HA Prometheus
- External long-term metrics storage
- Email/Slack/WeCom alert routing
- Custom dashboards beyond the chart defaults

## File Structure

- Create: `docs/k3s-rpi5b-monitoring-stack.md`
- Create: `docs/k3s-rpi5b-monitoring-values.yaml`
- Modify: `docs/k3s-rpi5b-commands.md`

## Components

### Monitoring namespace

Use a dedicated `monitoring` namespace to isolate stack resources from application workloads.

### Helm release

Install `kube-prometheus-stack` as a single release named `monitoring` so the stack can be upgraded or removed as one unit.

### Grafana

Expose Grafana through Traefik. Keep persistence enabled so dashboards, data sources, and user changes survive restarts.

### Prometheus

Expose Prometheus through Traefik for verification and troubleshooting. Keep persistence enabled so retention survives restarts.

### Alertmanager

Deploy Alertmanager as part of the stack, but do not expose it externally by default. Keep its default role focused on internal alert processing.

### Persistent storage

Use the existing k3s local-path storage class. This avoids introducing a heavier storage layer and matches the single-node setup.

## Data Flow

1. kube-state-metrics and node-exporter collect cluster and node metrics.
2. Prometheus scrapes those metrics inside the cluster.
3. Prometheus stores time series data on the local persistent volume.
4. Grafana queries Prometheus and renders dashboards.
5. Traefik routes browser requests to Grafana and Prometheus ingress hosts.

## Ingress and Access

- Grafana host: `grafana.rpi5b.local`
- Prometheus host: `prometheus.rpi5b.local`

Both hosts are handled by Traefik Ingress resources created by the Helm chart values.

If local DNS is not available, the setup document should instruct the user to add temporary `/etc/hosts` entries on the client machine.

## Configuration Strategy

The values file should:

- Enable Grafana ingress
- Enable Prometheus ingress
- Enable persistence for Grafana
- Enable persistence for Prometheus
- Keep CPU and memory requests modest
- Leave Alertmanager internal only

The document should explain how to reapply the same values file after reinstalling the cluster.

## Error Handling

Document the most likely failure modes:

- `ImagePullBackOff`: confirm the Pi can reach the configured image registries
- `Pending` PVCs: confirm `local-path-provisioner` and the `local-path` storage class are active
- `Ingress` not reachable: confirm Traefik is running and the client can resolve the hostnames
- `Grafana` login confusion: document the initial admin credential retrieval command
- `Prometheus` missing targets: check `kubectl get pods -n monitoring` and `kubectl get endpoints -n monitoring`

## Verification

The deployment is complete when all of the following are true:

- `kubectl get pods -n monitoring` shows Grafana, Prometheus, Alertmanager, kube-state-metrics, and node-exporter running
- `kubectl get ingress -n monitoring` shows Grafana and Prometheus ingress objects
- Grafana is reachable in a browser through Traefik
- Prometheus targets are `UP`
- The monitoring PVCs survive a k3s restart

## Testing

Manual verification is sufficient for this cluster:

- Install the chart with the values file
- Confirm pods become `Running`
- Confirm ingress routes resolve
- Confirm metrics are visible in Prometheus
- Confirm Grafana dashboards load

## Notes

- The stack is intentionally small and self-contained.
- Prefer chart defaults unless a value is needed for persistence, ingress, or single-node resource fit.
- Reuse the existing naming convention from the other k3s docs so the repository stays consistent.

