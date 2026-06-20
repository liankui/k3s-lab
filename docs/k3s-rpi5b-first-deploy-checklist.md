# k3s on Raspberry Pi 5B 首次部署清单

适用范围：k3s 已安装并且节点已 `Ready` 的单节点 Raspberry Pi 5B

## 核心内容

### 1. 创建首次部署命名空间

```bash
sudo k3s kubectl create namespace demo
```

### 2. 部署第一份工作负载

保存为 `demo.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello
  namespace: demo
spec:
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 80
```

应用：

```bash
sudo k3s kubectl apply -f demo.yaml
```

### 3. 验证服务

```bash
sudo k3s kubectl -n demo get pods,svc
sudo k3s kubectl -n demo get pod -l app=hello -o wide
sudo k3s kubectl -n demo describe deploy/hello
sudo k3s kubectl -n demo logs deploy/hello
```

本地访问：

```bash
sudo k3s kubectl -n demo port-forward svc/hello 8080:80
curl http://127.0.0.1:8080
```

### 4. 清理

```bash
sudo k3s kubectl delete -f demo.yaml
sudo k3s kubectl delete namespace demo
```

## 备注

- 如果要对外暴露 HTTPS，再加 `cert-manager`。
- 如果后面要做多服务管理，优先看 Argo CD GitOps：[`k3s-rpi5b-argocd-stack.md`](./k3s-rpi5b-argocd-stack.md)。
- 日常排障和命名速查请看 [`k3s-rpi5b-commands.md`](./k3s-rpi5b-commands.md)。
