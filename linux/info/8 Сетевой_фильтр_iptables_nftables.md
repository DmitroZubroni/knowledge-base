# Сетевой фильтр — iptables / nftables

> **Теги:** #linux #network #iptables #nftables #firewall #security #конспект

> [!abstract] Связи
> [[main]] | [[main Linux]]

---

## 🔹 Netfilter

Цепочки: `INPUT`, `OUTPUT`, `FORWARD`, `PREROUTING`, `POSTROUTING`.

---

## 🔹 iptables пример

```bash
iptables -P INPUT DROP
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
```

---

## 🔹 nftables пример

```bash
nft add table inet filter
nft add chain inet filter input '{ type filter hook input priority 0; policy drop; }'
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input tcp dport 22 accept
```

---

## 🔹 Итог

```
Шпаргалка firewall:
─────────────────────────────────────────
Netfilter chains: INPUT/OUTPUT/FORWARD
iptables legacy
nftables modern
allow ESTABLISHED,RELATED
open only required ports
```
