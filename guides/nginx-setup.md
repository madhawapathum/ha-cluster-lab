# Nginx Setup Guide

This guide walks through installing and configuring **Nginx on all 3 cluster nodes** so that the web service is always reachable via the Virtual IP, surviving any single node failure.

| What it provides |
|------------------|
| Nginx running on all 3 nodes simultaneously |
| Service always reachable via VIP `192.168.56.100` |
| Survives any single node failure — no manual intervention needed |

> **Navigation**
> - Previous: [Keepalived VRRP Setup](./keepalived-setup.md)

---

## Overview

Nginx runs independently on every node. The **Virtual IP (VIP)** managed by Keepalived always points to the current master node. When you access `http://192.168.56.100`, you hit the master's Nginx instance. If that node fails, Keepalived moves the VIP to a backup node within ~3 seconds — traffic is automatically routed to that node's Nginx.

```
Client → 192.168.56.100 (VIP)
              ↓
     [Current MASTER node]
              ↓
           Nginx
```

---

## Prerequisites

- All 3 nodes running Debian 12
- Keepalived VIP active (see [keepalived-setup.md](./keepalived-setup.md))
- Nodes reachable on `192.168.56.0/24`

---

## Step 1 — Install Nginx on All Nodes

Run on **every node**:

```bash
sudo apt update
sudo apt install -y nginx
```

---

## Step 2 — Enable and Start Nginx

Run on **every node**:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```

---

## Step 3 — Customise the Default Page (Optional but Recommended)

To easily identify which node is serving traffic during failover tests, customise the default page on each node.

On **node1**:

```bash
echo "<h1>node1 - 192.168.56.11</h1>" | sudo tee /var/www/html/index.html
```

On **node2**:

```bash
echo "<h1>node2 - 192.168.56.12</h1>" | sudo tee /var/www/html/index.html
```

On **node3**:

```bash
echo "<h1>node3 - 192.168.56.13</h1>" | sudo tee /var/www/html/index.html
```

---

## Step 4 — Verify Nginx is Reachable via the VIP

From any host on the `192.168.56.0/24` network (e.g. the PXE server):

```bash
curl http://192.168.56.100
```

You should see the page from whichever node currently holds the VIP.

---

## Step 5 — Test High Availability

1. **Confirm which node holds the VIP** (check for `192.168.56.100` on the interface):

```bash
# Run on each node until you find the one with the VIP
ip addr show enp0s8 | grep 192.168.56.100
```

2. **Check which node is serving** – request the VIP and note the response:

```bash
curl http://192.168.56.100
```

3. **Simulate a failure** — stop Nginx OR stop Keepalived on the master node:

```bash
sudo systemctl stop keepalived   # simulates full node failover
```

4. **Within ~3 seconds**, the VIP moves to the next backup node. Request again:

```bash
curl http://192.168.56.100
```

The response should now come from a different node — service was never interrupted.

5. **Restore the master:**

```bash
sudo systemctl start keepalived
```

Traffic automatically returns to node1 when it reclaims the VIP.

---

## Troubleshooting

See [`troubleshooting/issues-faced.md`](../troubleshooting/issues-faced.md) for common problems.

- `curl` times out → Check Keepalived is running and VIP is assigned to a node
- Wrong node responding → VIP may be on an unexpected node; check priorities in `keepalived.conf`
- Nginx not starting → Check `sudo journalctl -u nginx` for errors
