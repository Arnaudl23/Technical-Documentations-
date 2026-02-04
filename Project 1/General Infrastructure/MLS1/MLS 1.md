# MLS1

# 

MLS1 – Base Configuration (Cisco Multilayer Switch)

## 1. Objective

Deploy a clean, standardized, and secure base configuration on multilayer switch **MLS1**, including hostname, authentication, SSH access, VLAN SVI interfaces, routing, DHCP snooping/DAI, NTP, syslog, trunks, routed interfaces, IP SLA, and static routing.

---

## 2. Hardware

- Cisco Switch **WS-C3850-24T**

---

## 3. Hostname

MLS1(config)# hostname MLS1

---

## 4. Secure Privileged EXEC Mode

MLS1(config)# enable secret formation

---

## 5. Secure Console Access

MLS1(config)# line con 0
MLS1(config-line)# password formation
MLS1(config-line)# logging synchronous
MLS1(config-line)# login

---

## 6. Encrypt All Plaintext Passwords

MLS1(config)# service password-encryption

---

## 7. Login Attempt Protection

MLS1(config)# login block-for 30 attempts 3 within 15

---

## 8. MOTD Banner

MLS1(config)# banner motd $Authorized Access Only$

---

## 9. Set System Clock

MLS1# clock set 20:40:00 14 November 2025

---

## 10. SSH Configuration

### 10.1 Domain Name

MLS1(config)# ip domain-name infra.labo

### 10.2 Administrator User

MLS1(config)# username admin secret formation

### 10.3 Generate RSA Keys

MLS1(config)# crypto key generate rsa general-keys modulus 1024

### 10.4 Enable SSH v2

MLS1(config)# ip ssh version 2

### 10.5 Configure VTY Lines

MLS1(config)# line vty 0 15
MLS1(config-line)# transport input ssh
MLS1(config-line)# login local
MLS1(config-line)# exec-timeout 10 0
MLS1(config-line)# logging synchronous

---

## 11. Management SVI

MLS1(config)# interface vlan10
MLS1(config-if)# ip address 172.20.1.254 255.255.255.0
MLS1(config-if)# no shutdown

---

## 12. Local User Accounts

MLS1(config)# username constantin privilege 15 secret constantin1
MLS1(config)# username quentin privilege 15 secret quentin1
MLS1(config)# username arnaud privilege 15 secret arnaud1
MLS1(config)# username maxime privilege 15 secret maxime1
MLS1(config)# username philippe privilege 15 secret philippe1
MLS1(config)# username samuelg privilege 15 secret samuelg1
MLS1(config)# username samuell privilege 15 secret samuell1
MLS1(config)# username jeanfrancois privilege 15 secret jeanfrancois1
MLS1(config)# username mathieu privilege 15 secret mathieu1
MLS1(config)# username julien privilege 15 secret julien1
MLS1(config)# username solenne privilege 15 secret solenne1
MLS1(config)# username charles privilege 15 secret charles1

---

## 13. Enable L3 Routing

MLS1(config)# ip routing

---

## 14. NTP Configuration

MLS1(config)# ntp server 10.20.1.254
MLS1(config)# ntp update-calendar

---

## 15. DHCP Snooping & Dynamic ARP Inspection

MLS1(config)# ip dhcp snooping
MLS1(config)# ip dhcp snooping vlan 10,100,200
MLS1(config)# ip arp inspection vlan 10,100,200

### Trusted Interfaces

MLS1(config)# interface g1/0/1
MLS1(config-if)# ip dhcp snooping trust
MLS1(config-if)# ip arp inspection trust

MLS1(config)# interface g1/0/2
MLS1(config-if)# ip dhcp snooping trust
MLS1(config-if)# ip arp inspection trust

MLS1(config)# interface g1/0/3
MLS1(config-if)# ip dhcp snooping trust
MLS1(config-if)# ip arp inspection trust

MLS1(config)# interface g1/0/11
MLS1(config-if)# ip dhcp snooping trust
MLS1(config-if)# ip arp inspection trust

---

## 16. Syslog Forwarding

MLS1(config)# logging trap notifications
MLS1(config)# logging origin-id hostname
MLS1(config)# logging source-interface Loopback1
MLS1(config)# logging host 172.20.21.14

---

## 17. DHCP Relay (IP Helper)

MLS1(config)# interface vlan10
MLS1(config-if)# ip helper-address 172.20.21.67
MLS1(config-if)# ip helper-address 172.20.23.67

MLS1(config)# interface vlan100
MLS1(config-if)# ip helper-address 172.20.21.67
MLS1(config-if)# ip helper-address 172.20.23.67

MLS1(config)# interface vlan200
MLS1(config-if)# ip helper-address 172.20.21.67
MLS1(config-if)# ip helper-address 172.20.23.67

---

## 18. STP, VLANs & Loopback

### 18.1 STP Mode

MLS1(config)# spanning-tree mode rapid-pvst

### 18.2 VLANs

MLS1(config)# vlan 100
MLS1(config-vlan)# name User

MLS1(config)# vlan 200
MLS1(config-vlan)# name Guest

MLS1(config)# vlan 900
MLS1(config-vlan)# name Native

### 18.3 STP Root Primary

MLS1(config)# spanning-tree vlan 10,100,200,900 root primary

### 18.4 Loopback Interface

MLS1(config)# interface Loopback1
MLS1(config-if)# ip address 1.1.1.1 255.255.255.255

---

## 19. Additional SVIs

MLS1(config)# interface vlan100
MLS1(config-if)# ip address 172.20.10.254 255.255.255.0
MLS1(config-if)# no shutdown

MLS1(config)# interface vlan200
MLS1(config-if)# ip address 172.20.2.254 255.255.255.0
MLS1(config-if)# no shutdown

---

## 20. Trunk Ports

### g1/0/1

MLS1(config)# interface g1/0/1
MLS1(config-if)# switchport mode trunk
MLS1(config-if)# switchport nonegotiate
MLS1(config-if)# switchport trunk allowed vlan 10,100,200
MLS1(config-if)# switchport trunk native vlan 900
MLS1(config-if)# spanning-tree portfast trunk

### g1/0/2

MLS1(config)# interface g1/0/2
MLS1(config-if)# switchport mode trunk
MLS1(config-if)# switchport nonegotiate
MLS1(config-if)# switchport trunk allowed vlan 10,100,200
MLS1(config-if)# switchport trunk native vlan 900
MLS1(config-if)# spanning-tree portfast trunk

### g1/0/3

MLS1(config)# interface g1/0/3
MLS1(config-if)# switchport mode trunk
MLS1(config-if)# switchport nonegotiate
MLS1(config-if)# switchport trunk allowed vlan 10,100,200
MLS1(config-if)# switchport trunk native vlan 900
MLS1(config-if)# spanning-tree portfast trunk

---

## 21. Routed Interfaces

### g1/0/11 (towards BBA2)

MLS1(config)# interface g1/0/11
MLS1(config-if)# no switchport
MLS1(config-if)# ip address 10.20.3.2 255.255.255.0
MLS1(config-if)# no shutdown

### g1/0/5 (towards R1)

MLS1(config)# interface g1/0/5
MLS1(config-if)# no switchport
MLS1(config-if)# ip address 10.20.8.253 255.255.255.0
MLS1(config-if)# no shutdown

### g1/0/6 (towards R2)

MLS1(config)# interface g1/0/6
MLS1(config-if)# no switchport
MLS1(config-if)# ip address 10.20.6.253 255.255.255.0
MLS1(config-if)# no shutdown

---

## 22. IP SLA Tracking & Default Routes

### 22.1 Configure IP SLA

MLS1(config)# ip sla 1
MLS1(config-ip-sla)# icmp-echo 100.64.1.254 source-interface vlan10
MLS1(config-ip-sla)# frequency 5
MLS1(config-ip-sla)# timeout 5000
MLS1(config-ip-sla)# exit

### 22.2 Schedule IP SLA

MLS1(config)# ip sla schedule 1 life forever start-time now

### 22.3 Tracking Object

MLS1(config)# track 1 ip sla 1 reachability

---

## 23. Default Routes

MLS1(config)# ip route 0.0.0.0 0.0.0.0 10.20.3.254 track 1
MLS1(config)# ip route 0.0.0.0 0.0.0.0 10.20.1.254 200

---

## 24. Static Routes

MLS1(config)# ip route 10.20.1.0 255.255.255.0 10.20.10.1
MLS1(config)# ip route 10.20.2.0 255.255.255.0 10.20.10.1
MLS1(config)# ip route 10.20.4.0 255.255.255.0 10.20.3.254
MLS1(config)# ip route 10.20.5.0 255.255.255.0 10.20.10.1
MLS1(config)# ip route 100.64.1.0 255.255.255.0 10.20.10.1
MLS1(config)# ip route 100.64.2.0 255.255.255.0 10.20.3.254
MLS1(config)# ip route 172.20.3.0 255.255.255.0 10.20.10.1

---

## 25. Save Configuration

MLS1# copy run start

---

## 26. Validation Commands

show ip interface brief
show running-config
show vlan
show ip arp inspection
show ip dhcp snooping
show spanning-tree
show ip route
show ip sla statistics
