# PFSense

# Firewall rules

[PFSense Rules ](https://www.notion.so/PFSense-Rules-2be6c6954d7880599517f4761f9c9d4a?pvs=21)

# Syslog

We want to send the pfSense firewalls log  to our log forwarder (LOG1).

### Syslog configuration

Go to `Status / System Logs / Settings` and configure this in the `Remote Logging Options` ****section :

| Settings | Option |
| --- | --- |
| Enable Remote Logging | Yes |
| Source Address | LAN_LIN1 for PF1
WAN_MLS2 for PF2 |
| IP Protocol | IPv4 |
| Remote log servers | 172.20.21.14:514 |
| Remote Syslog Contents | System Events, Firewall Events, DNS Events, DHCP Events, Captive Portal Events, Gateway Monitor Events, NTP Events |