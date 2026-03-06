## 1. Why KMS? (The Self-Hosting Dilemma)

Point of self-hosting is to **own your data**. Using AWS KMS feels like letting Amazon keep a copy of your front door key.

**The Philosophical Conflict:**

* **Without KMS (Shamir's Keys):** You are the sovereign king of your data. If AWS disappears, you still have your keys. But, if your EC2 restarts at 3:00 AM, your website is **dead** until you wake up, find 3 separate physical keys, and manually type them in.
* **With KMS (Auto-Unseal):** You delegate "identity verification" to AWS. When the server boots, it says, "I'm a valid EC2 in this account, let me in." AWS KMS unlocks it.

**Why most "Self-Hosters" still use KMS:**
Most people use OpenBao to store keys for *other* apps. If OpenBao stays "Sealed" after a reboot, every other app you own also breaks. KMS isn't about giving AWS your secrets; it's about **automatic recovery**. AWS never sees your actual secrets—they only hold the "wrapper" key that encrypts your master key.

---

# 🛡️ OpenBao Production Guide: Cluster, Proxy, & Persistence

## 1. The Network "Phone Book"

For a cluster to work, every node needs to know where its teammates live. We use **Port 8200** for the API and **Port 8201** for the "Raft" sync.

| Component | Port | Description |
| --- | --- | --- |
| **Listener** | 8200 | Where Nginx and Clients talk to the node. |
| **Cluster Link** | 8201 | Where nodes vote and sync data (Internal only). |

---

## 2. Node Configuration (The Personality Files)

Each of your 3 EC2s needs its own `config.json`. **Do not copy-paste the exact same file.**

### Example for Node 1 (IP: 10.0.0.10)

```json
{
  "cluster_name": "bao-prod-cluster",
  "ui": true,
  "api_addr": "https://my-application.com",
  "cluster_addr": "https://10.0.0.10:8201",

  "storage": {
    "raft": {
      "path": "/var/lib/openbao/data",
      "node_id": "node1",
      "retry_join": [
        { "leader_api_addr": "https://10.0.0.11:8200" },
        { "leader_api_addr": "https://10.0.0.12:8200" }
      ]
    }
  },

  "listener": [{
    "tcp": {
      "address": "0.0.0.0:8200",
      "tls_disable": true 
    }
  }]
}

```

*Note: `tls_disable: true` is okay here **only** because Nginx is handling the SSL on the front and talking to the node over a private AWS network.*

---

## 3. Storage & Consensus (The "Brain")

We use **Raft integrated storage**.

* **`path`**: This is the folder on your EC2 SSD where the encrypted data lives.
* **`node_id`**: A unique nickname (node1, node2, node3).
* **`retry_join`**: The list of IPs. If Node 1 starts, it will keep "retrying" to join Node 2 and 3 until they answer.

---

## 4. The Reverse Proxy (The "Front Desk")

Since you want `my-application.com` to work, you install Nginx. Your Nginx config points to your EC2 cluster:

```nginx
server {
    listen 80;
    server_name my-application.com;
    return 301 https://$host$request_uri; # Force HTTPS
}

server {
    listen 443 ssl;
    server_name my-application.com;

    # SSL Certs go here (Certbot)

    location / {
        proxy_pass http://127.0.0.1:8200; # Send to local OpenBao
        proxy_set_header Host $host;
    }
}

```

---

## 5. Deployment Checklist (The "No-Skip" Steps)

1. **Launch 3 EC2s**: Ensure Security Groups allow **8200** and **8201**.
2. **Install OpenBao**: Place the unique `config.json` on each.
3. **Start Services**: `sudo systemctl start openbao`.
4. **Initialize (Once)**: On Node 1, run `bao operator init`.
* *Saves the 5 Unseal Keys and Root Token!*


5. **Unseal (Everywhere)**: On all 3 nodes, run `bao operator unseal` (repeat 3 times per node).
6. **Verify**: Run `bao operator raft list-peers`. You should see all 3 nodes listed.

---
