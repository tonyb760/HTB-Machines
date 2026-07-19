# Bedside: A Healer's Journey

> *"Every lock has a key. Every system has a pulse. Learn to listen."*

---

## The Diagnosis

You stand before the Bedside Clinic. The doors are open, but the path forward is obscured. Your mission: find the heartbeat of this machine, slip through the cracks in its armor, and claim what lies within.

This is not a guide of copy-paste commands. It is a map of the mind — a creative walkthrough of how one might approach this clinical network without handing over the surgery on a silver platter.

---

## Phase 1: The Initial Consultation (Reconnaissance)

Before you touch anything, you observe.

Imagine yourself as a physician conducting triage. The patient presents with only a few visible symptoms. Your first task is to identify the **surface area** — what ports are listening? What services are running? What banners are they whispering?

Think of banner grabbing as taking a patient's pulse. It tells you what technology stack you're dealing with. It gives you the first clue: is this a Linux host? Windows? What version of Apache? What language is the application written in?

The key insight here: **when only one port is open, that port is your entire universe.** Every path, every secret, every vulnerability lives there. Focus all your energy on understanding that single service completely before looking elsewhere.

### The Mindset
- Enumerate thoroughly but quietly
- Record every detail, even seemingly irrelevant ones
- Look for virtual hosts — sometimes the real patient is in the room next door
- Note headers, technology fingerprints, and any developer laziness (like `X-Powered-By` headers that reveal backend frameworks)

---

## Phase 2: The Research Portal (Web Enumeration)

You've discovered a portal. It's a place where files are exchanged — medical images, research documents, perhaps more. This is your examination room.

When you encounter a file upload interface, you are looking at a **trust boundary**. The system trusts you to provide benign data. Your goal is to make it trust you with something it shouldn't.

### The Fingerprinting Game
Upload something intentionally wrong. See how the system reacts. Error messages are the system's way of talking to you — sometimes they reveal:
- What file extensions are truly allowed (not just what the UI claims)
- The underlying file paths on the server
- What backend processing occurs on uploaded files
- Whether the system performs content analysis or just extension checks

### The PDF Paradox
If the system mentions PDF processing in any capacity, you've struck gold. PDFs are not benign documents — they are complex, programmable objects that can reference external resources, execute code, and carry hidden payloads. A system that parses PDFs is a system that runs untrusted code.

Think about what happens when a PDF is opened:
- Fonts are loaded
- Images are decoded
- External references may be resolved
- Embedded streams are decompressed

Each of these actions is a **potential sink** — a place where your malicious input can become the system's output, or even its execution.

### The Approach (Without Spoilers)
1. Understand what parser is being used (version matters)
2. Research known vulnerabilities in that parser
3. Determine if the parser loads external resources
4. Consider whether path traversal is possible in resource references
5. Think about serialization — if the parser loads objects, can you control what objects are loaded?

---

## Phase 3: The Injection (Foothold)

You've found a crack. Now you need a needle.

The concept here is **deserialization abuse**. When a program converts data back into objects, it often runs code automatically — constructors, `__reduce__` methods, initializers. If you control the serialized data, you control what code runs.

### The Two-Part Weapon
Your attack needs two components:
1. **The Payload**: A serialized object that, when deserialized, executes your chosen action. Think of this as the medicine you want to administer.
2. **The Trigger**: A document or request that causes the target system to deserialize your payload. Think of this as the injection mechanism.

### The Creative Angle
Don't think in terms of "exploit scripts." Think in terms of **causality**:
- What file must exist on the server?
- What path must the server traverse to reach it?
- What document reference will cause the server to make that traversal?
- How do you encode that reference so the parser understands it?

The encoding is crucial. PDF name objects have rules about character encoding. The `#` character followed by hex digits represents another character. This is not obfuscation — it's how the format works. Understanding the format's grammar is what separates script kiddies from practitioners.

### The Waiting Game
Once you've placed your pieces, patience is required. If a background worker processes files, you must wait for it to discover and consume your bait. The worker may poll directories, check for new files, or process a queue. Understanding the timing is part of understanding the system.

---

## Phase 4: The Container (Post-Exploitation)

You've gained access. But you're not home yet.

You may find yourself in a **container** — a virtualized environment that feels like a full system but is just a guest. The walls are thinner here.

### Container Enumeration
- Check who you are (`id`, `whoami`)
- Look for `.dockerenv` or examine `/proc/self/cgroup`
- Inspect mounts — shared filesystems between host and container are bridges
- Check network capabilities — can you reach services that are filtered from the outside?

### The Shared Mount Revelation
If a directory is mounted from the host into the container, changes you make there affect the host. This is not a vulnerability — it's architecture. But architecture can be exploited. If you can write to a directory that the host uses, you can influence the host's behavior.

---

## Phase 5: The Inner Sanctum (Privilege Escalation)

From the container, you discovered a service running on `localhost`. Services bound to localhost are not exposed to the world — but you are no longer "the world." You are inside.

### The Development Server
Development servers are often research tools left running. They serve content, handle requests, and sometimes trust the local environment more than they should. A development server may have:
- Directory traversal capabilities (the developer needed to access files)
- Debug endpoints (the developer needed to troubleshoot)
- Less restrictive access controls (it was never meant to face the internet)

The creative approach: use the container as a **pivot point**. Services that reject external connections may warmly welcome internal ones.

### Reading the File System
From your internal vantage point, you can now read files that would never be exposed externally. System files, user directories, configuration files, and — most importantly — SSH keys.

Think of SSH keys as master passes. If you can read a user's private key, you can become that user on the host system. No password needed. No brute force required.

### The Sudo Audit
Once you've established yourself as a legitimate user, examine your privileges:
- What can you run with `sudo`?
- Are there password-less sudo entries?
- What scripts or binaries are you allowed to execute as root?

Each sudo entry is a **contract** between the system and the user. It says: "I trust you to run this specific command as root." Your job is to determine whether that trust is well-placed.

### The Data Science Angle
If you discover a Python training script that runs as root, examine its dependencies. Does it load saved models? Does it use `torch.load()` or `pickle.load()`? If so, the same deserialization concept applies — but now with root privileges.

The creative twist: you may not be able to write the malicious file directly as your current user. But if a shared mount exists between the host and a container where you have write access, you can write the file from the container and trigger it from the host.

---

## Phase 6: The Cure (Root Access)

With root access, the system is yours. You have completed the diagnosis, administered the treatment, and now hold the keys to the clinic.

The flags are your proof of work — evidence that you understood the system deeply enough to bend it to your will.

---

## Key Lessons for the Practitioner

1. **One port is enough.** If the attack surface seems small, it means your focus must be absolute.

2. **File uploads are trust boundaries.** Every upload endpoint is a potential entry point. Understand what the server does with your files.

3. **Deserialization is code execution.** Any system that deserializes untrusted data is executing untrusted code. Find the serializer, find the exploit.

4. **Containers are not walls.** Shared mounts, network namespaces, and host filesystem access make containers permeable.

5. **Internal services are soft targets.** `localhost` services are built for convenience, not security. Once you're inside, they're yours.

6. **Sudo entries are contracts.** Read them carefully. A script that runs as root and loads files is a script that can be weaponized.

---

## Tools of the Trade (Mentioned in Principle)

- Network enumeration tools for port discovery
- Web proxies and fuzzers for endpoint discovery
- PDF analysis tools for understanding document structure
- Python for crafting serialized payloads
- Standard shell utilities for container enumeration
- SSH for authenticated access
- Standard privilege escalation enumeration scripts

---

> *"The best hackers are not those who memorize commands, but those who understand systems deeply enough to speak their language."*

**Happy hacking.**
