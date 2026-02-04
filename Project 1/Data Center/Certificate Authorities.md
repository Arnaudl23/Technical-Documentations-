# Certificate Authorities

<aside>
🎯

Set up a complete **internal PKI infrastructure** based on an offline Root CA and an online intermediate CA under Rocky Linux, in order to sign and manage internal TLS certificates for all services in the domain `infra.labo`.

</aside>

# Introduction

A PKI (*Public Key Infrastructure*) allows you to manage digital certificates used to secure internal communications via TLS/SSL.

In an enterprise environment, it is recommended to **separate the Root CA** from the **Intermediate CA**:

- The **Root CA** is **offline**, protected and used only to sign the intermediate CA.
- The **intermediate CA** signs the certificates of the internal servers (Teleport, Zabbix, etc.) and remains **active in production**.

# Prerequisites

### Have an up-to-date Rocky Linux server

```bash
sudo dnf update -y
```

### Check NTP synchronization

```bash
timedatectl
chronyc tracking
```

# Creation of the Root CA

This part must be carried out on an **isolated machine**. The Root CA should **never be connected to the network**.

### Create the folder structure

```bash
sudo mkdir -p /etc/pki/CA/root/{certs,crl,newcerts,private}
sudo touch /etc/pki/CA/root/index.txt
echo 1000 | sudo tee /etc/pki/CA/root/serial
```

### Assign permissions

```bash
sudo chmod 700 /etc/pki/CA/root/private
sudo chown -R admin:admin /etc/pki/CA/root
sudo chmod 700 /etc/pki/CA/root
```

### Configuration d’OpenSSL

Create a backup of the file `/etc/pki/tls/openssl.cnf` :

```bash
sudo cp /etc/pki/tls/openssl.cnf ~/openssl.cnf.bkp
```

Then edit the file `/etc/pki/tls/openssl.cnf` like this:

```bash
[ CA_default ]
dir               = /etc/pki/CA/root
certs             = $dir/certs
crl_dir           = $dir/crl
database          = $dir/index.txt
new_certs_dir     = $dir/newcerts
certificate       = $dir/certs/root.crt
serial            = $dir/serial
crlnumber         = $dir/crlnumber
crl               = $dir/crl/root.crl
private_key       = $dir/private/root.key

# Extensions in the CSR (e.g., SAN) are not copied into the final certificate.
copy_extensions = none

# Certificate validity period
default_days      = 3650
# Validity period of the CRL
default_crl_days  = 30

# Default hash algorithm to use
default_md        = sha512
preserve          = no

# Defines the security policy to use (see below)
policy            = policy_strict

[ policy_strict ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

# To create a certificate signing request (CSR) or a self-signed certificate
[ req ]
default_bits        = 4096
default_md          = sha512
distinguished_name  = req_distinguished_name
x509_extensions     = v3_ca

string_mask         = utf8only

[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
countryName_default             = BE

stateOrProvinceName             = State or Province Name
stateOrProvinceName_default     = Wallonie

localityName                    = Locality Name (eg, city)
localityName_default            = Ciney

0.organizationName              = Organization Name (eg, company)
0.organizationName_default      = Infra

organizationalUnitName          = Organizational Unit Name
organizationalUnitName_default  = Infra-CA1

[ v3_ca ]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, cRLSign, keyCertSign

[ v3_intermediate_ca ]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer
# This CA will not be able to sign other CAs
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, cRLSign, keyCertSign
```

### Generation of the private key

```bash
sudo openssl genrsa -aes256 -out /etc/pki/CA/root/private/root.key 4096
Enter PEM Passphrase : ************

sudo chmod 400 /etc/pki/CA/root/private/root.key
```

### Creating the self-signed root certificate

```bash
sudo openssl req -x509 -extensions v3_ca -new \
    -key /etc/pki/CA/root/private/root.key \
    -out /etc/pki/CA/root/certs/root.crt
```

```bash
Enter pass phrase for /etc/pki/CA/root/private/root.key: ************
-----
Country Name (2 letter code) [BE]:
State or Province Name (full name) [Wallonie]:
Locality Name (eg, city) [Ciney]:
Organization Name (eg, company) [Infra]:
Organizational Unit Name (eg, section) [Infra-CA1]:
Common Name (eg, your name or your server's hostname) []:CA1
Email Address []:
```

### Certificate verification

```bash
sudo openssl x509 -noout -text -in /etc/pki/CA/root/certs/root.crt
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            5f:b4:d2:8a:2b:ca:15:4e:47:23:ae:9d:89:51:8f:2f:68:3b:f5:01
        Signature Algorithm: sha512WithRSAEncryption
        Issuer: C=BE, ST=Wallonie, L=Ciney, O=Sam, OU=Sam-CA-Root, CN=CA-Root
        Validity
            Not Before: Nov  8 12:31:45 2025 GMT
            Not After : Dec  8 12:31:45 2025 GMT
        Subject: C=BE, ST=Wallonie, L=Ciney, O=Sam, OU=Sam-CA-Root, CN=CA-Root
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:a7:1a:e6:e1:d4:ed:3d:a8:b8:54:65:2f:27:d6:
                    ...
                    2c:31:0d
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                B9:EB:88:C4:55:E1:F9:30:90:A8:C0:C6:FD:4B:07:62:2B:E8:29:31
            X509v3 Authority Key Identifier:
                B9:EB:88:C4:55:E1:F9:30:90:A8:C0:C6:FD:4B:07:62:2B:E8:29:31
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
    Signature Algorithm: sha512WithRSAEncryption
    Signature Value:
        27:76:9a:97:7d:56:50:8e:97:9e:ce:62:a2:98:3e:13:74:79:
        ...
        e0:28:9c:5c:10:17:58:08
```

# Creation of the intermediate CA

### Create the folder structure

```bash
sudo mkdir -p /etc/pki/CA/intermediate/{certs,crl,csr,newcerts,private}
sudo touch /etc/pki/CA/intermediate/index.txt
echo 1000 | sudo tee /etc/pki/CA/intermediate/serial
echo 1000 | sudo tee /etc/pki/CA/intermediate/crlnumber
```

### Assign permissions

```bash
sudo chmod 700 /etc/pki/CA/intermediate/private
sudo chown -R admin:admin /etc/pki/CA/intermediate
sudo chmod 700 /etc/pki/CA/intermediate
```

### Configuration d’OpenSSL

Create a backup of the file `/etc/pki/tls/openssl.cnf` :

```bash
sudo cp /etc/pki/tls/openssl.cnf ~/openssl.cnf.bkp
```

Then edit the file `/etc/pki/tls/openssl.cnf` like this:

```bash
[ CA_default ]
dir               = /etc/pki/CA/intermediate
certs             = $dir/certs
crl_dir           = $dir/crl
database          = $dir/index.txt

new_certs_dir     = $dir/newcerts

certificate       = $dir/certs/intermediate.crt
serial            = $dir/serial
crlnumber         = $dir/crlnumber

crl               = $dir/crl/intermediate.crl
private_key       = $dir/private/intermediate.key

# Extensions in the CSR (e.g., SAN) are copied into the final certificate.
copy_extensions = copyall

# Certificate validity period
default_days      = 3650
# Validity period of the CRL
default_crl_days  = 30

# Default hash algorithm to use
default_md        = sha512
preserve          = no

# Define the security policy to be used (see below)
policy            = policy_loose

[ policy_loose ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

# To create a certificate signing request (CSR) or a self-signed certificate
[ req ]
default_bits        = 4096
default_md          = sha512
distinguished_name  = req_distinguished_name
x509_extensions     = v3_ca

# For greater compatibility
string_mask         = utf8only

[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
countryName_default             = BE

stateOrProvinceName             = State or Province Name
stateOrProvinceName_default     = Wallonie

localityName                    = Locality Name (eg, city)
localityName_default            = Ciney

0.organizationName              = Organization Name (eg, company)
0.organizationName_default      = Infra

organizationalUnitName          = Organizational Unit Name
organizationalUnitName_default  = Infra-CA2

[ v3_ca ]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer
# This CA will not be able to sign other CAs.
basicConstraints = critical, CA:true, pathlen:0 
keyUsage = critical, cRLSign, keyCertSign

[ usr_cert ]
basicConstraints=CA:FALSE
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection

[ server_cert ]
basicConstraints=CA:FALSE
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
```

### Generation of the private key

```bash
sudo openssl genrsa -aes256 -out /etc/pki/CA/intermediate/private/intermediate.key 4096
Enter PEM Passphrase : ************

sudo chmod 400 /etc/pki/CA/intermediate/private/intermediate.key
```

### Creation of the CSR (Certificate Signing Request)

```bash
sudo openssl req -new \
    -key /etc/pki/CA/intermediate/private/intermediate.key \
    -out /etc/pki/CA/intermediate/csr/intermediate.csr
```

```bash
Enter pass phrase for /etc/pki/CA/intermediate/private/intermediate.key: 
************
-----
Country Name (2 letter code) [BE]:
State or Province Name (full name) [Wallonie]:
Locality Name (eg, city) [Ciney]:
Organization Name (eg, company) [Infra]:
Organizational Unit Name (eg, section) [Infra-CA2]:
Common Name (eg, your name or your server's hostname) []:CA2
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

### CSR verification

```bash
cat /etc/pki/CA/intermediate/csr/intermediate.csr
-----BEGIN CERTIFICATE REQUEST-----
...
-----END CERTIFICATE REQUEST-----
```

# Signature of the CSR (Intermediate) by the Root CA

## Copying the CSR to the Root CA

Normally we will do this via a secure USB key but in the context of the laboratory, we will transfer it via SCP.

```bash
scp /etc/pki/CA/intermediate/csr/intermediate.csr admin@172.20.21.90:/home/admin
```

### Generate the certificate

```bash
sudo openssl ca -extensions v3_intermediate_ca -notext\
    -in /home/admin/intermediate.csr \
    -out /etc/pki/CA/root/certs/intermediate.crt
```

```bash
Enter pass phrase for /etc/pki/CA/root/private/root.key: *********************
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 4096 (0x1000)
        Validity
            Not Before: Nov  8 14:02:15 2025 GMT
            Not After : Nov  6 14:02:15 2035 GMT
        Subject:
            countryName               = BE
            stateOrProvinceName       = Wallonie
            organizationName          = Sam
            organizationalUnitName    = Sam-CA-Intermediate
            commonName                = CA-Intermediate
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                8B:75:F2:7B:F8:D3:DD:BD:76:2D:70:31:C4:36:88:52:47:5E:46:5E
            X509v3 Authority Key Identifier:
                7B:1A:DF:DB:3C:F6:22:3D:56:8C:BE:44:CD:90:22:A4:9C:C8:58:C9
            X509v3 Basic Constraints: critical
                CA:TRUE, pathlen:0
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
Certificate is to be certified until Nov  6 14:02:15 2035 GMT (3650 days)
Sign the certificate? [y/n]:y

1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Database updated
```

### Check certificate

```bash
sudo openssl x509 -noout -text -in /etc/pki/CA/root/certs/intermediate.crt
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4096 (0x1000)
        Signature Algorithm: sha512WithRSAEncryption
        Issuer: C=BE, ST=Wallonie, L=Ciney, O=Sam, OU=Sam-CA-Root, CN=CA-Root
        Validity
            Not Before: Nov  8 14:02:15 2025 GMT
            Not After : Nov  6 14:02:15 2035 GMT
        Subject: C=BE, ST=Wallonie, O=Sam, OU=Sam-CA-Intermediate, CN=CA-Intermediate
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:d0:36:4b:fa:0c:8b:3d:a0:cb:a7:42:45:fc:c4:
                    ...
                    b2:69:6f
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                8B:75:F2:7B:F8:D3:DD:BD:76:2D:70:31:C4:36:88:52:47:5E:46:5E
            X509v3 Authority Key Identifier:
                7B:1A:DF:DB:3C:F6:22:3D:56:8C:BE:44:CD:90:22:A4:9C:C8:58:C9
            X509v3 Basic Constraints: critical
                CA:TRUE, pathlen:0
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
    Signature Algorithm: sha512WithRSAEncryption
    Signature Value:
        85:91:ed:4e:e1:af:c1:2c:40:17:46:01:c8:ad:58:27:43:92:
        ...
        34:53:14:be:3b:c6:2f:4e
```

## Transfer of certificates to Intermediate CA

### Transfer of root certificate

```bash
scp /etc/pki/CA/root/certs/root.crt admin@172.20.23.90:/home/admin/
```

### Transfer of the intermediate certificate

```bash
scp /etc/pki/CA/root/certs/intermediate.crt admin@172.20.23.90:/home/admin/
```

# Trusted Chain Configuration

The `ca-chain.pem` file establishes the link between the certificates issued by the Intermediate CA and the Root CA.

For example, when a certificate is installed on a device (such as a Fortigate), it is signed by the Intermediate CA. However, a client (e.g., PC-HP) generally only knows the Root CA.

When a client initiates an HTTPS connection, Fortigate sends its certificate (`fortigate.crt`) and the chain of trust (`ca-chain.pem`).

This allows the client to verify that the Intermediate CA, which signed the Fortigate certificate, is itself validated by the Root CA, which it already trusts.

### Moving signed certificates

```bash
mv /home/admin/intermediate.crt /etc/pki/CA/intermediate/certs/
mv /home/admin/root.crt /etc/pki/CA/intermediate/certs/
```

### Creation of the Trusted Chain

```bash
cat /etc/pki/CA/intermediate/certs/intermediate.crt \
    /etc/pki/CA/intermediate/certs/root.crt \
    > /etc/pki/CA/intermediate/certs/ca-chain.crt
```

### Chain verification

```bash
openssl verify -CAfile /etc/pki/CA/intermediate/certs/ca-chain.crt \
  /etc/pki/CA/intermediate/certs/intermediate.crt

/etc/pki/CA/intermediate/certs/intermediate.crt: OK
```

# Signing a CSR issued by a server

## Introduction

We consider a *server* to be any machine that must present a certificate when providing a service: web server, administration interface (Proxmox, Fortigate, etc.).

A server must present to its clients:

1. Its **own certificate**, signed by the Intermediate CA.
2. The **chain of trust** (`ca-chain.pem`), which allows the client to go back to the Root CA, in which he already trusts.

Thus, the customer can validate the authenticity of the certificate presented:

### What a server needs

| Element | File | Role |
| --- | --- | --- |
| Certificate from CA Intermédiare | `intermediate.crt` | Allows the server to recognize the signature of its own certificate when returning the signed CSR. |
| Chain of Trust | `ca-chain.pem` | Used to prove to the client that the subordinate CA is itself validated by the root CA. The server transmits it to clients during TLS negotiation. |

### Importance of CSR (Distinguished Name) content

When generating the **CSR** (ex.`serveur.csr`) on the server, the *Distinguished Name* fields must be filled in (CN, Organization, Locality, etc.).

The section `policy` of the file `openssl.cnf` of the Intermediate CA defines which fields of the CSR must:

- match the fields of the subordinate CA certificate,
- simply be present,
- or can be ignored.

This guarantees identity consistency between the Intermediate CA and the certificates it issues, ensuring a controlled and controlled certification authority model.

## Signature of the CSR

```bash
openssl ca -extensions server_cert -notext \
  -in /etc/pki/CA/intermediate/csr/FW1.csr \
  -out /etc/pki/CA/intermediate/certs/FW1.crt
```

### Check certificate validity

```bash
openssl x509 -in /etc/pki/CA/intermediate/certs/FW1.crt -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4096 (0x1000)
        Signature Algorithm: sha512WithRSAEncryption
        Issuer: C=BE, ST=Wallonie, O=Sam, OU=Sam-CA-Intermediate, CN=CA-Intermediate
        Validity
            Not Before: Nov  8 15:16:06 2025 GMT
            Not After : Nov  6 15:16:06 2035 GMT
        Subject: C=BE, ST=Wallonie, L=Ciney, O=Sam, OU=FG-06, CN=10.6.0.253
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:ae:4c:a8:2f:ea:0f:ae:01:f2:e8:66:ca:a1:9d:
										...
                    c2:0b:77
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Subject Key Identifier:
                50:3A:32:D2:8C:EC:8A:32:3A:6A:48:0B:B7:B3:4D:C6:13:E4:08:2D
            X509v3 Authority Key Identifier:
                8B:75:F2:7B:F8:D3:DD:BD:76:2D:70:31:C4:36:88:52:47:5E:46:5E
            X509v3 Basic Constraints:
                CA:FALSE
            X509v3 Subject Alternative Name:
                DNS:fg-06.sam.lan, IP Address:172.20.68.254, IP Address:10.6.0.253
            X509v3 Key Usage:
                Digital Signature, Key Encipherment
    Signature Algorithm: sha512WithRSAEncryption
    Signature Value:
        67:53:2c:df:b1:a9:b6:78:44:fb:f6:e1:3e:34:66:de:03:d9:
	      ...
        87:14:d8:7d:02:c6:00:bd
```

# Installing the certificate on a client

## On Windows

### Install the certificate

1. Open file `root.crt`, which is the root certificate authority certificate
2. In the Assistant, choose ***Current User***
    
    ![image.png](image.png)
    
3. Then choose to place the certificate in ***Trusted Root Certification Authorities***
    
    ![image.png](image%201.png)
    

## On Rocky

### Install the certificate

1. Copy file `root.crt` on the machine and place it in the folder `/etc/pki/ca-trust/source/anchors/`
2. Afterwards :
    
    ```bash
    sudo update-ca-trust extract
    ```
    
3. Verify that the certificate is correctly installed
    
    ```bash
    sudo openssl verify /etc/pki/ca-trust/source/anchors/root.crt
    ```
    

# Creating a CSR for FortiGate

### Importing certificates

Go to ***System > Certificates*** and import the two CA certificates: `root.crt` and `intermediate.crt` and click on ***Create/Import > CA Certificate***

![image.png](image%202.png)

### Generate a CSR

Click on ***Create/Import > Generate CSR***

![image.png](image%203.png)

Fill out the form:

| Name | Description | Example |
| --- | --- | --- |
| Certificate Name | Name under which the certificate will appear in the fortigate | `FW1` |
| Subject Information | `IP` Or `FQDN`used to attach the web interface. | `fw1.infra.labo` |
| Optional information | These are the DNS of the certificates. According to the criteria of `policy`, they must be identical or not to those indicated on the Intermediate CA | `FW1` , `Infra`, `Wallonie` , `BE` |
| Subject Alternative Name | Allows you to enter several “aliases” to reach the FortiGate web interface. | `DNS:fw1.infra.labo,IP:10.20.1.254` |

Example :

![image.png](image%204.png)

### Download the CSR

In the list a local certificate should appear ***Pending***

![image.png](image%205.png)

Click on it then click on ***Download***

![image.png](image%206.png)

All that remains is to sign the CSR on the CA Intermediate

### Issue :

The certificate generated by the FortiGate is not in `UTF8`. Therefore, a problem may appear when trying to sign the certificate:

```bash
The stateOrProvinceName field is different between
CA certificate (Wallonie) and the request (Wallonie)
```

To correct this problem, you can temporarily change the `policy` of Intermediate CA by editing `/etc/pki/tls/openssl.cnf` :

```bash
[ CA_default ]
policy            = policy_anything

[ policy_anything ]
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
```

Once the certificate is signed, simply submit as before:

```bash
[ CA_default ]
policy          = policy_strict

[ policy_strict ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
```

# Creating a CSR for pfSense

### Importing certificates

Go to **System *> Certificates > Authorities*** and import the two CA certificates: `root.crt` and `intermediate.crt`

![image.png](image%207.png)

![image.png](image%208.png)

### Generate a CSR

Click on **System *> Certificates > Certificates***

![image.png](image%209.png)

Fill out the form:

| Name | Description | Example |
| --- | --- | --- |
| Descriptive Name | Name under which the certificate will appear in the fortigate | `pf.infra.labo` |
| Subject Information | `IP` or `FQDN` used to attach the web interface. | `pf.infra.labo` |
| External Signing Request | These are the DNS of the certificates. According to the criteria of `policy` , they must be identical or not to those indicated on the Intermediate CA | `pfSense` , `Infra`, `Wallonie` , `BE` |
| Alternative Name | Allows you to enter several “aliases” to reach the FortiGate web interface. | `IP address : 10.20.5.1` |

Example :

![image.png](image%2010.png)

![image.png](image%2011.png)

![image.png](image%2012.png)

### Download the CSR

Click on the arrow

![image.png](image%2013.png)

All that remains is to sign the CSR on the Intermediate CA.

![image.png](image%2014.png)

# Creation of a CSR for Proxmox

## Generate private key and CSR

### Create a folder to store keys and certificates

```bash
mkdir -p /etc/pve/local/ssl
```

### Generate the private key

```bash
openssl genrsa -out /etc/pve/local/ssl/pve1.key 4096
```

**Check :**

```bash
openssl rsa -in /etc/pve/local/ssl/pve1.key -noout -text
```

### Create the CSR

```bash
openssl req -new -sha512 -nodes \
	-key /etc/pve/local/ssl/pve1.key \
	-out /etc/pve/local/ssl/pve1.csr \
	-subj "/C=BE/ST=Wallonie/L=Ciney/O=Infra/OU=Infra-PVE1/CN=pve1.infra.labo" \
	-addext "subjectAltName=DNS:pve1.infra.labo,IP:10.20.5.253"
```

**Check :**

```bash
openssl req -in /etc/pve/local/ssl/pve1.csr -noout -text
```

## Sign the CSR

### Copy the CSR to the Intermediate CA

Copy file `proxmox.csr` in `/etc/pki/CA/intermediate/csr/`

### Sign the CSR

```bash
openssl ca -extensions server_cert -notext \
  -in /etc/pki/CA/intermediate/csr/pve1.csr \
  -out /etc/pki/CA/intermediate/certs/pve1.pem
```

**Check :**

```bash
openssl x509 -in /etc/pki/CA/intermediate/certs/proxmox.pem -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4098 (0x1002)
        Signature Algorithm: sha512WithRSAEncryption
        Issuer: C=BE, ST=Wallonie, O=Infra, OU=Infra-CA2, CN=CA-Intermediate
        Validity
            Not Before: Nov  8 17:00:33 2025 GMT
            Not After : Nov  6 17:00:33 2035 GMT
        Subject: C=BE, ST=Wallonie, L=Ciney, O=Infra, OU=Infra-PVE1, CN=pve1.infra.labo
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:b4:e4:2f:76:fd:b0:b1:be:f2:57:3a:62:a5:76:
                    ...
                    93:ef:8b
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            X509v3 Key Usage: critical
                Digital Signature, Non Repudiation, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Subject Key Identifier:
                C7:E4:9D:8F:51:43:D1:0F:27:F5:7C:33:8B:4E:30:7F:05:2A:1F:4E
            X509v3 Authority Key Identifier:
                8B:75:F2:7B:F8:D3:DD:BD:76:2D:70:31:C4:36:88:52:47:5E:46:5E
            X509v3 Subject Alternative Name:
                DNS:pve6.sam.lan, IP Address:10.60.0.253
    Signature Algorithm: sha512WithRSAEncryption
    Signature Value:
        2b:8d:a8:6c:47:39:b7:e3:00:77:57:f3:d0:2c:fa:f6:6f:d9:
        ...
        c8:d2:94:aa:78:3c:8c:80
```

### Create the chain of trust

From the Intermediate CA:

```bash
cat /etc/pki/CA/intermediate/certs/proxmox.pem \
    /etc/pki/CA/intermediate/certs/intermediate.crt \
    /etc/pki/CA/intermediate/certs/root.crt \
    > /home/samuel/proxmox-chain.pem
```

### Copy files to Proxmox

Copy `proxmox-chain.pem` in `/etc/pve/local/ssl/`

### Backup of old certificates

```bash
mkdir -p /etc/pve/local/bkp
mv /etc/pve/local/pve-ssl.key /etc/pve/local/bkp/pve-ssl.key.bkp
mv /etc/pve/local/pve-ssl.pem /etc/pve/local/bkp/pve-ssl.pem.bkp
```

### Copy of new certificates

```bash
cp /etc/pve/local/ssl/pve1.key /etc/pve/local/pve-ssl.key
cp /etc/pve/local/ssl/pve1-chain.pem /etc/pve/local/pve-ssl.pem
```

### Restart `pveproxy`

```bash
systemctl restart pveproxy
systemctl status pveproxy
```
