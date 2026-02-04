# HSRP/IPSLA

```arduino

standby version 2
standby 11 ip 172.20.11.10
standby 11 priority 110
standby 11 preempt

encapsulation dot1q 21
ip address 172.20.21.11 255.255.255.0
standby version 2
standby 21 ip 172.20.21.10
standby 21 priority 100
standby 21 preempt

ip address 172.20.31.12 255.255.255.0
standby version 2
standby 31 ip 172.20.31.10
standby 31 priority 90

ip address 100.64.1.1 255.192.0.0
ip route 0.0.0.0 0.0.0.0 g0/1 100.64.1.254

ip sla 11
icmp-echo 10.255.255.1
frequency 5
ip sla schedule 11 life forever start-time now
track 11 ip sla 11
exit
int g0/0.11
standby 11 track 11 decrement 20

ip sla 21
icmp-echo 10.255.255.1
frequency 5
ip sla schedule 21 life forever start-time now
track 21 ip sla 21
exit
int g0/0.21
standby 21 track 21 decrement 20

ip sla 31
icmp-echo 10.255.255.1
frequency 5
ip sla schedule 31 life forever start-time now
track 31 ip sla 31
exit
int g0/0.31
standby 31 track 31 decrement 20

```
