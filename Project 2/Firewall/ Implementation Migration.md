# Firewall Implementation/Migration

## 1. Project context

The company's infrastructure is spread across **three interconnected buildings (B1, B2, B3)**.

Each building has its own **router, firewall, layer 3 switch (MLS)**, **access switches**, and internal hosts.

The objective of this project is to:

- Implement a **FortiGate 60F firewall** between the **router** and the **MLS.**
- Migrate **HSRP to VRRP** on the firewall.
- Centralize **security and inter-VLAN routing** at the firewall level.
- Maintain inter-building connectivity via GR1/GR2 trunks.
- Add the necessary ACLs.

---

## 2. Overall network architecture

### Overview

- **WAN:** Internet access via the **BBA** router and **ISP**.
- **Building 1**:
    - Router: R1
    - Firewall: FortiGate 60F
    - Access switches: S1A / S1B
    - Server: HP 1
- **Interconnection:**
    - GR1 trunk between B1 and B2
    - GR2 trunk between B2 and B3

---

## 3. Initial state (before migration)

- The **HSRP protocol** provided redundancy between the routers in each building.
- **Router R1** had subinterfaces for each VLAN:
    - `G0/0.11 → 172.20.11.1`
    - `G0/0.12 → 172.20.12.1`
- Clients used the **HSRP VIP** as a gateway.

---

## 4. Target architecture (after migration)

- The **FortiGate 60F** becomes the **VRRP gateway** for VLANs 11 and 12.
- **Inter-VLAN routing** is now done on the firewall.
- The **MLS 1** remains an **L2 switch** (trunk between S1A, S1B, HP1).
- **Router R1** only maintains an **L3 link** to the firewall (network 100.64.1.0/30).
- The VLANs of the other buildings (21, 22, 31, 32) are also entered on the firewall to ensure inter-site continuity.

## 5. Preparation and prerequisites

| Step | Description | Objective |
| --- | --- | --- |
| 1 | Back up the current configuration of the router and MLS | Enable rollback |
| 2 | Identify the VLANs used | VLAN 11 (Data) and 12 (Management) |
| 3 | Prepare the IP addresses of both firewalls | `.2` and `.3` per VLAN |
| 4 | Determine the VRRP VIP for each VLAN | `.10` for VLAN 11, `.1` for VLAN 12 |
| 5 | Configure the MLS port as a trunk to the firewall | Transport of VLANs |
| 6 | Required ACLs | Filter |

## 6. FortiGate 60F implementation plan

### Operating mode

- **Routed mode (NAT disabled)**
- **Port10** interface connected to **MLS** in **802.1Q trunk**
- **Port1** interface connected to the **router (** L3 transit network)

| VLAN | Role | Network | FW1 IP | FW2 IP | VIP (VRRP) | Interface |
| --- | --- | --- | --- | --- | --- | --- |
| 11 | Data | 172.20.11.0/24 | 172.20.11.120 | 172.20.11.130 | 172.20.11.254 | port10.11 |
| 12 | Management | 172.20.12.0/24 | 172.20.12.120 | 172.20.12.130 | 172.20.12.254 | port10.12 |
| 21 | Data B2 | 172.20.21.0/24 | 172.20.21.120 | 172.20.21.130 | 172.20.21.254 | port10.21 |
| 22 | Mgmt B2 | 172.20.22.0/24 | 172.20.22.120 | 172.20.22.130 | 172.20.22.254 | port10.22 |
| 31 | Data B3 | 172.20.31.0/24 | 172.20.31.120 | 172.20.31.130 | 172.20.31.254 | port10.31 |
| 32 | Mgmt B3 | 172.20.32.0/24 | 172.20.32.120 | 172.20.32.130 | 172.20.32.254 | port10.32 |

## 🔸 FortiGate Configuration – FW1

```arduino
config system interface
edit Data11
config vrrp
set vrip 172.20.11.254
set priority 120
set preempt enable
next
end
next
```

```arduino
config system interface
edit Management21
config vrrp
set vrip 172.20.21.254
set priority 120
set preempt enable
next
end
next
end

```

## 8. MLS Trunk ↔ Firewall Configuration

```arduino
interface GigabitEthernet1/0/13
description Trunk vers Firewall
switchport mode trunk
switchport trunk allowed vlan 11,12,21,22,31,32
```

## 9. Firewall ↔ Router R1 Link

```arduino
config system interface
edit "port1"
set ip 100.64.0.2/30
set allowaccess ping ssh
end
```

### Router R1

```arduino
interface GigabitEthernet0/0
description Lien vers Firewall
ip address 172.20.70.1 255.255.255.0
ip nat inside
no shutdown
```

- The firewall becomes the **gateway to the Internet**.
- R1 only provides the **connection to the ISP/BBA**.

## 10. HSRP → VRRP migration (steps)

- Disable HSRP on R1.
- Create VLANs 11 and 12 on the firewall (with VRRP).
- Update the client gateway → VRRP VIP.
- Test the switchover (shut down FW1 → FW2 becomes master).
- Test the return (FW1 → resumes the master role).

## 11. Testing and validation

| Test | Objective | Expected result |
| --- | --- | --- |
| Client ping → VIP | Check VRRP | OK |
| Ping VLAN 11 ↔ VLAN 12 | Check inter-VLAN | OK |
| Ping to R1 | Check WAN link | OK |
| FW1 outage | Check VRRP failover | FW2 becomes master |
| FW1 restored | Check preempt | FW1 becomes master again |
| Internet test | Check WAN output | OK |

## 12. Rollback plan

- Disconnect the firewall.
- Reconnect the R1 ↔ MLS link.
- Reactivate the HSRP subinterfaces.
- Check connectivity return.

## 13. Key security points

- Enable ping and SSH only on management interfaces.
- Restrict LAN ↔ WAN policies as needed.
- Update FortiGate firmware.
- Configure VRRP alerts and syslog logs.

## 14. Appendix – Network Summary

| Network | Usage | Equipment involved |
| --- | --- | --- |
| 172.20.11.0/24 | Data B1 | S1A, S1B, HP1 |
| 172.20.12.0/24 | Management B1 | MLS 1, Switches, Firewalls |
| 172.20.21.0/24 | Data B2 | Trunk GR1 |
| 172.20.22.0/24 | Management B2 | Trunk GR1 |
| 172.20.31.0/24 | Data B3 | Trunk GR2 |
| 172.20.32.0/24 | Management B3 | Trunk GR2 |
| 100.64.0.0/30 | Transit R1–Firewall | R1 ↔ FortiGate |

## 15. Deployment plan and cost estimate

| Step | Description | Estimated duration | Network outage? | User impact | Estimated cost (€) |
| --- | --- | --- | --- | --- | --- |
| 1 | **Preliminary analysis & configuration backup (R1, MLS1, S1A/B)** | 1 hour | No | None | 0 |
| 2 | **FortiGate preparation (firmware update, IP plan, VRRP)** | 1.5 | No | None | 0 |
| 3 | **Trunk port configuration on MLS ↔ FortiGate** | 30 min | Yes (short) | Temporary interruption of the MLS ↔ FW link | 0 |
| 4 | **Creation of VLANs on the firewall and VRRP configuration** | 1 hour | No | None | 0 |
| 5 | **HSRP deactivation on R1 and MLS1** | 15 min | Yes | Temporary network outage (~10-15 min) | 0 |
| 6 | **Transit routing R1 ↔ FortiGate (network 100.64.0.0/30)** | 30 min | Yes (during failover) | Internet interruption (~5 min) | 0 |
| 7 | **VRRP tests, FW1 ↔ FW2 failover, inter-VLAN & WAN** | 45 min | No | None | 0 |
| 8 | **Final documentation and validation** | 1h | No | None | 0 |

### 🔹 Overall summary

| Element | Value |
| --- | --- |
| **Total implementation time** | ≈ 6 hours (1 workday) |
| **Estimated total network downtime** | 15 to 20 minutes maximum |
| **Recommended maintenance window** | Evening (after 6 p.m.) or weekend |
| **Hardware cost** | None (FortiGate already purchased) |
| **Operational cost** | ~$0 to $200 (depending on internal labor) |
| **Main risk** | Temporary loss of connectivity due to gateway change |
| **Rollback plan available** | Yes (HSRP restoration and direct cabling R1–MLS1) |

### 🔹 Impact details

- **VLAN 11 (Data) users**: will temporarily lose connectivity during the HSRP → VRRP switchover.
- **VLAN 12 (Management)**: virtually no impact, as migration is planned on trunk.
- **Inter-building link (GR1)**: will remain active, VLANs will still transit, only the routing will change.
- **Internet routing**: brief interruption during the R1 ↔ Firewall switchover.

### 🔹 Mitigation measures

| Risk | Preventive measure | Workaround |
| --- | --- | --- |
| Prolonged network outage | Schedule a maintenance window | Restore old cabling R1 ↔ MLS1 |
| Configuration loss | Full backup before migration | Restore pre-migration configuration |
| VRRP not functional | Preliminary lab testing | Manual switch to FW2 |
| Management access lost | Enable SSH + console on local port | Direct access via console cable |

### 🔹 Expected benefits post-migration

- Centralization of routing and filtering on a single secure device (FortiGate).
- Network simplification: end of HSRP, fewer MLS ↔ router dependencies.
- Improved visibility and logging via Splunk, Zabbix/Syslog.
- Greater flexibility to add VLANs (buildings 2 and 3).
- Reduced VRRP failover times (< 1 second).
