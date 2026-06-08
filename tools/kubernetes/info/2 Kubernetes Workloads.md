# Kubernetes — Workloads

> **Теги:** #kubernetes #deployment #configmap #secret #конспект

> [!abstract] Связи
> [[main]] | [[main Tools]]

---

## 🔹 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
```

---

## 🔹 Probes

- livenessProbe — жив ли контейнер
- readinessProbe — готов ли к трафику

---

## 🔹 ConfigMap и Secret

- ConfigMap: application config
- Secret: credentials, tokens

---

## 🔹 Итог

```
Шпаргалка Workloads:
─────────────────────────────────────────
Deployment + replicas
liveness/readiness probes
ConfigMap для конфига
Secret для секретов
kubectl rollout status/undo
```
