<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&height=240&color=0:0f172a,35:111827,70:2563eb,100:9FEF00&text=HTB%20Machines&fontColor=ffffff&fontSize=64&fontAlignY=38&desc=Retired%20Hack%20The%20Box%20writeups%20%E2%80%A2%20methodology%20notes%20%E2%80%A2%20clean-room%20learning&descSize=18&descAlignY=60&animation=fadeIn" width="100%" />

<br>

[![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-Writeups-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black)](https://www.hackthebox.com/)
![Security](https://img.shields.io/badge/Focus-Offensive%20Security-2563EB?style=for-the-badge&logo=gnometerminal&logoColor=white)
![Spoiler Policy](https://img.shields.io/badge/Policy-Retired%20Boxes%20Only-7C3AED?style=for-the-badge)
![Guides](https://img.shields.io/badge/Creative%20Guides-11-22C55E?style=for-the-badge&logo=markdown&logoColor=white)

<br>

### 🧠 Hack The Box writeups, clean-room reflections, and creative methodology guides.

<sub>Built for learning, documentation practice, methodology refinement, and defender-minded takeaways.</sub>

</div>

---

> [!IMPORTANT]
> **Spoiler policy:** Full walkthroughs are only published after machines retire. Active-box content is kept spoiler-light: no flags, passwords, hashes, exact object names, target IPs, or complete exploit chains.

---

# 🗺️ Repository Map

## 🧾 Writeups

| # | Machine | File | Type | Main Theme | Status |
|---:|---|---|---|---|---|
| 1 | **Blocky** | [`Blocky-HTB-Creative-Guide.md`](./Blocky-HTB-Creative-Guide.md) | Full creative guide | WordPress, plugin JAR secrets, sudo | ✅ Retired / Published |
| 2 | **Dog** | [`Dog-HTB-Creative-Guide.md`](./Dog-HTB-Creative-Guide.md) | Full creative guide | Exposed Git, Backdrop CMS, sudo `bee eval` | ✅ Retired / Published |
| 3 | **Down** | [`Down-HTB-Creative-Guide.md`](./Down-HTB-Creative-Guide.md) | Full creative guide | SSRF, curl option injection, vault disclosure, sudo | ✅ Retired / Published |
| 4 | **Editor** | [`Editor-HTB-Creative-Guide.md`](./Editor-HTB-Creative-Guide.md) | Full creative guide | XWiki RCE, config creds, Netdata `ndsudo` privesc | ✅ Retired / Published |
| 5 | **Fluffy** | [`Fluffy-HTB-Creative-Guide.md`](./Fluffy-HTB-Creative-Guide.md) | Full creative guide | ADCS, NTLM coercion, ESC16, shadow credentials | ✅ Retired / Published |
| 6 | **Lame** | [`Lame-HTB-Creative-Guide.md`](./Lame-HTB-Creative-Guide.md) | Full creative guide | Legacy Samba usermap script RCE | ✅ Retired / Published |
| 7 | **Love** | [`Love-HTB-Creative-Guide.md`](./Love-HTB-Creative-Guide.md) | Full creative guide | SSRF, Voting System upload RCE, AlwaysInstallElevated | ✅ Retired / Published |
| 8 | **Netmon** | [`Netmon-HTB-Creative-Guide.md`](./Netmon-HTB-Creative-Guide.md) | Full creative guide | Anonymous FTP, PRTG credentials, authenticated RCE | ✅ Retired / Published |
| 9 | **Support** | [`Support-HTB-Creative-Guide.md`](./Support-HTB-Creative-Guide.md) | Full creative guide | AD enumeration, LDAP secrets, RBCD to Administrator | ✅ Retired / Published |
| 10 | **TwoMillion** | [`TwoMillion-HTB-Creative-Guide.md`](./TwoMillion-HTB-Creative-Guide.md) | Full creative guide | API abuse, command injection, OverlayFS CVE-2023-0386 | ✅ Retired / Published |
| 11 | **Checkpoint** | [`Checkpoint_HTB_NoSpoilers.md`](./Checkpoint_HTB_NoSpoilers.md) | No-spoiler reflection | AD, evidence handling, branch control | ⚠️ Active-safe only |
| 12 | **Logging** | [`Logging-HTB.md`](./Logging-HTB.md) | GitHub-safe / active-box style | ADCS, gMSA, WSUS spoofing | ⚠️ Active-safe only |
| 13 | **Reactor** | [`Reactor.md`](./Reactor.md) | Clean-room methodology | Next.js RCE, SQLite, Node Inspector | ⚠️ Active-safe only |
| 14 | **Snapped** | [`Snapped_HTB_Writeup.html`](./Snapped_HTB_Writeup.html) | Creative HTML report | Visual report format | ✅ Published |
| 15 | **Orion** | [`Orion.md`](./Orion.md) | Full creative guide | Craft CMS RCE, CVE-2025-32432, telnetd injection | ✅ Retired / Published |

---

## 🧰 Methodology Themes

| Area | Practiced In |
|---|---|
| Active Directory enumeration | Support, Fluffy, Logging, Checkpoint |
| RBCD / Kerberos abuse | Support |
| ADCS / certificate abuse | Fluffy, Logging |
| Web application RCE | Editor, Love, Reactor, Orion |
| Source/config disclosure | Dog, Blocky, Editor |
| SSRF / file disclosure | Down, Love |
| Legacy service exploitation | Lame, Netmon, Orion |
| Linux privilege escalation | TwoMillion, Down, Editor, Dog, Blocky, Lame, Orion |
| Windows privilege escalation | Love, Support, Fluffy, Netmon |
| Evidence-driven analysis | Checkpoint, Netmon, Dog |
| PHP object injection / deserialization | Orion |
| CVE chaining (web + service) | Orion |
| Creative report writing | Snapped, all creative guides |

---

# 🧭 What This Repo Is

This repository is my public Hack The Box knowledge base. Each writeup is meant to capture more than just the final exploit path. I try to document:

- how I approached enumeration,
- what evidence mattered,
- what branches failed,
- why the final chain worked,
- what defenders could learn from the box,
- and what I would do differently next time.

<div align="center">

```text
Recon → Hypothesis → Validation → Pivot → Exploit → PrivEsc → Lessons
```

</div>

---

# 🔥 Featured Writeups

<table>
<tr>
<td width="33%" valign="top">

<h3 align="center">🧾 Support</h3>

<div align="center">

![Windows](https://img.shields.io/badge/Windows-Active%20Directory-0078D6?style=flat-square&logo=windows&logoColor=white)
![RBCD](https://img.shields.io/badge/RBCD-Kerberos-7C3AED?style=flat-square)
![LDAP](https://img.shields.io/badge/LDAP-Secrets-F97316?style=flat-square)

</div>

A clean AD chain from anonymous share exposure to LDAP secrets, BloodHound-style relationship analysis, and RBCD-based Administrator impersonation.

**Core lesson:** one dangerous ACL edge on a DC computer object can become full domain compromise.

➡️ [`Read Support`](./Support-HTB-Creative-Guide.md)

</td>
<td width="33%" valign="top">

<h3 align="center">🧪 Editor</h3>

<div align="center">

![Linux](https://img.shields.io/badge/Linux-Ubuntu-E95420?style=flat-square&logo=ubuntu&logoColor=white)
![XWiki](https://img.shields.io/badge/XWiki-Groovy%20RCE-F97316?style=flat-square)
![Netdata](https://img.shields.io/badge/Netdata-ndsudo%20Privesc-00AB44?style=flat-square)

</div>

A web-to-root Linux chain involving XWiki Groovy injection, configuration credential reuse, and Netdata `ndsudo` PATH injection.

**Core lesson:** application config secrets and helper binaries often complete the path after RCE.

➡️ [`Read Editor`](./Editor-HTB-Creative-Guide.md)

</td>
<td width="33%" valign="top">

<h3 align="center">🐶 Dog</h3>

<div align="center">

![Linux](https://img.shields.io/badge/Linux-Web%20Box-22C55E?style=flat-square&logo=linux&logoColor=white)
![Git](https://img.shields.io/badge/Exposed-.git-111827?style=flat-square&logo=git&logoColor=white)
![CMS](https://img.shields.io/badge/Backdrop-CMS-8B5CF6?style=flat-square)

</div>

A source-disclosure-driven solve: exposed Git leaks Backdrop CMS secrets, credentials reuse into SSH, and sudo `bee eval` becomes root command execution.

**Core lesson:** exposed source control can reveal the whole operational trust chain.

➡️ [`Read Dog`](./Dog-HTB-Creative-Guide.md)

</td>
</tr>
</table>

---

# 🧩 Writeup Style Guide

| Format | When I Use It | Includes |
|---|---|---|
| **Full Creative Guide** | Retired boxes with local root completion | Detailed chain, visual map, commands, lessons, defenses |
| **No-Spoiler Reflection** | Active/recent boxes or public-safe notes | Themes, lessons, high-level flow |
| **Clean-Room Methodology** | Technique-focused posts | Concepts and decision points without free-win payloads |
| **Creative Report** | Portfolio-style reports | HTML/visual storytelling, charts, narrative sections |

> [!TIP]
> My goal is to make every writeup useful even if you are not solving the exact same machine.

---

# 🕸️ Methodology Graph

```mermaid
flowchart LR
    A[Service Enumeration] --> B[Hypothesis Building]
    B --> C[Access Validation]
    C --> D{Branch Works?}
    D -- No --> E[Record Blocker]
    E --> B
    D -- Yes --> F[Exploit / Abuse Path]
    F --> G[Post-Exploitation Enumeration]
    G --> H[Privilege Escalation]
    H --> I[Evidence Capture]
    I --> J[Defensive Lessons]
```

---

# 🧠 Skills Practiced Across the Repo

<div align="center">

| Category | Examples |
|---|---|
| 🛰️ Recon | Nmap, HTTP fingerprinting, service/version triage |
| 🪪 Auth Testing | SMB/LDAP/WinRM/SSH validation |
| 🏛️ Active Directory | Users, groups, ACLs, BloodHound-style reasoning |
| 📜 ADCS | Template review, EKUs, enrollment rights |
| 📁 Evidence Handling | Logs, shares, SQLite DBs, offline artifacts, source repos |
| 🧪 Exploit Validation | CVE research, patch-level matching, safe proofing |
| 🧬 PrivEsc | gMSA, debug interfaces, scheduled/update workflows, sudo/SUID |
| 🛡️ Defense | Remediation notes and detection-minded takeaways |

</div>

---

# 🧰 Toolkit Highlight

```text
nmap        netexec / nxc     ldapsearch      smbclient
certipy-ad  bloodyAD          impacket        BloodHound
ffuf        feroxbuster       curl/httpx      hashcat
git-dumper  wpscan           sshpass         sqlite3
linpeas     winpeas           msfvenom        Python
PowerShell  certipy           responder       ntlmrelayx
```

---

# 🔐 Disclosure & Ethics

- I follow Hack The Box rules and avoid posting active-box spoilers.
- Flags are never included in public-safe posts.
- Credentials/hashes may be redacted or omitted depending on the box status.
- The purpose of this repo is education, documentation, and defensive learning.
- Do not use these techniques outside systems you own or are explicitly authorized to test.

---

# 📊 Current Coverage

<div align="center">

| OS / Category | Count | Examples |
|---|---:|---|
| Windows / AD | 4+ | Support, Fluffy, Logging, Checkpoint |
| Windows / Web + PrivEsc | 2+ | Love, Netmon |
| Linux / Web + PrivEsc | 8+ | TwoMillion, Down, Editor, Dog, Blocky, Reactor, Lame, Orion |
| Full Creative Markdown Guides | 11 | TwoMillion, Support, Fluffy, Down, Editor, Dog, Love, Netmon, Blocky, Lame, Orion |
| Active-safe / No-spoiler Posts | 3 | Logging, Checkpoint, Reactor |
| Creative HTML Reports | 1+ | Snapped |

<br>

![Creative Guides](https://img.shields.io/badge/Creative%20Guides-11-9FEF00?style=for-the-badge&logo=markdown&logoColor=black)
![Writeups](https://img.shields.io/badge/Total%20Writeups-15-2563EB?style=for-the-badge)
![Windows](https://img.shields.io/badge/Windows%20AD-Growing-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-Web%20%26%20PrivEsc-E95420?style=for-the-badge&logo=linux&logoColor=white)

</div>

---

# 📝 Suggested Reading Order

1. **Classic fundamentals:** [`Lame`](./Lame-HTB-Creative-Guide.md) → [`Blocky`](./Blocky-HTB-Creative-Guide.md)
2. **Linux web-to-root:** [`TwoMillion`](./TwoMillion-HTB-Creative-Guide.md) → [`Down`](./Down-HTB-Creative-Guide.md) → [`Editor`](./Editor-HTB-Creative-Guide.md) → [`Orion`](./Orion.md)
3. **Source/config disclosure:** [`Dog`](./Dog-HTB-Creative-Guide.md)
4. **Windows web/privesc:** [`Love`](./Love-HTB-Creative-Guide.md) → [`Netmon`](./Netmon-HTB-Creative-Guide.md)
5. **Active Directory chains:** [`Support`](./Support-HTB-Creative-Guide.md) → [`Fluffy`](./Fluffy-HTB-Creative-Guide.md)
6. **Spoiler-safe mindset posts:** [`Checkpoint`](./Checkpoint_HTB_NoSpoilers.md) → [`Reactor`](./Reactor.md) → [`Logging`](./Logging-HTB.md)
7. **Visual report style:** [`Snapped`](./Snapped_HTB_Writeup.html)

---

# 🚀 Roadmap

- [x] Add full creative guides for first 10 retired/rooted boxes
- [x] Add all writeups to the README writeups section
- [ ] Add difficulty badges for every machine row
- [ ] Add tags at the top of every guide
- [ ] Add a machine index grouped by OS and technique
- [ ] Add defensive detection notes to each retired-box walkthrough
- [ ] Add screenshots/diagrams where GitHub-safe
- [ ] Convert selected writeups into portfolio-style HTML reports

---

# 🤝 Connect / Feedback

If you spot an error, have a cleaner way to explain a technique, or want to compare approaches, feel free to open an issue or reach out.

<div align="center">

### ⭐ If this repo helps you learn, consider starring it.

<br>

<img src="https://capsule-render.vercel.app/api?type=waving&height=120&section=footer&color=0:9FEF00,45:2563eb,100:0f172a" width="100%" />

</div>
