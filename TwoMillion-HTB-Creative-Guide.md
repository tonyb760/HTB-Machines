<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&height=230&color=0:0f172a,45:9FEF00,100:111827&text=HTB%20TwoMillion&fontColor=ffffff&fontSize=58&fontAlignY=38&desc=API%20nostalgia,%20admin%20command%20injection,%20and%20OverlayFS%20privesc&descAlignY=60&animation=fadeIn" width="100%" />

<br>

![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-Retired-9FEF00?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Linux-22C55E?style=for-the-badge)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-9FEF00?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Rooted-16A34A?style=for-the-badge)

<br>

### API nostalgia, admin command injection, and OverlayFS privesc

</div>

---

> [!WARNING]
> **Spoiler warning:** This is a full retired-box walkthrough. Flags are intentionally redacted for GitHub, but the exploitation chain and key techniques are documented.

---

## 🧭 Snapshot

| Field | Value |
|---|---|
| Machine | `TwoMillion` |
| Platform | Hack The Box |
| Retirement Status | Confirmed retired via HTB MCP |
| OS | Linux |
| Difficulty | Easy |
| Tags | `web` `api` `command-injection` `linux` `cve-2023-0386` |

---

## ⚡ TL;DR

TwoMillion starts as a web/API puzzle around the HTB invite flow. After account creation and API enumeration, an admin endpoint can be abused for command execution, leading to Linux credentials. Root is obtained with CVE-2023-0386 OverlayFS/FUSE privilege escalation.

---

## 🛰️ Attack Surface

| Port | Service | Note |
|---:|---|---|
| `22/tcp` | SSH | OpenSSH 8.9p1 |
| `80/tcp` | HTTP | nginx / 2million.htb |

---

## 🧩 Attack Chain

| Step | Action |
|---:|---|
| 1 | Enumerate web app and identify invite-code workflow |
| 2 | Reverse/abuse invite API to register an account |
| 3 | Enumerate authenticated API routes |
| 4 | Abuse admin/VPN-generation command injection for RCE |
| 5 | Recover application/environment credentials |
| 6 | SSH as the recovered user |
| 7 | Compile/run CVE-2023-0386 OverlayFS exploit |
| 8 | Read root flag |

---

## 🕸️ Visual Attack Graph

```mermaid
flowchart TD
    A[Enumerate web app and identify invite-code workflow]
    B[Reverse/abuse invite API to register an account]
    A --> B
    C[Enumerate authenticated API routes]
    B --> C
    D[Abuse admin/VPN-generation command injection for RCE]
    C --> D
    E[Recover application/environment credentials]
    D --> E
    F[SSH as the recovered user]
    E --> F
    G[Compile/run CVE-2023-0386 OverlayFS exploit]
    F --> G
    H[Read root flag]
    G --> H
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

<details>
<summary><b>Useful commands and technique anchors</b></summary>

- `ffuf/ferox + browser/API review of http://2million.htb`
- `curl -X POST /api/v1/invite/generate`
- `curl authenticated API endpoints to discover admin functions`
- `Inject shell metacharacters into vulnerable admin VPN generation parameter`
- `ssh admin@<target>`
- `compile CVE-2023-0386 helper and trigger OverlayFS/FUSE privesc`

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
| 1 | Never trust role-sensitive functionality purely client-side/API-discoverable |
| 2 | Validate and safely compose shell arguments in backend admin tooling |
| 3 | Do not store reusable credentials in readable environment files |
| 4 | Patch kernels affected by CVE-2023-0386 |

---

## 🏁 Final Path

```text
Enumerate web app and identify invite-code workflow → Reverse/abuse invite API to register an account → Enumerate authenticated API routes → Abuse admin/VPN-generation command injection → Recover application/environment credentials → SSH as the recovered user → Compile/run CVE-2023-0386 OverlayFS exploit → Read root flag
```

<div align="center">

### ✅ TwoMillion rooted — full guide complete

<img src="https://capsule-render.vercel.app/api?type=waving&height=120&section=footer&color=0:111827,45:9FEF00,100:0f172a" width="100%" />

</div>
