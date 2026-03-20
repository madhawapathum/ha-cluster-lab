# Issues Faced

A log of problems encountered during the lab and how they were resolved.

---

### 1. PXE Boot Not Starting

**Problem**
The node attempted to boot but repeatedly returned to the BIOS screen after displaying `Start PXE over IPv4`. The PXE installation never began.

**Cause**
VirtualBox was using UEFI firmware, which caused compatibility issues with the syslinux-based PXE setup.

**Fix**
- Disabled UEFI in VM Settings → System → Motherboard → uncheck *Enable EFI*
- Set boot order to prioritise **Network** first

After these changes the node successfully started the PXE installation.

---

### 2. DHCP Conflict on Host-Only Network

**Problem**
PXE clients were not receiving the correct DHCP/PXE boot response from dnsmasq.

**Cause**
VirtualBox has its own built-in DHCP server enabled on the host-only network by default. It was racing with dnsmasq and winning, sending responses that lacked PXE boot options.

**Fix**
Disabled the VirtualBox built-in DHCP server for the host-only adapter:

> VirtualBox → File → Tools → Network Manager → Select adapter → DHCP Server tab → Uncheck *Enable*

After disabling it, dnsmasq became the sole DHCP server and nodes received correct PXE instructions.

---

### 3. Debian Package Download Failure During Install

**Problem**
The Debian installer failed to download packages from the mirror during automated installation.

**Cause**
Nodes were only connected to the host-only network, which has no route to the internet.

**Fix**
Added a second **NAT adapter** (`eth1`) to each node VM so it could reach external Debian mirrors. The host-only adapter (`eth0`) remains the provisioning and cluster interface.

---

### 4. Host-Only Adapter Goes Inactive After Installation

**Problem**
After OS installation completed and the node rebooted, the host-only adapter (`eth0`) was not brought up automatically, breaking cluster network connectivity.

**Current Status**
Under investigation. Likely needs a persistent network interface config in the preseed or a post-install script to ensure `eth0` is configured at boot.

---

### 5. Corosync Binding to localhost (127.0.0.1)

**Problem**
Corosync showed as active and running, but nodes appeared offline in Pacemaker.

**Cause**
The most damaging problem was Corosync binding to `127.0.0.1` instead of the cluster network interface. This happened because the `/etc/hosts` file had node hostnames mapped to the loopback address, and Corosync resolved its own ring address to localhost. It could talk to itself but never reached the other nodes.

**Fix**
Replaced hostnames with explicit IP addresses in `corosync.conf`.

---

### 6. Pacemaker STONITH Fencing Requirement

**Problem**
Pacemaker refused to manage resources without a fencing device, marking all nodes as unclean.

**Cause**
VirtualBox has no proper fence agent, but STONITH is enabled by default.

**Fix**
Disabled STONITH for the lab and set `no-quorum-policy` appropriately to bootstrap the cluster.

---