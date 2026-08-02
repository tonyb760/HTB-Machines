# DARKZERO//RETURNS

### A field journal from an active box — atmosphere, tools, and lessons. Zero spoilers.

> **Status:** box is active, so this journal contains **no solution details**.
> No flags, no exploit chains, no internal names, no "how I did it."
> Just the vibe, the skill map, and the habits that carried me through.

---

```text
██████╗  █████╗ ██████╗ ██╗  ██╗███████╗██████╗  ██████╗
██╔══██╗██╔══██╗██╔══██╗██║ ██╔╝██╔════╝██╔══██╗██╔═══██╗
██║  ██║███████║██████╔╝█████╔╝ █████╗  ██████╔╝██║   ██║
██║  ██║██╔══██║██╔══██╗██╔═██╗ ██╔══╝  ██╔══██╗██║   ██║
██████╔╝██║  ██║██║  ██║██║  ██╗███████╗██║  ██║╚██████╔╝
╚═════╝ ╚═╝  ╚═╝╚═╝  ╚═╝╚═╝  ╚═╝╚══════╝╚═╝  ╚═╝  ╚═════╝
```

---

## The brief

| Field | Value |
|---|---|
| Platform | Hack The Box |
| Rating | Hard |
| Theme | A campaign platform with a developer who left things unfinished |
| Scale | Multi-host · Linux and Windows · two separate AD forests |
| Mood | One identity quietly handing you the next one |

The story, as best as I can tell without spoiling anything: a small "campaigns"
platform ships to production with a list of features still marked **Coming Soon**.
Somewhere behind that list is a developer who broke a little more than they knew.

The box is not one vuln. It is a relay race where every checkpoint hands you
the credentials, identity, or context you need for the next hop — until the last
hop crosses a forest boundary you weren't even expecting to exist.

---

## What this box actually practices

If you attempt it yourself, here is the skill map. This is public knowledge you
can verify from any box page or dashboard — no solutions, just the syllabus.

### The skills

| Skill | Why you'll care |
|---|---|
| Template engine forensics | The app processes user input in a way that is *not* a normal string. Think about how parsers represent templates internally. |
| Reading an app like a detective | The "Coming Soon" page is not filler. It is a roadmap of what the developer built and broke. |
| Kerberos basics | Tickets, caches, keytabs, clock skew, SPNs. If you do not speak Kerberos yet, this box will teach you. |
| SSO / SPNEGO | The Git platform does not do ordinary password login for this path. Look at how the Linux host already authenticates to the domain. |
| CI/CD trust boundaries | Forked content, review events, runner scoping. Automation identities are identities too. |
| Linux → AD bridging | A Linux host participating in a Windows domain is a weird, wonderful bridge. Keytabs and Kerberos caches live in surprising places. |
| Directory permissions | The interesting AD permission here is not the loud one. It is something quieter. |
| Forest trusts and SID filtering | The final trust filter does not block everything. It blocks *categories*. Know the RID ranges. |
| SMB semantics | Sometimes the file is there and the ACL is there and the ticket is there — and the *intent flags* are the difference. |

### The habits that mattered more than any single tool

1. **Read the hints the box gives you.** The dashboard "Coming Soon" list and its
   hint lines are the closest thing to a table of contents.
2. **Follow the identity, not the port.** The chain is web → service account →
   domain user → automation runner → local admin → domain admin → other forest.
   If you think in ports, you will miss the chain.
3. **Every credential is a hypothesis, not a fact.** A password found in one
   place may be a Kerberos password, an SSH password, a Git password, or none
   of the above. Test it in the right protocol.
4. **Cache everything, label everything.** Ticket caches, keytabs, config files,
   session cookies — name them, save them, and know which identity each one
   represents.
5. **Do not panic when a tool says WRONG_REALM.** Cross-realm Kerberos has its
   own etiquette. Sometimes the tool is the wrong shape for the job and the
   native client is right.

---

## Public clues (from the dashboard — no spoilers)

The box itself publishes these hint lines. They are fair game for anyone who
has started the box:

> - Multi-host, multi-domain: you move between Linux and Windows, then between two separate AD forests.
> - Kerberos everywhere: SSPI, keytabs, ticket caches, clock skew, and SPN management are all required.
> - Subtle trust rules: the final step depends on understanding exactly which SIDs survive a "Treat as External" forest filter. Most built-in groups are blocked; one specific high-RID group is not.
> - Chained logic bugs: the CI/CD bypass requires understanding how the Git platform handles fork context, review events, and runner scoping simultaneously.
> - The template injection is not a string payload. Think about how the parser represents templates internally.
> - The Git service does not use password login. Look at how the Linux host already authenticates to the domain.
> - The build runner's home directory is not the only place its identity exists on disk.
> - The OU permission is not WriteDacl or GenericAll. It is something quieter.

That last line was the one I kept coming back to.

---

## Field notes (generic, spoiler-free)

Things I wish someone had told me before I started:

- **Spend time in the app before touching a scanner.** The web app tells you
  its own story: an unfinished feature list, a disabled input, a field the UI
  hides but the server accepts. Browsers hide more than they show.
- **Kerberos is a state machine with a clock.** If a request fails with a
  cryptic code, check your cache first, then your realm config, then your
  clock. In that order.
- **The runner is not "just CI."** A build runner with domain credentials is a
  service account with a shell. Treat automation as a full identity, because
  the box does.
- **Permissions are inherited, delegated, and nested.** The user you control
  may not be the user who owns the ACL. Effective access is what matters.
- **The forest is not the border you think it is.** Trusts exist to share —
  and filtering exists to limit *what* is shared. The interesting boundary is
  the one between "filtered" and "forgotten."
- **When a normal read fails, change the semantics, not the account.** If the
  platform has a concept of backup access, the flag may be reachable only
  through that intent — not through brute force.

---

## Toolbox glossary

| Tool / concept | One-line field note |
|---|---|
| `kinit` | Get a ticket. The beginning of most Kerberos sentences. |
| `klist` | Know which identity you are wearing right now. |
| `kvno` | Ask a service "who are you?" — and watch what realm answers. |
| `ldapsearch` | The directory will tell you almost everything, if you ask politely. |
| `bloodyAD` / `samba-tool` | Directory management from Linux. Also: permission inspection. |
| `impacket-secretsdump` | The classic "can I replicate?" question. |
| `hashcat` | Some secrets are stored as work, not as text. |
| `ss` / `nc` / `curl` | Every shell begins with one of these. |
| Ticket caches & keytabs | The Linux side of Windows authentication. Respect them, label them. |

I will not tell you which of these appear in the solve or in which order. That
is the spoiler.

---

## The no-spoiler promise

This box is still active, and half the fun is that the answer does not look
like the question. Publishing the solve now would rob every future player of
the best parts: the "wait, the *field* accepts that?" moment, the "this CI
runner is a *person*?" moment, and the "there are *two forests*?" moment.

So:

- **No flags.** They belong to the box until it retires.
- **No commands that constitute the chain.** Generic tools and habits only.
- **No internal names.** The domain structure is part of the discovery.

A complete walkthrough — the full chain, the commands, the flags, the
screenshots — will be published here after the box retires.

---

## Afterword

The best boxes do not test whether you know a tool. They test whether you can
read a system the way a developer left it: the unfinished feature list, the
credential that authenticates in only one protocol, the permission that is
quiet because nobody audited it, and the trust that was designed to share —
but shared slightly too much.

DarkZeroReturns was one of those. It is a box about **identity**, in every
sense: whose key, whose token, whose permission, whose forest, whose intent.

Until the box retires — happy hunting. Stay curious, and check the
**Coming Soon** list.

---

*Field journal · DarkZeroReturns · HTB · spoiler-free edition*
*Full walkthrough follows after retirement.*
