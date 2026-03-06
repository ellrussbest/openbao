Here’s a **comprehensive, readable Markdown guide** on OpenBao configuration with clusters, proxying, and ports explained, based on the official docs you linked.

---

# OpenBao Configuration & Clusters Guide

## 1. Overview

OpenBao servers are configured via **JSON or HCL files**. Key sections:

* `storage`: where secrets are stored
* `listener`: how API requests are received
* `api_addr`: public endpoint for clients
* `cluster_addr`: internal communication for multi-node clusters
* `seal`, `plugin`, `telemetry`, `audit`, etc.

Multiple configuration files can be loaded from a directory. ([docs](https://openbao.org/docs/configuration/))

---

## 2. What is a Cluster?

A **cluster** is multiple OpenBao nodes working together:

* Nodes replicate state across the cluster
* One node is **active**, others **standby**
* Provides **high availability** and **failover**
* Can handle more requests than a single node

**Cluster Example:**

```
         Clients
            │
            ▼
      Load Balancer
       /    |    \
      ▼     ▼     ▼
  Node1    Node2  Node3
(Active) (Standby) (Standby)
```

**Why clusters help:**

* **Node failure**: clients are redirected to standby nodes
* **Load distribution**: multiple nodes can serve read requests
* **Maintenance**: nodes can be upgraded without downtime

---

## 3. api_addr vs cluster_addr

| Field          | Purpose                                                |
| -------------- | ------------------------------------------------------ |
| `api_addr`     | Public endpoint clients connect to (may include port)  |
| `cluster_addr` | Internal communication between nodes (cluster traffic) |

### Important Notes on Ports:

* `api_addr` must include the **port OpenBao listens on**, e.g., `https://vault.company.com:8200`
* DNS alone cannot include a port; `vault.company.com` maps to an IP, port is handled by listener or proxy
* For friendly URLs without ports, use a **reverse proxy**:

```
Client
   │
   ▼
https://vault.company.com (443)
   │
   ▼
Reverse Proxy → OpenBao (8200)
```

---

## 4. Listener Example

**TCP Listener with TLS:**

```json
{
  "listener": [
    {
      "tcp": {
        "address": "0.0.0.0:8200",
        "tls_cert_file": "/etc/openbao/tls/server.crt",
        "tls_key_file": "/etc/openbao/tls/server.key"
      }
    }
  ]
}
```

**Development Listener (no TLS):**

```json
{
  "listener": [
    {
      "tcp": {
        "address": "127.0.0.1:8200",
        "tls_disable": true
      }
    }
  ]
}
```

---

## 5. Storage Backends

### Raft (integrated, recommended for clusters)

```json
"storage": {
  "raft": {
    "path": "/var/lib/openbao/data",
    "node_id": "node1"
  }
}
```

### PostgreSQL (external DB)

```json
"storage": {
  "postgresql": {
    "connection_url": "postgres://openbao:password@db:5432/openbao?sslmode=require",
    "table": "openbao_store"
  }
}
```

---

## 6. Example Configurations

### config-dev.json

```json
{
  "ui": true,
  "api_addr": "http://127.0.0.1:8200",
  "storage": { "raft": { "path": "./data", "node_id": "dev-node" } },
  "listener": [{ "tcp": { "address": "127.0.0.1:8200", "tls_disable": true } }]
}
```

### config-prod.json

```json
{
  "ui": true,
  "api_addr": "https://vault.company.com:8200",
  "cluster_addr": "https://vault.company.com:8201",
  "storage": { "raft": { "path": "/var/lib/openbao/data", "node_id": "node1" } },
  "listener": [{ "tcp": { "address": "0.0.0.0:8200", "tls_cert_file": "/etc/openbao/tls/server.crt", "tls_key_file": "/etc/openbao/tls/server.key" } }]
}
```

### Multi-Node Cluster Example

**Node 1**

```json
{
  "api_addr": "https://node1.example.com:8200",
  "cluster_addr": "https://node1.example.com:8201",
  "storage": { "raft": { "path": "/var/lib/openbao/data", "node_id": "node1" } }
}
```

**Node 2**

```json
{
  "api_addr": "https://node2.example.com:8200",
  "cluster_addr": "https://node2.example.com:8201",
  "storage": { "raft": { "path": "/var/lib/openbao/data", "node_id": "node2" } }
}
```

**Node 3**

```json
{
  "api_addr": "https://node3.example.com:8200",
  "cluster_addr": "https://node3.example.com:8201",
  "storage": { "raft": { "path": "/var/lib/openbao/data", "node_id": "node3" } }
}
```

---

## 7. How Clusters and Load Balancers Work Together

* Clients connect to **load balancer** (HTTPS, port 443)
* Load balancer forwards traffic to cluster nodes (port 8200)
* Cluster handles **active/standby logic**
* Standby nodes redirect writes to active node

**Key Takeaways:**

* Cluster = **state coordination + failover**
* Load balancer = **traffic routing**
* DNS points to **load balancer**, not cluster nodes directly

---

## 8. Benefits of Clusters

| Situation         | Benefit                                |
| ----------------- | -------------------------------------- |
| Node fails        | Automatic failover to standby          |
| High traffic      | Load can be distributed across nodes   |
| Maintenance       | Nodes can be upgraded without downtime |
| Disaster recovery | Cluster replication ensures redundancy |

## 9. Retry Join 

```json
{
  "api_addr": "https://node1.example.com:8200",
  "cluster_addr": "https://node1.example.com:8201",
  "storage": {
    "raft": {
      "path": "/var/lib/openbao/data",
      "node_id": "node1",
      "retry_join": [
        { "leader_api_addr": "https://node2.example.com:8200" },
        { "leader_api_addr": "https://node3.example.com:8200" }
      ]
    }
  }
}
```

---

## References

* [OpenBao Configuration Docs](https://openbao.org/docs/configuration/)

