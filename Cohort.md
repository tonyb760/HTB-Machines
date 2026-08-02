# THE COHORT EXPERIMENT

*A field journal, lightly redacted.*

> **Spoiler level: 🌫️ — a fog, not a map.**
> This box is still alive. Everything here is a hint, a whisper, or a feeling.
> The exact commands, versions, and mechanics are withheld until retirement.
> If you want to solve it yourself, read the *Field Notes* at the end and stop there.

---

## 01 — The Experiment Begins

They call it a **cohort** — a group of organisms observed over time, sharing
something in common. In this case, the cohort is a small cluster of processes
on a quiet Linux machine, and someone forgot to tell them the experiment ended.

Three doors stand open on the street:
a **shell** (guarded, but present),
a **web server** (twice — once with a letter, once with armor).

The interesting one wears armor. TLS. A single vhost, hidden behind the
entrance, answers only if you know its name.

*Recon is just learning to ask the right questions in the right voice.*

---

## 02 — The Oracle

Every lab has an oracle. This one asks for a URL and — generously — goes to
look at it for you.

It believes what you tell it. It will visit *localhost* if you ask nicely,
because localhost is just another address, and the oracle is not the suspicious
type.

**Lesson:** when a service fetches URLs on your behalf, the map of the world
you're allowed to see is smaller than the map of the world it can see.
Ask it about *its* world.

The oracle's answer includes a **name**. Names are the keys to vhosts, and
vhosts are the keys to rooms.

*One host header, whispered correctly, changes everything.*

---

## 03 — The Notebook

Behind the name: a **notebook**.

Not paper. Code. A Python notebook that runs in the browser — cells, plots,
variables, all of it alive. The kind of tool a researcher leaves running while
they go get coffee.

It has a terminal built in. And here's the quiet horror of research
infrastructure: **the terminal does not check who you are.**

You're a user now. Not a loud one — a *marimo*, floating in the current.
But a user.

```
$ id
uid=1000(marimo) ...
```

*Foothold achieved. The experiment has a new test subject.*

---

## 04 — The Held Specimen

Inside the machine, you start looking at what's pinned to the wall.

Most specimens are clean — patched, current, boring. The kernel: boring.
The disk tools: boring. The container snap: boring.

But one jar on the shelf is different. A **package manager's daemon**, running
as **root**, listening on the system bus — and it's been *held* in place.
`hold`. Someone deliberately pinned a vulnerable version and told apt:
*do not touch this one. it is important.*

A version from spring 2024. A bug in it from that same spring.
The fix exists. The machine just never got it.

**Lesson:** on a box, a package in `hold` status is a signpost with a
floodlight on it. Box authors don't hold packages by accident — they hold
*the branch you're supposed to take*.

---

## 05 — The Race

And now, the heart of it. The daemon has a flaw that only exists in the
**space between two messages**.

It keeps notes about what a task is allowed to do. The notes are written
*once, eagerly* — and read *later, lazily*. Two different moments in time.
If you can send **two requests** so close together that the second one rewrites
the notes *before* the first one is acted on…

…then you can ask the daemon, in the same breath:

> *"Simulate this. Just pretend. No permission needed, right?"*

and then, before it can answer:

> *"…actually, do it for real."*

The daemon is a system that trusts the *last word* more than the *first
promise*. It performs the second request with the permission of the first.
That is the whole trick. That is the whole box.

Getting the two messages to land "in the same breath" took three tries:

1. **One at a time** — polite, sequential. The daemon finished the first
   request before the second arrived. No race. No dice.
2. **Fire and forget** — shouted both at once, but *forgot to listen*,
   so the words never actually left my mouth. They sat in a queue, unspoken.
3. **Two words, one heartbeat** — both messages sent together, with an
   event loop keeping the channel alive. The daemon heard them back-to-back,
   exactly as intended.

*The first attempt of the third try won.*

A small, crafted package — a **post-install script** wearing a disguise —
ran as **root**.

```
POSTINST_RAN_AS=0
```

And that was the end of the experiment. Or rather: the beginning of the end.

---

## 06 — Results

Two flags were recovered and submitted. Both accepted.

```
USER: fda8…6331
ROOT: 507d…7ee6
```

*(Redacted. Go find your own.)*

The malicious package, the SUID shell, the race scripts — all cleaned up
afterward. The lab is tidy. The cohort is complete.

---

## Field Notes — the hints, collected

| Step | Hint |
|------|------|
| 1 | Three ports. One of them asks you for a URL. |
| 2 | The URL fetcher can visit its own neighborhood. Ask it about `localhost`. |
| 3 | The leak reveals a name. Use it as a `Host` header. |
| 4 | You get a notebook with a built-in terminal. It trusts you. |
| 5 | Enumeration: look for packages that are *held* in place. |
| 6 | The held package runs as root on the system bus and is a few patches behind. |
| 7 | Its flaw is a race between two messages — permissions written early, read late. |
| 8 | Send two requests in one heartbeat: one that needs no permission, one that does. |
| 9 | A fake package's `postinst` script runs as root. Say hello. |

---

## What I learned

- **A held package is a confession.** Maintainers pin what matters.
- **Race conditions are real.** TOCTOU isn't a textbook word — it's a window
  of milliseconds between *"this is allowed"* and *"this is actually running."*
- **Event loops matter.** Async calls without a main loop are words never
  spoken. The exploit was right three times; only the delivery changed.
- **Research tools are not security tools.** A notebook server with a terminal
  is a foothold with a user interface.
- **Clean up after yourself.** The lab should look untouched.

---

*Cohort — pwned 2026-08-02. Full technical writeup after retirement.*
