# HTB-puppy

<img src="images/00-banner.png" width="600">

Difficulty: Medium

OS: Windows

IP: 10.129.232.75

Date: 17/05/2025

An initial nmap scan revealed the target was a Domain Controller. Using the provided credentials, we ran BloodHound and discovered we had GenericWrite privileges over the developers group. We added ourselves to this group, which gave us access to an SMB share containing a KeePass database file. We then cracked the KeePass password, logged into the database and recovered the plaintext password of the ant.edwards user.

Logging back into BloodHound, we discovered that ant.edwards had GenericAll privileges over adam.silver where We forced a password change before using an LDIF file to re-enable the account and subsequently log into the machine.

Next, we discovered a backup directory and downloaded a zip file which contained the password for steph.cooper.

Finally, while logged in as steph.cooper, we discovered DPAPI credentials were being stored on the machine. We transfered the encrypted password file and key to our local machine and used Impacket's DPAPI module to recover the planitext password of the steph.cooper_adm user before logging in and completing the machine.
