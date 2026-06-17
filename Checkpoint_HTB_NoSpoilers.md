# 🛡️ Hack The Box — Checkpoint

> **Status:** Completed  
> **Platform:** Hack The Box  
> **Category:** Windows / Active Directory  
> **Write-up type:** Public, non-spoiler reflection  

<p align="center">
  <img alt="HTB" src="https://img.shields.io/badge/Hack%20The%20Box-Checkpoint-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black">
  <img alt="Windows" src="https://img.shields.io/badge/Windows-Active%20Directory-0078D4?style=for-the-badge&logo=windows&logoColor=white">
  <img alt="No Spoilers" src="https://img.shields.io/badge/No%20Spoilers-Safe%20Review-blueviolet?style=for-the-badge">
</p>

---

## ✨ Overview

**Checkpoint** was a polished Windows/Active Directory machine that rewarded patience, clean enumeration, and careful validation of assumptions.

Rather than being a box where one obvious exploit immediately leads to the finish line, Checkpoint felt like a layered investigation. Each step provided a small piece of the bigger picture, and progress came from connecting those pieces instead of forcing a single technique repeatedly.

This post intentionally avoids flags, credentials, hashes, object names, exact commands, and exploitation steps.

---

## 🧭 High-Level Journey

```text
Initial Access
      ↓
Domain Enumeration
      ↓
Access Review
      ↓
Controlled Execution Path
      ↓
Privilege Escalation Research
      ↓
Evidence Review
      ↓
Administrative Access
      ↓
Root
```

The solve path involved several common enterprise themes:

- Understanding Active Directory relationships
- Validating account privileges instead of assuming them
- Reviewing available evidence with purpose
- Handling tooling edge cases
- Knowing when to stop repeating a failing path and pivot

---

## 🔍 What Made It Interesting

### 1. Enumeration mattered

Checkpoint emphasized that enumeration is not just about collecting output. The important part was interpreting what the data implied:

- Which users or groups mattered?
- Which access looked intentional?
- Which permissions were actually useful?
- Which paths were dead ends despite looking promising?

The box rewarded taking notes and building a mental map of the environment.

---

### 2. The obvious route was not always the correct one

A few paths looked attractive at first but did not lead directly to progress. The useful lesson was to treat failures as evidence:

```text
Failed attempt → verify assumptions → inspect permissions → choose a new branch
```

This is especially true in Active Directory environments, where similar-looking permissions can have very different practical impact.

---

### 3. Tooling required adjustment

Some tooling behaved differently than expected, and not every failure meant the idea was wrong. In a couple of cases, the concept was sound, but the implementation details needed adjustment.

That made the box feel closer to a real assessment, where you often have to understand both the technique and the tool you are using.

---

### 4. Evidence was part of the story

Checkpoint included useful evidence that was easy to overlook if you only focused on live services. The final stretch required treating available files as evidence and asking:

- Why is this here?
- Who can access it?
- What does it reveal about the environment?
- Can it be analyzed safely and efficiently?

That evidence-analysis angle was one of the most enjoyable parts of the box.

---

## 🧠 Lessons Learned

### ✅ Verify both sides of a relationship

In AD, it is common to find a relationship that looks useful from one perspective but is incomplete from another. Always verify the full chain before assuming a privilege escalation path is valid.

---

### ✅ Validate credentials and access with low-noise checks

Before going deeper, confirm what an account can actually do:

- Can it authenticate?
- Can it list shares?
- Can it execute remotely?
- Is it local admin or only a remote management user?
- Does it have a specific privilege or only group membership that sounds useful?

Small validation checks can save a lot of time.

---

### ✅ Do not over-download or over-exploit by default

When large artifacts are available, inspect metadata and choose the smallest useful path first. Efficient evidence handling matters, even in a lab.

---

### ✅ Keep a clean troubleshooting log

Several attempted paths did not work immediately. Tracking errors, assumptions, and branch decisions made it much easier to avoid loops and return to promising leads later.

---

## 🧰 Skills Practiced

- Windows service enumeration
- Active Directory privilege analysis
- SMB share review
- Kerberos-aware troubleshooting
- Remote access validation
- Evidence triage
- Credential hygiene concepts
- Access validation concepts
- Methodical note-taking and branch control

---

## 🧪 My Favorite Part

The best part of Checkpoint was how the final path felt earned. It was not about firing a single exploit and collecting a flag. It required:

1. Finding the right access
2. Understanding why that access mattered
3. Fixing incorrect assumptions
4. Reading artifacts carefully
5. Validating the result cleanly

That kind of progression is exactly what makes AD-focused boxes fun.

---

## 🧾 Public Notes

This repository version is intentionally spoiler-free.

It does **not** include:

- Flags
- Passwords
- NTLM/AES hashes
- Exact object names from the escalation path
- Full exploitation commands
- Direct root path instructions

If you are working through the box, my recommendation is to slow down and document every permission or artifact you find. The path becomes much clearer once the environment is mapped properly.

---

## 🏁 Final Thoughts

Checkpoint was a strong reminder that Active Directory attacks are often about chaining small, defensible observations rather than relying on a single obvious vulnerability.

Great box, great learning value, and a very satisfying finish.

---

## 🏷️ Tags

`HackTheBox` `ActiveDirectory` `WindowsSecurity` `Authentication` `SMB` `PrivilegeEscalation` ``RedTeam` `Pentesting` `CyberSecurity`

---

<p align="center">
  <b>Completed with notes, patience, and a lot of assumption-checking.</b>
</p>
