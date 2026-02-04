# NTP

```arduino

R1(config)#ntp master 3
R1(config)#clock timezone utc +1
R1(config)#clock summer-time CEST recurring last Sun Mar 2:00 last Sun Oct 3:00
R1#clock set 13:35:00 june 02 2025

S1A(config)#ntp server 172.20.12.1
S1A(config)#clock timezone utc +1
S1A(config)#clock summer-time CEST recurring last Sun Mar 2:00 last Sun Oct 3:00
S1A(config)#service timestamps debug datetime ms localtime
S1A(config)#service timestamps log datetime ms localtime

S1B(config)#ntp server 172.20.12.1
S1B(config)#clock timezone utc +1
S1B(config)#clock summer-time CEST recurring last Sun Mar 2:00 last Sun Oct 3:00
S1B(config)#service timestamps debug datetime ms localtime
S1B(config)#service timestamps log datetime ms localtime

MLS1(config)#ntp server 172.20.12.1
MLS1(config)#clock timezone utc +1
MLS1(config)#clock summer-time CEST recurring last Sun Mar 2:00 last Sun Oct 3:00

```
