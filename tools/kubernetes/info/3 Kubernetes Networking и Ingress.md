# Kubernetes — Networking и Ingress

> **Теги:** #kubernetes #networking #ingress #service #конспект

> [!abstract] Связи
> [[main]] | [[main Tools]]

---

## 🔹 Service types

- ClusterIP
- NodePort
- LoadBalancer

---

## 🔹 Ingress

Ingress роутит HTTP host/path к Service.

---

## 🔹 DNS и discovery

`<service>.<namespace>.svc.cluster.local`

---

## 🔹 Итог

```
Шпаргалка K8s Networking:
─────────────────────────────────────────
Service ClusterIP
Ingress host/path routing
Ingress Controller обязателен
DNS внутри кластера
TLS через secret
```
