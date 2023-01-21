# Legacy

Machine Level: Easy <br />
OS: Windows

## Scanning
I ran an aggressive NMAP scan and found that it was a Windows XP box. 
```
ajread@aj-ubuntu:~$ nmap -A [TARGET IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2023-01-21 13:38 EST
Nmap scan report for [TARGET IP]
Host is up (0.015s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Microsoft Windows XP microsoft-ds
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_clock-skew: 5d01h57m38s
|_ms-sql-info: ERROR: Script execution failed (use -d to debug)
|_nbstat: NetBIOS name: LEGACY, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:b9:e6:f7 (VMware)
|_smb-os-discovery: ERROR: Script execution failed (use -d to debug)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 57.36 seconds
```
## Initial Access 
I found [this](https://www.binarytides.com/hack-windows-xp-metasploit/) website that helped guide me for what to do next and exploit MS08-067 for Netapi. In metasploit, I had to set the target to be ```Windows XP SP3 English (AlwaysOn NX)``` or else it would not work. After running the exploit, I was dropped into a system shell! 
```
meterpreter > pwd
C:\WINDOWS\system32
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```
And I was able to retrieve the user and administrator flag! There was no need to conduct privilege escalation on this box. 
