# Keepalived VRRP Setup Guide

This guide walks through setting up **Keepalived** on all cluster nodes to provide a Virtual IP (VIP) with automatic failover using VRRP.

| What it provides |
|------------------|
| Virtual IP `192.168.56.100` always reachable regardless of which node is active |
| Automatic failover in **~3 seconds** when the master node goes down |
| Automatic failback when the master node recovers |

> **Navigation**
> - Previous: [Pacemaker + Corosync Setup](./pacemaker-corosync-setup.md)
> - Next: [Nginx Setup](./nginx-setup.md)

---

## Overview

**VRRP (Virtual Router Redundancy Protocol)** elects one node as the **MASTER** which holds the VIP. All other nodes are **BACKUP** and monitor the master. If the master fails, the highest-priority backup immediately takes the VIP.

| Role     | Node    | IP               | Priority |
|----------|---------|------------------|----------|
| MASTER   | node1   | 192.168.56.11    | 110      |
| BACKUP   | node2   | 192.168.56.12    | 100      |
| BACKUP   | node3   | 192.168.56.13    | 90       |

> The node with the **highest priority** wins the election and becomes MASTER.
> Each node must have a **unique priority**, and backup nodes must always have a **lower priority than the master**.

---

## Prerequisites

- All 3 nodes running Debian 12
- Pacemaker + Corosync cluster set up (see [pacemaker-corosync-setup.md](./pacemaker-corosync-setup.md))
- Nodes reachable on `192.168.56.0/24`

---

## Step 1 — Install Keepalived on All Nodes

Run on **every node**:

```bash
sudo apt update
sudo apt install -y keepalived
```

---

## Step 2 — Configure Keepalived

The base configuration is in the repo at [`cluster/keepalived.conf`](../cluster/keepalived.conf).

> **This file must be present on every node.** Adjust `state` and `priority` per node as described below.

Copy to all nodes:

```bash
sudo cp cluster/keepalived.conf /etc/keepalived/keepalived.conf
```

### On `node1` (MASTER)

```
state MASTER
priority 110
```

### On `node2` (BACKUP)

```
state BACKUP
priority 100
```

### On `node3` (BACKUP)

```
state BACKUP
priority 90
```

> The `interface` field should match the cluster network interface name on each node (e.g. `enp0s8`).

---

## Step 3 — Enable and Start Keepalived

Run on **every node**:

```bash
sudo systemctl enable keepalived
sudo systemctl start keepalived
sudo systemctl status keepalived
```

---

## Step 4 — Verify the Virtual IP

From the **master node** (`node1`), verify the VIP is assigned:

```bash
ip addr show enp0s8
```

You should see `192.168.56.100` listed on the interface.

From the **PXE server or any host** on the same network, verify the VIP is reachable:

```bash
ping 192.168.56.100
```

---

## Step 5 — Test Failover

1. **Check which node holds the VIP:**

```bash
ip addr show enp0s8 | grep 192.168.56.100
```

2. **Simulate a master failure** — stop Keepalived on `node1`:

```bash
sudo systemctl stop keepalived
```

3. **Within ~3 seconds**, `node2` should take the VIP. Verify on `node2`:

```bash
ip addr show enp0s8 | grep 192.168.56.100
```

4. **Test failback** — restart Keepalived on `node1`:

```bash
sudo systemctl start keepalived
```

`node1` reclaims the VIP automatically (preemption is enabled by default).

---

## Troubleshooting

See [`troubleshooting/issues-faced.md`](../troubleshooting/issues-faced.md) for common problems.

- VIP not appearing → Check `interface` name matches the actual adapter name on the node
- No failover happening → Ensure all nodes have keepalived running and priorities are unique
- Failback not working → Confirm `nopreempt` is NOT set in the config (it disables failback)
