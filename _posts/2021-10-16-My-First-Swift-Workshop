---
layout: post
title: My First Swift Workshop
subtitle: A Pentesting Lab
thumbnail-img: /assets/img/SWIFT.png
share-img: /assets/img/SWIFT.jpg
tags: [SWIFT]
---
#### 10 minute read



## Introduction

It's been about three months into my time at SWIFT and jumping from knowing nothing to assisting the creation of a CTF(Capture The Flag) workshop was insane. A CTF, for those who don't know, is an activity where there are vulnerable services or machines. A player has to utilize these to their advantage and a flag is found at a checkpoint that is located at a point only accessible if they utilize the intended vulnerability. Note that this is a type of CTF oriented around our presentation topic this week, which was Penetration Testing; other CTF's relating to computer stuff might be similar to finding small, specific details, such as finding image metadata or cracking a hash. A CTF event usually takes quite a bit of effort to set up, especially with the effort that our Vice President of Operations, Alex, wanted us to put into this. As a workshop is apart of our weekly meetings, we usually have a short bit of time to set them up, and factoring in the effort that Alex had wanted us to put in, this was an incredibly difficult undertaking. Each member of our team (Alex, Justin, Luis, and me), along with Robinson, Evan, Jacob, and Taylor, had to put in an incredible amount of work in this short amount of time, resulting in something beautiful. While I haven't seen much from SWIFT as I'm relatively new, considering the time we had allotted along with the result and turnout, I'd say that we did something amazing for "just a weekly workshop".



## Setting up the CTF

The infrastructure for the CTF was held on vSphere, a cloud computer virtualization platform. We had a Kali machine set up, along with three machines representing a level, with the higher levels corresponding to having harder flags. The first machine was a Linux machine, the second a Windows, and the last one was also a Linux machine. We cloned the machines, along with the Kali one, four times, one for each breakout room that we would host during the actual meeting. To actually connect to the machines a player had to ssh into our jumpbox which was open to the public, and from there they could ssh into the actual machine. From there we had the three machines available to each group. Players would use their Kali machine to scan the box that they were on, find vulnerabilities, exploit them to gain a foothold, and then escalate privileges from other vulnerabilities we left on the system. Once they rooted a machine they could find information to help them get started on the next box. At each stage of the penetration process, there would be a flag awarded that they could submit to our CTFd website (a Capture The Flag framework), and a scoreboard would show who was in the lead.

<img src="https://github.com/susMdT/Nigerald/blob/master/assets/images/CTF_Score.PNG?raw=true" width="100%" height="100%" unselectable="on" />
<p class="subtitle"> The scoreboard </p>

## The Vulnerabilities

Each level had four flags (except the last level) that were awarded for successfully completing a stage of a penetration test on the current machine the player was on. These flags were: Reconnaissance, Exploitation, Privilege Escalation, Lateral Movement.
  
<img src="https://github.com/susMdT/Nigerald/blob/master/assets/images/CTF_Challenges.PNG?raw=true" width="100%" height="100%" unselectable="on" />
<p class="subtitle"> The challenges </p>


| Level 1:              | Linux |
| -------------------- | ------------------------------------------------------------ |
| Reconnaissance       | An nmap scan would reveal that the machine was running vsftpd 2.3.4, and anonymous ftp was enabled |
| Exploitation         | Using metasploit or using exploit 49757 on Exploit-DB, the player would gain a foothold through exploiting vsftpd 2.3.4 to gain a bind shell |
| Privilege Escalation | Reading /etc/crontabs would show that an automated task was running a script as root, and that by checking the permissions on the script, the player would find that they could modify it to run any command on root. |
| Lateral Movement     | Now as root, if the player enumerated further, they would find credentials for an MsSql user along with the lateral movement flag. |

| Level 2:             | Windows |
| -------------------- | ------------------------------------------------------------ |
| Reconaissance        | An nmap scan would reveal a MsSql server open along with an SMB server. Enumerating further, the player would find out that the guest account was enabled in SMB and that they could find the recon flag in one of the shares, along with a note that leaves a hint about xp_cmdshell being an attack vector since the creds found in the previous box could be used to access a sysadmin account. |
| Exploitation         | Using the information from recon, the player could use Metasploit (exploit/windows/mssql/mssql_payload) along with the MsSql creds to gain a reverse shell |
| Privilege Escalation | The flag hint on the CTFd website mentioned the service "Development_Test" having an unquoted service path vulnerability. Creating a payload in msfvenom, placing it in root directory as "New.exe" and then starting the service would give the user a reverse shell as System. |
| Lateral Movement     | The player had to upload Mimikatz from their Kali machine and dump hashes with it to gain the last flag. The password hashes weren't meant to be cracked nor were they used to authenticate to the third machine |

| Level 3:               | Linux |
| ---------------------- | ------------------------------------------------------------ |
| Reconnaissance         | An nmap scan would reveal that the machine was running an apache web server, and curling it would show the credentials to the manager in the documentation. |
| Exploitation           | The player could manually craft a POST request through curl to upload a reverse shell through the manager page, or use Metasploit (exploit/multi/http/tomcat_mgr_upload) to do this for them. If the player sent a POST request through curl, they would then have to access the location of the upload to active the shell. |
| Privilege Escalation   | After enumerating the machine, the player would notice that there was an executable script that they could run as another user, and that this file had a path injection vulnerability. By modifying their path variable and creating a malicious file to be executed, the player would gain access to a more privileged user. |
| Privilege Escalation 2 | The new user was in the lxd group, and by uploading an image, initializing it with privileges, and mounting it on the root file system, the player would gain root. |

## Backend Challenges

There were two main challenges that I faced personally when contributing to this workshop. The first was our original privilege escalation vulnerability for the Windows box. We had intended to use AlwaysInstallElevated as the privilege escalation vector, but it would not seem to work at all due to our usage of xp_cmdshell. Since the initial foothold was made through xp_cmdshell, attempting to install an msi file to abuse this vulnerability would result in the error "Client-side and UI is none or basic: Running entire install on the server.". Even trying to circumvent this through spawning a reversehell through xp_cmdshell did not change this. This was incredibly frustrating and if it wasn't for Brice's idea of using unquoted service path, along with making a vulnerable service himself, I'm not sure how we would have made the Windows box work.

The second challenge was an unexpected one that came from running a multiple player environment. Since the CTF was essentially set up as four machines to be shared by four people, we ran into some issues. Most notably, in the first level, the bind shell created by the vsftpd 2.3.4 exploit set up a listener on the victim on port 6200. Since a port can't be used by multiple services or processes, only one person was able to exploit the first box at a time. A player would have to wait for another person to completely root it before being able to exploit the box. Every time someone finished rooting the box, I had to remember to kill the bind shell process since it didn't go away even after they killed their shell, which was weird. 
<img src="https://github.com/susMdT/Nigerald/blob/master/assets/images/CTF_IR.PNG?raw=true" width="100%" height="100%" unselectable="on" />
<p class="subtitle"> My attempts at "IR" via killing the hanging bind shell sessions </p>


## Final  Thoughts

This was a very stressful and fun experience. The team and I placed a lot of effort into this event, and we had a very short time to do so (just about two days). There were very few hours of sleep made in that second half of the week. However, the event had a much larger turnout that usual workshops, and I had a great time guiding people through the boxes. Some people even stayed very late after hours to try to get as many flags as possible, and it was very satisfying seeing that amount of passion coming from people who were just as new as I was to all this stuff three months ago. I want to thank Jacob and Taylor for figuring out the networking magic so that the infrastructure was actually functional, and to Brice for saving our Windows box. I also want to thank everyone else who spent countless hours on these machines to make the event a success. Thanks Robinson, Luis, Alex, and Justin for all the help. It was really cool to be a part of all of this.

