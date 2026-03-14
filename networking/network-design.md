# Networking Design

## Overview

This lab uses a **simple but practical network design** to support automated provisioning and cluster communication.

The environment is built in VirtualBox and separates **three main networking purposes**:

1. PXE provisioning
2. Cluster communication
3. Internet access for package downloads

Keeping these roles separated helps simulate how real infrastructure environments are usually designed.

---

## Network Layout

| Component     | Network Type        | Purpose                                                               |
| ------------- | ------------------- | --------------------------------------------------------------------- |
| PXE Server    | Host-Only + NAT     | Provides provisioning services and allows SSH management              |
| Cluster Nodes | Host-Only + NAT     | Join the PXE network and access Debian mirrors                        |
| PXE Network   | 192.168.56.0/24     | Internal network used for boot provisioning and cluster communication |

---

## Host-Only Network

The **Host-Only network** acts as the internal lab network.

All cluster nodes and the PXE server connect to this network.
This is where the PXE boot process takes place.

Responsibilities of this network:

* PXE boot requests from nodes
* DHCP responses from the PXE server
* Delivery of boot files through TFTP
* Retrieval of installation configuration files
* Communication between cluster nodes

Example addressing used in the lab:

| System     | IP Address    |
| ---------- | ------------- |
| PXE Server | 192.168.56.10 |
| Node01     | 192.168.56.21 |
| Node02     | 192.168.56.22 |
| Node03     | 192.168.56.23 |

---

## NAT Adapter

Cluster nodes also use a **NAT adapter**.

This adapter allows nodes to reach external Debian mirrors during installation so that packages can be downloaded.

Without this connection, the installer would fail when attempting to retrieve system packages.

The NAT adapter is **not used for cluster communication**, only for internet access.

---

## Bridged Adapter (PXE Server)

The PXE server includes a **bridged adapter** to allow:

* SSH access from the host machine
* External management of the PXE services

This adapter operates independently from the internal PXE network.

---

## DHCP Considerations

The PXE server provides DHCP services using dnsmasq.

To prevent conflicts:

* The VirtualBox built-in DHCP service on the host-only network is disabled.
* All IP assignments for PXE boot are handled by the PXE server itself.

This ensures nodes receive the correct PXE boot instructions.

---

## Cluster Communication

After provisioning, the same host-only network is used for cluster operations.

Cluster components running on the nodes include:

* Pacemaker
* Corosync

These services exchange heartbeat and state information across the internal network to monitor node health and manage failover operations.

---

## Virtual IP for High Availability

The cluster exposes services through a **Virtual IP address**.

Clients access services through this single IP rather than connecting directly to individual nodes.

If a node fails:

* Pacemaker detects the failure
* The Virtual IP is moved to another healthy node
* Services continue running without changing the client endpoint

---

## Summary

The lab network separates responsibilities while keeping the design simple:

* **Host-Only Network** → provisioning and cluster communication
* **NAT Adapter** → external package downloads
* **Bridged Adapter** → PXE server management access

This design allows automated node provisioning and supports the high-availability cluster built later in the project.
