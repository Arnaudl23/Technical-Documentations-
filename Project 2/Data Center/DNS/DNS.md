# DNS

```arduino

IP du serveur DNS : 172.20.17.17

dnf install bind bind-utils -y
systemctl enable --now named
firewall-cmd --permanent --add-service=dns
firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
firewall-cmd --reload
sudo firewall-cmd --list-all
```

```arduino
⚙️ Configuration de BIND

nano etc/named.conf

//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
        listen-on port 53 { 127.0.0.1;172.20.17.17;172.20.11.0;172.20.12.0;172.20.13.0;172.20.16.0;172.20.17.0; };
        listen-on-v6 port 53 { none; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { localhost; 172.20.11.0; 172.20.12.0; 172.20.13.0; 172.20.16.0; 172.20.17.0; };
        forwarders      { 8.8.8.8; };
        forward only;

        /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable
           recursion.
         - If your recursive DNS server has a public IP address, you MUST enable access
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface
        */
        recursion yes;

        dnssec-validation yes;

        managed-keys-directory "/var/named/dynamic";
        geoip-directory "/usr/share/GeoIP";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";

        /* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
        include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";

//forward zone
zone "cqa.lan" IN{
        type master;
        file  "cqa.lan.db";
        allow-update { none; };
        allow-query {any; };
};
//reverse zone
zone "17.20.172.in-addr.arpa" IN {
        type master;
        file "cqa.lan.rev";
        allow-update { none; };
        allow-query { any; };
};

```

```arduino
🌐 Configuration de la Zone directe du DNS

nano /var/named/cqa.lan.db

$TTL 86400
@       IN SOA localhost.cqa.lan. ar.cqa.lan. (
                                        2019061800      ; serial
                                        3600    ; refresh
                                        1800    ; retry
                                        604800  ; expire
                                        86400   ; minimum
)

;Name Server Information
@ IN NS localhost.cqa.lan.

;IP for Name Server
localhost IN A 172.20.17.17

;A Record for IP address to Hostname
wp IN A 172.20.17.16
www IN A 172.20.17.16
www.wp  IN A 172.20.17.16
```

```arduino
🌐 Configuration de la Zone inverse du DNS

nano /var/named/cqa.lan.rev

$TTL 86400
@       IN SOA localhost.cqa.lan. ar.cqa.lan. (
                                        2019061800      ; serial
                                        3600    ; refresh
                                        1800    ; retry
                                        604800  ; expire
                                        86400   ; minimum
)

;Name Server Information
@ IN NS localhost.cqa.lan.

;Reverse lookup for Name Server
17 IN PTR localhost.cqa.lan.

;PTR Record for IP address to Hostname
16 IN PTR www.wp.cqa.lan.
```
