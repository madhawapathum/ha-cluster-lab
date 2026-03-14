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

Each VM has **two network adapters**:
- **Host-Only (`eth0`)** — provisioning and cluster communication
- **NAT (`eth1`)** — internet access for package downloads

---

## Components

### PXE Server

The PXE server automates OS installation across all cluster nodes. It runs three services:

| Service       | Purpose |
|---------------|---------|
| **dnsmasq**   | Acts as the DHCP server and sends PXE boot instructions to booting nodes |
| **TFTP**      | Serves the kernel (`linux`) and initial ramdisk (`initrd.gz`) to the node over the network |
| **Apache HTTP** | Hosts the Debian preseed file so the installer can fetch it automatically |

### Cluster Nodes

Three identical Debian VMs provisioned automatically via PXE. After installation, they are the target hosts for Pacemaker/Corosync cluster configuration.

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
| 3 | Pacemaker + Corosync cluster formation | 🔜 Planned |
| 4 | Virtual IP (VIP) failover | 🔜 Planned |
| 5 | Fencing (STONITH) to prevent split-brain | 🔜 Planned |
| 6 | Failure simulation and recovery testing | 🔜 Planned |

---

## Planned: High-Availability Cluster (Phase 3+)

Once all nodes are provisioned, the cluster will be configured with:

- **Corosync** — handles low-level cluster messaging and quorum between nodes
- **Pacemaker** — manages cluster resources (services, virtual IPs) and decides where they run
- **Virtual IP (VIP)** — a floating IP address that moves to a healthy node automatically when one fails
- **STONITH Fencing** — if a node stops responding, the cluster fences (forcibly powers off) the unreachable node before moving resources, preventing data corruption from a split-brain scenario

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
| Debian Linux    | OS for PXE server and all cluster nodes |
| dnsmasq         | DHCP server + PXE boot instructions |
| TFTP            | Network boot file delivery |
| Apache HTTP     | Preseed file hosting |
| Debian Preseed  | Unattended OS installation |
| Pacemaker       | Cluster resource manager *(Phase 3)* |
| Corosync        | Cluster communication and quorum *(Phase 3)* |

---

## Learning Objectives

- Understand the full PXE boot chain (DHCP → TFTP → installer)
- Practice unattended OS installation using preseed
- Learn Pacemaker/Corosync cluster fundamentals
- Understand quorum, virtual IP failover, and split-brain protection
- Simulate real failure scenarios and observe automated recovery
