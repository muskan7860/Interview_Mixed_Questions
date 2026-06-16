# LINUX — TOP 20 INTERVIEW QUESTIONS (4 YOE DevOps)
*Format for every question: (1) Simple explanation with WHY it works this way, (2) What the interviewer is actually testing, (3) Polished interview answer.*

> Read ONE question at a time. Don't move to Q2 until Q1 fully makes sense. This is not a list to memorize — it's a list to understand.

---

## Q1. What is the Linux boot process? (Power button → login prompt)

### Simple Explanation (the WHY)
Think about starting a car. You don't just turn the key and the engine instantly runs — there's a sequence: battery powers up → starter motor cranks → engine ignites → dashboard lights come on → car is ready to drive.

A Linux server boots the same way, in stages:

1. **BIOS/UEFI** — This is like the car's battery check. The motherboard's firmware does a basic hardware check (is RAM there? is the disk there?) before handing control to anything else.
2. **Bootloader (GRUB)** — This is like the starter motor. It's a small program whose ONLY job is to find the actual operating system (the Linux kernel) on the disk and load it into memory.
3. **Kernel loads** — The kernel is the "engine" — the core brain of Linux. It initializes hardware drivers, sets up memory management, and gets the system into a basic working state.
4. **init / systemd starts** — This is the "ignition" — the very first process (PID 1) that starts everything else: networking, logging, SSH service, etc. On modern Linux, this is `systemd`.
5. **Runlevel/Target reached, login prompt appears** — Car is idling, ready to drive — system is now ready for you to log in.

**Why this matters in real life:** When a server won't boot or hangs during boot, knowing WHICH stage it's stuck at tells you WHERE to look. Stuck before kernel loads = hardware/bootloader issue. Stuck after kernel but before login = a service in systemd is hanging.

### What the Interviewer is Testing
They want to know if you understand *layers of responsibility* — do you blindly restart servers, or do you actually understand what "booting" means so you can troubleshoot a boot failure intelligently? This question also often leads into "what is systemd" and "how do you check why a service failed to start" — so it's a gateway question.

### How to Answer in Interview
> "When a Linux system boots, BIOS or UEFI first does a hardware check and hands off to the bootloader, typically GRUB. GRUB locates and loads the Linux kernel into memory. The kernel initializes hardware and mounts the root filesystem, then starts the first process, PID 1 — on modern systems that's `systemd`. Systemd then starts all other services in the correct order based on dependencies, and once essential services are up, we get to a login prompt. If a server hangs during boot, I'd check which stage it's stuck at — for example, `journalctl -xb` after the fact shows boot logs, which helps pinpoint exactly which service failed to start."

---

## Q2. What's the difference between a Process and a Thread?

### Simple Explanation (the WHY)
Imagine a **process** is an entire restaurant — it has its own kitchen, its own staff, its own storage, its own address. It's completely independent from the restaurant next door.

A **thread** is like a chef working INSIDE that restaurant. You can have multiple chefs (threads) working in the SAME kitchen, sharing the SAME ingredients and storage (memory), just doing different tasks at the same time — one chopping vegetables, one grilling meat. They're faster to coordinate because they share everything, but if one chef makes a mess (crashes), it can mess up the shared kitchen for everyone else.

If you wanted two completely separate restaurants instead (two processes), they'd each need their own full kitchen setup — slower to start, but if one restaurant burns down, the other is completely unaffected.

**Why this matters:** This is exactly why a crashing thread can sometimes crash the entire application, but a crashing process (like one Docker container) doesn't affect other processes/containers on the same machine.

### What the Interviewer is Testing
This tests basic OS fundamentals — they want to confirm you're not JUST a "ran commands from a wiki" DevOps person, but actually understand what's running underneath. It often comes up when discussing why an application uses high memory, or how containers provide isolation.

### How to Answer in Interview
> "A process is an independent unit of execution with its own memory space — completely isolated from other processes. A thread is a smaller unit of execution WITHIN a process, and multiple threads in the same process share the same memory space, which makes communication between them fast, but also means a misbehaving thread can affect the entire process. This is relevant in DevOps because process-level isolation is exactly why containers — which are essentially isolated processes with their own resource limits — provide much stronger isolation than just running multiple threads of an app."

---

## Q3. How do you check and kill a process by name or PID?

### Simple Explanation (the WHY)
Every running program on Linux gets a unique ID number called a PID (Process ID) — like a token number at a hospital. If a program is misbehaving (frozen, stuck, consuming resources), you need a way to find its token number and tell it to stop.

`ps aux | grep <name>` is like asking the reception desk "show me everyone with the name John" — it searches all running processes for a name match and shows you their PID.

`kill <PID>` sends a polite request — "please shut yourself down gracefully" (this is signal 15, SIGTERM — like asking someone to finish their current task and leave).

`kill -9 <PID>` is NOT polite — it's "stop immediately, no questions asked" (SIGKILL). The OS forcefully terminates it without giving the program a chance to clean up — which can sometimes leave behind incomplete files or unsaved data.

**Why this matters:** Senior engineers know to TRY `kill` (graceful) FIRST, and only use `kill -9` (forceful) if the process refuses to die — because force-killing a database process mid-write, for example, can corrupt data.

### What the Interviewer is Testing
They want to see if you know the DIFFERENCE between graceful and forceful termination — using `kill -9` as your default answer (which many juniors do) is actually a red flag, not a good answer. It signals you don't think about consequences.

### How to Answer in Interview
> "I'd use `ps aux | grep <process-name>` to find the PID, or `pgrep <name>` for a quicker lookup. To stop it, I'd first try `kill <PID>`, which sends SIGTERM — a graceful shutdown signal that lets the process clean up properly, like closing open files or finishing a database write. Only if the process doesn't respond after a few seconds would I escalate to `kill -9`, which sends SIGKILL and forcefully terminates it immediately — I treat that as a last resort since it can leave things in an inconsistent state, especially for stateful services like databases."

---

## Q4. What is a Zombie process and an Orphan process?

### Simple Explanation (the WHY)
Imagine a manager (parent process) assigns a task to an employee (child process). The employee finishes the task and reports back: "I'm done, here's the result." But the manager is busy and hasn't acknowledged/collected that report yet. The employee is technically done working, but their "exit report" is still sitting on the manager's desk, unprocessed. That employee is now a **Zombie process** — finished, but not yet cleaned up by the parent.

An **Orphan process** is the opposite situation — the employee is STILL working, but their manager suddenly quit/got fired (parent process died) before the employee finished. The employee doesn't have a manager anymore, so the very top-level boss (PID 1 / systemd/init) "adopts" them and becomes their new manager.

**Why this matters:** A few zombie processes are harmless and normal. But IF YOU SEE HUNDREDS of zombies piling up, it usually means the parent application has a bug — it's not properly "collecting" finished children, and this is a real production issue you might need to debug or restart the parent process to fix.

### What the Interviewer is Testing
This is a "do you actually know Linux internals or did you just learn `kill -9` from YouTube" question. Most junior candidates have never heard the term zombie process. Knowing this signals genuine depth.

### How to Answer in Interview
> "A zombie process is one that has finished executing, but its parent hasn't yet called `wait()` to read its exit status, so it stays in the process table with a 'Z' state until cleaned up. A few zombies are completely normal and harmless. An orphan process is the reverse case — its parent died before the child finished, so the child gets re-parented to PID 1 or systemd, which takes over as its new parent. I'd worry if I saw a large, growing number of zombies — that usually points to a bug in the parent application not properly reaping its children, and I'd flag that to the dev team since restarting the parent process is often the only fix."

---

## Q5. What's the difference between `/etc`, `/var`, `/usr`, and `/tmp`? (Linux directory structure)

### Simple Explanation (the WHY)
Think of the Linux filesystem like a well-organized office building, where every floor has a specific PURPOSE — this isn't random, it's a convention every Linux system follows so any engineer can walk into ANY Linux server and know where to look.

- **`/etc`** = the "settings/config" floor. This is where almost ALL configuration files live — network settings, user accounts, application configs (Nginx config is in `/etc/nginx/`). If you want to CHANGE how something behaves, you're usually editing something here.
- **`/var`** = the "variable data" floor — meaning data that changes/grows constantly while the system runs. Logs (`/var/log`), databases' data files, mail spools, cache. This is the floor that fills up your disk over time — which is exactly why `/var` is the #1 place disk-full issues happen.
- **`/usr`** = the "installed programs" floor — most of your actual installed software and its supporting files live here (`/usr/bin`, `/usr/lib`). Think of it as "user-installed system resources," NOT personal user files.
- **`/tmp`** = the "scratch pad" floor — temporary files that any program can write to, and which usually get cleared on reboot. Don't store anything important here.

**Why this matters:** When troubleshooting, knowing this map means you instinctively know WHERE to look. Disk full? Check `/var` first. App not behaving as expected? Check its config in `/etc`. Need to check what version of a tool is installed? Look in `/usr/bin`.

### What the Interviewer is Testing
This tests if you can navigate a Linux server confidently without hand-holding — a very practical, "have you actually worked on real servers" check.

### How to Answer in Interview
> "`/etc` holds configuration files for the system and installed applications — that's where I'd go to change how something behaves. `/var` holds variable data that grows over time — logs, caches, spool files — and it's the most common place disk space issues come from, so it's usually my first stop when troubleshooting a full disk. `/usr` contains installed programs and their supporting libraries. `/tmp` is for temporary files that don't need to persist, and most systems clear it on reboot. Knowing this layout helps me navigate any Linux server quickly, even one I've never touched before."

---

# LINUX — TOP 20 INTERVIEW QUESTIONS — BATCH 2 (Q6-10)
*Same story-style format: Easy Explanation → What Interviewer is Testing → Interview Answer*

---

## Q6. What's the difference between a Process and a Thread?

### Easy Explanation — The School Story
Think of a **Process** as an entire separate school building — School A has its own classrooms, its own staff room, its own store room, its own gate, completely separate from School B next door. If School A catches fire, School B is totally unaffected, because nothing is shared between them.

A **Thread** is like a teacher working INSIDE one school building. You can have multiple teachers (threads) in the SAME school, all sharing the SAME staff room, SAME store room, SAME supplies (this shared space is called "memory"). They can pass notes to each other instantly because they're in the same building — much faster coordination than calling a teacher in a different school entirely.

But here's the catch: if one teacher in School A causes a fire in the shared staff room, it can affect ALL the other teachers in that SAME school, because they share the same space. Meanwhile, School B (a different process) remains completely safe.

**Why this matters in real DevOps work:** This is exactly why a crashing thread inside one application can sometimes crash the WHOLE application, but a crashing process (like one Docker container) doesn't affect other separate containers running on the same machine — because containers are built around process-level isolation, the "separate school building" model.

### What the Interviewer is Testing
They want to confirm you understand basic operating system fundamentals — not just "I ran commands I copied from a tutorial," but "I actually understand what's happening underneath." This question often leads naturally into discussing why containers provide strong isolation, or why an application's memory usage behaves a certain way.

### How to Answer in Interview
> "A process is an independent unit of execution with its own dedicated memory space, completely isolated from other processes. A thread is a smaller unit of execution that runs WITHIN a process, and multiple threads in the same process share that process's memory space — which makes communication between them fast, but also means one misbehaving thread can affect the entire process. This concept is directly relevant in DevOps because containers are built on process-level isolation — that's exactly why one container crashing doesn't bring down other containers on the same host, the same way one school building catching fire doesn't affect the building next door."

---

## Q7. How do you check and stop (kill) a process by name or PID?

### Easy Explanation — The School Story
Every student in the school is given a unique roll number — that's exactly what a **PID (Process ID)** is for every running program on Linux. If a particular student is misbehaving in class (a process is frozen or eating up resources), the teacher first needs to find that student's roll number before doing anything.

`ps aux | grep <name>` is like the teacher asking the school office: "Show me every student named Rahul" — it searches through everyone currently in the school and shows you their roll numbers (PIDs).

`kill <PID>` is the teacher **politely** telling that student: "Please finish what you're doing and leave the classroom calmly." This is signal 15 (SIGTERM) — a graceful request that gives the process a chance to save its work, close files properly, and exit cleanly.

`kill -9 <PID>` is NOT polite at all — it's like calling security to physically drag the student out immediately, no warning, no chance to pack their bag. This is signal 9 (SIGKILL) — the operating system terminates the process instantly, without letting it clean up. If that student (process) was in the middle of writing something important (like a database mid-save), forcing them out abruptly can leave things half-finished or corrupted.

**The senior-level instinct:** always try the polite request (`kill`) first, and only escalate to the forceful drag-out (`kill -9`) if the process refuses to respond after a few seconds.

### What the Interviewer is Testing
This question is a trap for juniors who jump straight to `kill -9` as their go-to answer. That's actually a red flag — it tells the interviewer you don't think about consequences. They want to hear that you understand graceful vs forceful termination and WHY the order matters.

### How to Answer in Interview
> "I'd use `ps aux | grep <process-name>` to find the PID, or `pgrep <name>` for a quicker lookup. To stop it, I'd first try `kill <PID>`, which sends SIGTERM — a graceful shutdown signal that allows the process to clean up properly, like closing open file handles or completing a database write. Only if the process doesn't respond after a few seconds would I escalate to `kill -9`, which sends SIGKILL and forcefully terminates it immediately. I treat that as a last resort, since it can leave things in an inconsistent state — especially risky for stateful services like databases that might be mid-write."

---

## Q8. What is a Zombie process and an Orphan process?

### Easy Explanation — The School Story
Imagine a teacher (parent process) assigns homework to a student (child process). The student finishes the homework and submits it on the teacher's desk: "I'm done, here's my completed work." But the teacher is busy with something else and hasn't actually picked up and reviewed that submission yet. The student has technically finished their task, but their "completion report" is just sitting there, unprocessed. That student is now a **Zombie process** — finished working, but not yet acknowledged/cleaned up by the parent.

An **Orphan process** is the opposite situation — the student is STILL working on their homework, but their teacher suddenly resigns and leaves the school (parent process dies) before the student is done. The student doesn't have a teacher anymore to report to. So the Principal (PID 1, systemd) automatically steps in and "adopts" that student, becoming their new responsible authority until they finish.

**Why this matters:** A few zombie students sitting around occasionally is completely normal and harmless — every school has a few. But if you see HUNDREDS of zombie students piling up every single day, it means there's a real problem — a teacher (the parent application) has a bug and isn't properly reviewing/clearing finished students' work. This is a genuine production issue, and the fix is often restarting the parent application so it starts properly cleaning up after itself again.

### What the Interviewer is Testing
This is a "do you actually understand Linux internals, or did you just memorize `kill -9` from a YouTube video" filter question. Most junior candidates have never even heard the term "zombie process." Knowing this signals real depth of understanding, not surface-level memorization.

### How to Answer in Interview
> "A zombie process is one that has finished executing, but its parent hasn't yet called `wait()` to read its exit status, so it remains in the process table with a 'Z' state until the parent acknowledges it. A few zombies are completely normal and harmless. An orphan process is the reverse — its parent died before the child finished, so the child gets re-parented to PID 1 or systemd, which takes over as its new parent. I'd be concerned if I saw a large, continuously growing number of zombies — that usually points to a bug in the parent application not properly reaping its children, and I'd flag that to the development team, since restarting the parent process is often the practical fix."

---

## Q9. What's the Linux Directory Structure? (`/etc`, `/var`, `/usr`, `/tmp`, `/home`)

### Easy Explanation — The School Story
Think of the entire Linux filesystem as one big school building, where every floor has ONE specific, well-known purpose — and every Linux school in the world is built with the exact same floor plan, so any teacher (engineer) can walk into ANY Linux school and immediately know where everything is.

- **`/etc`** — the **Principal's office / rule book floor**. This is where all the official rules and settings are kept — how the canteen should operate, who's allowed in which room. In Linux terms, this is where almost ALL configuration files live (`/etc/nginx/nginx.conf`, `/etc/passwd` for user accounts). If you want to CHANGE how something behaves, you go here.
- **`/var`** — the **store room that keeps filling up**. "Var" stands for "variable" — meaning the contents constantly grow as the school runs day to day: daily attendance records (logs in `/var/log`), the cafeteria's used-up supplies (cache), exam answer sheets piling up (databases). This is the #1 floor that fills up and causes "we're out of space" problems.
- **`/usr`** — the **library / textbook floor**. Most of the actual installed programs and their supporting materials live here (`/usr/bin`, `/usr/lib`). Despite sounding like "user," it's not personal files — it actually stands for "Unix System Resources."
- **`/tmp`** — the **scratch notebook floor**. Temporary notes any student/teacher can quickly jot down and discard — gets wiped clean when the school restarts for a new term (reboot). Never store anything important here.
- **`/home`** — **each student's personal locker**. `/home/yourname` is your own personal space, similar to "My Documents" — your own files, separate from the school's shared resources.

**The instinct to build:** Disk full? Check `/var` first — it's almost always the cause. App misbehaving? Check its rule book in `/etc`. Need to know if a tool/program is installed? Look in `/usr/bin`.

### What the Interviewer is Testing
They want to know if you can navigate ANY Linux server confidently without hand-holding, even one you've never seen before — a very practical "have you actually worked hands-on with real servers" filter.

### How to Answer in Interview
> "`/etc` holds configuration files for the system and installed applications — that's where I'd go to change how something behaves. `/var` holds variable data that grows over time — logs, caches, spool files — and it's the most common place disk space issues come from, so it's usually my first stop when troubleshooting a full disk. `/usr` contains installed programs and their supporting libraries. `/tmp` is for temporary files that don't need to persist, and most systems clear it on reboot. `/home` holds individual users' personal files. Knowing this layout helps me navigate any Linux server quickly, even one I've never touched before."

---

## Q10. What's the difference between a Soft Link (Symbolic Link) and a Hard Link?

### Easy Explanation — The School Story
Imagine a student named Rahul has a locker (the actual file/data) somewhere in the school. The locker number on the door is like the "file name" — it's just a label pointing to where the actual stuff is kept.

A **Hard Link** is like putting up a SECOND nameplate on a COMPLETELY DIFFERENT door, but that door secretly opens into the exact SAME locker space as Rahul's original door. Both nameplates are EQUALLY real and official — there's no "original" vs "copy," they're both genuine entrances to the same storage. If you remove ONE nameplate/door, the locker contents are still safe, because the other door still leads to it. BUT, you can't do this trick across two different school buildings (filesystems), and you can't make a hard link to an entire room/section (a directory) — only to individual lockers (files).

A **Soft Link (Symbolic Link / Symlink)** is completely different — it's like a small sticky note pinned to a wall saying "Rahul's locker is actually in Room 204, Building B." This sticky note isn't a real door at all — it's just a pointer, a note with directions. It CAN point across different buildings (filesystems), and it CAN point to an entire room (a directory), not just one locker. But if Rahul's actual locker gets removed or moved somewhere else, that sticky note now points to nothing — it becomes a "dangling" note pointing at an empty wall.

**Why this matters practically:** Hard links are rarely used day-to-day, but symlinks are EXTREMELY common in DevOps — for example, pointing `/usr/bin/python` to `/usr/bin/python3.11`, so when you upgrade Python versions, you just update the symlink instead of moving actual files around.

### What the Interviewer is Testing
This is a fundamentals check — they want to see if you understand HOW Linux actually stores and references files underneath, which becomes relevant when discussing things like Docker image layers, software version management, or why deleting a "linked" file sometimes doesn't free up disk space the way you'd expect.

### How to Answer in Interview
> "A hard link is another directory entry pointing to the exact same inode as the original file — they're indistinguishable from each other, there's no original versus copy. Deleting one doesn't affect the other, since the actual data is only removed when ALL hard links to it are deleted. It can't span across filesystems and can't point to a directory. A symbolic link is a completely separate, small file that just stores a path to the target — it CAN cross filesystems and CAN point to a directory, but if the target is deleted or moved, the symlink becomes a dangling reference pointing to nothing. In practice, I use symlinks far more often — for example, managing multiple versions of a tool by pointing a generic path like `/usr/bin/python` to a specific installed version, so upgrades just mean updating the symlink."

---


