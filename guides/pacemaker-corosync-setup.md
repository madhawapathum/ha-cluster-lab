# Pacemaker + Corosync Cluster Setup Guide

This guide walks through setting up a **3-node Pacemaker + Corosync high-availability cluster** on the nodes provisioned by the PXE server.

The cluster provides quorum-based fault tolerance and automatic node failure detection.

> **Navigation**
> - Previous: [PXE Server Setup](./pxe-server-setup.md)
> - Next: [Keepalived VRRP Setup](./keepalived-setup.md)

---

## Overview

| Component      | Role |
|----------------|------|
| **Corosync**   | Cluster communication layer — keeps nodes in sync and detects failures |
| **Pacemaker**  | Cluster resource manager — starts, stops, and moves resources between nodes |

Together they provide:
- **Quorum-based fault tolerance** — the cluster stays active as long as a majority of nodes are healthy
- **Automatic node failure detection** — a failed node is isolated and its resources are moved within seconds

---

## Prerequisites

- 3 nodes provisioned via PXE, each running Debian 12
- All nodes reachable on the host-only network (`192.168.56.0/24`):
  - `node1` → `192.168.56.11`
  - `node2` → `192.168.56.12`
  - `node3` → `192.168.56.13`
- SSH access to all nodes
- Internet access for nodes via the PXE server's NAT forwarding:

```
Internet → enp0s3 (NAT) → PXE Server → enp0s8 (host-only) → nodes
```

> The PXE server forwards traffic from the nodes out to the internet through its NAT interface. Nodes do **not** need their own NAT adapter.

---

## Step 0 — Enable NAT Forwarding on the PXE Server

> Run this **on the PXE server**, not on the nodes.

### Enable IP forwarding

```bash
# Enable immediately (runtime)
sudo sysctl -w net.ipv4.ip_forward=1

# Make it permanent across reboots
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```

### Set up NAT masquerading with iptables

Masquerade traffic coming in from the host-only network (`enp0s8`) and send it out via the NAT interface (`enp0s3`):

```bash
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

### Make the iptables rules persistent

```bash
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

> When prompted during install, answer **Yes** to save current rules.

### Configure nodes to use the PXE server as their default gateway

On **each node**, set the PXE server (`192.168.56.10`) as the default gateway:

```bash
sudo ip route add default via 192.168.56.10
```

To make it permanent, edit `/etc/network/interfaces` on the node and add:

```
gateway 192.168.56.10
```

### Verify internet access from a node

```bash
ping -c 3 8.8.8.8
```

Once packets flow through, the nodes can reach package mirrors and `apt install` will work normally.

---

## Step 1 — Install Pacemaker and Corosync on All Nodes


Run the following on **every node**:

```bash
sudo apt update
sudo apt install -y pacemaker corosync pcs
```

| Package       | Purpose |
|---------------|---------|
| `corosync`    | Cluster messaging and quorum |
| `pacemaker`   | Resource management and failover |
| `pcs`         | CLI tool for configuring Pacemaker and Corosync |

---

## Step 2 — Set the `hacluster` User Password

`pcs` authenticates nodes using the `hacluster` system user. Set the same password on **every node**:

```bash
sudo passwd hacluster
```

> Use the same password on all nodes.

---

## Step 3 — Configure Corosync

The Corosync configuration is in the repo at [`cluster/corosync.conf`](../cluster/corosync.conf).

Copy it to all nodes:

```bash
sudo cp cluster/corosync.conf /etc/corosync/corosync.conf
```

> **Important:** Use explicit IP addresses in `corosync.conf`, not hostnames.
> If `/etc/hosts` maps hostnames to `127.0.0.1`, Corosync will bind to localhost
> and nodes will never be able to reach each other.
> See [`troubleshooting/issues-faced.md`](../troubleshooting/issues-faced.md) — Issue #5 for details.

Enable and start Corosync on all nodes:

```bash
sudo systemctl enable corosync
sudo systemctl start corosync
```

Verify Corosync is running and nodes can see each other:

```bash
sudo corosync-cmapctl | grep members
sudo corosync-quorumtool
```

---

## Step 4 — Start and Authenticate Pacemaker

Enable and start Pacemaker on **all nodes**:

```bash
sudo systemctl enable pacemaker
sudo systemctl start pacemaker
```

From **one node**, authenticate all cluster nodes with `pcs`:

```bash
sudo pcs host auth 192.168.56.11 192.168.56.12 192.168.56.13 -u hacluster
```

---

## Step 5 — Set Up the Cluster

From **one node only**, initialise the cluster:

```bash
sudo pcs cluster setup ha-cluster 192.168.56.11 192.168.56.12 192.168.56.13
sudo pcs cluster start --all
sudo pcs cluster enable --all
```

Verify all nodes appear:

```bash
sudo pcs status
```

---

## Step 6 — Disable STONITH and Configure Quorum Policy

> VirtualBox has no fence agent, so STONITH must be disabled for lab use.
> See [`troubleshooting/issues-faced.md`](../troubleshooting/issues-faced.md) — Issue #6 for details.

```bash
sudo pcs property set stonith-enabled=false
sudo pcs property set no-quorum-policy=ignore
```

---

## Step 7 — Verify the Cluster

```bash
sudo pcs status
sudo pcs status nodes
sudo crm_verify -L
```

Expected output: all 3 nodes in **Online** state, no errors.

---

## Troubleshooting

See [`troubleshooting/issues-faced.md`](../troubleshooting/issues-faced.md) for common problems:

- Nodes appear offline in Pacemaker despite Corosync running → Check `/etc/hosts` and use explicit IPs in `corosync.conf`
- Pacemaker marks nodes as unclean → Disable STONITH for lab environment
