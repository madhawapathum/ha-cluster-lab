# HA Cluster Lab

> A self-healing Linux cluster built in VirtualBox to learn high-availability concepts hands-on — from bare-metal PXE provisioning to automated failover with Pacemaker and Corosync.

---

## Project Goal

Build a small, fully automated Linux cluster that can detect node failures and recover without manual intervention. The lab covers every layer: network provisioning, OS deployment, cluster formation, virtual IP failover, and fencing.

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                      VirtualBox Host                      │
│                                                          │
│   ┌──────────────┐       Host-Only Network               │
│   │  PXE Server  │       192.168.56.0/24                 │
│   │  Debian      │◄────────────────────────────────┐     │
│   │  192.168.56.10│                                │     │
│   └──────┬───────┘                         ┌───────┴───┐ │
│          │ NAT                             │  node-1   │ │
│          │ (internet access                │  .101     │ │
│          │  for all VMs)                   ├───────────┤ │
│          │                                 │  node-2   │ │
│          │                                 │  .102     │ │
│          │                                 ├───────────┤ │
│          │                                 │  node-3   │ │
│          │                                 │  .103     │ │
│          └─────────────────────────────────┴───────────┘ │
└──────────────────────────────────────────────────────────┘
```

| Machine    | Role                    | Host-Only IP    |
|------------|-------------------------|-----------------|
| pxe-server | Provisioning server     | 192.168.56.10   |
| node-1     | HA cluster member       | 192.168.56.101  |
| node-2     | HA cluster member       | 192.168.56.102  |
| node-3     | HA cluster member       | 192.168.56.103  |

**Network setup:**
- **Host-Only (`enp0s8`)** — provisioning and cluster communication (`192.168.56.0/24`)
- Nodes access the internet through the **PXE server's NAT interface** (`enp0s3`) via IP forwarding — nodes do not need their own NAT adapter

---

## Components

| Component       | Role |
|-----------------|------|
| **PXE Server**  | Provisions nodes automatically over the network — no manual OS install needed |
| **Corosync**    | Keeps nodes in constant communication and detects failures |
| **Pacemaker**   | Decides where resources live based on current cluster state |
| **Keepalived**  | Manages the Virtual IP (VIP), moves it automatically when the master changes |
| **Nginx**       | Serves the actual service — always reachable via the VIP regardless of which node is active |

---

## Provisioning Workflow

When a node VM is powered on with no OS installed, the following sequence happens automatically:

```
1. Node powers on → NIC broadcasts DHCP request
2. dnsmasq assigns an IP and returns PXE boot instructions
3. Node downloads kernel + initrd via TFTP
4. Debian installer starts
5. Installer fetches the preseed file from Apache over HTTP
6. Preseed drives fully unattended Debian installation
7. Node reboots into a configured Debian system
```

No keyboard input is required on the node at any point.

---

## Project Phases

| # | Phase | Status |
|---|-------|--------|
| 1 | PXE provisioning server setup | ✅ Complete |
| 2 | Automated node OS installation via preseed | ✅ Complete |
| 3 | Pacemaker + Corosync cluster formation | ✅ Complete |
| 4 | Keepalived VRRP — Virtual IP failover | ✅ Complete |
| 5 | Nginx on all nodes — service via VIP | ✅ Complete |
| 6 | Failure simulation and recovery testing | 🔜 Planned |

---

## How the Stack Works Together

```
Node powers on
    ↓
PXE Server provisions Debian automatically over the network
    ↓
Corosync forms the cluster — nodes communicate and monitor each other
    ↓
Pacemaker manages cluster resources based on node health
    ↓
Keepalived holds Virtual IP 192.168.56.100 on the master node
    ↓
Nginx serves the service — always reachable via the VIP
    ↓
Node fails → Corosync detects it → Keepalived moves VIP → traffic continues
```

---

## Repository Structure

```
ha-cluster-lab/
├── README.md                   # Project overview (this file)
├── architecture/
│   └── overview.md             # Detailed component and design notes
├── networking/
│   └── network-design.md       # Network topology and adapter setup
├── pxe-server/
│   ├── dnsmasq/                # dnsmasq config (DHCP + PXE)
│   ├── tftp/                   # TFTP boot file layout
│   └── http/                   # Apache config and preseed file
├── cluster/                    # Pacemaker/Corosync configs (Phase 3+)
├── scripts/                    # Automation helper scripts
└── troubleshooting/
    └── issues-faced.md         # Issues encountered and how they were resolved
```

---

## Tech Stack

| Technology      | Role |
|-----------------|------|
| VirtualBox      | Hypervisor for the lab environment |
| Debian Linux    | OS for the PXE server and all cluster nodes |
| dnsmasq         | DHCP server + PXE boot instructions |
| TFTP            | Network boot file delivery |
| Apache HTTP     | Preseed file hosting |
| Debian Preseed  | Unattended OS installation |
| Corosync        | Cluster communication layer — detects node failures |
| Pacemaker       | Cluster resource manager — moves resources on failure |
| Keepalived      | VRRP — manages the Virtual IP and moves it on failover |
| Nginx           | Web service — always reachable via the VIP |

---

## Learning Objectives

- Understand the full PXE boot chain (DHCP → TFTP → installer)
- Practice unattended OS installation using preseed
- Learn Pacemaker/Corosync cluster fundamentals
- Understand quorum, virtual IP failover, and split-brain protection
- Simulate real failure scenarios and observe automated recovery
