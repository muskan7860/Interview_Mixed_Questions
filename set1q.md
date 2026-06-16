# DevOps Interview Prep — Master File
*(4 YOE — Atos DBA/Junior DevOps Support → Infosys Mid-Level DevOps)*

> Structure: This file has 4 sections — **Day 1 (Linux + Networking)**, **Day 2 (Git/GitHub)**, **Mixed Round 1 (Linux + Networking + Git combined)**, and a bonus **Easy-Explanation Layer** at the very end for Day 1 topics that were confusing. Read top to bottom, or jump to whichever section you're revising.

---
---

# SECTION 1: DAY 1 — Linux Fundamentals + Core Networking

### Q1. Walk me through what happens when you type a command like `ls -l` in the terminal.
**Expected answer:** The shell parses the command, checks if it's a built-in or searches `$PATH` for the binary, forks a child process, execs the `ls` binary in that process, which makes system calls to the kernel (like `getdents`, `stat`) to read directory entries and file metadata, then writes formatted output to stdout.

### Q2. How do you check which process is consuming the most CPU/memory on a server?
**Expected answer:** `top` or `htop` for a live view; `ps aux --sort=-%cpu | head` or `ps aux --sort=-%mem | head` for a snapshot. For memory specifically, also check `free -h` for overall usage and `vmstat` for trends.

### Q3. A server's disk is almost full. How do you find what's eating space?
**Expected answer:** Start with `df -h` to see which filesystem/mount is full. Then `du -sh /*` (or `du -h --max-depth=1 /path | sort -rh`) to drill down directory by directory until you find the culprit. Common offenders: log files (`/var/log`), Docker images/containers, old kernel versions.

### Q4. Explain file permissions — what does `-rwxr-xr--` mean, and how do you change them?
**Expected answer:** First character is file type (`-` = regular file). Then three groups of three: owner (rwx = read/write/execute), group (r-x = read/execute), others (r-- = read only). Numerically that's 754. Changed with `chmod 754 file` or `chmod u+x,g-w file`. Ownership changed with `chown user:group file`.

### Q5. What's the difference between a hard link and a symbolic link?
**Expected answer:** A hard link is another directory entry pointing to the same inode — same file, no distinction between "original" and "link", can't cross filesystems, doesn't work on directories. A symlink is a separate file containing a path to the target — can cross filesystems, can point to directories, breaks if the target is deleted/moved (dangling link).

### Q6. Explain the difference between TCP and UDP, with an example of when you'd use each.
**Expected answer:** TCP is connection-oriented, reliable, ordered (uses handshake, acknowledgments, retransmission) — used for HTTP, SSH, database connections where data integrity matters. UDP is connectionless, faster, no guarantee of delivery/order — used for DNS queries, video streaming, VoIP where speed matters more than occasional packet loss.

### Q7. Walk me through what happens when you visit a URL in a browser (high level — focus on DNS and connection).
**Expected answer:** Browser checks DNS cache, then queries a DNS resolver which walks through root → TLD → authoritative nameservers to resolve the domain to an IP. Browser opens a TCP connection to that IP on port 443 (HTTPS), performs a TLS handshake to establish an encrypted channel, then sends an HTTP request and receives the response (HTML/assets).

### Q8. How do you check if a remote port is open/reachable from a server?
**Expected answer:** `telnet <host> <port>` or `nc -zv <host> <port>` for a quick check. `curl -v https://host:port` to test HTTP-level connectivity. `ss -tuln` or `netstat -tuln` to see what's listening locally. If it's failing, also check security groups/firewall rules (`iptables -L` or cloud security groups).

### Q9. What's the difference between `systemctl restart` and `systemctl reload` for a service?
**Expected answer:** `restart` stops the process entirely and starts a new one — connections drop, brief downtime. `reload` sends a signal (usually SIGHUP) to the running process to re-read its config without dropping connections — used for services like Nginx where you want zero-downtime config changes. Not all services support reload.

### Q10. Scenario: You SSH into a server and it's extremely slow to respond. What's your troubleshooting approach?
**Expected answer:** Check `uptime` for load average, `top`/`htop` for CPU and memory hogs, `free -h` for memory/swap usage (heavy swapping is a common cause), `df -h` for disk space, `dmesg | tail` for kernel-level errors (OOM killer events are common), and `iostat`/`vmstat` for disk I/O bottlenecks. Also check network with `ss -s` for connection counts — sometimes it's a flood of connections, not a resource issue.

---
---

# SECTION 2: DAY 2 — Git & Branching Strategies

### Q1. What's the difference between `git merge` and `git rebase`?
**Concept:** Merge takes two branches and creates a new "merge commit" that ties their histories together — keeps full history but can get messy with lots of merge commits. Rebase takes your commits and replays them on top of the latest target branch, making history look linear and clean — but it rewrites commit history, which is dangerous on shared branches.
**Interview answer:** "Merge preserves the complete history and is safe for shared branches. Rebase rewrites commit history to create a linear timeline — I use it for cleaning up my local feature branch before raising a PR, but never on branches others are working on, since rewriting shared history causes conflicts for everyone else."

### Q2. Explain Git branching strategies — Gitflow vs trunk-based development.
**Concept:** Gitflow has multiple long-lived branches (main, develop, feature/*, release/*, hotfix/*) — good for projects with scheduled releases. Trunk-based development means everyone works off `main` with short-lived feature branches merged quickly, often combined with feature flags — better for continuous delivery.
**Interview answer:** "In my experience, we followed [Gitflow/trunk-based — pick whichever you actually used]. Feature branches were created from `develop`, merged via PR after review, and releases were cut from `release/*` branches into `main`. For hotfixes, we branched directly off `main` and merged back into both `main` and `develop`."

### Q3 — SCENARIO (important): "You pushed a commit to `main` that broke production. The pipeline already deployed it. Walk me through what you do — right now, in order."
**Concept:** Tests calm-under-pressure + Git knowledge + process awareness. Priority order: stop the bleeding (rollback/revert) → communicate → fix properly → prevent recurrence.
**Interview answer:** "First priority is restoring service, not fixing the root cause immediately. I'd run `git revert <commit-hash>` to create a new commit that undoes the bad change — safer than `reset` because it doesn't rewrite history others may have pulled. Push that revert, let the pipeline redeploy. In parallel, I'd notify the team/on-call that a rollback is in progress. Once production is stable, I'd investigate the actual bug on a separate branch, fix it properly, and add a test or pipeline check to catch this category of issue going forward."

### Q4 — SCENARIO: "Two developers modified the same file and now there's a merge conflict. How do you resolve it?"
**Concept:** Git marks conflicting sections with `<<<<<<<`, `=======`, `>>>>>>>` markers showing both versions. You manually edit the file to decide the final code, remove the markers, then stage and commit.
**Interview answer:** "I'd open the conflicted file, review both versions marked by Git's conflict markers, and discuss with the other developer if unclear which change should win. After editing to the correct final state, I run `git add <file>` and `git commit`. For complex conflicts, I might use `git mergetool` or VS Code's merge editor."

### Q5 — SCENARIO: "You accidentally committed a file containing AWS credentials and pushed it to a public repo. What do you do?"
**Concept:** Tests security awareness — removing it from the latest commit isn't enough; it's still in history and could already be scraped by bots.
**Interview answer:** "First, I'd immediately rotate/revoke those credentials in AWS — assume they're compromised the second they're pushed. Then remove the file and add it to `.gitignore`. To remove it from Git history, I'd use `git filter-repo` (or `BFG Repo-Cleaner`) to scrub it from all commits, then force-push — only after coordinating with the team. Going forward, I'd add secrets-scanning to the CI pipeline (`git-secrets` or `trufflehog`)."

### Q6. What's `git stash` and when would you use it?
**Concept:** Stash temporarily shelves uncommitted changes so you have a clean working directory, without committing half-done work.
**Interview answer:** "If I'm mid-way through a change and need to quickly switch branches — say to fix an urgent bug — I run `git stash` to save my work-in-progress, switch branches, do the fix, switch back, and run `git stash pop` to restore my changes."

### Q7 — SCENARIO: "How would you find which commit introduced a specific bug, out of hundreds of commits?"
**Concept:** `git bisect` does a binary search through commit history — you mark a known-good and known-bad commit, it checks out the midpoint, you test and tell it good/bad, repeats until it finds the exact commit.
**Interview answer:** "I'd use `git bisect`. I mark a known-good commit and the current known-bad commit, Git checks out the midpoint automatically, I test and run `git bisect good` or `git bisect bad`, and it repeats — narrowing down in log(n) steps instead of checking every commit manually. Once found, `git bisect reset` ends the session."

---
---

# SECTION 3: MIXED ROUND 1 — Linux + Networking + Git/GitHub Combined

### Q1 (Linux/Scenario). "A deployment script that worked yesterday fails today with 'Permission denied.' Nothing in the script changed. What do you check?"
**Concept:** Permissions issues often come from things outside the script — file ownership changed, the script lost its execute bit, SELinux/AppArmor policies, or the user running it changed (e.g., cron runs as a different user than your shell).
**Interview answer:** "I'd first check `ls -l` on the script to confirm the execute bit (`x`) is still set and ownership is correct. Next, I'd check which user is running it — if it's via cron, cron might run as a different user with fewer privileges than my interactive shell. I'd also check if it's writing to a directory it no longer has access to. If on a system with SELinux, I'd check `getenforce` and `audit.log` for denials."

### Q2 (Networking/Scenario). "Your application can't connect to the database, but `ping` to the DB server works fine. What's your next step?"
**Concept:** `ping` only tests ICMP (basic reachability) — doesn't confirm the actual service/port is open or that the application can authenticate.
**Interview answer:** "Since ping works, basic network connectivity and routing are fine, so I'd move up the stack. I'd use `nc -zv <db-host> <port>` or `telnet <db-host> 1433/3306` to check if the database port itself is reachable — if that fails, it's likely a security group or firewall blocking that specific port. If the port is open, I'd check the application's connection string/credentials, and check the DB server's logs for failed connection attempts."

### Q3 (Git/GitHub). "What's the difference between `git fetch` and `git pull`?"
**Concept:** `fetch` downloads new commits from the remote but doesn't touch your working files or current branch. `pull` is `fetch` + `merge` (or rebase) — it actually updates your current branch.
**Interview answer:** "`git fetch` retrieves the latest changes without modifying my working directory — a safe way to see what's changed before integrating it. `git pull` does a fetch and then immediately merges those changes into my current branch. I usually prefer fetch first when working on something important, so I can review incoming changes before merging."

### Q4 (GitHub/Scenario). "Walk me through how you'd set up branch protection on `main` so nobody can directly push broken code to production."
**Concept:** GitHub branch protection rules enforce that changes to a branch only happen through reviewed, tested pull requests.
**Interview answer:** "I'd enable 'Require pull request before merging' so no one can push directly. I'd require at least one approving review, require status checks to pass — meaning the CI pipeline must succeed before merge. I'd also enable 'Require branches to be up to date before merging,' and optionally restrict who can merge to senior team members for production branches."

### Q5 (Linux/Scenario). "You need to find all files modified in the last 24 hours under `/var/log`. How?"
**Concept:** The `find` command with time-based filters is one of the most practically useful Linux tools for troubleshooting "what changed recently."
**Interview answer:** `find /var/log -type f -mtime -1` — finds regular files modified within the last 1 day. "If I needed files modified in the last hour during active troubleshooting, I'd use `-mmin -60` instead."

### Q6 (Networking). "Explain what a load balancer does and the difference between Layer 4 and Layer 7 load balancing."
**Concept:** A load balancer distributes incoming traffic across multiple servers so no single server is overwhelmed, and provides failover. Layer 4 operates at TCP/UDP level (forwards based on IP/port), Layer 7 operates at the application level (HTTP) and can route based on URL paths, headers, cookies.
**Interview answer:** "A load balancer sits in front of servers and distributes incoming requests based on health and load. Layer 4 load balancing works at the transport layer — routes based on IP/port without understanding traffic content, so it's faster but less flexible. Layer 7 works at the application layer — for HTTP, it can route based on URL path, inspect headers, do SSL termination — this is AWS's ALB, versus NLB which is Layer 4."

### Q7 (Git/Scenario). "You need to undo the last 3 commits, but you've already pushed them to the remote branch others are using. How do you handle this safely?"
**Concept:** Combines the revert lesson with a real constraint — once pushed to a shared branch, you can't force-rewrite history without disrupting others.
**Interview answer:** "Since the commits are already on a shared branch, I wouldn't use `git reset` + force push, as that rewrites history others have already pulled. Instead, I'd use `git revert` for each of the three commits, or `git revert HEAD~3..HEAD` to revert a range. Then push normally. If it were a brand-new branch nobody else pulled yet, `git reset --hard HEAD~3` + force push would be acceptable."

### Q8 (Linux/Networking — Revision check). "If a server is responding slowly, what does a load average of '8.00' mean on a 4-core server, and is that good or bad?"
**Concept:** Load average represents the average number of processes waiting for CPU time, compared against the number of CPU cores.
**Interview answer:** "On a 4-core server, a load average of 8.00 means on average 8 processes are competing for CPU at once, but only 4 cores are available — roughly double the processes the system can handle simultaneously are queued. That's CPU saturation, and I'd investigate which processes are consuming CPU with `top`, sorted by `%CPU`."

---
---

# SECTION 4: EASY-EXPLANATION LAYER (Bonus — for concepts that needed simplifying)

> Use this section when Section 1 (Day 1) still feels confusing. Read the analogy first, then go back and re-read the professional answer above — it'll click faster.

## Networking Layers (OSI Model) — The Foundation Concept

**In Simple Words:** Imagine sending a letter to a friend. You write the message, put it in an envelope with an address, the postman picks a route, and it travels on roads. Each layer doesn't care what the layer above is doing — the postman doesn't read your letter, he just delivers the envelope. Computer networking works the same way — data gets wrapped in layers, sent, then unwrapped layer by layer at the other end.

| Layer | Name | What it does | DevOps relevance |
|---|---|---|---|
| 1 | Physical | Cables, wifi, electricity | Rarely touched in cloud work |
| 2 | Data Link | Local network device-to-device via MAC address | Switches; rarely touched directly |
| 3 | Network | Cross-network communication via **IP address**, routing | VPCs, subnets, routing tables, `ping` |
| 4 | Transport | TCP (reliable) vs UDP (fast), introduces **ports** | Security Groups, firewalls, Layer 4 Load Balancers, `telnet`/`nc` |
| 5 | Session | Keeps the "conversation" open | SSH sessions, login sessions |
| 6 | Presentation | Formats/encrypts data | SSL/TLS |
| 7 | Application | App-level protocol — HTTP, DNS, FTP | Your website/API, Nginx, Layer 7 Load Balancers |

**Key idea:** Going UP layers = smarter/more specific. Going DOWN = dumber but faster. Layer 3 only knows IP (where), Layer 4 adds port (which app), Layer 7 understands content (URL paths, headers).

**Why it matters:** When something breaks, ask "at which layer is this failing?"
- Can't reach server at all → Layer 3 (IP/routing/VPC)
- Server reachable, specific service not responding → Layer 4 (port blocked, security group)
- Service responds but behaves wrong → Layer 7 (app config, Nginx rules, LB routing)

**Interview answer:** "Networking works in layers — I use the OSI model mentally. Layer 3 is IP addressing and routing, basically 'can I reach this machine.' Layer 4 adds ports and TCP/UDP — 'can I reach this specific service.' Layer 7 is the application layer — HTTP — where load balancers make smart routing decisions based on URL paths. When troubleshooting connectivity, I go layer by layer: ping for IP-level reachability, then telnet/nc for port-level, then check application logs if the port is open but something's still wrong."

**Follow-up:** *"Is ping Layer 3 or 4?"* → "Ping uses ICMP, a Layer 3 protocol — it confirms the host is reachable but tells you nothing about a specific port or application, which is why ping working doesn't guarantee your application will work."

---

## `ls -l` Walkthrough — Simplified

**In Simple Words:** The shell is like a restaurant waiter. You ask for `ls -l`. The waiter checks "do I know this myself (built-in)? No? Let me check the kitchen list (`$PATH`)." Finds `ls` in `/bin/ls`, creates a helper process (fork), tells the helper to run the actual program (exec). That program asks the kernel (building manager) for the file list (system call), and the result is printed to your screen.

**Key terms:** Shell = command reader. `$PATH` = folders searched for programs. Fork = create helper process. Exec = run the actual program inside that helper. System call = program asking kernel to do something.

---

## CPU/Memory Check — Simplified

**In Simple Words:** Server is a busy kitchen. `top`/`htop` = live dashboard of which chef (process) is using which stove burner (CPU) or fridge space (Memory). `ps aux --sort=-%cpu | head` = a quick photo instead of a live feed.

---

## Disk Full — Simplified

**In Simple Words:** Disk = a house with many rooms. `df -h` = hallway sensor telling you which room (partition) is full. `du -sh` = walking into that room and checking each drawer until you find what's overflowing — usually old logs, Docker leftovers, or junk files.

---

## File Permissions — Simplified

**In Simple Words:** A file is a house with 3 key sets: Owner, Group, Others. Each key opens 3 lock types: Read, Write, eXecute.
`-rwxr-xr--`: `-`=regular file, `rwx`=owner full access, `r-x`=group read+execute only, `r--`=others read only.
Numbers: Read=4, Write=2, Execute=1 → Owner(7), Group(5), Others(4) = **754**.

---

## Hard Link vs Symlink — Simplified

**In Simple Words:** A file is a house, the filename is an address signboard.
- **Hard link** = a second signboard pointing to the SAME house. Both equally real. Can't cross cities (filesystems) or point to a whole neighborhood (directories).
- **Symlink** = a sticky note saying "the house is actually at [other address]." If that house is demolished/moved, the sticky note points to nothing (dangling link).

---

## TCP vs UDP — Simplified

**In Simple Words:** TCP = trusted courier with tracking and signature — confirms pickup and delivery, resends if lost. Slower but guaranteed. UDP = shouting across a noisy room — said once, no confirmation, but instant. Fine for video calls/streaming where a few dropped words don't matter.

---

## Visiting a URL — Simplified

**In Simple Words:** You know a person's name (domain) but not their house address (IP). DNS is the directory service that looks up the address — checking its own contacts (cache) first, then asking bigger directories (root → TLD → authoritative) if needed. Once you have the address, your browser "calls" it (TCP connection), does a secret handshake (TLS) to encrypt the chat, then asks "send me your homepage" and gets HTML back.

---

## Checking a Remote Port — Simplified

**In Simple Words:** Like knocking on a specific door (port) of a building (server). `nc -zv host port` = literally knocking and listening. No answer = either nobody's home (service not running) or a security guard (firewall/security group) won't let you near that door.

---

## restart vs reload — Simplified

**In Simple Words:** Restart = turning the machine fully OFF and ON — everyone using it gets kicked out briefly. Reload = telling the machine "just re-read your instructions" without shutting down — nobody notices a gap. Not every machine supports "reload mode."

---

## Slow SSH Server — Simplified

**In Simple Words:** Like a doctor doing a checkup on a patient who "feels slow." Check vitals in order: heart rate (CPU load via `uptime`), out of breath (memory/swap via `free -h`), blocked arteries (disk I/O via `iostat`), anything external (network flood via `ss -s`). Structured checklist, not random guessing.

---

# Quick Self-Test (cover answers, try out loud)
1. Difference between Layer 3 and Layer 4 in networking?
2. Disk is full — what are your exact first 2 commands?
3. What does `chmod 754` mean in letters (rwx form)?
4. Ping works but app can't connect to DB — why, in terms of layers?
5. Difference between `git merge` and `git rebase` — when would you NOT rebase?
6. You pushed a bad commit to `main` and it auto-deployed — first command you run?
7. `git fetch` vs `git pull` — which is safer and why?

---

*Next: Day 3 (Docker) will be added below this file or as a continuation — mixed with revision questions from Sections 1-3 so nothing gets forgotten.*
