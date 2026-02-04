# DNS Servers

# Objective

<aside>
🎯

DNS (Domain Name System) servers are responsible for ensuring:

- The **internal resolution** of infrastructure domain names `infra.labo`.
- The **delegation** of requests to domain controllers for the AD zone : `ad.infra.labo`.
- **Securing** DNS access by filtering authorized networks.
- **Redundancy** via two DNS servers: `DNS1` And `DNS2`.

This documentation describes the **complete implementation of BIND9**, configuration of zones, ACLs, logs, permissions, DNS delegation, and validation tests

</aside>

# **Material used**

### Physical

| **Name** | **VM** |
| --- | --- |
| **pve1** | DNS1, AD1 |
| **pve2** | DNS2, AD2 |

### **Virtual**

| **Name** | **Status** | **Authority over** |
| --- | --- | --- |
| DNS1 | Master | `infra.labo` |
| DNS2 | Slavic | `infra.labo` |
| DC1 | Master | `ad.infra.labo` |
| DC2 | Slavic | `ad.infra.labo` |

# **Functioning**

Here is a clear overview of the path of a DNS query:

1. **User request**
    
    The user enters a URL and the DNS resolver configured on their machine is requested.
    
2. **Checking cache**
    
    The resolver checks if it already has the answer in memory. If so, he responds immediately.
    
3. **Hierarchical query**
    
    If not cached, the resolver contacts successively:
    
    - a **root server**,
    - a **TLD server** (.com, .fr…),
    - then the **authoritative server** for the domain.
4. **Obtaining the IP address**
    
    The authoritative server provides the IP address corresponding to the requested name.
    
5. **Response to customer**
    
    The resolver passes the IP address to the browser.
    
6. **Caching**
    
    The response is stored at a TTL to speed up future requests.
    

# **Topology & Addressing**

### **Network equipment**

| **Equipment** | **Management IP** |
| --- | --- |
| SA1 | 172.20.1.1 |
| SA2 | 172.20.1.2 |
| MLS1 | 172.20.1.254 |
| MLS2 | 172.20.1.253 |
| FW1 | 10.20.1.254 |
| FW2 | 10.20.3.254 |
| R1 | 10.20.2.254 |
| R2 | 10.20.4.254 |

### **Servers**

| **Server** | **Network** | **Machine** | **IP Address** | **Gateway** |
| --- | --- | --- | --- | --- |
| pve1 | WAN_MLS1 | PF1 | 10.20.5.1/24 | 10.20.5.254 |
| pve1 | WAN_MLS1 | pve1 | 10.20.5.253/24 | 10.20.5.254 |
| pve1 | LAN_LIN1 | NS1 | 172.20.21.53/24 | 172.20.21.254 |
| pve1 | LAN_LIN1 | DHCP1 | 172.20.21.67/24 | 172.20.21.254 |
| pve1 | LAN_LIN1 | CA1 | 172.20.21.90/24 | 172.20.21.254 |
| pve1 | LAN_LIN1 | TFTP1 | 172.20.21.69/24 | 172.20.21.254 |
| pve1 | LAN_LIN1 | LOG1 | 172.20.21.14/24 | 172.20.21.254 |
| pve1 | LAN_WIN1 | DC1 | 172.20.20.1/24 | 172.20.20.254 |
| pve1 | LAN_WIN1 | FS1 | 172.20.20.2/24 | 172.20.20.254 |
| pve1 | LAN_BRWS1 | browser | 172.20.30.1/24 | 172.20.30.254 |
| pve2 | WAN_MLS2 | PF2 | 10.20.8.1/24 | 10.20.8.254 |
| pve2 | WAN_MLS2 | pve2 | 10.20.8.253/24 | 10.20.8.254 |
| pve2 | LAN_LIN2 | NS2 | 172.20.23.53/24 | 172.20.23.254 |
| pve2 | LAN_LIN2 | DHCP2 | 172.20.23.67/24 | 172.20.23.254 |
| pve2 | LAN_LIN2 | CA2 | 172.20.23.90/24 | 172.20.23.254 |
| pve2 | LAN_WIN2 | DC2 | 172.20.22.1/24 | 172.20.22.254 |
| pve2 | LAN_WIN2 | FS2 | 172.20.22.2/24 | 172.20.22.254 |
| web | DMZ | web | 172.20.3.80/24 | 172.20.3.254 |
| bastion | BASTION | bastion | 172.20.4.1/24 | 172.20.4.254 |

# **Configuring BIND**

### Installing BIND

```bash
sudo dnf install bind bind-utils -y
```

### **Creation of files**

We will create a folder for the logs and for the zone files:

```bash
sudo mkdir -p /var/named/{log,zones}
chown named:named /var/named/zones/
sudo chmod 750 /var/named/zones
```

### **Definition of authorized networks (ACL)**

In the main configuration file:`/etc/named.conf`add this:

```bash
acl internal-net {
        localhost;
        172.20.1.0/24;   # VLAN MGMT
        172.20.10.0/24;  # VLAN USER
        172.20.4.0/24;   # LAN BASTION
        172.20.20.0/24;  # PVE1 - Windows
        172.20.21.0/24;  # PVE1 - Linux
        172.20.30.0/24;  # Browser
        172.20.22.0/24;  # PVE2 - Windows
        172.20.23.0/24;  # PVE2 - Linux
        10.20.0.0/16;    # Material
};
```

### **Global options**

In the same file`/etc/named.conf` edit this:

```bash
options {
    listen-on port 53 { localhost; 172.20.21.53; };
    listen-on-v6 port 53 { ::1; };

    directory "/var/named";

    allow-query { internal-net; };
    allow-recursion { internal-net; };

    # Configures the DNS servers to which BIND should redirect queries.
    forwarders { 195.238.2.21; 195.238.2.22; };
    # If the forwarders do not respond, bind will attempt recursive resolution.
};
```

### **Logs configuration**

Always in the same file `/etc/named.conf` add this:

```bash
logging {
    category notify { zone_transfer_log; };
    category xfer-in { zone_transfer_log; };
    category xfer-out { zone_transfer_log; };
		category queries { query_log; };
		
    channel zone_transfer_log {
        file "/var/named/log/transfer.log" versions 10 size 50M;
        print-time yes;
        print-category yes;
        print-severity yes;
        severity info;
    };
    channel query_log {
        file "/var/named/log/query.log" versions 5 size 10M;
			  print-time yes;
        print-category yes;
        print-severity yes;
        severity info;
		};
};
```

**Assign the correct permissions**

```bash
sudo chown named:named /var/named/log
sudo chmod 700 /var/named/log
```

### Add a rule to the firewall

Authorize the`port 53`on Rocky Linux:

```bash
sudo firewall-cmd --add-service=dns --permanent
sudo firewall-cmd --reload

```

### **Configuration check**

```bash
sudo named-checkconf
```

If the command returns no results, the syntax of the main configuration file is correct.

### **Enabling DNS service**

```bash
sudo systemctl enable --now named
sudo systemctl status named
```

# Creation of zones

### **Main zone declaration**

In the `/etc/named.conf` file add this:

```bash
zone "infra.labo" IN {
        type master;
        file "zones/infra.labo.zone";
        allow-query { internal-net; };
        allow-transfer { none; };
        allow-update { none; };
};
```

### Configuring a zone file

Then create a file`/var/named/zones/infra.labo.zone`and add this:

```bash
$TTL 8h
@                       IN      SOA     ns1.infra.labo.         admin.infra.labo. (
                                2025112301      ; Serial
                                3h              ; Refresh
                                1h              ; Retry
                                1w              ; Expire
                                1h )            ; Minimum TTL

@                       IN      NS      ns1.infra.labo.

; --- Server for infra.labo ---
ns1                     IN      A       172.20.21.53
ns2                     IN      A       172.20.23.53

dhcp1                   IN      A       172.20.21.67
dhcp2                   IN      A       172.20.23.67

log1                    IN      A       172.20.21.14
tftp1                   IN      A       172.20.21.69

browser                 IN      A       172.20.30.1
web                     IN      A       172.20.3.80
bastion                 IN      A       172.20.4.1

; --- Network devices for infra.labo —

pve1                    IN      A       10.20.5.253
pve2                    IN      A       10.20.8.253

pf1                     IN      A       10.20.5.1
pf2                     IN      A       10.20.8.1
sa1                     IN      A       172.20.1.1
sa2                     IN      A       172.20.1.2

mls1                    IN      A       172.20.1.254
mls2                    IN      A       172.20.1.253

fw1                     IN      A       10.20.1.254
fw2                     IN      A       10.20.3.254

r1                      IN      A       10.20.2.254
r2                      IN      A       10.20.4.254

; --- Website for infra.labo ---

site1                   IN      A       172.20.3.80
www.site1               IN      CNAME   site1

site2                   IN      A       172.20.3.80
www.site2               IN      CNAME   site2
```

### **Assign the correct permissions**

```bash
sudo chown root:named /var/named/zones/*.zone
sudo chmod 640 /var/named/zones/*.zone
```

### **Assign the correct SELinux context**

```bash
sudo semanage fcontext -a -t named_zone_t "/var/named/zones(/.*)?"
sudo restorecon -Rv /var/named/zones
```

### **Syntax check of the zone**

```bash
sudo named-checkzone infra.labo /var/named/zones/infra.labo.zone
```

# Creating reverse zones

| **Network** | **Areas** | **Affected machines** |
| --- | --- | --- |
| 172.20.1.0/24 | 1.20.172.in-addr.arpa | MLS1, MLS2, SA1, SA2 |
| 172.20.3.0/24 | 3.20.172.in-addr.arpa | web |
| 172.20.4.0/24 | 4.20.172.in-addr.arpa | bastion |
| 172.20.21.0/24 | 21.20.172.in-addr.arpa | NS1, DHCP1, CA1, SCP1, LOG1 |
| 172.20.23.0/24 | 23.20.172.in-addr.arpa | NS2, DHCP2, CA2 |
| 172.20.30.0/24 | 30.20.172.in-addr.arpa | browser |
| 10.20.1.0/24 | 1.20.10.in-addr.arpa | MLS1, FW1 |
| 10.20.2.0/24 | 2.20.10.in-addr.arpa | R1, FW1 |
| 10.20.3.0/24 | 3.20.10.in-addr.arpa | MLS2, FW2 |
| 10.20.4.0/24 | 4.20.10.in-addr.arpa | R2, FW2 |
| 10.20.5.0/24 | 5.20.10.in-addr.arpa | MLS1, PVE1, PF1 |
| 10.20.8.0/24 | 8.20.10.in-addr.arpa | MLS2, PVE2, PF2 |
| 10.20.10.0/24 | 10.20.10.in-addr.arpa | MLS1, MLS2 |

### **Reverse zone declarations**

For each reverse zone, add this to the file `/etc/named.conf` :

```bash
zone "1.20.172.in-addr.arpa" IN {
        type master;
        file "zones/1.20.172.in-addr.arpa.zone";
        allow-query { internal-net; };
        allow-transfer { none; };
        allow-update { none; };
};
```

### **Creation of reverse zone files**

For each reverse  zone, add a file in `/var/named/zones`

Example :`1.20.172.in-addr.arpa.zone`

```bash
$TTL 8h
@                       IN      SOA     ns1.infra.labo.         admin.infra.labo. (
                                2025112301      ; Serial
                                3h              ; Refresh
                                1h              ; Retry
                                1w              ; Expire
                                1h )            ; Minimum TTL

@                       IN      NS      ns1.infra.labo.

1                       IN      PTR     sa1.infra.labo.
2                       IN      PTR     sa2.infra.labo.
254                     IN      PTR     mls1.infra.labo.
253                     IN      PTR     mls2.infra.labo.
```

### **Assign the correct permissions**

```bash
sudo chown root:named /var/named/zones/*.zone
sudo chmod 640 /var/named/zones/*.zone
```

### **Syntax check of the zone**

```bash
sudo named-checkzone 1.20.172.in-addr.arpa \\
  /var/named/zones/1.20.172.in-addr.arpa.zone
```

# **DNS delegation to Active Directory**

In our infrastructure, the BIND servers (DNS1 and DNS2) manage the main zone: `infra.labo`

The zone `ad.infra.labo` is managed by **Windows domain controllers** (DC1 and DC2).

### **Creation of a “forward” zone**

Add this to the file`/etc/named.conf` :

```bash
zone "ad.infra.labo" {
    type forward;
    forwarders { 172.20.20.1; 172.20.22.1; };
    forward only;
};
```

### **Delegation in parent zone**

The parent zone is `infra.labo` we need to add this in our zone file to allow delegation:

```bash
; --- Delegation for ad.infra.labo ---

ad.infra.labo.          IN      NS      dc1.ad.infra.labo.
ad.infra.labo.          IN      NS      dc2.ad.infra.labo.

dc1.ad.infra.labo.      IN      A       172.20.20.1
dc2.ad.infra.labo.      IN      A       172.20.22.1
```

### **On the active directory**

You must configure the DNS of the DCs to forward to DNS1 and DNS2

# **DNS redundancy**

### **Basic DNS2 configuration**

1. Do the same configuration as for the master server but without configuring the zones (except for delegation).
2. In`/etc/named.conf` assigned the DNS2 IP addresses:
    
    ```bash
    options {
        listen-on port 53 { localhost; 172.20.23.53; };
        …
    };
    ```
    

### **Generate the TSIG key on the MASTER**

On DNS1 (`172.20.21.53`) :

```bash
tsig-keygen -a hmac-sha256 transfer-key > /etc/named/tsig.key
```

Copy key to DNS2:

```bash
sudo scp /etc/named/tsig.key admin@172.20.23.53:/etc/named/
```

### **Include the TSIG key on both servers**

In `/etc/named.conf` add :

```bash
include "/etc/named/tsig.key";
```

### **Configure permissions on both servers**

```bash
sudo chown root:named /etc/named/tsig.key
sudo chmod 640 /etc/named/tsig.key
sudo restorecon -RF /etc/named
```

### **Configure MASTER zones on DNS1**

For **each zone** (except delegation), add this:

```bash
zone "infra.labo" IN {
    type master;
    file "zones/infra.labo.zone";
    allow-query { internal-net; };
    **allow-transfer { key transfer-key; };**
    allow-update { none; };
    **notify yes;**
    **also-notify { 172.20.23.53 key transfer-key; };**
};
```

### **Configure SLAVE zones on DNS2**

For **each zone** (except delegation), put this in `/etc/named.conf` :

```bash
zone "infra.labo" IN {
    type slave;
    masters {
        172.20.21.53 key transfer-key;
    };
    file "slaves/infra.labo.zone";
};
```

# Verification

### Check configuration

On both servers do:

```bash
sudo named-checkconf
```

If no errors are returned, it is good and we can restart the service

```bash
sudo systemctl restart named
```

### Check name resolution

From the BIND server, check that it resolves the domain names

```bash
dig +short @localhost sa1.infra.labo
dig +short @localhost pf1.infra.labo
```

From another machine, configure the BIND server IP as DNS. Then check that it resolves the domain names correctly

```bash
dig +short @172.20.21.53 sa1.infra.labo
dig +short @172.20.23.53 pf1.infra.lab
```

### Check reverse resolution

From the BIND server, check that it resolves the IP addresses correctly

```bash
dig +short @localhost -x 172.20.1.1
dig +short @localhost -x 10.20.5.253
```

From another machine, configure the BIND server IP as DNS. Then check that it resolves the IP addresses correctly

```bash
dig +short @172.20.21.53 -x 172.20.1.1
dig +short @172.20.23.53 -x 10.20.5.253
```

### Check delegation

```bash
dig +short @localhost ad.infra.labo NS
dig +short @localhost pc1.ad.infra.labo
```

### **Verification of the transfer from the SLAVE**

```bash
dig @172.20.21.53 infra.labo AXFR
```

The files should appear:

```bash
ls -l /var/named/slaves/
```

### **Force a transfer (optional)**

On DNS1:

```bash
rndc reload
rndc notify infra.labo
```

Verification on DNS2:

```bash
sudo journalctl -u named -f
```
