# Etherchannel

```arduino

MLS1(config-if-range)#duplex full
MLS1(config)#int range g1/0/1-2
MLS1(config-if-range)#channel-group 1 mode passive
MLS1 (conf) # int port-channel 1
MLS1(config-if-range)#sw mode trunk
MLS1(config-if-range)#sw trunk allowed vlan 11,21,31,50
MLS1(config-if-range)#exit
MLS1(config-if-range)#sw nonegotiate
MLS1(config-if-range)#lacp rate fast

```
