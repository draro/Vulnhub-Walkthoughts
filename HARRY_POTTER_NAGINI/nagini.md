# Walkthrough VULNHUB --> HARRY POTTER: NAGINI

First step is to scan our Netwok to find the machine IP, in my case the host has ip 192.168.1.172.

We use nmap to scan the target machine with the command:

```bash
nmap -sC -sV -p- 192.168.1.174
```

![Alt text](./img/nmap.PNG?raw=true "NMAP Results")
