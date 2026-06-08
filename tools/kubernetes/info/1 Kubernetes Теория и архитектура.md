# Kubernetes — Теория и архитектура

> **Теги:** #kubernetes #theory #architecture #pods #конспект

> [!abstract] Связи
> [[main]] | [[main Tools]]

---

## 🔹 Что такое Kubernetes

> [!note] Определение
> Kubernetes — оркестратор контейнеров: деплой, масштабирование, self-healing, service discovery.

---

## 🔹 Архитектура

- Control Plane: API Server, etcd, Scheduler, Controller Manager
- Worker Node: kubelet, kube-proxy, Pods

---

## 🔹 Базовые сущности

- Pod
- Deployment
- Service
- Ingress
- ConfigMap / Secret

---

## 🔹 Итог

```
Шпаргалка K8s basics:
─────────────────────────────────────────
Pod = минимальная единица
Deployment = desired replicas
Service = стабильный endpoint
Ingress = внешний HTTP вход
kubectl get/apply/describe/logs
```
