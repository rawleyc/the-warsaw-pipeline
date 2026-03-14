# 🚀 Azure Automation Lab: Scaling n8n Behind Cloudflare Zero Trust

**A Cloud Engineering Case Study in Troubleshooting, Scaling, and Security**

> This project demonstrates deploying a self-hosted **n8n** automation instance on **Azure**, secured with **Cloudflare Zero Trust (2FA)**, while solving real-world authentication and performance bottlenecks.

**Skills demonstrated:** Layer 7 routing · Zero Trust security design · OAuth2 integration · Docker orchestration · Cloud infrastructure scaling · Reverse proxy TLS termination

---

## Table of Contents

- [Mission](#-mission)
- [Architecture Overview](#-architecture-overview)
- [Phase 1: From 1 GB RAM Failure → 4 GB Resilient Infrastructure](#-phase-1-from-1-gb-ram-failure--4-gb-resilient-infrastructure)
- [Phase 2: Solving the "2FA Callback Deadlock"](#-phase-2-solving-the-2fa-callback-deadlock-with-zero-trust-longest-path-matching)
- [Phase 3: Fixing SSL "Death Loops" (HTTP 525/520)](#-phase-3-fixing-ssl-death-loops-http-525520)
- [Phase 4: Docker Permission Hardening](#-phase-4-docker-permission-hardening)
- [Deployment Performance Metrics](#-deployment-performance-metrics)
- [Project Structure](#-project-structure)
- [Key Takeaways](#-key-takeaways)

---

## 📌 Mission

Deploy a high-performance, secure **n8n** instance with:

- Zero Trust 2FA for administrative security
- Path-specific bypass for third-party OAuth callbacks
- Resilient infrastructure on Azure
- Automated TLS management via Caddy reverse proxy
- Scalable Node.js memory management

---

## 🏗️ Architecture Overview

### End-to-End Request Flow

```mermaid
flowchart LR
    User((User)) -->|HTTPS| CF[Cloudflare\nZero Trust]
    CF -->|2FA Challenge| Auth{Authenticated?}
    Auth -->|Yes| Caddy[Caddy\nReverse Proxy]
    Auth -->|No| Block[Access Denied]
    Caddy -->|TLS Termination| n8n[n8n Docker\nContainer]
    n8n -->|Persistent Volume| Data[(n8n_data\nUID 1000)]
    n8n --- Azure[Azure VM\nB2als_v2 · 4 GB RAM]

    style CF fill:#f48120,color:#fff
    style Caddy fill:#22b573,color:#fff
    style n8n fill:#ea4b71,color:#fff
    style Azure fill:#0078d4,color:#fff
    style Block fill:#c0392b,color:#fff
```

### Internal Component Stack

```mermaid
block-beta
    columns 3
    block:cloud:3
        columns 3
        A["☁️ Cloudflare CDN + Zero Trust"]
        B["🔒 2FA / Access Policies"]
        C["🌐 DNS (Orange Cloud)"]
    end
    space:3
    block:vm:3
        columns 3
        D["🔁 Caddy Reverse Proxy\nPorts 80 / 443\nAuto TLS"]
        E["⚙️ n8n (Docker)\nNode.js · V8 Heap 2048 MB\nUID 1000"]
        F["💾 Persistent Volume\nn8n_data\nEncryption Keys"]
    end
    space:3
    block:azure:3
        columns 3
        G["🖥️ Azure VM Standard_B2als_v2\n4 GB RAM · AMD EPYC™"]
        H["🛡️ NSG Rules\n80/443 Allow\nSSH Admin-only"]
        I["💿 2 GB Swap\nvm.swappiness=10"]
    end

    style cloud fill:#f5a623,color:#000
    style vm fill:#4a90d9,color:#fff
    style azure fill:#0078d4,color:#fff
```

---

## 🛠️ Phase 1: From 1 GB RAM Failure → 4 GB Resilient Infrastructure

The project initially launched on an **Azure B1ls** (1 GB RAM) and immediately crashed due to **Out Of Memory (OOM)** errors.

### Scaling Timeline

```mermaid
gantt
    title Infrastructure Scaling Timeline
    dateFormat X
    axisFormat %s

    section Diagnosis
    OOM crash detected           :crit, done, 0, 1
    Identified 1 GB RAM limit    :done, 1, 2

    section Scaling
    Hot resize to B2als_v2 4 GB  :done, 2, 3
    Static IP migration          :done, 3, 4

    section Hardening
    Kernel tuning swappiness     :done, 4, 5
    2 GB swap provisioned        :done, 5, 6
    Node V8 heap set to 2048 MB  :done, 6, 7
```

### Infrastructure Upgrade & Fixes

| Action | Detail |
|---|---|
| **Hot Resize** | Azure instance scaled to **Standard_B2als_v2** (4 GB RAM, AMD EPYC™) |
| **Static IP Migration** | Prevented DNS de-propagation during resize |
| **Kernel Tuning** | `vm.swappiness=10` to prioritize RAM over swap |
| **Swapfile Provisioning** | 2 GB swap to safeguard memory-intensive workflows |
| **Node.js Heap Limiting** | V8 heap set to 2048 MB for stability |

```bash
# Swap provisioning commands
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

---

## 🔐 Phase 2: Solving the "2FA Callback Deadlock" with Zero Trust Longest Path Matching

Cloudflare Zero Trust's **2FA** blocked the **Google OAuth2** callback, creating a transport error in n8n.

### The Problem: OAuth Deadlock

```mermaid
sequenceDiagram
    participant User as 👤 User
    participant n8n as ⚙️ n8n
    participant Google as 🔵 Google OAuth
    participant CF as 🟠 Cloudflare ZT

    User->>n8n: 1. Trigger OAuth connect
    n8n->>Google: 2. Redirect to Google login
    Google->>CF: 3. Redirect to /rest/oauth2-credential/callback
    CF->>CF: 4. Intercept — present 2FA challenge
    CF--xGoogle: ❌ OAuth handshake fails
    Note over CF,Google: Google cannot complete<br/>the 2FA challenge on<br/>behalf of the user
```

### The Solution: Zero Trust Child Application

```mermaid
sequenceDiagram
    participant User as 👤 User
    participant n8n as ⚙️ n8n
    participant Google as 🔵 Google OAuth
    participant CF as 🟠 Cloudflare ZT

    User->>n8n: 1. Trigger OAuth connect
    n8n->>Google: 2. Redirect to Google login
    Google->>CF: 3. Redirect to /rest/oauth2-credential/callback
    CF->>CF: 4. Longest-path match → Child App (Bypass)
    CF->>n8n: 5. ✅ Pass through to n8n
    n8n->>User: 6. OAuth credential saved
    Note over CF: Only the callback path<br/>bypasses 2FA. All other<br/>routes remain protected.
```

### Zero Trust Access Policy Hierarchy

```mermaid
flowchart TD
    Root["n8n.dmpoco.lol\n🔒 Parent App"] -->|All routes| ZT["Zero Trust 2FA\nGroup: Admins"]
    Root -->|Longest path match| Child["n8n.dmpoco.lol/rest/oauth2-credential/callback\n🟢 Child App"]
    Child --> Bypass["Bypass Policy\nGroup: Everyone"]

    ZT --> Protected["✅ Full 2FA Protection\nDashboard · Workflows · API"]
    Bypass --> OAuth["✅ OAuth Callback\nGoogle · Microsoft · etc."]

    style Root fill:#e74c3c,color:#fff
    style Child fill:#27ae60,color:#fff
    style ZT fill:#f39c12,color:#fff
    style Bypass fill:#2ecc71,color:#fff
    style Protected fill:#3498db,color:#fff
    style OAuth fill:#1abc9c,color:#fff
```

> **Security Insight:** Cloudflare prioritizes the most specific path. Only the OAuth callback bypasses 2FA; all other requests remain fully protected.

---

## ⚠️ Phase 3: Fixing SSL "Death Loops" (HTTP 525/520)

Initial deployment caused SSL errors due to ALPN negotiation failure with Cloudflare's Orange Cloud proxy.

### SSL Resolution Sequence

```mermaid
sequenceDiagram
    participant Caddy as 🟢 Caddy
    participant CF as 🟠 Cloudflare
    participant LE as 🔐 Let's Encrypt

    Note over Caddy,CF: ❌ PROBLEM: Orange Cloud blocks ACME challenge
    Caddy->>CF: Request TLS cert via ACME
    CF--xCaddy: ALPN negotiation fails (HTTP 525)

    Note over Caddy,CF: ✅ FIX: Temporarily switch to DNS-only
    CF->>CF: Switch to Gray Cloud (DNS Only)
    Caddy->>LE: ACME challenge succeeds
    LE->>Caddy: TLS certificate issued
    CF->>CF: Re-enable Orange Cloud proxy
    Caddy->>CF: Full (Strict) SSL active
    Note over Caddy,CF: ✅ HTTPS end-to-end established
```

### Resolution Steps

1. Temporarily switched DNS to **Gray Cloud** (DNS Only) to allow Caddy to obtain a certificate
2. Re-enabled proxy with **Full (Strict) SSL**
3. End-to-end encryption established without further 525/520 errors

---

## 📂 Phase 4: Docker Permission Hardening

n8n initially failed to save internal encryption keys:

```
EACCES: permission denied
```

| Root Cause | Resolution |
|---|---|
| Docker volume owned by `root` | Recursive permission reassignment to UID 1000 |
| n8n runs as non-root `node` user | Match volume ownership to container user |

```bash
sudo chown -R 1000:1000 ./n8n_data
```

### Docker Container Security Model

```mermaid
flowchart TD
    Host["Azure Host OS\n(root)"] --> Docker["Docker Engine"]
    Docker --> Container["n8n Container\nuser: node (UID 1000)"]
    Container --> Vol["Persistent Volume\n./n8n_data"]
    Vol --> Keys["🔑 Encryption Keys"]
    Vol --> DB["📊 Workflow DB"]
    Vol --> Creds["🔐 Credential Store"]

    Host -.->|"chown -R 1000:1000"| Vol

    style Host fill:#2c3e50,color:#fff
    style Docker fill:#2496ed,color:#fff
    style Container fill:#ea4b71,color:#fff
    style Vol fill:#f39c12,color:#fff
```

---

## 📊 Deployment Performance Metrics

| Metric | Original (Failed) | Optimized (Current) |
|---|---|---|
| Instance | Azure B1ls (1 GB RAM) | Azure B2als_v2 (4 GB RAM) |
| Swap Space | 0 GB | 2 GB Active |
| Node V8 Heap | Default (Unstable) | 2048 MB |
| SSL Setup | HTTP 525 Error | Full (Strict) via Caddy |
| Auth | Blocked by 2FA | Seamless Path Bypass |

### Before vs. After

```mermaid
quadrantChart
    title Deployment Health: Before vs After
    x-axis Low Reliability --> High Reliability
    y-axis Low Security --> High Security
    quadrant-1 Target State
    quadrant-2 Secure but Fragile
    quadrant-3 Needs Work
    quadrant-4 Reliable but Exposed
    Before: [0.15, 0.3]
    After: [0.85, 0.9]
```

---

## 📁 Project Structure

```
the-warsaw-pipeline/
├── README.md              # This document — architecture & case study
├── docker-compose.yml     # n8n + Caddy service definitions
├── Caddyfile              # Reverse proxy & TLS configuration
└── n8n_data/              # Persistent volume (UID 1000)
    ├── .n8n/              # Encryption keys & credentials
    └── database.sqlite    # Workflow definitions
```

### Docker Compose Layout

```mermaid
flowchart TB
    subgraph compose["docker-compose.yml"]
        direction TB
        subgraph caddy_svc["caddy service"]
            Caddy["Caddy\nPorts: 80, 443\nVolume: ./Caddyfile"]
        end
        subgraph n8n_svc["n8n service"]
            N8N["n8n\nPort: 5678 (internal)\nVolume: ./n8n_data\nUser: node (1000)"]
        end
        caddy_svc -->|"proxy to :5678"| n8n_svc
    end
    Internet((Internet)) -->|"80 / 443"| caddy_svc

    style compose fill:#1a1a2e,color:#fff
    style caddy_svc fill:#22b573,color:#fff
    style n8n_svc fill:#ea4b71,color:#fff
```

---

## 💡 Key Takeaways

| Lesson | Detail |
|---|---|
| **Zero Trust Nuance** | You can secure an entire domain while selectively bypassing critical OAuth endpoints using child applications and longest-path matching |
| **Hardware Resilience** | Node.js requires 4 GB+ RAM for production automation workflows; swap + V8 heap tuning provides the safety net |
| **Log-Driven Troubleshooting** | Errors like 520/525 only make sense after reviewing reverse proxy logs alongside Cloudflare diagnostics |
| **Security-First Docker** | Running containers as non-root with matched volume ownership is non-negotiable for production |

---

## 🧰 Tech Stack

![Azure](https://img.shields.io/badge/Azure-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare-F48120?style=for-the-badge&logo=cloudflare&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![n8n](https://img.shields.io/badge/n8n-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)
![Caddy](https://img.shields.io/badge/Caddy-22B573?style=for-the-badge&logo=caddy&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)

---

<details>
<summary><strong>📌 Full Phase Progression Overview (click to expand)</strong></summary>

```mermaid
flowchart TD
    Start([🚀 Project Start]) --> P1
    subgraph P1["Phase 1: Infrastructure"]
        A1["Deploy Azure B1ls 1 GB"] --> A2["OOM Crash ❌"]
        A2 --> A3["Hot Resize → B2als_v2 4 GB"]
        A3 --> A4["Swap + Kernel Tuning"]
        A4 --> A5["V8 Heap → 2048 MB ✅"]
    end
    P1 --> P2
    subgraph P2["Phase 2: Zero Trust Auth"]
        B1["Enable Cloudflare 2FA"] --> B2["OAuth Callback Blocked ❌"]
        B2 --> B3["Create Child App"]
        B3 --> B4["Longest Path Bypass ✅"]
    end
    P2 --> P3
    subgraph P3["Phase 3: SSL / TLS"]
        C1["Enable Orange Cloud"] --> C2["HTTP 525 / 520 ❌"]
        C2 --> C3["Gray Cloud + ACME"]
        C3 --> C4["Full Strict SSL ✅"]
    end
    P3 --> P4
    subgraph P4["Phase 4: Docker Hardening"]
        D1["n8n Write Fails"] --> D2["EACCES ❌"]
        D2 --> D3["chown 1000:1000"]
        D3 --> D4["Persistent Volume ✅"]
    end
    P4 --> Done([🎯 Production Ready])

    style P1 fill:#0078d4,color:#fff
    style P2 fill:#f48120,color:#fff
    style P3 fill:#22b573,color:#fff
    style P4 fill:#ea4b71,color:#fff
    style Start fill:#2c3e50,color:#fff
    style Done fill:#27ae60,color:#fff
```

</details>