# VPN/Fortinet

# FortiGate IPsec VPN (IKEv2) — Remote Access to Jump Server

This document describes the full configuration of an **IPsec Remote Access VPN (IKEv2)** on a FortiGate firewall, allowing authenticated external users to securely access an internal **Jump Server** only.

---

## 1. General Principle

- External user connects using **FortiClient (IPsec IKEv2)**
- FortiGate authenticates the user
- FortiGate assigns a VPN IP from a dedicated pool
- Firewall rules restrict traffic **only to the Jump Server**
- No access to the internal LAN

**Flow:**

`User → Internet → Public IP → FortiGate → IPsec Tunnel → Jump Server`

---

## 2. Access to the Bastion (Guacamole)

The VPN will allow access to:

- *[http://172.20.4.1:8080/guacamole**](http://172.20.4.1:8080/guacamole**)

---

## 3. Creating Local Users

Menu:

**User & Authentication → User Definition**

For each VPN user:

- **Name:** e.g., charle
- **Type:** Local
- **Password:** xxx
- **Two‑factor:** optional

### Example Screens

![image.png](image.png)

![image.png](image%201.png)

---

## 4. Create User Group for VPN

Menu:

**User & Authentication → User Groups → Create New**

- **Name:** `VPN_Admins`
- **Type:** Firewall
- **Members:** select authorized users (e.g., Charle, Ju)

![image.png](image%202.png)

---

## 5. Define IP Pool for VPN Clients

Menu:

**Policy & Objects → Addresses → Create New**

- **Name:** `IPSEC-pool`
- **Type:** IP Range
- **Interface:** WAN1
- **Range:** `10.100.1.10–10.100.1.50`

![image.png](image%203.png)

---

## 6. Create Object for Jump Server

Menu:

**Policy & Objects → Addresses → Create New**

- **Name:** `Bastion`
- **Type:** Subnet
- **IP/Mask:** `172.20.4.1/32`
- **Interface:** INT5

![image.png](image%204.png)

---

## 7. Create the Remote Access IPsec Tunnel

Menu:

**VPN → IPsec Wizard**

---

### Step 1 — General Configuration

- **Template:** Remote Access
- **VPN Type:** IPsec
- **Incoming Interface:** WAN1
- **Authentication method:** Pre‑shared Key
- **Pre‑shared key:** `JUJU123!`
- **User Group:** `VPN_Admins`

![image.png](image%205.png)

---

![image.png](image%206.png)

### Step 2 — Phase 1 (IKEv2)

Configure IKEv2 parameters:

- **IKE Version:** IKEv2
- **Encryption:** AES‑256
- **Authentication:** SHA‑256
- **DH Group:** 14
- **NAT Traversal:** Enabled

---

### Step 3 — Policy & Routing

- **Local interface:** Bastion
- **Local address:** Bastion-1
- **Client Address Range:** `10.100.1.10–10.100.1.50`
- **Subnet Mask:** `255.255.255.0`
- **Enable IPv4 Split Tunneling:** ON
→ Only traffic to Jump Server will pass
- **DNS:** Use System DNS

![image.png](image%207.png)

### Client Options

- **Keep Alive:** closes tunnel when inactive

![image.png](image%208.png)

---

## 8. Firewall Policy (VPN → Jump Server)

Menu:

**Policy & Objects → Firewall Policy**

| Setting | Value |  |
| --- | --- | --- |
| **Name** | `VPN_TO_JUMP` |  |
| **Incoming Interface** | IPSEC‑TUNNEL |  |
| **Outgoing Interface** | internal |  |
| **Source** | IPSEC‑TUNNEL_range |  |
| **Destination** | Bastion |  |
| **Services** | RDP / SSH / HTTPS |  |
| **Action** | ACCEPT |  |
| **NAT** | OFF |  |

---

![image.png](image%209.png)

## 9. FortiClient Installation

Download from the official link:

[https://links.fortinet.com/forticlient/win/vpnagent](https://links.fortinet.com/forticlient/win/vpnagent)

Choose: **FortiClient VPN only**

---

### Connection Parameters

- **Gateway:** `194.78.154.149`
- **Shared Key:** `JUJU123!`
- **Username:** provided separately
- **Password:** Formation1

![image.png](image%2010.png)

![image.png](image%2011.png)

---

![image.png](image%2012.png)

## 10. Communication Flow

### 1) External user PC

FortiClient sends an IKE request to the **public IP** of the FortiGate.

### 2) Upstream router

A NAT rule redirects traffic:

- UDP **500** → FortiGate
- UDP **4500** → FortiGate

---

This Markdown file is structured cleanly and ready for conversion to **PDF**, **HTML**, or import into **VS Code / Notion / GitHub**.
