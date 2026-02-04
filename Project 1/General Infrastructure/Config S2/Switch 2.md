# Config S2

# SA2 – Base Configuration (Cisco Switch)

## 1. Objective

Deploy a clean, standardized, and secure base configuration on Cisco switch **SA2**, including hostname, authentication, SSH access, VLANs, trunking, syslog, NTP, and port security.

---

## 2. Hardware

- Cisco Switch **WS-C2960+24TC-L**

---

## 3. Hostname

```
SA2(config)# hostname SA2

```

---

## 4. Secure Privileged EXEC Mode

```
SA2(config)# enable secret formation

```

---

## 5. Secure Console Access

```
SA2(config)# line con 0
SA2(config-line)# password formation
SA2(config-line)# logging synchronous
SA2(config-line)# login

```

---

## 6. Encrypt All Plaintext Passwords

```
SA2(config)# service password-encryption

```

---

## 7. Login Attempt Protection

```
SA2(config)# login block-for 30 attempts 3 within 15

```

---

## 8. MOTD Banner

```
SA2(config)# banner motd $Authorized Access Only$

```

---

## 9. Set System Clock

```
SA2# clock set 20:40:00 14 November 2025

```

---

## 10. User Accounts

```
SA2(config)# username constantin privilege 15 secret constantin1
SA2(config)# username quentin privilege 15 secret quentin1
SA2(config)# username arnaud privilege 15 secret arnaud1
SA2(config)# username maxime privilege 15 secret maxime1
SA2(config)# username philippe privilege 15 secret philippe1
SA2(config)# username samuelg privilege 15 secret samuelg1
SA2(config)# username samuell privilege 15 secret samuell1
SA2(config)# username jeanfrancois privilege 15 secret jeanfrancois1
SA2(config)# username mathieu privilege 15 secret mathieu1
SA2(config)# username julien privilege 15 secret julien1
SA2(config)# username solenne privilege 15 secret solenne1
SA2(config)# username charles privilege 15 secret charles1

```

---

## 11. SSH Configuration

### 11.1 Domain Name

```
SA2(config)# ip domain-name infra.labo

```

### 11.2 Administrator User

```
SA2(config)# username admin secret formation

```

### 11.3 Generate RSA Keys

```
SA2(config)# crypto key generate rsa general-keys modulus 1024

```

### 11.4 Enable SSH v2

```
SA2(config)# ip ssh version 2

```

### 11.5 Configure VTY Lines

```
SA2(config)# line vty 0 15
SA2(config-line)# transport input ssh
SA2(config-line)# login local
SA2(config-line)# exec-timeout 10 0
SA2(config-line)# logging synchronous

```

---

## 12. Management SVI

```
SA2(config)# interface vlan10
SA2(config-if)# ip address 172.20.1.2 255.255.255.0
SA2(config-if)# no shutdown

```

---

## 13. Default Gateway

```
SA2(config)# ip default-gateway 172.20.1.253

```

---

## 14. NTP Configuration

```
SA2(config)# clock timezone UTC 1
SA2(config)# clock summer-time CEST recurring last Sun Mar 2:00 last Sun Oct 3:00
SA2(config)# ntp server 10.20.2.254

```

---

## 15. VLAN Configuration

```
SA2(config)# vlan 200
SA2(config-vlan)# name Guest
SA2(config)# vlan 100
SA2(config-vlan)# name User
SA2(config)# vlan 10
SA2(config-vlan)# name Management
SA2(config)# vlan 900
SA2(config-vlan)# name Native

```

---

## 16. Access Ports

```
SA2(config)# interface range f0/1 - 22
SA2(config-if)# switchport mode access
SA2(config-if)# switchport access vlan 100
SA2(config-if)# switchport nonegotiate
SA2(config-if)# spanning-tree bpduguard enable
SA2(config-if)# spanning-tree portfast

```

### VLAN 200 Port

```
SA2(config)# interface f0/23
SA2(config-if)# switchport mode access
SA2(config-if)# switchport access vlan 200
SA2(config-if)# switchport nonegotiate
SA2(config-if)# spanning-tree bpduguard enable
SA2(config-if)# spanning-tree portfast

```

### Management VLAN Port

```
SA2(config)# interface f0/24
SA2(config-if)# switchport mode access
SA2(config-if)# switchport access vlan 10
SA2(config-if)# switchport nonegotiate
SA2(config-if)# spanning-tree bpduguard enable
SA2(config-if)# spanning-tree portfast
SA2(config-if)# no shutdown

```

---

## 17. Trunk Ports

### Gi0/1

```
SA2(config)# interface g0/1
SA2(config-if)# switchport mode trunk
SA2(config-if)# switchport nonegotiate
SA2(config-if)# switchport trunk native vlan 900
SA2(config-if)# switchport trunk allowed vlan 10,100,200,900
SA2(config-if)# no shutdown

```

### Gi0/2

```
SA2(config)# interface g0/2
SA2(config-if)# switchport mode trunk
SA2(config-if)# switchport nonegotiate
SA2(config-if)# switchport trunk native vlan 900
SA2(config-if)# switchport trunk allowed vlan 10,100,200,900
SA2(config-if)# no shutdown

```

---

## 18. Save Configuration

```
SA2# copy run start

```

---

## 19. Syslog Forwarding

```
SA2(config)# logging trap notifications
SA2(config)# logging origin-id hostname
SA2(config)# logging host 172.20.21.14

```

---

## 20. Port Security (Optional)

```
SA2(config)# interface range fa0/1 - 24
SA2(config-if)# switchport port-security
SA2(config-if)# switchport port-security maximum 2
SA2(config-if)# switchport port-security violation restrict
SA2(config-if)# switchport port-security aging time 2
SA2(config-if)# switchport port-security aging type inactivity

```

---

## 21. Validation

Not implemented due to lab conditions. Typical validation commands:

- show vlan
- show ip int brief
- show spanning-tree
- show port-security

---
