# Command Cheatsheet — Starting Point Tier 1

| Phase | Command | Purpose |
|-------|---------|---------|
| Recon | `nmap -sV -sC <ip>` | Service + default-script scan |
| Recon | `nmap -p- --min-rate=2000 <ip>` | Full TCP port sweep |
| Web | `gobuster dir -u http://<ip> -w <wordlist>` | Directory brute force |
| Access | `ssh <user>@<ip>` | Shell with harvested creds |

<!-- Add techniques here as you learn them on each box. -->
