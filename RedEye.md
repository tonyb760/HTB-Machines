---
title: "Unclaimed — Spoiler-Free Flight Journal (HTB Season 12)"
published: false
tags: [hackthebox, htb, season-12, windows, medium, methodology, spoiler-free]
---

# ✈️ Flight Journal — Hack The Box: *Unclaimed*

> **Spoiler-free edition.** Season 12 · Windows · Medium · Rooted ✅
>
> No flags. No passwords. No walkthrough. Just the flight plan, the lessons, and the parts that made this box worth an evening.

```text
┌──────────────────────────────────────────────────────────────────┐
│  BOARDING PASS                                    HTB · S12      │
├──────────────────────────────────────────────────────────────────┤
│  PASSENGER : operator                                            │
│  FROM      : a terminal, somewhere                               │
│  TO        : UNCLAIMED                                           │
│  FLIGHT    : HTB-1012         CLASS : Medium / Windows           │
│  GATE      : the friendly chatbot                                │
│  SEAT      : user → root      STATUS : ROOTED ✅                 │
│  WRITE-UP  : spoiler-free while the box is active                │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🎙️ Cabin announcement

A lost-baggage assistant. A data layer with opinions. A Windows host that trusts the wrong things for the right reasons.

*Unclaimed* is one of those boxes where the story is better than the exploit. It looks like a web challenge, smells like a Windows classic, and quietly teaches a lesson about trust boundaries that applies far beyond CTFs:

> **The thing that talks to users should never be the thing that decides what they are allowed to do.**

If you like apps that reason, databases that remember, and privileges that travel quietly, this is your flight.

---

## 🗺️ Flight plan (methodology, zero answers)

| Phase | The question I kept asking | What it really tests |
| --- | --- | --- |
| **Boarding** | What is this app *actually* doing? | Recon discipline, reading over scanning |
| **In flight** | Who authorizes what — user, model, app, or data? | Trust boundaries, LLM app security |
| **Layover** | Where does input stop being data? | Data-layer reasoning |
| **Transfer** | Who am I now, and who else is onboard? | Windows identity and privilege mapping |
| **Final approach** | Which right beats an ACL? | Windows privilege model |
| **Landing** | Can I prove it *and* leave no trace? | Evidence, hygiene, discipline |

**What I will not tell you (yet):** the vulnerability, the credential, the escalation path, or the flags. The box is still active — and the next player deserves the same "oh." moment you did.

---

## 🎒 Overhead bin — what I actually used

- **A shell and `curl`** — to talk to the app like a curious passenger.
- **A data-layer client** — because every assistant has a favourite language.
- **NetExec** — to validate an identity, politely and once.
- **A remote shell** — one clean session, zero drama.
- **Notes, a timer, and a diagram** — the diagram did more work than any tool.
- **Patience** — the real exploit framework.

No zero-days. No brute force. Just reading, mapping, and one well-earned "huh."

---

## 🧠 Black box — lessons worth stealing

1. **Prompts are not permissions.**
   If a language model decides what a user may do, the model *is* your authorization layer — and that is the bug, not the clever prompt that proved it.

2. **"Read-only" describes intent, not impact.**
   A restricted role can still be loud if it can borrow someone else's context. Least privilege is about *reach*, not labels.

3. **Every escalation is an identity change.**
   Follow the identities, not the exploits. The map of *who can become whom* is usually shorter than the list of CVEs.

4. **Some Windows rights are administrator rights in a trench coat.**
   A handful of user rights exist specifically to ignore file permissions. Learn them before you learn another kernel exploit.

5. **Encrypted data is bytes, not text.**
   Length, encoding, and padding matter. A single missing zero byte can turn a valid secret into a very convincing failure.

6. **Clean up after yourself.**
   Restore the settings you changed, delete what you staged, and close the doors you opened. A good operator leaves the plane as they found it.

---

## 🌪️ Turbulence report

- **Difficulty:** fair medium. The path is logical; the noise is real.
- **Where I lost time:** assuming the obvious Windows escalation. The box politely said no — twice.
- **What unblocked me:** treating the application as a *document* instead of a target. Read what it says about itself.
- **Would I recommend it?** Yes — especially to anyone who thinks "AI feature" and "security boundary" belong in the same sentence.
- **Would I do it again?** Already did. The instance expired mid-journal, so I rebuilt the whole chain from scratch. Muscle memory is a feature.

---

## 📚 In-flight reading

- [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/)
- Microsoft Learn — *Access control and least privilege on Windows*
- Microsoft Learn — *Service accounts and service isolation*
- Microsoft Learn — *Windows security principals and user rights*
- [Hack The Box — write-up and streaming rules](https://help.hackthebox.com/en/articles/5188925-streaming-writeups-walkthrough-guidelines)

---

## 🛬 Landing report

| Item | Status |
| --- | --- |
| User flag | Claimed ✅ (value withheld) |
| Root flag | Claimed ✅ (value withheld) |
| Machine | Stopped |
| Artifacts | Removed |
| Settings | Restored |
| Write-up | Spoiler-free by design |

A clean landing is part of the flight. The flags are the souvenir; the hygiene is the skill.

---

## 🫥 The no-spoiler pledge

This journal exists for three reasons:

1. **Respect the box.** It is active. Public flags and full paths ruin the game for everyone still boarding.
2. **Respect the learner.** A hint you have to think about is worth ten copy-pasted commands.
3. **Respect the craft.** Methodology is more portable than a walkthrough.

When the machine retires, the full technical write-up goes up — with diagrams, dead ends, and the one-line fix that would have prevented the whole chain.

<details>
<summary><strong>Three spoiler-free hints (open at your own risk)</strong></summary>

1. Read the assistant's own documentation. All of it. Twice.
2. Draw the identities on paper before you run another tool — every hop changes who you are.
3. The last step is a *right*, not a bug. Look for the group whose job is to ignore permissions.

</details>

---

## 🔗 Links

- **Machine:** [Hack The Box — Unclaimed](https://app.hackthebox.com/machines/Unclaimed)
- **Author:** *add your GitHub / blog here*
- **Series:** *add your write-up index here*

---

<p align="center"><em>Flight journal by <strong>your name</strong> · 2026-11-17 · Seat 1A, somewhere over the data layer.</em></p>
