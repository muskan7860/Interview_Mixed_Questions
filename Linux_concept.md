# Linux Boot Process & Directory Structure — Easy Explanation

---

## PART 1: Boot Process (Power Button → Login Prompt)

Think of starting a Linux server like starting a car, with 5 stages — each stage finishes its job and hands control to the next, like relay runners passing a baton.

### Stage 1 — BIOS/UEFI (the ignition check)
When you press power, this is firmware built into the motherboard itself — not Linux yet. It does a basic hardware sanity check: is RAM present, is the disk present. Like turning the key and checking dashboard lights before the engine starts.

### Stage 2 — Bootloader/GRUB (the starter motor)
BIOS doesn't know what Linux is — it just knows "go look at the disk and find something to load." GRUB is a small program sitting at the start of the disk whose only job is: find the Linux kernel file, load it into RAM, and hand over control.

### Stage 3 — Kernel loads (the engine starts)
The kernel is the actual brain of Linux. It takes control of the hardware, sets up memory management, loads device drivers, and mounts the root filesystem (`/`) so the rest of the system can read files.

### Stage 4 — systemd / init starts (ignition fully on)
The kernel starts exactly ONE process — PID 1, which on modern Linux is `systemd`. This single process then starts everything else in the correct order — networking, SSH, logging, your application services — based on defined dependencies (e.g., "don't start the app until networking is up").

### Stage 5 — Login prompt (car idling, ready to drive)
Once essential services are running, you get a login screen, or SSH access starts working.

### Why This Matters Practically
If a server won't come up, knowing these stages tells you exactly where to look:
- Stuck before kernel loads → hardware or GRUB/bootloader problem
- Kernel loads but hangs after → a systemd service is hanging or failing — check `journalctl -xb` to see exactly which one

### How to Answer in Interview
> "When a Linux system boots, BIOS or UEFI first does a hardware check and hands off to the bootloader, typically GRUB. GRUB locates and loads the Linux kernel into memory. The kernel initializes hardware and mounts the root filesystem, then starts the first process, PID 1 — on modern systems that's `systemd`. Systemd then starts all other services in the correct order based on dependencies, and once essential services are up, we get a login prompt. If a server hangs during boot, I'd check which stage it's stuck at — `journalctl -xb` after the fact shows boot logs, which helps pinpoint exactly which service failed."

---

## PART 2: Linux Directory Structure

Think of the Linux filesystem as a building where every floor has one specific job — and every Linux machine in the world follows this same layout, so once you learn it, you can navigate ANY server confidently.

| Path | Easy Analogy | What It Actually Holds |
|---|---|---|
| `/` | The ground floor / whole building | Everything else lives inside this — the root of the entire filesystem |
| `/etc` | The settings office | Almost all configuration files — `/etc/nginx/nginx.conf`, `/etc/passwd` (user accounts). If you want to CHANGE behavior, you're usually here |
| `/var` | The storage room that keeps filling up | "Var" = variable — data that grows while the system runs: logs (`/var/log`), databases, caches, mail. **#1 place disk-full issues happen** |
| `/usr` | The installed software floor | Most actual programs and libraries (`/usr/bin`, `/usr/lib`). Not personal files — stands for "Unix System Resources" |
| `/tmp` | The scratch pad | Temporary files any program can dump here, usually cleared on reboot. Never store anything important here |
| `/home` | Personal user folders | `/home/yourname` — your actual personal files, like "My Documents" |
| `/root` | Admin's personal folder | Home directory specifically for the `root` (admin) user, separate from regular users |
| `/bin`, `/sbin` | Emergency toolbox | Essential command binaries needed even in recovery mode (`ls`, `cp`, `reboot`) |
| `/opt` | Third-party software shelf | Vendor software that wants to keep all its files in one self-contained folder |

### The Instinct to Build
- Disk full? → Check `/var` first
- App misbehaving? → Check its config in `/etc`
- Need to know if a tool is installed? → Look in `/usr/bin`

### How to Answer in Interview
> "`/etc` holds configuration files for the system and installed applications — that's where I'd go to change how something behaves. `/var` holds variable data that grows over time — logs, caches, spool files — and it's the most common place disk space issues come from, so it's usually my first stop when troubleshooting a full disk. `/usr` contains installed programs and their supporting libraries. `/tmp` is for temporary files that don't need to persist, and most systems clear it on reboot. Knowing this layout helps me navigate any Linux server quickly, even one I've never touched before."

---

## Quick Self-Test
1. Server hangs right after power-on, before any OS message appears — which stage is likely failing?
2. You SSH in and the disk is full — which directory do you check first, and why?
3. What's the difference between `/usr` and `/home`?
4. What is PID 1 on a modern Linux system, and what's special about it?
5. Where would you go to change Nginx's configuration?
