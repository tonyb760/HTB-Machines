# ⚛️ HTB: Reactor — Writeup

> A clean-room methodology writeup. No flags, no secrets, no live exploit
> payloads — just the *how* and the *why* so others can learn the technique
> chain without getting a free win on an active box.

| Field | Value |
|---|---|
| Platform | Hack The Box |
| Difficulty | Medium |
| OS | Linux (Ubuntu) |
| IP | `10.129.x.x` (lab-only) |
| Attack surface | Web (Next.js) + SSH |
| Key CVEs | **CVE-2025-55182** (Next.js SSR RCE) |
| Key privesc | Node.js Inspector protocol abuse |
| Tags | `web`, `nextjs`, `cve`, `nodejs`, `privesc` |

> ⚠️ **Spoiler-light policy.** This file describes *techniques* and *decision
> points* only. Hashes, plaintext credentials, exact file contents, and ready-
> to-run exploit commands are intentionally omitted. Reproduce them on your
> own lab.

---

## 🧭 TL;DR

Reactor is a two-stage Linux box: an **internet-facing Next.js dashboard**
sitting on a non-standard high port, and an SSH service. The web app
ships a vulnerable version of Next.js that falls to a public pre-auth RCE
(*React2Shell* family). That gives a foothold as the `node` service user.
Inside the app's working directory is a small SQLite database whose only
sensitive column is a single unsalted MD5 password hash. Crack it, SSH in
as `engineer`, and pivot to root by abusing a **Node Inspector process
bound to localhost** that the app's root-owned service started on launch.

```
[ Next.js :3000 ]  --(CVE-2025-55182)-->  RCE as `node`
                                            |
                                            v
                                [/opt/reactor-app/reactor.db]
                                            |
                                  (MD5 crack, wordlist)
                                            |
                                            v
                                  SSH as `engineer`
                                            |
                                            v
                            Node Inspector @ 127.0.0.1:9229
                                            |
                                            v
                                       root shell
```

---

## 1. Recon

### 1.1 Port scan

Two ports. That's the whole attack surface.

```text
22/tcp   OpenSSH 9.6p1 Ubuntu
3000/tcp Next.js (X-Powered-By: Next.js, app router headers)
```

A high TCP port with `X-Powered-By: Next.js` is a screaming hint to
fingerprint the exact Next.js build and look up known advisories before
doing anything else.

### 1.2 Service fingerprinting

A normal `GET /` on `:3000` returns a long, prerendered HTML page with the
usual Next.js chunks:

```text
/_next/static/chunks/main-app-*.js
/_next/static/chunks/517-*.js          <- framework polyfill chunk
/_next/static/chunks/webpack-*.js
/_next/static/css/<hash>.css
```

A useful trick: the framework version string is sometimes embedded in
the polyfill chunk or in the build manifest. Pulling
`/_next/static/L<deploymentId>/_buildManifest.js` exposes the page
registry; comparing the chunk hashes against the official Next.js release
manifest quickly narrows the version.

> 💡 **Tip:** When `robots.txt` returns 404 and the app router has no
> obvious API surface, the polyfill chunk (`517-*.js`) often leaks the
> framework version. Grep for `next.version` or `window.next=` literals.

### 1.3 Subdomain / vhost discovery

A vhost fuzz with `ffuf` against `<word>.reactor.htb` and a 5k subdomain
list returns nothing meaningful on the live target in the standard
enumeration window. The interesting surfaces are *inside* the app
itself, not on new hostnames.

### 1.4 Tech stack

- **Server:** Next.js 15.x (App Router, RSC enabled)
- **DB:** SQLite file (`reactor.db` in the app's working directory)
- **Process model:** `node` process running as the unprivileged `node`
  service user; a *separate* root-owned Node process with the
  `--inspect` flag listening on `127.0.0.1:9229`

---

## 2. Foothold — Pre-auth RCE via CVE-2025-55182

### 2.1 Vulnerability in one paragraph

CVE-2025-55182 is a server-side request forgery / unsafe deserialization
flaw in Next.js's React Server Components (RSC) flight protocol. The
server deserializes attacker-controlled references inside the
`Next-Action` flight payload without sufficient origin checks, allowing
a crafted multipart RSC request to coerce the server into evaluating
attacker-supplied code. In the **React2Shell** exploit family this is
typically weaponized as: a benign-looking `POST` to a RSC endpoint with
a poisoned `Next-Action` header and a body whose `$$typeof` chain
references server-only modules.

Affected versions: Next.js ≤ 15.0.3 (per the public advisory) and any
patched-but-misconfigured App Router deployment that re-enabled the
unsafe deserializer.

### 2.2 Decision point

Once the framework version is confirmed, you have two reasonable
branches:

| Branch | Evidence | Pros | Cons |
|---|---|---|---|
| **A. Pre-auth RCE (CVE-2025-55182)** | Version is in the vulnerable window, server returns RSC headers | One-shot foothold, no creds needed | Requires the correct exploit variant for the exact patch level |
| **B. Look for an auth/admin surface in the app** | RSC action endpoints may exist for users | More "by-the-book" | The app's only meaningful data is the local SQLite — you'd still need LFI/RCE eventually |

Branch A wins because the version is in the vulnerable window and the
exploit is public, deterministic, and pre-auth. We don't burn time
enumerating RSC actions.

### 2.3 Exploitation (high level only)

> The full request body is omitted on purpose — see the official
> advisory / patch diff for the canonical PoC. Below is just the
> *shape* of the attack so you can recognize it in PCAP.

The exploit is a single HTTP request to an RSC endpoint, typically
`POST /` with:

- `Next-Action: <action-id>` — references a real server action the app
  actually exposes, or a special header that the vulnerable serializer
  follows without validation
- `Content-Type: multipart/form-data; boundary=...` — body is a
  multipart payload mimicking the RSC flight format
- Inside one of the parts: a `$$typeof`-tagged reference whose
  prototype chain points to a server-only module path; the deserializer
  walks it and ends up `require()`-ing attacker-controlled JS

A successful run gives an unauthenticated RCE as the `node` user — no
reverse shell needed if you can do command execution inline.

### 2.4 Confirming the foothold

1. `id` → `uid=... (node)`
2. `hostname` / `cat /etc/os-release` to confirm the box identity
3. `ls /opt/reactor-app/` to find the app source + SQLite DB
4. Record the user, IP, and timestamp in your notes

> At this point the box is "rooted in spirit" — you have code execution
> on the target. Everything after this is just unlocking nicer
> primitives (TTY, persistence, root).

---

## 3. Lateral Movement — DB Loot → SSH as `engineer`

### 3.1 What we find

Inside `/opt/reactor-app/`:

```text
reactor.db          <- SQLite, world-readable
package.json
next.config.js
server.js           <- the entrypoint
```

The interesting schema is tiny: a `users` table with `(id, username,
password_hash, role)`. The hash column is **a single round of MD5**, no
salt — the kind of thing a 30-second `hashcat -m 0` run will spit out.

> 🔑 **Lesson:** When you get RCE, the *first* thing you should do
> after confirming identity is `ls` the application's working
> directory, look for a `*.db` / `*.sqlite` / `*.sqlite3` file, and
> `sqlite3` it. App credentials are almost always there before they
> are in the OS.

### 3.2 Cracking the hash

It's an MD5 — every rainbow table since 2008 has it. A short wordlist
suffices; do not over-engineer this. (No hash printed here, obviously.)

### 3.3 SSH as `engineer`

The recovered plaintext works directly for SSH. From here we have:

- A real TTY
- Persistence between sessions
- Access to the `engineer` user's files (low-value here, but a
  prerequisite for the next step)

---

## 4. Privilege Escalation — Node Inspector Abuse

### 4.1 The misconfiguration

While enumerating the box, you'll notice a second Node.js process
owned by `root` that was started with `--inspect=127.0.0.1:9229`. The
app's `server.js` (or a sibling service file) launches it on boot as a
"development convenience" for whoever maintains the app.

**Node Inspector is a full V8 debugging protocol.** When bound to
`127.0.0.1` it is *not* exposed to the network, but **any local user
that can reach the port has root-level code execution** through
`Runtime.evaluate`. There is no auth on the inspector protocol by
default. The intent was "only I can reach it" — the reality is "anyone
on the box can reach it."

> 🧨 **This is the box's core lesson.** A root-owned Node process
> with the inspector enabled is functionally equivalent to leaving a
> root shell listening on a TCP port. The `127.0.0.1` binding stops
> remote attackers, not local users.

### 4.2 Tree-of-Thought for privesc

Before committing to a path, score your options:

| Option | Evidence | Cost | Verdict |
|---|---|---|---|
| Node Inspector attach (`127.0.0.1:9229`) | `ss -tlnp` shows a Node process listening there as root; process cmdline has `--inspect` | One Python script using `websocket-client` and the DevTools protocol | **✅ Choose this** |
| `sudo -l` | `engineer` has no sudo rules | 0 | ❌ |
| SUID binaries | None of interest | 0 | ❌ |
| Writable cron / systemd timers | No obvious writable scheduled task | Low | ❌ |
| Kernel exploit | Out of scope, no hint in the box | High | ❌ |

### 4.3 The attack (shape only)

The Node Inspector protocol is JSON-RPC over WebSocket. The relevant
method is `Runtime.evaluate`, which takes a JavaScript expression and
returns its value from inside the running process.

Pseudo-flow (not the real request — go read the DevTools Protocol spec
to author your own):

```text
1. GET http://127.0.0.1:9229/json           # discover the websocket URL
2. Open the websocket
3. Send {"id":1, "method":"Runtime.evaluate",
        "params":{"expression": "<spawn a shell or read /root/...>"}}
4. Receive the result
```

Concretely, you can either:

- **Read the flag** by evaluating a `require('fs').readFileSync(...)`
  expression that returns the file contents, **or**
- **Get an interactive root shell** by evaluating
  `process.mainModule.require('child_process').spawn(...)` with a
  reverse-shell payload

Either works. The flag-read approach is cleaner for HTB and avoids
spraying shells.

> 🔬 **Stabilization note:** The inspector is sensitive to the calling
> TTY. If your first attach drops the request silently, retry over
> `ssh -T` (no PTY) or pipe your client through a script — the DevTools
> protocol does not need a real terminal.

---

## 5. Full Attack Chain (one-pager)

```
┌────────────────────────────────────────────────────────────────────┐
│ Phase 0 — Recon                                                    │
│   nmap -p-  → 22/ssh, 3000/http (Next.js)                          │
│   banner grab  → X-Powered-By: Next.js 15.x                        │
│   chunk dump   → confirms vulnerable minor version                 │
├────────────────────────────────────────────────────────────────────┤
│ Phase 1 — Foothold (pre-auth)                                      │
│   React2Shell (CVE-2025-55182) against :3000                       │
│   → RCE as `node`                                                  │
├────────────────────────────────────────────────────────────────────┤
│ Phase 2 — Credential discovery                                     │
│   /opt/reactor-app/reactor.db → users table → MD5 hash             │
│   hashcat -m 0 → cracked                                          │
├────────────────────────────────────────────────────────────────────┤
│ Phase 3 — Lateral                                                  │
│   ssh engineer@<target> using recovered password                   │
├────────────────────────────────────────────────────────────────────┤
│ Phase 4 — Privilege escalation                                     │
│   ss -tlnp → 127.0.0.1:9229 owned by root node --inspect           │
│   attach via DevTools protocol → Runtime.evaluate → root code exec │
└────────────────────────────────────────────────────────────────────┘
```

---

## 6. Key Lessons

1. **Read the version, then read the advisory.** A `X-Powered-By`
   header and a chunk hash are sometimes all you need to pick the
   right public exploit.
2. **App working directories are gold.** RCE + `ls` + `sqlite3` is
   the entire credential path on most web-boxes. Don't tunnel
   straight to a reverse shell.
3. **Unsalted single-round MD5 is not a password, it's a search
   query.** Any hash in this format on a public-facing app is
   compromised by definition.
4. **`--inspect` on a root-owned process is a root shell.** Treat
   any open inspector port on a multi-user box the same way you'd
   treat an unauthenticated SSH key in `/root/.ssh/authorized_keys`.
5. **`127.0.0.1` is not a security boundary** against local users.
   It's a *network* boundary, nothing more. If your privesc model
   relies on "only root can reach the socket", you don't have a
   privesc model.

---

## 7. Hardening Notes (defender perspective)

| Layer | Fix |
|---|---|
| Next.js | Patch to a non-vulnerable release; disable RSC on public-facing apps that don't need it; put the app behind a reverse proxy that strips `Next-Action` from untrusted origins |
| Database | Use bcrypt / argon2 with a per-row salt; never store a hash that pre-dates 2010's standards |
| Node Inspector | Never run with `--inspect` in production. If you must, require `NODE_OPTIONS=--inspect-brk=false --require=./auth.js` and gate the port behind a Unix socket + group ACL |
| Process model | Run the app and any dev tools under a dedicated low-priv user; never share a UID between an app and a debugging surface |

---

## 8. References

- CVE-2025-55182 advisory and patch diff
- Next.js security best-practices (App Router, RSC)
- "React2Shell" — public PoC for CVE-2025-55182 (search the GitHub advisory thread)
- Node.js Inspector / DevTools Protocol spec
- HackTricks: *NodeJS Inspector* abuse
- PayloadsAllTheThings: *Hashcat example hashes* (mode 0)

---

*Written for educational purposes. Don't paste this into a script and
expect a flag — that's not the point.*
