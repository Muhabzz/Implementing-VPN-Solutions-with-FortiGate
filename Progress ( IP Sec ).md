# FortiGate HQ ↔ Branch Lab — Setup & Verification

## Overview

This lab simulates a site-to-site IPsec VPN between two FortiGate devices (HQ and Branch) inside VMware Workstation.
It demonstrates how to configure networking (Bridged + Host-only VMnets), configure FortiGate interfaces, verify connectivity, and transfer files between machines on different sides of the tunnel.

**Goals**

* Create HQ and Branch environments inside VMware.
* Configure FortiGate interfaces for WAN (Bridged) and LAN (Host-only).
* Establish/verify IPsec tunnel and routing.
* Transfer files between Windows (HQ) and Ubuntu (Branch) to validate real traffic over the tunnel.

---

## Table of Contents

1. Topology
2. VMware network settings
3. FortiGate interface configuration (summary)
4. DHCP configuration (FortiGate)
5. Verifications & diagnostic commands
6. File transfer: Windows → Ubuntu (SMB)
7. Troubleshooting
8. Optional methods for file transfer
9. Lab notes & recommendations

---

## 1. Topology (logical)

```
                [Internet / Bridged network]
                         │
         ┌───────────────┴───────────────┐
         │                               │
    [FortiGate HQ]                 [FortiGate Branch]
    port1 - Bridged                port1 - Bridged
    port2 - Host-only (vmnet1)     port2 - Host-only (vmnet8)
        │                               │
[Windows Device (HQ)]            [Ubuntu Device (Branch)]
        │                               │
     vmnet1                          vmnet8
```

**Sample IPs used in this doc**

* HQ LAN: `10.10.10.0/24` — FortiHQ port2 = `10.10.10.1`
* Branch LAN: `10.20.20.0/24` — FortiBranch port2 = `10.20.20.1`
* WAN interfaces use Bridged mode (host network), public or host-assigned addresses.

---

## 2. VMware network settings

* `vmnet0` — Bridged (used for WAN connectivity).
* `vmnet1` — Host-only (HQ LAN).
* `vmnet8` — Host-only (Branch LAN).

**Important**

* Host-only networks are isolated per VMnet. VMs attached to `vmnet1` cannot see VMs attached to `vmnet8` unless routing is performed by FortiGates or another router.
* If VMware reports `The host-only adapter driver does not seem to be running`, open **Virtual Network Editor** and click **Restore Defaults**, then reboot host. Ensure VMware Host-Only adapters exist in Windows Device Manager and are enabled.

---

## 3. FortiGate interface configuration (summary)

### HQ FortiGate (CLI examples)

```text
config system interface
edit "port1"
    set vdom "root"
    set mode dhcp
    set allowaccess ping https ssh http
    set alias "WAN"
next
edit "port2"
    set vdom "root"
    set ip 10.10.10.1 255.255.255.0
    set allowaccess ping https ssh http
    set alias "LAN"
    set role lan
next
end
```

### Branch FortiGate (CLI examples)

```text
config system interface
edit "port1"
    set vdom "root"
    set mode dhcp
    set allowaccess ping https ssh http
    set alias "WAN"
next
edit "port2"
    set vdom "root"
    set ip 10.20.20.1 255.255.255.0
    set allowaccess ping https ssh http
    set alias "LAN"
    set role lan
next
end
```

---

## 4. DHCP configuration (FortiGate)

If you want the FortiGate to hand out DHCP addresses on LAN:

```text
config system dhcp server
edit 1
    set interface "port2"
    set lease-time 86400
    config ip-range
        edit 1
            set start-ip 10.20.20.50
            set end-ip 10.20.20.200
        next
    end
    set netmask 255.255.255.0
    set default-gateway 10.20.20.1
next
end
```

Repeat on HQ (adjust IP range and gateway for `10.10.10.0/24`).

---

## 5. Verification & diagnostic commands

### FortiGate CLI (useful commands)

* Check interface summary:

```text
get system interface | grep "port2"
```

* Check physical NIC status:

```text
diagnose hardware deviceinfo nic port2
```

* Ping from FortiGate:

```text
execute ping 10.10.10.50
```

* Sniff traffic on a FortiGate interface or tunnel:

```text
diagnose sniffer packet port2 'host 10.10.10.50' 4
```

### Linux/Windows clients

* On Linux:

```bash
ip a
ping 10.20.20.1
sudo dhclient -v eth0   # to request DHCP
```

* On Windows (PowerShell):

```powershell
Test-NetConnection -ComputerName 10.10.10.50 -Port 445
ping 10.10.10.50
```

**Note:** Use `Test-NetConnection -Port 445` to validate SMB connectivity.

---

## 6. File transfer: Windows (HQ) → Ubuntu (Branch) via SMB

### A. On Windows (HQ) — create and share a folder

1. Create a folder to share, e.g. `C:\Share`.

2. Right-click `C:\Share` → Properties → Sharing → Advanced Sharing.

   * Check **Share this folder**.
   * Share name: `Share`.

3. Click **Permissions**, add `Everyone` and grant **Full Control** (for testing).

4. (Optional) For easier test: Control Panel → Network and Sharing Center → Advanced sharing settings → Under *All Networks* → Turn off *Password protected sharing* (ONLY for lab/testing).

5. Ensure Windows Firewall allows File and Printer Sharing:

```powershell
# Run as Administrator
Enable-NetFirewallRule -DisplayGroup "File and Printer Sharing"
```

### B. On Ubuntu (Branch) — access the share

1. Install cifs-utils (if you want to mount):

```bash
sudo apt update
sudo apt install cifs-utils
```

2. Temporary browser access:

   * Open Files → Other Locations → Connect to Server: `smb://10.10.10.50/Share`
3. To mount using CLI:

```bash
mkdir -p ~/hq-share
sudo mount -t cifs //10.10.10.50/Share ~/hq-share -o username=WINDOWS_USER,password=WINDOWS_PASS,vers=3.0
```

* Replace `WINDOWS_USER` and `WINDOWS_PASS`. Adjust `vers=` parameter if necessary (SMB v2/v3 recommended).

4. To copy file from Ubuntu to the share:

```bash
cp ~/localfile.txt ~/hq-share/
```

---

## 7. Troubleshooting (common issues & fixes)

| Symptom                                  | Cause                                                     | Fix                                                                                                           |
| ---------------------------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `Host-only adapter driver not running`   | VMware host-only service / adapter missing                | Virtual Network Editor → Restore Defaults → Reboot host; enable adapter in Device Manager                     |
| FortiGate port `down`                    | VM network adapter not connected to correct VMnet         | Edit VM → Network Adapter → Custom -> select correct VMnet; enable Connected/Connect at power on              |
| DHCPREQUEST receives no reply            | No DHCP configured on FortiGate or Host-only driver issue | Configure FortiGate DHCP server on `port2` or ensure VMware DHCP is enabled; verify host adapters             |
| SMB access denied / path not found       | Firewall, SMB version mismatch, or credentials            | Enable File and Printer Sharing; open port 445; ensure correct credentials; use `vers=3.0` when mounting CIFS |
| Tunnel shows up but traffic doesn’t pass | Missing firewall policy or NAT applied                    | Add firewall policies HQ↔Branch, disable NAT on inter-site policies                                           |

---

## 8. Optional file-transfer methods

* **SCP / SSH**

  * Install `openssh-server` on Ubuntu and use `scp` from Windows (use `scp` from PowerShell or use WinSCP).
* **HTTP / Simple Web Server**

  * On sender: `python3 -m http.server 8080` then from receiver browse `http://<IP>:8080`.
* **Robocopy (Windows)** — robust copy tool:

```cmd
robocopy "C:\FolderToCopy" "\\10.10.10.50\Share" /MIR /Z /W:5 /R:3
```

* **VMware Shared Folders** — share host filesystem into guest (not recommended to test network-level connectivity).

---

## 9. Lab notes & recommendations

* Prefer SMBv2/v3 (modern and secure). Avoid enabling SMBv1.
* For production VPNs:

  * Use IKEv2 where possible.
  * Use NAT-T if either side is behind NAT.
  * Use strong crypto (AES-GCM, SHA2, Diffie-Hellman group > 19) and a secure pre-shared key/certs.
* When testing: temporarily loosen firewall rules to confirm connectivity, then tighten to least privilege later.
* Keep FortiGate firmware and VMware tools updated.

---

## Example: Quick checklist before file transfer

1. `ping` passes both ways between test hosts.
2. FortiGate interfaces are `status: up` (`get system interface` / `diagnose hardware deviceinfo nic port2`).
3. Firewall policies allow traffic between LAN and VPN.
4. For SMB: port 445 reachable (`Test-NetConnection -Port 445`).
5. Windows folder is shared and firewall rules allow File and Printer Sharing.

---

## Appendix — Useful commands recap

### FortiGate

```text
get system interface
get system interface | grep "port2"
diagnose hardware deviceinfo nic port2
diagnose sniffer packet port2 'host 10.10.10.50' 4
```

### Linux

```bash
ip a
ping 10.20.20.1
sudo dhclient -v eth0
```

### Windows (PowerShell)

```powershell
Test-NetConnection -ComputerName 10.10.10.50 -Port 445
Enable-NetFirewallRule -DisplayGroup "File and Printer Sharing"
net use Z: \\10.10.10.50\Share /user:10.10.10.50\username password
```

---


