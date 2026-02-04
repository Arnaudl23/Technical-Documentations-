# Free Radius

```arduino
Sur les Switchs :

aaa new-model
radius-server host 172.20.17.133 auth-port 1812 acct-port 1813 
radius-server key pass
aaa authentication login default group radius local

```

```arduino
dnf install -y freeradius
dnf install -y freeradius-utils

cd /etc/raddb/certs
./bootstrap

cd /etc/raddb/

nano users
Arnaud    Cleartext-Password := "arnaud"

nano clients.conf

client 172.20.12.4 {
        secret = pass
}
client 172.20.12.3 {
        secret = pass
}
client 172.20.12.2 {
        secret = pass
}
systemctl enable --now radiusd
```

```arduino
Test sur le serveur :

radtest Arnaud arnaud localhost 0 testing123

```