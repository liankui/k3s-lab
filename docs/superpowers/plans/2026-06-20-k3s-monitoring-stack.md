# k3s Monitoring Stack Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Install a reusable Prometheus + Grafana monitoring stack on the Raspberry Pi 5B k3s cluster, expose it through Traefik, and document the deployment so it can be repeated later.

**Architecture:** Use `kube-prometheus-stack` as one Helm release in the `monitoring` namespace. Keep the configuration in a dedicated values file so installation and reinstallation use the same inputs. Expose Grafana and Prometheus through Traefik Ingress, keep persistence enabled for both, and keep the resource footprint modest for a single-node Pi.

**Tech Stack:** `k3s`, `Helm`, `kube-prometheus-stack`, `Traefik`, `Prometheus`, `Grafana`, `local-path-provisioner`, Markdown

---

### Task 1: Create the reusable Helm values

**Files:**
- Create: `docs/k3s-rpi5b-monitoring-values.yaml`

- [ ] **Step 1: Write the Helm values file**

```yaml
fullnameOverride: monitoring

grafana:
  enabled: true
  persistence:
    enabled: true
    type: sts
    storageClassName: local-path
    accessModes:
      - ReadWriteOnce
    size: 1Gi
  ingress:
    enabled: true
    ingressClassName: traefik
    hosts:
      - grafana.rpi5b.local
    path: /
    pathType: Prefix
  resources:
    requests:
      cpu: 50m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi

prometheus:
  enabled: true
  prometheusSpec:
    retention: 7d
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 300m
        memory: 512Mi
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 2Gi
  ingress:
    enabled: true
    ingressClassName: traefik
    hosts:
      - prometheus.rpi5b.local
    paths:
      - /
    pathType: Prefix

alertmanager:
  enabled: true

kubeStateMetrics:
  enabled: true

nodeExporter:
  enabled: true

defaultRules:
  create: true

prometheusOperator:
  resources:
    requests:
      cpu: 50m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi
```

- [ ] **Step 2: Check the file for duplicate keys**

Run:

```bash
sed -n '1,240p' docs/k3s-rpi5b-monitoring-values.yaml
```

Expected:
- The file contains one `grafana:` block only
- The file contains the ingress and persistence settings for Grafana
- The file contains the Prometheus storage and ingress settings

- [ ] **Step 3: Fix any duplicate or conflicting values**

If the file contains duplicate top-level keys, merge them so the final YAML keeps only one `grafana:` block with both persistence and resource settings.

### Task 2: Write the reusable deployment guide

**Files:**
- Create: `docs/k3s-rpi5b-monitoring-stack.md`
- Modify: `docs/k3s-rpi5b-commands.md`

- [ ] **Step 1: Draft the deployment document**

```md
# k3s on Raspberry Pi 5B Monitoring Stack

## What this installs

This guide installs `kube-prometheus-stack` into the `monitoring` namespace and exposes Grafana and Prometheus through Traefik.

## Prerequisites

- k3s cluster is already running
- `helm` is installed on the machine used to run the commands
- `grafana.rpi5b.local` and `prometheus.rpi5b.local` resolve to the Pi via DNS or `/etc/hosts`

## Install

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --version 86.3.2 \
  --values docs/k3s-rpi5b-monitoring-values.yaml
```

## Verify

```bash
kubectl get pods -n monitoring
kubectl get ingress -n monitoring
kubectl get pvc -n monitoring
kubectl get svc -n monitoring
```

## Access

- Grafana: `http://grafana.rpi5b.local`
- Prometheus: `http://prometheus.rpi5b.local`

## Grafana admin password

```bash
kubectl get secret -n monitoring monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

## Uninstall

```bash
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```
```

- [ ] **Step 2: Add a short monitoring section to the command cheat sheet**

Append the following block to `docs/k3s-rpi5b-commands.md`:

```md
### Monitoring stack

```bash
kubectl get pods -n monitoring
kubectl get ingress -n monitoring
kubectl get pvc -n monitoring
kubectl get secret -n monitoring monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
```
```

- [ ] **Step 3: Review the document for repetition**

Run:

```bash
sed -n '1,240p' docs/k3s-rpi5b-monitoring-stack.md
sed -n '1,260p' docs/k3s-rpi5b-commands.md
```

Expected:
- The deployment guide contains install, verify, access, and uninstall steps
- The cheat sheet only adds a short monitoring block without repeating the install guide

### Task 3: Install and verify the stack on the Pi

**Files:**
- Modify: `docs/k3s-rpi5b-monitoring-stack.md` (final notes if needed)

- [ ] **Step 1: Deploy the chart**

Run on the operator machine:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --version 86.3.2 \
  --values docs/k3s-rpi5b-monitoring-values.yaml
```

- [ ] **Step 2: Wait for workloads**

Run:

```bash
kubectl get pods -n monitoring -w
```

Expected:
- Grafana, Prometheus, Alertmanager, kube-state-metrics, and node-exporter all reach `Running`

- [ ] **Step 3: Verify ingress and credentials**

Run:

```bash
kubectl get ingress -n monitoring
kubectl get secret -n monitoring monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

Expected:
- Grafana and Prometheus ingress objects exist
- Grafana admin password is printed

- [ ] **Step 4: Confirm browser access**

Open:

```text
http://grafana.rpi5b.local
http://prometheus.rpi5b.local
```

Expected:
- Grafana login page loads
- Prometheus UI loads

- [ ] **Step 5: Commit the monitoring docs**

```bash
git add docs/k3s-rpi5b-monitoring-stack.md docs/k3s-rpi5b-monitoring-values.yaml docs/k3s-rpi5b-commands.md docs/superpowers/plans/2026-06-20-k3s-monitoring-stack.md
git commit -m "docs: add monitoring stack deployment guide"
```
