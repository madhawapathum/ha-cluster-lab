# PXE Server Setup Guide

This guide walks through building a PXE provisioning server from scratch on **Debian 12 (Bookworm) — netinst image**.
When complete, the server will automatically install Debian on any node VM that boots from the network.

---

## Overview

The PXE server runs three services that work together:

| Service     | Role |
|-------------|------|
| **dnsmasq** | DHCP server + sends PXE boot instructions to clients |
| **TFTP**    | Serves the kernel and initrd to the booting node |
| **Apache**  | Hosts the preseed file the installer fetches over HTTP |

---

## Prerequisites

- A VM or machine running **Debian 12 (netinst)** — this becomes the PXE server
- Two network adapters on the PXE server VM:
  - **Adapter 1 (enp0s3)** — NAT (internet access to download packages)
  - **Adapter 2 (enp0s8)** — Host-Only `192.168.56.10` (provisioning network)
- VirtualBox Host-Only network: `192.168.56.0/24`
- VirtualBox built-in DHCP server **disabled** for the host-only adapter

> **Disable VirtualBox DHCP first:**
> VirtualBox → File → Tools → Network Manager → Select adapter → DHCP Server tab → uncheck *Enable*

---

## Step 1 — Assign a Static IP to the Host-Only Interface

Edit the network interfaces file:

```bash
sudo nano /etc/network/interfaces
```

Add the following for `enp0s8`:

```
auto enp0s8
iface enp0s8 inet static
    address 192.168.56.10
    netmask 255.255.255.0
```

Apply the changes:

```bash
sudo ifdown enp0s8 && sudo ifup enp0s8
```

Verify:

```bash
ip addr show enp0s8
```

---

## Step 2 — Install Required Packages

```bash
sudo apt update
sudo apt install -y dnsmasq apache2 pxelinux syslinux-common wget
```

| Package          | Purpose |
|------------------|---------|
| `dnsmasq`        | DHCP + TFTP + PXE instructions |
| `apache2`        | Serve the preseed file over HTTP |
| `pxelinux`       | PXE bootloader (`pxelinux.0`) |
| `syslinux-common`| Syslinux modules used by pxelinux |
| `wget`           | Download Debian netboot files |

---

## Step 3 — Set Up the TFTP Root Directory

Create the TFTP directory structure:

```bash
sudo mkdir -p /srv/tftp/debian-installer/amd64
sudo mkdir -p /srv/tftp/pxelinux.cfg
```

Copy the PXE bootloader files:

```bash
sudo cp /usr/lib/PXELINUX/pxelinux.0 /srv/tftp/
sudo cp /usr/lib/syslinux/modules/bios/{ldlinux.c32,libcom32.c32,libutil.c32,menu.c32} /srv/tftp/
```

Download the Debian 12 netboot kernel and initrd:

```bash
cd /srv/tftp/debian-installer/amd64

sudo wget http://ftp.debian.org/debian/dists/bookworm/main/installer-amd64/current/images/netboot/debian-installer/amd64/linux

sudo wget http://ftp.debian.org/debian/dists/bookworm/main/installer-amd64/current/images/netboot/debian-installer/amd64/initrd.gz
```

---

## Step 4 — Configure the PXE Boot Menu

The boot menu config is in the repo at [`pxe-server/tftp/pxelinux.cfg/default`](../pxe-server/tftp/pxelinux.cfg/default).

Copy it to the TFTP directory:

```bash
sudo cp pxe-server/tftp/pxelinux.cfg/default /srv/tftp/pxelinux.cfg/default
```

Or create the file manually:

```bash
sudo nano /srv/tftp/pxelinux.cfg/default
```

> Use the contents from [`pxe-server/tftp/pxelinux.cfg/default`](../pxe-server/tftp/pxelinux.cfg/default) in this repo.
> The `preseed/url` points to this server's Apache instance — the installer fetches the preseed file automatically.

---

## Step 5 — Configure dnsmasq

The dnsmasq config is in the repo at [`pxe-server/dnsmasq/dnsmasq.conf`](../pxe-server/dnsmasq/dnsmasq.conf).

Back up the default config, then copy from the repo:

```bash
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.bak
sudo cp pxe-server/dnsmasq/dnsmasq.conf /etc/dnsmasq.conf
```

Or open the file manually and use the contents from the repo:

```bash
sudo nano /etc/dnsmasq.conf
```

> See [`pxe-server/dnsmasq/dnsmasq.conf`](../pxe-server/dnsmasq/dnsmasq.conf) for the full configuration.

Enable and restart dnsmasq:

```bash
sudo systemctl enable dnsmasq
sudo systemctl restart dnsmasq
sudo systemctl status dnsmasq
```

---

## Step 6 — Create the Preseed File

The preseed file drives fully unattended OS installation. It is in the repo at [`pxe-server/http/preseed.cfg`](../pxe-server/http/preseed.cfg).

Copy it to the Apache web root:

```bash
sudo cp pxe-server/http/preseed.cfg /var/www/html/preseed.cfg
```

Or create it manually:

```bash
sudo nano /var/www/html/preseed.cfg
```

> Use the contents from [`pxe-server/http/preseed.cfg`](../pxe-server/http/preseed.cfg) in this repo.

> ⚠️ The default password in the preseed file is `changeme` — update it before use in any non-lab environment.

Enable and start Apache:

```bash
sudo systemctl enable apache2
sudo systemctl start apache2
```

Verify the preseed file is reachable:

```bash
curl http://192.168.56.10/preseed.cfg
```

---

## Step 7 — Verify the Full Setup

Check all three services are running:

```bash
sudo systemctl status dnsmasq
sudo systemctl status apache2
```

Confirm the TFTP directory has the required files:

```bash
ls /srv/tftp/
ls /srv/tftp/debian-installer/amd64/
ls /srv/tftp/pxelinux.cfg/
```

Expected output:

```
/srv/tftp/
  pxelinux.0  ldlinux.c32  libcom32.c32  libutil.c32  menu.c32
  debian-installer/  pxelinux.cfg/

/srv/tftp/debian-installer/amd64/
  linux  initrd.gz

/srv/tftp/pxelinux.cfg/
  default
```

---

## Step 8 — Boot a Node VM

1. Create a new VirtualBox VM (no OS, 1–2 GB RAM, 20 GB disk)
2. Add **two network adapters**:
   - Adapter 1: **Host-Only** (`vboxnet0`) — must be first so PXE boot works
   - Adapter 2: **NAT** — for internet access during package download
3. In VM Settings → System → Motherboard:
   - **Disable EFI** (use BIOS mode)
   - Set boot order: **Network → Hard Disk**
4. Start the VM

The node will:
1. Broadcast a DHCP request → dnsmasq responds with an IP and PXE instructions
2. Download `pxelinux.0` via TFTP → load the boot menu
3. Download `linux` + `initrd.gz` via TFTP → start the Debian installer
4. Fetch `preseed.cfg` from Apache → run fully automated installation
5. Reboot into a configured Debian 12 system

---

## Troubleshooting

See [`troubleshooting/issues-faced.md`](../troubleshooting/issues-faced.md) for common problems:

- PXE boot not starting → disable UEFI, set Network as first boot device
- Node gets wrong DHCP response → disable VirtualBox built-in DHCP server
- Installer can't download packages → add NAT adapter to the node VM
