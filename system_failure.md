# Walkthrough VULNHUB --> SYSTEM FAILURE

First step is to scan our Netwok to find the machine IP, in my case the host has ip 192.168.1.109.

We use nmap to scan the target machine with the command:

```bash
nmap -sC -sV -p- 192.168.1.109
```

```bash
┌──(root💀kali)-[/home/animale]
└─# nmap -sC -sV -p- 192.168.1.109
Starting Nmap 7.91 ( https://nmap.org ) at 2021-06-14 07:07 EDT
Nmap scan report for SystemFailure.homenet.telecomitalia.it (192.168.1.109)
Host is up (0.00016s latency).
Not shown: 65530 closed ports
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 3.0.3
22/tcp  open  ssh         OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 bb:02:d1:ee:91:11:fe:a0:b7:90:e6:e0:07:49:95:85 (RSA)
|   256 ef:e6:04:30:01:50:07:5d:2d:17:99:d1:00:3d:f2:d6 (ECDSA)
|_  256 80:7f:c5:96:0e:3d:66:b9:d6:a8:6f:59:fa:ca:86:36 (ED25519)
80/tcp  open  http        Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Site doesn't have a title (text/html).
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.9.5-Debian (workgroup: WORKGROUP)
MAC Address: 08:00:27:50:19:B3 (Oracle VirtualBox virtual NIC)
Service Info: Host: SYSTEMFAILURE; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 1h20m01s, deviation: 2h18m34s, median: 0s
|_nbstat: NetBIOS name: SYSTEMFAILURE, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.9.5-Debian)
|   Computer name: systemfailure
|   NetBIOS computer name: SYSTEMFAILURE\x00
|   Domain name: homenet.telecomitalia.it
|   FQDN: systemfailure.homenet.telecomitalia.it
|_  System time: 2021-06-14T07:08:11-04:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2021-06-14T11:08:11
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.45 seconds
```

Let's run gobuster to see what's in the web server

```bash
┌──(root💀kali)-[/home/animale]
└─# gobuster dir --url http://192.168.1.109 -x .php,.txt,.js,.html --wordlist /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 50
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.1.109
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              js,html,php,txt
[+] Timeout:                 10s
===============================================================
2021/06/14 07:10:01 Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 161]
/server-status        (Status: 403) [Size: 278]
/area4                (Status: 301) [Size: 314] [--> http://192.168.1.109/area4/]

===============================================================
2021/06/14 07:13:05 Finished
===============================================================
```

We navigate to the index.html and we have the following:
![Index.html](./System_Failure/index.PNG?raw=true "index.html")

Nothing there!

Let's try on to see area4:
![area4](./System_Failure/area4.PNG?raw=true "area4")

Nothing there!

Let's check for shares on the host :

```bash
┌──(root💀kali)-[/home/animale]
└─# smbmap -u victim -p s3cr3t -H 192.168.1.109
[+] Guest session       IP: 192.168.1.109:445   Name: SystemFailure.homenet.telecomitalia.it
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        anonymous                                               READ, WRITE     open
        IPC$                                                    NO ACCESS       IPC Service (Samba 4.9.5-Debian)
```

It appears that anonymous user has Read and Write permission

Let's connect through smbclient:

```bash
┌──(root💀kali)-[/home/animale]
└─# smbclient //192.168.1.109/anonymous
Enter WORKGROUP\root's password:
Try "help" to get a list of possible commands.
smb: \> pwd
Current directory is \\192.168.1.109\anonymous\
smb: \> ls
  .                                   D        0  Mon Jun 14 07:13:41 2021
  ..                                  D        0  Wed Dec 16 09:58:53 2020
  share                               N      220  Thu Dec 17 16:25:14 2020

                7205476 blocks of size 1024. 5291292 blocks available
smb: \> ls
  .                                   D        0  Mon Jun 14 07:13:41 2021
  ..                                  D        0  Wed Dec 16 09:58:53 2020
  share                               N      220  Thu Dec 17 16:25:14 2020

                7205476 blocks of size 1024. 5291292 blocks available

```

We see only one file called **share**. Let's see what is in it

```bash
smb: \> more share
```

![share](./System_Failure/message.PNG?raw=true "share")

We go on [https://crackstation.net/](https://crackstation.net/) and we paste the string we found

![crack station](./System_Failre/crackstation.PNG?raw=true "Crack Station")
