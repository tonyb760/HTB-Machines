# Paperwork — A Tale of Three Daemons

**Machine:** Paperwork (HTB, Easy)  
**Role:** The printer that talked too much

---

## The Premise

Some boxes hit you over the head with a vulnerable service on a silver platter. Paperwork isn't one of them. It asks you to read — not just scan, not just spray, but actually *read*. Source code, log files, socket messages, the spaces between the lines.

Three daemons. One chain. One assumption that just needed a second look.

---

## Act I — The Open Door

Port 1515 speaks a protocol from another era. A relic of the printing age, sitting wide open on the public interface. It asks for a queue name — a gatekeeper that looks like it's checking something important.

The check is the first lesson.

> *"If you're going to validate input, make sure you're actually validating something."*

An empty string behaves differently than you might expect when checked against another string in Python. This is not a bug in the language. It's a bug in the *assumption* that the check works the way it reads.

One line of shell. That's all it takes to go from "lp" to "let's see what else is here."

---

## Act II — The Printer on the Loopback

Inside the box, a second daemon listens on port 9100. It speaks JetDirect — the language of enterprise printers. It's meant to be internal, localhost-only, invisible from the outside.

But you're already inside.

This daemon thinks it's safe. It trusts the loopback interface like a diary under the bed. It serves a filesystem abstraction with a translation function that has a quiet bug — a path normalization that doesn't normalize quite enough.

The second lesson:

> *"A security boundary only works if both sides respect it."*

One daemon was supposed to be the guard. The other was supposed to be the vault. Instead, the guard handed out keys to anyone who asked the right question.

---

## Act III — The Root That Watches Everything

The third daemon runs as root. It sits on a Unix socket, monitoring everything the second daemon does. It has a security mechanism: watch for certain keywords in the logs, and if you see them, go into lockdown mode.

Lockdown mode is interesting. It gathers evidence. It bundles up file descriptors — including one to a sensitive config file — and sends them to whoever triggered the alarm.

The third lesson:

> *"A panic button that hands you the fire extinguisher *and* the safe combination is not a security mechanism — it's a delivery service."*

The config file held the password. The password unlocked root.

Three daemons. Three assumptions. Three cracks in the wall.

---

## What Makes It Great

Paperwork is Easy-rated but it's not *simple*. It rewards:

- Reading source code when you find it
- Understanding how Unix primitives (file descriptors, sockets, ancillary data) actually work
- Not stopping at the first shell
- Asking "what else is listening here?"

The box doesn't cheat. Every piece of the puzzle exists for a reason. The LPD service isn't just "a vulnerable thing" — it's a functional print spooler with a design flaw. The JetDirect daemon isn't just "a file reader" — it's a printer filesystem with a path traversal that makes perfect sense once you understand what it was trying to do. The root daemon isn't "a free flag dispenser" — it's a security monitor whose failure mode is informative.

This is what good box design looks like.

---

## Key Takeaways

1. **Know your `in` operator.** String containment checks in Python are not type-enforced. The wrong type of input makes them behave unintuitively.

2. **Localhost is not a security boundary.** Services bound to 127.0.0.1 are one `curl` away once you have any execution context.

3. **Path normalization is subtle.** `lstrip("/")` strips *all* leading slashes. `../` with no leading slash is a completely different thing.

4. **SCM_RIGHTS is powerful.** Passing file descriptors over Unix sockets is elegant and terrifying in equal measure.

5. **Read the logs.** The box tells you what it's doing. Every step of the way.

---

*"The most dangerous assumption in security is that something works the way you think it does."*

Flag count: 2/2  
Lessons learned: more than that
