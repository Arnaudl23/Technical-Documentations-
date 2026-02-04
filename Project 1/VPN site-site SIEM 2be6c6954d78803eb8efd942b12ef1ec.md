# VPN/site-site/SIEM

# IPsec VTI VPN (IKEv2) — Site-to-Site R2 ↔ R3

This document describes the configuration of an **IPsec VTI site-to-site VPN (IKEv2)** between routers **R2** and **R3**, providing a secure tunnel for management traffic and log forwarding between the test infrastructure and the SIEM.

---

## 1. Objective

- Secure all **management** and **log** traffic between:
    - The **test infrastructure** behind R2
    - The **SIEM / SOC infrastructure** behind R3
- Use an **IPsec VTI (Virtual Tunnel Interface)** with **IKEv2**
- Ensure:
    - Confidentiality (encryption)
    - Integrity
    - Mutual authentication
- Route internal networks **only through the tunnel**

---

## 2. General Principle

- R2 and R3 are connected via a WAN segment `100.64.2.0/24`.
- An IPsec VTI (Tunnel10) is configured on both routers with IPs:
    - R2: `10.200.10.1/30`
    - R3: `10.200.10.2/30`
- Static routes send internal networks across this tunnel.
- The tunnel is protected by:
    - **IKEv2 Phase 1**: negotiation, authentication, key exchange
    - **IPsec Phase 2**: encryption of data traffic

**Logical flow:**

`LAN R2 ↔ R2 ↔ VTI Tunnel ↔ R3 ↔ LAN R3 (SIEM)`

---

## 3. Topology

### 3.1 WAN Interfaces

- **R2 — G0/0 (WAN):** `100.64.2.1/24`
- **R3 — G0/2 (WAN):** `100.64.2.2/24`

### 3.2 Tunnel Interfaces

- **Tunnel10 (R2):** `10.200.10.1/30`
- **Tunnel10 (R3):** `10.200.10.2/30`

### 3.3 Example Internal Networks

- **Behind R2 (test infra):**`10.20.0.0/16`
- **Behind R3 (SIEM side):**`192.168.1.0/24192.168.2.0/24`

These networks are only reachable **through the IPsec tunnel**.

---

## 4. IKEv2 / IPsec Design

### 4.1 Common IKEv2 Settings

- **Version:** IKEv2
- **Encryption:** AES-256 (CBC)
- **Integrity:** SHA-256
- **DH Group:** 14
- **Authentication:** Pre-Shared Key (PSK)

### 4.2 Pre-Shared Key

Both routers use the same PSK (example):

```
Pre-shared-key: R2-R3-VPN

```

> This is a lab value and must be replaced in production.
> 

---

## 5. R3 Configuration (SIEM Side)

### 5.1 IKEv2 Proposal

```
crypto ikev2 proposal IKEV2-PROP
 encryption aes-cbc-256
 integrity sha256
 group 14

```

### 5.2 IKEv2 Policy

```
crypto ikev2 policy 1
 proposal IKEV2-PROP

```

### 5.3 Keyring

```
crypto ikev2 keyring IKEV2-KEYRING-R2
 peer R2
  address 100.64.2.1
  pre-shared-key R2-R3-VPN

```

### 5.4 IKEv2 Profile

```
crypto ikev2 profile IKEV2-PROFILE-R3-R2
 match identity remote address 100.64.2.1 255.255.255.255
 identity local address 100.64.2.2
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEV2-KEYRING-R2
 crypto ikev2 policy 1
 proposal IKEV2-PROP

```

---

### 5.5 IPsec Transform-Set (Phase 2)

```
crypto ipsec transform-set TS-R3-R2 esp-aes 256 esp-sha256-hmac
 mode tunnel

```

### 5.6 IPsec Profile

```
crypto ipsec profile PROF-R3-R2
 set transform-set TS-R3-R2
 set ikev2-profile IKEV2-PROFILE-R3-R2

```

---

### 5.7 Tunnel Interface (R3)

```
interface Tunnel10
 ip address 10.200.10.2 255.255.255.252
 ip tcp adjust-mss 1360
 tunnel source GigabitEthernet0/2
 tunnel destination 100.64.2.1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF-R3-R2

```

### 5.8 Static Route (R3 → R2 LAN)

```
ip route 10.20.0.0 255.255.0.0 10.200.10.1

```

---

## 6. R2 Configuration (Test Infrastructure Side)

### 6.1 IKEv2 Proposal

```
crypto ikev2 proposal IKEV2-PROP
 encryption aes-cbc-256
 integrity sha256
 group 14

```

### 6.2 IKEv2 Policy

```
crypto ikev2 policy 1
 proposal IKEV2-PROP

```

### 6.3 Keyring

```
crypto ikev2 keyring IKEV2-KEYRING-R3
 peer R3
  address 100.64.2.2
  pre-shared-key R2-R3-VPN

```

### 6.4 IKEv2 Profile

```
crypto ikev2 profile IKEV2-PROFILE-R2-R3
 match identity remote address 100.64.2.2 255.255.255.255
 identity local address 100.64.2.1
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEV2-KEYRING-R3
 crypto ikev2 policy 1
 proposal IKEV2-PROP

```

---

### 6.5 IPsec Transform-Set

```
crypto ipsec transform-set TS-R2-R3 esp-aes 256 esp-sha256-hmac
 mode tunnel

```

### 6.6 IPsec Profile

```
crypto ipsec profile PROF-R2-R3
 set transform-set TS-R2-R3
 set ikev2-profile IKEV2-PROFILE-R2-R3

```

---

### 6.7 Tunnel Interface (R2)

```
interface Tunnel10
 ip address 10.200.10.1 255.255.255.252
 ip tcp adjust-mss 1360
 tunnel source GigabitEthernet0/0
 tunnel destination 100.64.2.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF-R2-R3

```

### 6.8 Static Routes (R2 → SIEM LANs)

```
ip route 192.168.1.0 255.255.255.0 10.200.10.2
ip route 192.168.2.0 255.255.255.0 10.200.10.2

```

---

## 7. Verification Commands

### 7.1 Check IKEv2 SA

```
show crypto ikev2 sa

```

### 7.2 Check IPsec SA

```
show crypto ipsec sa

```

### 7.3 Check Tunnel Interface

```
show interface Tunnel10

```

### 7.4 Ping Tests

```
ping 10.200.10.2
ping 192.168.1.1

```

---

## 8. Traffic Flow Summary

1. Host behind R2 sends traffic to SIEM network.
2. R2 forwards packets into the encrypted tunnel.
3. WAN path `100.64.2.x` transports encrypted ESP packets.
4. R3 decapsulates packets and forwards to SIEM LAN.
5. Return traffic flows back through the tunnel.

---