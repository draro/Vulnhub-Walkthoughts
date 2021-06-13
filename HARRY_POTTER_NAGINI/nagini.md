# Walkthrough VULNHUB --> HARRY POTTER: NAGINI

First step is to scan our Netwok to find the machine IP, in my case the host has ip 192.168.1.172.

We use nmap to scan the target machine with the command:

```bash
nmap -sC -sV -p- 192.168.1.174
```

![Alt text](./img/nmap.PNG?raw=true "NMAP Results")

I navigate to the web page to see what it is and to check if we find any visible vulnerability on that page.

![Alt text](./img/web_page_port80.PNG?raw=true "Port 80")

We run a scan on the port 80 with gobuster to search for interesting things.

```bash
gobuster dir --url http://192.168.1.174 -x .php,.js,.txt,.html --wordlist /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 50
```

![Alt text](./img/gobuster.PNG?raw=true "GOBUSTER SCAN")

We start browsing the /note.txt and we find as follow:

```
Hello developers!!


I will be using our new HTTP3 Server at https://quic.nagini.hogwarts for further communications.
All developers are requested to visit the server regularly for checking latest announcements.


Regards,
site_amdin
```

**WE NEED HTTP3!!!!!**

We also need to add **quic.nagini.hogwarts** in our **/etc/hosts**.

After spending some time on internet trying to figure out how to use this http3, I found this guide to enable **curl** with http3 support.

[https://github.com/curl/curl/blob/master/docs/HTTP3.md](https://github.com/curl/curl/blob/master/docs/HTTP3.md)

**NOW I built a new curl with http3 support**

Using curl we do a GET request on https://quic.nagini.hogwarts:

```bash
./curl --http3 https://quic.nagini.hogwarts
```

![Alt text](./img/http3.PNG?raw=true "CURL HTTP3")
