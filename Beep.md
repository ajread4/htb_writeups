# Beep

Machine Level: Easy <br />
OS: Linux/CentOS

## Scanning 
I started an aggressive NMAP scan to see what ports and services are open. 
```
ajread@aj-ubuntu:~/hackthebox$ nmap -A [TARGET IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2023-02-17 13:27 EST
Nmap scan report for [TARGET IP]
Host is up (0.013s latency).
Not shown: 988 closed ports
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 4.3 (protocol 2.0)
| ssh-hostkey: 
|   1024 ad:ee:5a:bb:69:37:fb:27:af:b8:30:72:a0:f9:6f:53 (DSA)
|_  2048 bc:c6:73:59:13:a1:8a:4b:55:07:50:f6:65:1d:6d:0d (RSA)
25/tcp    open  smtp       Postfix smtpd
|_smtp-commands: beep.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, ENHANCEDSTATUSCODES, 8BITMIME, DSN, 
80/tcp    open  http       Apache httpd 2.2.3
|_http-server-header: Apache/2.2.3 (CentOS)
|_http-title: Did not follow redirect to https://[TARGET IP]/
|_https-redirect: ERROR: Script execution failed (use -d to debug)
110/tcp   open  pop3       Cyrus pop3d 2.3.7-Invoca-RPM-2.3.7-7.el5_6.4
|_pop3-capabilities: UIDL PIPELINING STLS TOP AUTH-RESP-CODE RESP-CODES IMPLEMENTATION(Cyrus POP3 server v2) USER EXPIRE(NEVER) LOGIN-DELAY(0) APOP
|_ssl-cert: ERROR: Script execution failed (use -d to debug)
|_ssl-date: ERROR: Script execution failed (use -d to debug)
|_sslv2: ERROR: Script execution failed (use -d to debug)
|_tls-alpn: ERROR: Script execution failed (use -d to debug)
|_tls-nextprotoneg: ERROR: Script execution failed (use -d to debug)
111/tcp   open  rpcbind    2 (RPC #100000)
143/tcp   open  imap       Cyrus imapd 2.3.7-Invoca-RPM-2.3.7-7.el5_6.4
|_imap-capabilities: Completed ID BINARY MULTIAPPEND OK NO URLAUTHA0001 THREAD=ORDEREDSUBJECT LIST-SUBSCRIBED ATOMIC LISTEXT IDLE STARTTLS UIDPLUS IMAP4 LITERAL+ UNSELECT CATENATE ANNOTATEMORE THREAD=REFERENCES SORT=MODSEQ SORT CHILDREN CONDSTORE RENAME MAILBOX-REFERRALS X-NETSCAPE IMAP4rev1 ACL NAMESPACE RIGHTS=kxte QUOTA
|_imap-ntlm-info: ERROR: Script execution failed (use -d to debug)
|_ssl-cert: ERROR: Script execution failed (use -d to debug)
|_ssl-date: ERROR: Script execution failed (use -d to debug)
|_sslv2: ERROR: Script execution failed (use -d to debug)
|_tls-alpn: ERROR: Script execution failed (use -d to debug)
|_tls-nextprotoneg: ERROR: Script execution failed (use -d to debug)
443/tcp   open  ssl/https?
|_ssl-date: 2023-02-17T19:30:47+00:00; +1h00m00s from scanner time.
993/tcp   open  ssl/imap   Cyrus imapd
|_imap-capabilities: CAPABILITY
995/tcp   open  pop3       Cyrus pop3d
|_ssl-cert: ERROR: Script execution failed (use -d to debug)
|_ssl-date: ERROR: Script execution failed (use -d to debug)
|_ssl-known-key: ERROR: Script execution failed (use -d to debug)
|_sslv2: ERROR: Script execution failed (use -d to debug)
|_tls-alpn: ERROR: Script execution failed (use -d to debug)
|_tls-nextprotoneg: ERROR: Script execution failed (use -d to debug)
3306/tcp  open  mysql      MySQL (unauthorized)
4445/tcp  open  upnotifyp?
10000/tcp open  http       MiniServ 1.570 (Webmin httpd)
|_http-server-header: MiniServ/1.570
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
Service Info: Hosts:  beep.localdomain, 127.0.0.1, example.com

Host script results:
|_clock-skew: 59m59s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 355.77 seconds
```
## Enumeration 
I wanted to take a closer look at the HTTP service running on port 80 and 443. I ran gobuster against the service running on port 443 with a directory list from SecLists.
```
ajread@aj-ubuntu:~/hackthebox$ gobuster -u https://[TARGET IP] -w ~/resources/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt -k

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : https://[TARGET IP]/
[+] Threads      : 10
[+] Wordlist     : /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2023/02/17 13:52:05 Starting gobuster
=====================================================
/images (Status: 301)
/admin (Status: 301)
/modules (Status: 301)
/themes (Status: 301)
/help (Status: 301)
/var (Status: 301)
/mail (Status: 301)
/static (Status: 301)
/lang (Status: 301)
/libs (Status: 301)
/panel (Status: 301)
/configs (Status: 301)
/recordings (Status: 301)
/vtigercrm (Status: 301)
2023/02/17 13:55:08 [!] parse "https://[TARGET IP]/error\x1f_log": net/url: invalid control character in URL
=====================================================
2023/02/17 13:55:55 Finished
=====================================================
```
I wanted to hone in on the ```/vtigercrm``` subdirectory. 
## Initial Access
I was able to find an LFI that worked on that version of Elastix and vtigercrm. Exploit located [here](https://www.exploit-db.com/exploits/37637). 