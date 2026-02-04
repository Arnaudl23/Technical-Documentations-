# Config S1

# SA1 – Base Configuration (Cisco Switch)

## 1. Objective

Deploy a clean, standardized, and secure base configuration on Cisco switch **SA1**, including device identity, local authentication, SSH security, VLAN structure, trunking, access ports, syslog, and NTP.

---

## 2. Hardware

- Cisco Switch **WS-C2960+24TC-L**

---

## 3. Hostname

```
SA1(config)# hostname SA1

```

---

## 4. Secure Privileged EXEC Mode

```
SA1(config)# enable secret formation

```

---

## 5. Secure Console Access

```
SA1(config)# line con 0
SA1(config-line)# password formation
SA1(config-line)# logging synchronous
SA1(config-line)# login

```

---

## 6. Encrypt All Plaintext Passwords

```
SA1(config)# service password-encryption

```

---

## 7. Login Attempt Protection

```
SA1(config)# login block-for 30 attempts 3 within 15

```

---

## 8. MOTD Banner

```
SA1(config)# banner motd $Authorized Access Only$

```

---

## 9. Set System Clock

```
SA1# clock set 20:40:00 14 November 2025

```

---

## 10. Create User Accounts

```
SA1(config)# username constantin privilege 15 secret constantin1
SA1(config)# username quentin privilege 15 secret quentin1
SA1(config)# username arnaud privilege 15 secret arnaud1
SA1(config)# username maxime privilege 15 secret maxime1
SA1(config)# username philippe privilege 15 secret philippe1
SA1(config)# username samuelg privilege 15 secret samuelg1
SA1(config)# username samuell privilege 15 secret samuell1
SA1(config)# username jeanfrancois privilege 15 secret jeanfrancois1
SA1(config)# username mathieu privilege 15 secret mathieu1
SA1(config)# username julien privilege 15 secret julien1
SA1(config)# username solenne privilege 15 secret solenne1
SA1(config)# username charles privilege 15 secret charles1

```

---

## 11. SSH Configuration

### 11.1 Domain Name

```
SA1(config)# ip domain-name infra.labo

```

### 11.2 Administrator User

```
SA1(config)# username admin secret formation

```

### 11.3 Generate RSA Keys

```
SA1(config)# crypto key generate rsa general-keys modulus 2048

```

### 11.4 Enable SSH v2

```
SA1(config)# ip ssh version 2

```

### 11.5 Configure VTY Lines

```
SA1(config)# line vty 0 15
SA1(config-line)# transport input ssh
SA1(config-line)# login local
SA1(config-line)# exec-timeout 10
SA1(config-line)# logging synchronous

```

---

## 12. Management SVI

```
SA1(config)# interface vlan10
SA1(config-if)# ip address 172.20.1.1 255.255.255.0
SA1(config-if)# no shutdown

```

---

## 13. Default Gateway

```
SA1(config)# ip default-gateway 172.20.1.254

```

---

## 14. NTP Configuration

```
SA1(config)# clock timezone UTC 1
SA1(config)# clock summer-time CEST recurring last Sun Mar 2:00 last Sun Oct 3:00
SA1(config)# ntp server 10.20.2.2show 54

```

---

## 15. VLAN Configuration

```
SA1(config)# vlan 200
SA1(config-vlan)# name Guest
SA1(config)# vlan 100
SA1(config-vlan)# name User
SA1(config)# vlan 10
SA1(config-vlan)# name Management
SA1(config)# vlan 900
SA1(config-vlan)# name Native

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
SA1(config)# interface g0/1
SA1(config-if)# switchport mode trunk
SA1(config-if)# switchport nonegotiate
SA1(config-if)# switchport trunk native vlan 900
SA1(config-if)# switchport trunk allowed vlan 10,100,200,900
SA1(config-if)# no shutdown

```

### Gi0/2

```
SA1(config)# interface g0/2
SA1(config-if)# switchport mode trunk
SA1(config-if)# switchport nonegotiate
SA1(config-if)# switchport trunk native vlan 900
SA1(config-if)# switchport trunk allowed vlan 10,100,200,900
SA1(config-if)# no shutdown

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
SA1(config)# interface range fa0/1 - 24
SA1(config-if)# switchport port-security
SA1(config-if)# switchport port-security maximum 2
SA1(config-if)# switchport port-security violation restrict
SA1(config-if)# switchport port-security aging time 2
SA1(config-if)# switchport port-security aging type inactivity

```

---

## 21. Validation

Not implemented due to lab conditions. Common validation commands:

- show vlan
- show ip int brief
- show spanning-tree
- show port-security

---
