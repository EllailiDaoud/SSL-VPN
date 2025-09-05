# SSL & VPN — pfSense SSL-VPN Lab

A hands-on lab that builds a production-style **SSL/TLS VPN** (OpenVPN on pfSense), including PKI, server + client config, and verification. The lab also clarifies how **SSL/TLS** and **VPN** differ and complement each other.

> Authors: **Daoud ELLAILI**, **Anas EL GHANDOUR** • Supervisor: **M. Mohammed BOUGRINE**

---

## 1) Overview

- **SSL/TLS** secures point-to-point application traffic (e.g., HTTPS).
- **VPN** secures an entire network path (IP layer), enabling remote access to private resources.
- **SSL-VPN** (OpenVPN over TLS) combines both: a VPN tunnel authenticated and encrypted with TLS.

---

## 2) Lab Architecture

```
[Client Laptop] --(Internet, UDP/1194 over TLS)--> [pfSense (OpenVPN Server)] -- [LAN Services]
      10.8.0.0/24 (VPN)
                                    192.168.10.0/24 (LAN)
```

**Components**
- **pfSense**: Firewall/VPN appliance (OpenVPN server, CA & certificates, firewall rules).
- **Client**: Windows/macOS/Linux with **OpenVPN client**.
- **LAN target(s)**: Web server, file share, or any internal service to test access.

---

## 3) Prerequisites

- pfSense 2.6+ / 2.7+ with:
  - WAN reachable from the Internet (public IP or port-forward).
  - **UDP/1194** open on WAN (default OpenVPN).
- Time synced (NTP) on pfSense and client.
- Optional: Domain/DNS record for the VPN server (e.g., `vpn.example.local`).

---

## 4) Network Plan (example)

| Item                  | Value                  |
|----------------------|------------------------|
| OpenVPN Port/Proto   | `1194/udp`             |
| Tunnel Network       | `10.8.0.0/24`          |
| Local (LAN) Network  | `192.168.10.0/24`      |
| TLS Version          | Min TLS 1.2            |
| Cipher               | AES-256-GCM            |
| HMAC Auth            | TLS-Crypt (preferred)  |

---

## 5) Quick Start (TL;DR)

1. **Create CA & certs** → *System → Cert. Manager*.
2. **OpenVPN Wizard** → *VPN → OpenVPN*; choose **Remote Access (SSL/TLS + User Auth)**.
3. **Server settings**:
   - `UDP/1194`, Tunnel `10.8.0.0/24`, Local Networks `192.168.10.0/24`
   - TLS-Crypt enabled, TLS 1.2+, AES-256-GCM, Auth SHA256, ECDH P-256
4. **Firewall**:
   - WAN rule to allow `UDP/1194` to pfSense.
   - OpenVPN tab: allow traffic to LAN (or least-privilege rules).
5. **Users** → *System → User Manager*: create user, issue **User Cert**.
6. **Export Client** → install **OpenVPN Client Export** package; download `.ovpn`.
7. **Connect & Test**: `ping`, `traceroute`, access internal apps.

---

## 6) Step-by-Step Setup (pfSense)

### 6.1 Create the PKI
- **System → Cert. Manager → CAs → Add**
  - Create an **Internal CA** (CN like `Lab-Root-CA`).
- **System → Cert. Manager → Certificates → Add/Sign**
  - Create **Server Certificate** (CN `vpn.pfsense.local`, type **Server**).
  - For each user, create a **User Certificate** (type **User**).

### 6.2 OpenVPN Server (Wizard)
- **VPN → OpenVPN → Wizards**
  1. Select your CA.
  2. Server cert = your **OpenVPN Server** cert.
  3. **Type**: *Remote Access (SSL/TLS + User Auth)*.
  4. **Protocol/Port**: `UDP/1194`.
  5. **Tunnel Network**: `10.8.0.0/24`.
  6. **Local networks** (accessible over VPN): `192.168.10.0/24`.
  7. **TLS**: enable **TLS-Crypt**, require **TLS 1.2+**.
  8. **Crypto**: **AES-256-GCM**, Auth **SHA256**, ECDH **prime256v1**.
  9. **Client Settings**:
     - **DNS** (optional): push internal DNS (e.g., `192.168.10.1`) and domain.
     - **Compression**: **Disabled**.
     - **Topology**: `subnet`.

> **Split-tunnel vs Full-tunnel**:  
> - Split = only **local networks** go through VPN (recommended).  
> - Full = push default route via VPN (`Redirect Gateway`) to send all Internet traffic through the tunnel.

### 6.3 Firewall Rules
- **Firewall → Rules → WAN**: Add `UDP/1194` → **OpenVPN server**.
- **Firewall → Rules → OpenVPN**: Allow from `OpenVPN net` to **LAN** (or use granular rules).

### 6.4 User & Authentication
- **System → User Manager**:
  - Add user (e.g., `alice`), assign **User Certificate**.
- (Optional) **MFA (TOTP)**: *System → User Manager → Users → Enable TOTP* and enforce MFA in OpenVPN server.

### 6.5 Export Client Profiles
- **System → Package Manager**: Install **openvpn-client-export**.
- **VPN → OpenVPN → Client Export**:
  - Select the **Server Instance**.
  - Export the user’s **Inline Config (.ovpn)** (includes certs/keys).

### 6.6 Client Installation & Connection
- **Windows/macOS**: Install the OpenVPN client, import `.ovpn`, connect.
- **Linux (CLI)**:
  ```bash
  sudo openvpn --config ~/Downloads/alice-lab.ovpn
  ```

---

## 7) Verification

- Check VPN IP:
  ```bash
  ip addr | grep 10.8.
  ```
- Reach LAN host:
  ```bash
  ping 192.168.10.10
  ```
- Route table (Linux):
  ```bash
  ip route
  ```
- If **Full-tunnel** enabled, verify public IP changes:
  ```bash
  curl ifconfig.io
  ```

---

## 8) Troubleshooting (What we usually see)

- **Can’t connect**: WAN rule missing, ISP blocks UDP/1194 → change to TCP/443 temporarily.
- **TLS errors**: Wrong time (NTP), cert CN mismatch, expired certs.
- **Connected but no LAN access**: OpenVPN or LAN rules missing; wrong **Local Networks**; server or client **topology** mismatch.
- **Name resolution fails**: DNS not pushed; fix in server **Client Settings**.

---

## 9) Security Notes / Hardening

- Prefer **TLS-Crypt** (hides handshake) and **AES-GCM** ciphers.
- Enforce **TLS 1.2+**, disable legacy ciphers/compression.
- Use **per-user certs** + **MFA**; revoke compromised certs promptly.
- Scope access with **granular OpenVPN firewall rules** (least privilege).
- Monitor logs: *Status → System Logs → OpenVPN*.

---

## 10) Project Structure (suggested)

```
.
├─ docs/
│  ├─ slides.pdf
│  └─ topology.md
├─ pki/
│  ├─ ca.crt
│  ├─ server.crt
│  └─ revocation-list.crl
├─ client-profiles/
│  ├─ alice.ovpn
│  └─ bob.ovpn
├─ examples/
│  └─ server.conf.sample
└─ README.md
```

---

## 11) Example `server.conf` (reference)

```conf
port 1194
proto udp
dev tun
topology subnet

server 10.8.0.0 255.255.255.0
push "route 192.168.10.0 255.255.255.0"

tls-version-min 1.2
tls-cipher TLS-ECDHE-ECDSA-WITH-AES-256-GCM-SHA384
tls-crypt /var/etc/openvpn/server1.tls-crypt
cipher AES-256-GCM
auth SHA256
dh none
ecdh-curve prime256v1
user nobody
group nobody
persist-key
persist-tun
keepalive 10 60
explicit-exit-notify 1
```

---

## 12) Credits & Reference

- Based on our project slides: *Introduction → SSL → VPN → Differences → Combined → SSL-VPN Lab (Architecture, pfSense Configuration, Certificate Generation, Mode Configuration, User Authentication, Issues, Conclusion).*

