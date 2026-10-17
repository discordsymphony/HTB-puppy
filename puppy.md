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

In Bloodhound, we perform an analysis of Group Delegated Object Control, which reveals the following graph:

<img src="images/01-hr-developers.png" width="500">


