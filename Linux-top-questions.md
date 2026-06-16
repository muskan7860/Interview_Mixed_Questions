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

*[End of Batch 1 — Q1 to Q5. Once these feel solid, say "next batch" and I'll give Q6-10.]*
