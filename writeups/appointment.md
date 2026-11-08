# Appointment — HackTheBox Starting Point Writeup

![Platform](https://img.shields.io/badge/Platform-HackTheBox-9FEF00?logo=hackthebox&logoColor=black)
![Tier](https://img.shields.io/badge/Tier-1-red)
![OS](https://img.shields.io/badge/OS-Linux-blue?logo=linux&logoColor=white)
![Flags](https://img.shields.io/badge/Flags-1-orange)
![Foothold](https://img.shields.io/badge/Foothold-SQLi%20auth%20bypass-red)

> **Target:** `10.129.85.138`
> **Foothold:** **SQL injection** auth bypass on the web login form (`http://10.129.85.138/`).

---

## 🧭 Overview

Appointment serves an **Apache 2.4.38** web app with a **login form** that builds its SQL
query from unsanitised user input. By injecting a classic authentication-bypass payload,
the `WHERE` clause is forced always-true (or the password check is commented out), logging
in without valid credentials. The page then returns **"Congratulations"** and the flag.

---

## 1. Reconnaissance

```bash
nmap -sC -sV 10.129.85.138
```

```text
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Login
```

Only **port 80** — a web server titled **Login**. Browse to it:

```text
http://10.129.85.138/
```

A username/password login form is presented.

---

## 2. Foothold — SQL Injection Auth Bypass

The form almost certainly builds a query like:

```sql
SELECT * FROM users WHERE username='<input>' AND password='<input>';
```

**What didn't work — `admin'--`:** this assumes a row with username `admin` actually
exists. It comments out the password check, but if there is no `admin` user the query
returns nothing, so login fails.

**What worked — `' OR 1=1-- `** in **both** the username and password fields:

```text
' OR 1=1-- 
```

This turns the condition into `... WHERE username='' OR 1=1-- ...`. The `OR 1=1` is
**always true**, so the query matches **every** row and returns the first user — logging
in regardless of which accounts exist. (In MySQL/MariaDB the `--` comment needs a trailing
space; `#` also works, e.g. the intended payload `admin'#`.)

The app responds with **"Congratulations"** and reveals the flag.

---

## 3. Flag

<details>
<summary>🚩 Click to reveal — FLAG</summary>

```text
e3d0796d002a446c0e622226f42e9672
```
</details>

---

## 📝 Task Questions & Answers

<details>
<summary>📋 Click to reveal — guided task answers</summary>

| # | Question | Answer |
|---|----------|--------|
| 1 | What does the acronym SQL stand for? | Structured Query Language |
| 2 | What is one of the most common types of SQL vulnerabilities? | SQL injection (SQLi) |
| 3 | What is the 2021 OWASP Top 10 classification for this vulnerability? | A03:2021 – Injection |
| 4 | What does Nmap report as the service and version on port 80? | Apache httpd 2.4.38 ((Debian)) |
| 5 | What is the standard port used for the HTTPS protocol? | 443 |
| 6 | What is a folder called in web-application terminology? | Directory |
| 7 | What is the HTTP response code returned for `Not Found` errors? | 404 |
| 8 | What Gobuster switch specifies we're brute-forcing directories (not subdomains)? | dir |
| 9 | What single character can be used to comment out the rest of a line in MySQL? | # |
| 10 | Use a comment to log in as admin. What is the first word on the returned page? | Congratulations |
| 11 | Submit flag | `e3d0796d002a446c0e622226f42e9672` |

</details>

---

## 🛡️ Lessons / Remediation
- **Use parameterised queries / prepared statements** — never concatenate user input into SQL.
- **Validate and escape input** and apply least-privilege on the DB account.
- **Hash passwords** and verify them in application code, not via a raw SQL match.
- Consider a **WAF** and generic error messages to limit injection feedback.
