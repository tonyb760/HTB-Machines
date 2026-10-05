# 🛫 HTB Touch — Spoiler-Free Progressive Guide

> **Progressive hint sheet for Hack The Box — Touch**  
> No flags. No exploit payloads. No direct credential dumps.  
> **Open only the hints you need, then stop reading.**

---

## 🎯 Objective

This guide is designed to **nudge you toward the intended attack path** without turning the machine into a copy/paste walkthrough.

The hints become progressively more revealing.

> [!TIP]
> Start with **Hint 1** and work downward only when you're genuinely stuck.

---

## 🧳 What You Should Already Know

If you completed **Layover**, you should have picked up two useful pieces of information:

| Context | Value |
|---|---|
| 👤 Passenger | `Jenny Crawford` |
| 🎫 Booking Code | `KS7X2M` |

Don't assume they're simply credentials.

Instead, think about what **customer, booking, or device context** they could help you uncover on Touch.

---

# 🔎 Phase 1 — Reconnaissance

### Your first goal

Identify the service that appears most closely connected to the airport/device ecosystem.

<details>
<summary><strong>💡 Hint 1 — Where should I focus?</strong></summary>

<br>

Pay particular attention to the web service running on:

```text
TCP/8443
```

Test the protocol rather than assuming it.

A TLS-related failure does **not** necessarily mean there isn't a useful web application listening there.

</details>

---

<details>
<summary><strong>💡 Hint 2 — Is authentication my first obstacle?</strong></summary>

<br>

The service identifies itself as:

```text
Nexion DeviceHub
```

Before spending too much time attacking the login page, enumerate the application's **API routes**.

Ask yourself:

- Are all endpoints authenticated?
- Can you retrieve device status?
- Can you retrieve inventory information?
- Does the API expose identifiers that the UI normally hides?

Device metadata may reveal more than it initially appears to.

</details>

---

# 🔐 Phase 2 — DeviceHub Authentication

Once you've identified the exposed information, compare it with how the login system behaves.

<details>
<summary><strong>💡 Hint 3 — How do I reach the dashboard?</strong></summary>

<br>

Look closely at the identifiers exposed through the DeviceHub API.

Then test whether the authentication mechanism makes an unsafe assumption about one of them.

> A **device identifier** may be trusted more than it should be.

Think weak credential design rather than password cracking.

</details>

---

# 🧩 Phase 3 — Inspect the Application

Getting into the dashboard is **not the end of the web portion**.

It's the beginning of another enumeration step.

<details>
<summary><strong>💡 Hint 4 — I'm inside DeviceHub. What now?</strong></summary>

<br>

Inspect what the application sends to your browser.

Pay particular attention to:

```text
JavaScript
Client-side configuration
Static assets
Embedded application settings
```

You are not necessarily looking for another server-side vulnerability.

Instead, ask:

> **What information has the browser already been trusted with?**

Look for information that could provide access to another exposed service.

</details>

---

# 🖥️ Phase 4 — Windows Foothold

At this stage, your attack path should start moving away from the web application.

### Intended chain so far

```text
┌──────────────────────┐
│   Nexion DeviceHub   │
│       :8443          │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   API Enumeration    │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ DeviceHub Dashboard  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Browser-side Secrets │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│      RDP :3389       │
└──────────────────────┘
```

<details>
<summary><strong>💡 Hint 5 — What do I do with the account I found?</strong></summary>

<br>

The disclosed account belongs to another exposed service:

```text
TCP/3389
```

Think **RDP**.

Once connected, don't expect a normal unrestricted Windows desktop.

You'll land in a **restricted kiosk environment**, and the airport check-in application is part of the intended path.

> [!NOTE]
> Repeated password guessing against RDP is unlikely to move you forward.

</details>

---

# 🛂 Phase 5 — Escaping the Kiosk Trail

The kiosk itself contains another clue.

<details>
<summary><strong>💡 Hint 6 — Where is the user flag?</strong></summary>

<br>

Interact with the airport check-in workflow.

You'll eventually encounter a problem involving the:

```text
Badge Reader
```

The device isn't working correctly.

Instead of treating the error as a dead end:

1. Read the error carefully.
2. Follow the support path it provides.
3. Inspect the browser environment.
4. Inspect the local user context already available to you.

The **user flag** belongs to the currently logged-in Windows account and is located on that user's desktop.

</details>

---

# 🧭 Keep the Attack Path Focused

If you're getting lost in enumeration, return to this chain:

```text
8443
  ↓
Nexion DeviceHub
  ↓
API information disclosure
  ↓
Weak DeviceHub authentication
  ↓
Dashboard
  ↓
Browser-delivered information
  ↓
RDP
  ↓
Windows kiosk
  ↓
Local foothold
```

### Things you probably don't need

Avoid burning time on:

- RDP password spraying
- unrelated network services
- repeated brute-force attempts
- hunting for another web RCE after reaching DeviceHub
- attacking services before inspecting what the browser already knows

> [!IMPORTANT]
> Touch rewards **following information between systems** more than blindly attacking every exposed port.

---

# 👑 Phase 6 — Privilege Escalation

Once you have an interactive Windows foothold, the machine changes character.

Your next objective is:

```text
Kiosk User
     ↓
Local Application
     ↓
Database
     ↓
Privileged Service Context
     ↓
SYSTEM
```

---

<details>
<summary><strong>💡 Hint 7 — Where should I look after getting a shell?</strong></summary>

<br>

Get an interactive command prompt as the kiosk account.

Then investigate the local database environment.

Look for:

```text
MySQL
```

Pay attention not only to the database itself, but also to:

- how the service is configured;
- which Windows identity runs it;
- how the Airways application connects to it.

The MySQL service operates under a **significantly more privileged Windows identity** than your kiosk user.

That difference matters.

</details>

---

<details>
<summary><strong>💡 Hint 8 — How do I authenticate to MySQL?</strong></summary>

<br>

Explore the Airways application's configuration data.

A particularly interesting location is:

```text
ProgramData
```

Look for application configuration as well as readable:

```text
refresh
sync
configuration
```

files.

One of these contains credentials used by the application to communicate with the local database.

> [!TIP]
> The database does not need to be remotely exposed for it to be useful.

The service is reachable **locally from the compromised Windows machine**.

</details>

---

# ⚙️ Phase 7 — Database to SYSTEM

At this stage, think less about extracting application data and more about what the **database service itself can do on the operating system**.

<details>
<summary><strong>💡 Hint 9 — What's the privilege-escalation primitive?</strong></summary>

<br>

The obvious MySQL file read/write approach is restricted by the server's file import/export configuration.

Don't stop there.

Instead, inspect the MySQL:

```text
plugin directory
```

Then check its filesystem permissions.

Consider the combination:

```text
Writable MySQL plugin directory
             +
MySQL running as LocalSystem
             =
       Interesting
```

Research **MySQL User-Defined Functions (UDFs)** and how native plugins can expose operating-system command execution through SQL.

</details>

---

<details>
<summary><strong>💡 Hint 10 — How do I confirm SYSTEM access?</strong></summary>

<br>

You're looking for a **64-bit MySQL-compatible system UDF**.

Once loaded, verify the execution context before doing anything else.

For example, conceptually your first question should be:

```text
Who is executing this command?
```

If you've followed the privilege boundary correctly, the resulting execution context should reflect the Windows identity used by the MySQL service.

From there, the Administrator desktop becomes accessible through the SQL-backed execution primitive.

> [!TIP]
> Windows path escaping can become annoying when passing commands through SQL.
>
> Encoding the command representation first can help avoid problems caused by backslashes and nested quoting.

</details>

---

# 🏁 Full Intended Route

If you've finished the box, the overall chain should make sense as:

```text
Layover clue
    │
    ▼
Touch reconnaissance
    │
    ▼
TCP/8443
    │
    ▼
Nexion DeviceHub
    │
    ▼
Unauthenticated API enumeration
    │
    ▼
Device information disclosure
    │
    ▼
Weak DeviceHub authentication
    │
    ▼
Authenticated dashboard
    │
    ▼
Client-side JavaScript / configuration
    │
    ▼
RDP credentials
    │
    ▼
TCP/3389
    │
    ▼
Restricted Windows kiosk
    │
    ▼
Badge-reader / support trail
    │
    ▼
User foothold
    │
    ▼
Airways ProgramData
    │
    ▼
Database credentials
    │
    ▼
Local MySQL
    │
    ▼
Writable plugin directory
    │
    ▼
MySQL UDF
    │
    ▼
LocalSystem command execution
    │
    ▼
Administrator context 👑
```

---

## 🧠 Lessons From Touch

Touch is a nice example of how a compromise can emerge from several individually small weaknesses.

The important ideas are:

| Stage | Security Lesson |
|---|---|
| 🌐 DeviceHub | APIs can disclose data the UI hides |
| 🔑 Authentication | Identifiers should never double as secrets |
| 📦 Frontend | Anything shipped to the browser should be considered exposed |
| 🖥️ Kiosk | Restricted interfaces don't necessarily create a strong security boundary |
| 📁 Configuration | Application directories frequently expose reusable credentials |
| 🗄️ Database | Local-only services still matter after gaining a foothold |
| 🔧 Permissions | Writable directories used by privileged services are dangerous |
| 👑 Privilege | Service identity determines the impact of service compromise |

> **Core takeaway:**  
> The interesting part of Touch isn't one spectacular exploit. It's the way **trust leaks from one component into the next** until a low-impact disclosure becomes full system compromise.

---

## ⚠️ Instance Note

> [!WARNING]
> The **user flag** was refreshed on instance:
>
> ```text
> 10.129.100.245
> ```
>
> The recorded **root escalation** was completed on the earlier instance:
>
> ```text
> 10.129.100.153
> ```
>
> HTB instance resets may regenerate flags or change instance-specific state.

---

### ✅ Spoiler Policy

This guide intentionally avoids:

- exact credentials;
- flags;
- ready-to-run exploit commands;
- exploit payloads;
- direct copy/paste privilege-escalation commands.

If a hint gives you enough direction, **close it and keep hacking.** 🏴
