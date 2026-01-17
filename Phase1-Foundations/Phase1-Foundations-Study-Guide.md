# 📚 Phase 1: Foundations Study Guide
## Linux, Containers & Kubernetes - From Zero to Hero

---

## Table of Contents

1. [Introduction](#introduction)
2. [Module 1: Linux Fundamentals](#module-1-linux-fundamentals)
3. [Module 2: Container Fundamentals](#module-2-container-fundamentals)
4. [Module 3: Kubernetes Fundamentals](#module-3-kubernetes-fundamentals)
5. [Practice Labs](#practice-labs)
6. [Quick Reference Cheat Sheets](#quick-reference-cheat-sheets)

---

## Introduction

### Why This Matters

Before diving into OpenShift and Operators, you need to understand the foundation they're built on. Think of it like building a house:

```
┌─────────────────────────────────────────────────────────────┐
│                     YOUR GOAL: OpenShift Expert             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   🏠 OpenShift / Operators          (The House)             │
│   ─────────────────────────                                 │
│        ↑                                                    │
│   🧱 Kubernetes                      (The Walls)            │
│   ─────────────────────────                                 │
│        ↑                                                    │
│   📦 Containers                      (The Bricks)           │
│   ─────────────────────────                                 │
│        ↑                                                    │
│   🐧 Linux                           (The Foundation)       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**If your foundation is weak, everything above it will be shaky!**

---

# Module 1: Linux Fundamentals

## 1.1 What is Linux?

**Simple Explanation**: Linux is an operating system (OS) - the software that manages your computer's hardware and lets you run programs. It's like the manager of a hotel who controls which guests (programs) can use which rooms (resources).

### Key Components of Linux

```
┌──────────────────────────────────────────────────────────────────┐
│                      Linux Architecture                          |
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                     Applications                           |  |
│  │           (Firefox, Terminal, Your Programs)               |  |
│  └────────────────────────────────────────────────────────────┘  │
│                             ↓                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                        Shell                               │  │
│  │            (bash, zsh - Command Interpreter)               │  │
│  └────────────────────────────────────────────────────────────┘  │
│                             ↓                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                        Kernel                              │  │
│  │       (The Core - Manages CPU, Memory, Devices)            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                             ↓                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                       Hardware                             │  │
│  │            (CPU, RAM, Disk, Network Card)                  │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1.2 Processes - The Heart of Linux

### What is a Process?

**Simple Explanation**: A process is a running program. When you open Firefox, that's a process. When you run a command in terminal, that's a process.

**Real-life Analogy**: Think of a recipe book (the program) vs actually cooking (the process). The recipe just sits there, but cooking is the active work happening.

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Program (File on Disk)              Process (Running)          │
│   ─────────────────────              ──────────────────          │
│                                                                  │
│   /usr/bin/firefox          →→→     Firefox running with         │
│   (Just a file,                      PID 1234, using CPU,        │
│    doing nothing)                    memory, open files          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Process ID (PID)

Every process gets a unique number called PID (Process ID). It's like an employee ID at a company.

```bash
# See your current processes
ps aux

# Output explained:
# USER       PID  %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
# sayadas   1234  2.0  1.5  500000 30000 ?        S    10:00   0:05 firefox

# USER = Who started it
# PID = Process ID (unique number)
# %CPU = How much CPU it's using
# %MEM = How much memory it's using
# COMMAND = What program is running
```

### Parent and Child Processes

Every process (except PID 1) has a parent. When you open terminal and run a command, the terminal is the parent, and your command is the child.

```
┌──────────────────────────────────────────────────────────────────┐
│                        Process Tree                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   PID 1: systemd (The "Adam" of all processes)                   │
│       │                                                          │
│       ├── PID 100: sshd (SSH server)                             │
│       │       │                                                  │
│       │       └── PID 500: bash (Your terminal)                  │
│       │               │                                          │
│       │               └── PID 600: vim (Editor you opened)       │
│       │                                                          │
│       ├── PID 200: nginx (Web server)                            │
│       │       │                                                  │
│       │       ├── PID 201: nginx worker                          │
│       │       └── PID 202: nginx worker                          │
│       │                                                          │
│       └── PID 300: postgres (Database)                           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Hands-on: Process Commands

```bash
# 1. See all processes
ps aux

# 2. See process tree (parent-child relationships)
pstree

# 3. See only your processes
ps -u $(whoami)

# 4. Find a specific process
ps aux | grep firefox

# 5. See processes in real-time (like Task Manager)
top
# Press 'q' to quit

# 6. Better version of top
htop
# (You may need to install: sudo dnf install htop)

# 7. See process details
cat /proc/1/status | head -20
# /proc is a special "fake" filesystem that shows process info
```

### Killing Processes

Sometimes processes get stuck. Here's how to stop them:

```bash
# Gentle kill (asks process to stop)
kill 1234        # Replace 1234 with actual PID

# Force kill (immediately stops - use as last resort)
kill -9 1234

# Kill by name
pkill firefox
killall firefox
```

---

## 1.3 Namespaces - The Magic Behind Containers

### What are Namespaces?

**Simple Explanation**: Namespaces let you create isolated environments. A process in one namespace can't see processes in another namespace.

**Real-life Analogy**: Think of namespaces like separate apartments in a building. Each apartment has its own rooms, bathroom, kitchen. People in apartment A can't see inside apartment B.

```
┌──────────────────────────────────────────────────────────────────┐
│                     Without Namespaces                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   All processes live together, can see each other:               │
│                                                                  │
│   ┌────────────────────────────────────────────────────────────┐ │
│   │  Process A    Process B    Process C    Process D          │ │
│   │     │             │            │            │              │ │
│   │     └─────────────┴────────────┴────────────┘              │ │
│   │              All share the same view                       │ │
│   └────────────────────────────────────────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                      With Namespaces                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Processes are isolated, can't see each other:                  │
│                                                                  │
│   ┌─────────────────────┐       ┌─────────────────────┐          │
│   │    Namespace 1      │       │    Namespace 2      │          │
│   │                     │       │                     │          │
│   │   Process A         │       │   Process C         │          │
│   │   Process B         │       │   Process D         │          │
│   │                     │       │                     │          │
│   │   "I can only       │       │   "I can only       │          │
│   │    see A and B"     │       │    see C and D"     │          │
│   └─────────────────────┘       └─────────────────────┘          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Types of Namespaces

Linux has 7 types of namespaces. Each isolates different things:

| Namespace | What it Isolates | Analogy |
|-----------|------------------|---------|
| **PID** | Process IDs | Each apartment has its own room numbers |
| **NET** | Network (IPs, ports) | Each apartment has its own address |
| **MNT** | Mount points (filesystems) | Each apartment has its own storage |
| **UTS** | Hostname | Each apartment has its own name |
| **IPC** | Inter-process communication | Each apartment has its own phone system |
| **USER** | User IDs | Each apartment has its own resident list |
| **CGROUP** | Resource limits | Each apartment has its own utility meters |

### Hands-on: See Namespaces

```bash
# See namespaces of a process
ls -la /proc/self/ns/

# Output:
# lrwxrwxrwx. 1 user user 0 Jan 15 10:00 cgroup -> 'cgroup:[4026531835]'
# lrwxrwxrwx. 1 user user 0 Jan 15 10:00 ipc -> 'ipc:[4026531839]'
# lrwxrwxrwx. 1 user user 0 Jan 15 10:00 mnt -> 'mnt:[4026531840]'
# lrwxrwxrwx. 1 user user 0 Jan 15 10:00 net -> 'net:[4026531992]'
# lrwxrwxrwx. 1 user user 0 Jan 15 10:00 pid -> 'pid:[4026531836]'
# lrwxrwxrwx. 1 user user 0 Jan 15 10:00 user -> 'user:[4026531837]'
# lrwxrwxrwx. 1 user user 0 Jan 15 10:00 uts -> 'uts:[4026531838]'

# The numbers in brackets are namespace IDs
# Processes with same ID share that namespace
```

---

## 1.4 Cgroups - Resource Limits

### What are Cgroups?

**Simple Explanation**: Cgroups (Control Groups) let you limit how much CPU, memory, and disk a process can use.

**Real-life Analogy**: Think of your computer as an **apartment building** with limited resources (electricity, water, space). Cgroups are like giving each apartment its own utility meters and limits.

```
🏢 Your Computer (The Building)
│
├── 👨‍👩‍👧‍👦 user.slice (All Residents)
│   │
│   ├── 🏠 user-1000.slice (YOUR Apartment - User ID 1000)
│   │   │
│   │   ├── 🖥️ app.slice (Your Apps Room)
│   │   │   │
│   │   │   └── 📺 terminal.scope (Your Terminal Window)
│   │   │        ↑
│   │   │        YOU ARE HERE! 👋
```

### Why Cgroups Matter

```
┌────────────────────────────────────────────────────────────────────┐
│                        Without Cgroups                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                    System Resources                           │ │
│  │                    CPU: 100%, RAM: 16GB                       │ │
│  └───────────────────────────────────────────────────────────────┘ │
│       ↑            ↑            ↑            ↑                     │
│       │            │            │            │                     │
│  Process A    Process B    Process C    Process D                  │
│  (greedy!)    (greedy!)    (starving)   (starving)                 │
│  uses 70%    uses 25%     gets 3%      gets 2%                     │
│                                                                    │
│  Problem: Greedy processes can starve others!                      │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│                         With Cgroups                               │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                    System Resources                           │ │
│  │                    CPU: 100%, RAM: 16GB                       │ │
│  └───────────────────────────────────────────────────────────────┘ │
│       ↑            ↑            ↑            ↑                     │
│  ┌────┴────┐  ┌────┴────┐  ┌────┴────┐  ┌────┴────┐                │
│  │ Limit:  │  │ Limit:  │  │ Limit:  │  │ Limit:  │                │
│  │ 25% CPU │  │ 25% CPU │  │ 25% CPU │  │ 25% CPU │                │
│  │ 4GB RAM │  │ 4GB RAM │  │ 4GB RAM │  │ 4GB RAM │                │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘                │
│       │            │            │            │                     │
│  Process A    Process B    Process C    Process D                  │
│  (limited)    (limited)    (fair share) (fair share)               │
│                                                                    │
│  Solution: Each process gets fair, guaranteed resources!           │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Cgroups v1 vs v2

Modern Linux systems (Fedora, RHEL 9+, Ubuntu 22+) use **Cgroups v2** (unified hierarchy):

```
CGROUPS v1 (Old):                    CGROUPS v2 (Modern):

/sys/fs/cgroup/                      /sys/fs/cgroup/
├── 📁 cpu/        ← Separate        ├── 📄 cpu.stat      ← Files
├── 📁 memory/     ← Separate        ├── 📄 cpu.pressure
├── 📁 blkio/      ← Separate        ├── 📄 memory.stat
│                                    ├── 📄 memory.max
🏚️ Multiple trees (messy)            🏠 One unified tree (clean!)
```

### Hands-on: Exploring Cgroups

```bash
# See which cgroup your shell is in
cat /proc/self/cgroup
# Output: 0::/user.slice/user-1000.slice/...

# See the cgroup control panel
ls /sys/fs/cgroup/

# Note: In cgroups v2, there are no separate folders!
# ❌ ls /sys/fs/cgroup/cpu/     <- doesn't exist
# ✅ ls /sys/fs/cgroup/cpu.*    <- use this instead
```

### The Cgroup Filesystem Explained

```
/sys/fs/cgroup/
│
├── 🎛️ CONTROLLERS (what can be controlled)
│   ├── cpu.*          → ⚡ CPU limits & stats
│   ├── memory.*       → 🧠 RAM limits & stats
│   ├── io.*           → 💾 Disk I/O limits & stats
│   └── cpuset.*       → 🔢 Which CPUs to use
│
├── 📊 MONITORING (*.pressure files = "is it stressed?")
│   ├── cpu.pressure   → Is CPU overloaded?
│   ├── memory.pressure→ Is memory stressed?
│   └── io.pressure    → Is disk I/O slow?
│
└── 📁 SLICES (groups of processes)
    ├── user.slice/    → 👤 User processes
    ├── system.slice/  → ⚙️ System services
    └── machine.slice/ → 🐳 Containers/VMs
```

### Key Cgroup Files

| File | What it shows | Try it! |
|------|---------------|---------|
| `cpu.stat` | CPU time used | `cat /sys/fs/cgroup/cpu.stat` |
| `cpu.pressure` | CPU stress level | `cat /sys/fs/cgroup/cpu.pressure` |
| `memory.stat` | Memory usage | `cat /sys/fs/cgroup/memory.stat` |
| `memory.pressure` | Memory stress | `cat /sys/fs/cgroup/memory.pressure` |

### Understanding Pressure Files

```bash
cat /sys/fs/cgroup/cpu.pressure
# Output:
# some avg10=0.31 avg60=3.81 avg300=2.12 total=4043062763
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

| Field | Meaning |
|-------|---------|
| `some` | Some processes had to wait |
| `full` | ALL processes were stuck |
| `avg10` | Pressure in last 10 seconds (%) |
| `avg60` | Pressure in last 60 seconds (%) |
| `avg300` | Pressure in last 5 minutes (%) |

**Health Check Scale:**
```
0-5%    🟢 Excellent - System running smoothly
5-20%   🟡 Moderate  - Some contention
20-50%  🟠 High      - Performance impact
50%+    🔴 Critical  - Severe bottleneck
```

### Why This Matters for OpenShift 🚀

When you set resource limits in Kubernetes:

```yaml
resources:
  limits:
    memory: "512Mi"
    cpu: "500m"
```

Kubernetes creates cgroup entries:
- `memory.max` → set to 512MB
- `cpu.max` → set to 50% of one CPU

**Same cgroup technology, just automated!**

---

## 1.5 Networking Basics

### IP Addresses

**Simple Explanation**: An IP address is like a home address for computers. It tells other computers where to send data.

```
┌──────────────────────────────────────────────────────────────────┐
│                      IP Address Basics                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   IPv4 Address:  192 . 168 .  1  . 100                           │
│                   │     │     │     │                            │
│                   │     │     │     └─── Host (your computer)    │
│                   │     │     └───────── Subnet                  │
│                   │     └─────────────── Network                 │
│                   └───────────────────── Organization            │
│                                                                  │
│   Each number: 0-255 (8 bits = 1 byte)                           │
│   Total: 4 bytes = 32 bits                                       │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│   Special IPs:                                                   │
│                                                                  │
│   127.0.0.1      = localhost (yourself)                          │
│   192.168.x.x    = Private network (home/office)                 │
│   10.x.x.x       = Private network (often in containers)         │
│   172.16-31.x.x  = Private network                               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Ports

**Simple Explanation**: If IP address is your home address, port is the apartment number. One computer can have many services, each on a different port.

```
┌──────────────────────────────────────────────────────────────────┐
│                        Ports Explained                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Server: 192.168.1.100                                          │
│                                                                  │
│   ┌────────────────────────────────────────────────────────────┐ │
│   │                                                            │ │
│   │   Port 22   ──►  SSH (Remote login)                        │ │
│   │   Port 80   ──►  HTTP (Web server)                         │ │
│   │   Port 443  ──►  HTTPS (Secure web)                        │ │
│   │   Port 5432 ──►  PostgreSQL (Database)                     │ │
│   │   Port 6443 ──►  Kubernetes API                            │ │
│   │   Port 8443 ──►  OpenShift Console                         │ │
│   │                                                            │ │
│   └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│   Total ports available: 0 - 65535                               │
│   Ports 0-1023:  Reserved (need root to use)                     │
│   Ports 1024+:   Anyone can use                                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### DNS - Domain Name System

**Simple Explanation**: DNS is like a phone book. It converts names (google.com) to IP addresses (142.250.185.46).

```
┌──────────────────────────────────────────────────────────────────┐
│                        How DNS Works                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   You type: apps.cluster.example.com                             │
│                                                                  │
│   ┌─────────────┐                      ┌─────────────┐           │
│   │   Browser   │  ──── Step 1 ────►   │ DNS Server  │           │
│   │             │  "What's the IP?"    │             │           │
│   └─────────────┘                      └─────────────┘           │
│         │                                    │                   │
│         │                                    │                   │
│         │         ◄──── Step 2 ────          │                   │
│         │         "It's 34.29.173.203"       │                   |
|         |                                    |                   | 
│         │                                    |                   │
│         ▼                                    ▼                   │
│   ┌─────────────┐      Step 3              ┌─────────────┐       │
│   │   Browser   │  ──────────────────►     │   Server    │       │
│   │             │  Connect to IP           │34.29.173.203│       │
│   └─────────────┘                          └─────────────┘       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Hands-on: Networking Commands

```bash
# 1. See your IP addresses
ip addr show
# or shorter
ip a

# 2. See network interfaces
ip link show

# 3. Test connectivity
ping google.com -c 3

# 4. DNS lookup
dig google.com
# or simpler
nslookup google.com

# 5. See what ports are listening
ss -tulpn
# -t = TCP
# -u = UDP
# -l = listening
# -p = show process
# -n = show numbers (not names)

# 6. See active connections
netstat -an | head -20

# 7. Trace route to a server
traceroute google.com

# 8. See firewall rules
sudo iptables -L -n
```

---

## 1.6 Filesystem Basics

### Linux Directory Structure

```
┌──────────────────────────────────────────────────────────────────┐
│                  Linux Filesystem Hierarchy                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   /                           ← Root (top of everything)         │
│   ├── bin/                    ← Essential commands (ls, cp)      │
│   ├── boot/                   ← Boot files, kernel               │
│   ├── dev/                    ← Devices (disk, usb)              │
│   ├── etc/                    ← Configuration files ⭐           │
│   │   ├── hosts               ← Local DNS entries                │
│   │   ├── passwd              ← User accounts                    │
│   │   └── kubernetes/         ← K8s configs                      │
│   ├── home/                   ← User home directories            │
│   │   └── sayadas/            ← Your home                        │
│   ├── proc/                   ← Process info (virtual) ⭐        │
│   │   ├── 1/                  ← Info about PID 1                 │
│   │   └── self/               ← Info about current process       │
│   ├── root/                   ← Root user's home                 │
│   ├── sys/                    ← Kernel/hardware info ⭐          │
│   │   └── fs/cgroup/          ← Cgroup controls                  │
│   ├── tmp/                    ← Temporary files                  │
│   ├── usr/                    ← User programs                    │
│   │   ├── bin/                ← More commands                    │
│   │   └── local/              ← Locally installed software       │
│   └── var/                    ← Variable data                    │
│       ├── log/                ← Log files ⭐                     │
│       └── lib/                ← Application data                 │
│                                                                  │
│   ⭐ = Important for OpenShift/K8s work                          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### File Permissions

```
┌──────────────────────────────────────────────────────────────────┐
│                      File Permissions                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   -rw-r--r--  1  sayadas  users  1024  Jan 15  file.txt          │
│   ││  │  │       │        │                                      │
│   ││  │  │       │        └───── Group owner                     │
│   ││  │  │       └──────────────  User owner                     │
│   ││  │  └───────────────────────  Others: r-- (read only)       │
│   ││  └──────────────────────────  Group:  r-- (read only)       │
│   │└─────────────────────────────  Owner:  rw- (read + write)    │
│   └──────────────────────────────  Type:   - = file              │
│                                            d = directory         │
│                                            l = link              │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│   Permission Values:                                             │
│                                                                  │
│   r = read    (4)     Example: chmod 755 file                    │
│   w = write   (2)              = rwxr-xr-x                       │
│   x = execute (1)              = 7(rwx) + 5(r-x) + 5(r-x)        │
│                                                                  │
│   Common Examples:                                               │
│   ─────────────────────────────────────────────                  │
│   chmod 755 file  →  rwxr-xr-x  (owner:all, others:read+exec)    │
│   chmod 644 file  →  rw-r--r--  (owner:rw, others:read)          │
│   chmod 600 file  →  rw-------  (only owner can read+write)      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Hands-on: Filesystem Commands

```bash
# 1. Navigate
pwd                    # Where am I?
cd /etc                # Go to /etc
cd ~                   # Go home
cd -                   # Go to previous directory

# 2. List files
ls -la                 # List all with details
ls -lh                 # Human readable sizes

# 3. File operations
cp file1 file2         # Copy
mv file1 file2         # Move/rename
rm file                # Delete (careful!)
rm -rf directory       # Delete directory (VERY careful!)

# 4. View files
cat file               # Show entire file
head -20 file          # First 20 lines
tail -20 file          # Last 20 lines
tail -f file           # Follow file (great for logs!)
less file              # Scrollable view (q to quit)

# 5. Find files
find /etc -name "*.conf"      # Find by name
find / -type f -size +100M    # Files larger than 100MB

# 6. Search inside files
grep "error" /var/log/messages    # Find lines with "error"
grep -r "password" /etc/          # Search recursively
grep -i "error" file              # Case-insensitive
```

---

## 1.7 Systemd - Service Management

### What is Systemd?

**Simple Explanation**: Systemd is the first process (PID 1) that starts when Linux boots. It manages all other services.

**Real-life Analogy**: Systemd is like a hotel manager who wakes up first, then starts all the services - front desk, housekeeping, restaurant, etc.

### Hands-on: Systemd Commands

```bash
# 1. See all services
systemctl list-units --type=service

# 2. Check status of a service
systemctl status sshd
systemctl status docker
systemctl status kubelet

# 3. Start/stop/restart services
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx

# 4. Enable/disable auto-start on boot
sudo systemctl enable nginx     # Start on boot
sudo systemctl disable nginx    # Don't start on boot

# 5. View logs
journalctl -u nginx              # Logs for nginx
journalctl -u nginx -f           # Follow logs
journalctl -u nginx --since "1 hour ago"
journalctl -xe                   # Recent errors

# 6. See what failed
systemctl --failed
```

---

## 1.8 Linux Quiz - Test Yourself!

### Questions

1. **What is a process?**
   - A) A file on disk
   - B) A running program
   - C) A user account
   - D) A network connection

2. **What does PID 1 usually run?**
   - A) Your terminal
   - B) The web browser
   - C) systemd (init system)
   - D) The kernel

3. **What do namespaces provide?**
   - A) Resource limits
   - B) Isolation between processes
   - C) File encryption
   - D) Network speed

4. **What do cgroups provide?**
   - A) Process isolation
   - B) Resource limits (CPU, memory)
   - C) User authentication
   - D) File permissions

5. **What port does HTTPS use by default?**
   - A) 22
   - B) 80
   - C) 443
   - D) 8080

6. **Which command shows listening ports?**
   - A) `ps aux`
   - B) `ss -tulpn`
   - C) `ls -la`
   - D) `cat /etc/hosts`

7. **What does DNS do?**
   - A) Encrypts network traffic
   - B) Converts domain names to IP addresses
   - C) Limits CPU usage
   - D) Manages file permissions

8. **What permission value means read+write+execute?**
   - A) 4
   - B) 6
   - C) 7
   - D) 9

### Answers
1. B, 2. C, 3. B, 4. B, 5. C, 6. B, 7. B, 8. C

---

# Module 2: Container Fundamentals

## 2.1 What is a Container? (The REAL Answer)

### Common Misconception

❌ **Wrong**: "A container is a lightweight VM"

✅ **Correct**: "A container is just a regular Linux process with extra isolation using namespaces and cgroups"

```
┌──────────────────────────────────────────────────────────────────┐
│                  VM vs Container - The Truth                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Virtual Machine                     Container                  │
│   ───────────────                     ─────────                  │
│                                                                  │
│   ┌───────────────────┐              ┌───────────────────┐       │
│   │ App A   │  App B  │              │ App A   │  App B  │       │
│   ├─────────┼─────────┤              ├─────────┴─────────┤       │
│   │ Guest   │  Guest  │              │ Shared Container  │       │
│   │ OS      │  OS     │              │ Runtime (CRI-O)   │       │
│   │ (full!) │ (full!) │              ├───────────────────┤       │
│   ├─────────┴─────────┤              │ Shared Host       │       │
│   │    Hypervisor     │              │ Linux Kernel      │       │
│   │   (VMware/KVM)    │              ├───────────────────┤       │
│   ├───────────────────┤              │ Host OS           │       │
│   │    Host OS        │              ├───────────────────┤       │
│   ├───────────────────┤              │ Hardware          │       │
│   │    Hardware       │              └───────────────────┘       │
│   └───────────────────┘                                          │
│                                                                  │
│   Each VM has its own                Containers share the        │
│   complete OS (heavy!)               host kernel (light!)        │
│                                                                  │
│   ⏱️ Boot: Minutes                    ⏱️ Boot: Milliseconds      │
│   💾 Size: Gigabytes                  💾 Size: Megabytes         │
│                                                                  | 
└──────────────────────────────────────────────────────────────────┘
```

### A Container is Just:

```
┌──────────────────────────────────────────────────────────────────┐
│                 Container = Process + Isolation                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Container = Regular Linux Process                              │
│               + PID namespace (isolated process list)            │
│               + NET namespace (isolated network)                 │
│               + MNT namespace (isolated filesystem)              │
│               + UTS namespace (isolated hostname)                │
│               + USER namespace (isolated user IDs)               │
│               + cgroups (resource limits)                        │
│               + Overlay filesystem (layered storage)             │
│                                                                  │
│   That's it! No magic, no VMs, just Linux features! ✨           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2.2 Container Images

### What is a Container Image?

**Simple Explanation**: An image is a packaged filesystem with your application and everything it needs to run.

**Real-life Analogy**: An image is like a shipping container packed with everything needed - the product, packaging materials, instructions. You can ship this container anywhere and unpack it.

### Image Layers

Images are built in layers. Each layer is a change from the previous one.

```
┌──────────────────────────────────────────────────────────────────┐
│                        Image Layers                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Your Application Image                                         │
│                                                                  │
│   ┌──────────────────────────────────┐  Layer 4: Your app        │
│   │     Your application code        │  (ADD app.py)             │
│   ├──────────────────────────────────┤                           │
│   │     Python packages              │  Layer 3: Dependencies    │
│   │     (pip install)                │  (RUN pip install)        │
│   ├──────────────────────────────────┤                           │
│   │     Python runtime               │  Layer 2: Runtime         │
│   │     (/usr/bin/python)            │  (FROM python:3.9)        │
│   ├──────────────────────────────────┤                           │
│   │     Base OS (Alpine/Ubuntu)      │  Layer 1: Base            │
│   │     (/bin/sh, basic utils)       │                           │
│   └──────────────────────────────────┘                           │
│                                                                  │
│   Benefits of layers:                                            │
│   • Shared between images (saves space)                          │
│   • Cached (faster builds)                                       │
│   • Only changed layers need to be rebuilt                       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Dockerfile Example

```dockerfile
# Layer 1: Start from base image
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest

# Layer 2: Set working directory
WORKDIR /app

# Layer 3: Install dependencies
RUN microdnf install -y python3 && \
    microdnf clean all

# Layer 4: Copy application
COPY app.py .

# Layer 5: Set default command
CMD ["python3", "app.py"]
```

### Container Registry

**Simple Explanation**: A registry is like Docker Hub or App Store - a place to store and download images.

```
┌──────────────────────────────────────────────────────────────────┐
│                     Container Registry                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Image Name Format:                                             │
│                                                                  │
│   registry.example.com / namespace / image : tag                 │
│   ─────────┬────────── / ────┬──── / ──┬── / ─┬─                 │
│            │                 │         │      │                  │
│            │                 │         │      └── Version        │
│            │                 │         │          (latest, v1.0) │
│            │                 │         └── Image name            │
│            │                 └── Organization/project            │
│            └── Registry server                                   │
│                                                                  │
│   Examples:                                                      │
│   • docker.io/library/nginx:latest                               │
│   • quay.io/openshift/origin-cli:latest                          │
│   • registry.access.redhat.com/ubi9/ubi-minimal:latest           │
│   • ghcr.io/spiffe/spire-server:1.9.0                            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2.3 Container Runtime

### What is a Container Runtime?

**Simple Explanation**: The runtime is the software that actually creates and runs containers. OpenShift uses CRI-O.

```
┌──────────────────────────────────────────────────────────────────┐
│                   Container Runtime Stack                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │             Kubernetes / OpenShift                         │  │
│  │        (Orchestrator - manages many containers)            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                             │                                    │
│                             │ CRI (Container Runtime Interface)  │
│                             ▼                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │             CRI-O / containerd                             │  │
│  │        (High-level runtime - manages images)               │  │
│  └────────────────────────────────────────────────────────────┘  │
│                             │                                    │
│                             │ OCI (Open Container Initiative)    │
│                             ▼                                    │
│  ┌────────────────────────────────────────────────────────────┐  |
│  │                       runc                                 │  │
│  │      (Low-level runtime - creates namespaces/cgroups)      │  │
│  └────────────────────────────────────────────────────────────┘  │
│                             │                                    │
│                             ▼                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    Linux Kernel                            │  │ 
│  │       (namespaces, cgroups, seccomp, capabilities)         │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Hands-on: Working with Containers

### Using Podman (Red Hat's Docker alternative)

```bash
# 1. Pull an image
podman pull nginx:latest

# 2. List images
podman images

# 3. Run a container
podman run -d --name my-nginx -p 8080:80 nginx:latest
# -d = detached (run in background)
# --name = give it a name
# -p 8080:80 = map host port 8080 to container port 80

# 4. List running containers
podman ps

# 5. List all containers (including stopped)
podman ps -a

# 6. See container logs
podman logs my-nginx

# 7. Execute command inside container
podman exec -it my-nginx /bin/bash
# -i = interactive
# -t = allocate terminal

# 8. See what's happening inside
podman exec my-nginx ps aux
podman exec my-nginx cat /etc/nginx/nginx.conf

# 9. Stop container
podman stop my-nginx

# 10. Remove container
podman rm my-nginx

# 11. Remove image
podman rmi nginx:latest
```

### Inspecting Container Internals

```bash
# Run a container
podman run -d --name test nginx:latest

# Find the container's process on host
podman inspect test --format '{{.State.Pid}}'
# Let's say it returns 12345

# See namespaces of container
ls -la /proc/12345/ns/

# Compare with your shell's namespaces
ls -la /proc/self/ns/
# Notice the different namespace IDs!

# See what the container sees as PID 1
podman exec test cat /proc/1/cmdline
# It sees nginx as PID 1, but on host it's PID 12345!
```

---

## 2.5 Container Quiz - Test Yourself!

### Questions

1. **A container is:**
   - A) A lightweight VM
   - B) A process with namespaces and cgroups
   - C) A virtual network
   - D) A type of hypervisor

2. **What provides process isolation in containers?**
   - A) Cgroups
   - B) Namespaces
   - C) Firewall
   - D) Encryption

3. **What provides resource limits in containers?**
   - A) Namespaces
   - B) Images
   - C) Cgroups
   - D) DNS

4. **Container images are made of:**
   - A) Virtual disks
   - B) Layers
   - C) VMs
   - D) Partitions

5. **What does CRI-O do?**
   - A) Builds images
   - B) Runs containers
   - C) Manages networks
   - D) Stores data

6. **What command runs a container?**
   - A) `podman build`
   - B) `podman run`
   - C) `podman images`
   - D) `podman ps`

### Answers
1. B, 2. B, 3. C, 4. B, 5. B, 6. B

---

# Module 3: Kubernetes Fundamentals

## 3.1 What is Kubernetes?

**Simple Explanation**: Kubernetes (K8s) is a system that manages many containers across many machines. It handles deployment, scaling, and operations automatically.

**Real-life Analogy**: Kubernetes is like an airline operations center. It decides which planes (containers) go to which airports (nodes), handles delays (failures), and scales up during holidays (auto-scaling).

### Why Kubernetes?

```
┌──────────────────────────────────────────────────────────────────┐
│                    Without Kubernetes                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   You have 100 containers across 20 servers.                     │
│                                                                  │
│   Questions you must answer manually:                            │
│   • Which server should container X run on?                      │
│   • What if Server 5 dies?                                       │
│   • How do I update containers without downtime?                 │
│   • How do containers find each other?                           │
│   • How do I scale to 200 containers?                            │
│                                                                  │
│   Result: 😱 Chaos! Manual work! Errors! Late nights!            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                     With Kubernetes                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   You tell Kubernetes: "I want 100 nginx containers"             │
│                                                                  │
│   Kubernetes automatically:                                      │
│   ✅ Decides which servers to use                                │
│   ✅ Restarts containers if they crash                           │
│   ✅ Moves containers if a server dies                           │
│   ✅ Updates containers with zero downtime                       │
│   ✅ Provides DNS for service discovery                          │
│   ✅ Scales up/down based on rules                               │
│                                                                  │
│   Result: 😴 You sleep well! Automation handles everything!      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Kubernetes Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                 Kubernetes Cluster Architecture                   │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                     CONTROL PLANE                            │ │
│  │                (The Brain - Makes Decisions)                 │ │
│  │                                                              │ │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ │ │
│  │  │ API Server │ │    etcd    │ │ Scheduler  │ │ Controller │ │ │
│  │  │            │ │            │ │            │ │  Manager   │ │ │
│  │  │ Front door │ │  Database  │ │  Decides   │ │ Maintains  │ │ │
│  │  │  for all   │ │ Stores all │ │ which node │ │  desired   │ │ │
│  │  │  requests  │ │   state    │ │  runs what │ │   state    │ │ │
│  │  └────────────┘ └────────────┘ └────────────┘ └────────────┘ │ │
│  │                                                              │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                              │                                    │
│                              │ Network                            │
│                              ▼                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                      WORKER NODES                            │ │
│  │                (The Workers - Run Containers)                │ │
│  │                                                              │ │
│  │  ┌────────────────────┐       ┌────────────────────┐         │ │
│  │  │   Worker Node 1    │       │   Worker Node 2    │         │ │
│  │  │                    │       │                    │         │ │
│  │  │  ┌──────────────┐  │       │  ┌──────────────┐  │         │ │
│  │  │  │   kubelet    │  │       │  │   kubelet    │  │         │ │
│  │  │  │ (Node agent) │  │       │  │ (Node agent) │  │         │ │
│  │  │  └──────────────┘  │       │  └──────────────┘  │         │ │
│  │  │  ┌──────────────┐  │       │  ┌──────────────┐  │         │ │
│  │  │  │    CRI-O     │  │       │  │    CRI-O     │  │         │ │
│  │  │  │  (Runtime)   │  │       │  │  (Runtime)   │  │         │ │
│  │  │  └──────────────┘  │       │  └──────────────┘  │         │ │
│  │  │  ┌─────┐ ┌─────┐   │       │  ┌─────┐ ┌─────┐   │         │ │
│  │  │  │Pod A│ │Pod B│   │       │  │Pod C│ │Pod D│   │         │ │
│  │  │  └─────┘ └─────┘   │       │  └─────┘ └─────┘   │         │ │
│  │  │                    │       │                    │         │ │
│  │  └────────────────────┘       └────────────────────┘         │ │
│  │                                                              │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### Component Explanations

| Component | Role | Analogy |
|-----------|------|---------|
| **API Server** | Front door for all requests | Hotel reception |
| **etcd** | Database storing all cluster state | Hotel's guest registry |
| **Scheduler** | Decides which node runs each pod | Room assignment desk |
| **Controller Manager** | Maintains desired state | Hotel maintenance staff |
| **kubelet** | Agent on each node, runs pods | Floor manager |
| **CRI-O** | Container runtime | Actually runs the containers |

---

## 3.3 Core Concepts

### Pod - The Smallest Unit

**Simple Explanation**: A Pod is one or more containers that run together and share resources.

**Real-life Analogy**: A Pod is like a hotel room. It can have one bed (container) or multiple beds (containers), but they all share the bathroom (network) and mini-fridge (storage).

```yaml
# Simple Pod with one container
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

```yaml
# Pod with two containers (sidecar pattern)
# This is what you used with spiffe-helper!
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: main-app              # Main application
    image: my-app:latest
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: sidecar               # Helper container
    image: helper:latest
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data           # Shared between containers
    emptyDir: {}
```

### Deployment - Managing Pods

**Simple Explanation**: A Deployment manages multiple identical Pods and handles updates.

**Real-life Analogy**: A Deployment is like a hotel chain. You say "I want 3 identical hotels (Pods)", and it creates and maintains them.

```
┌──────────────────────────────────────────────────────────────────┐
│                         Deployment                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   You say: "I want 3 replicas of nginx"                          │
│                                                                  │
│   ┌────────────────────────────────────────────────────────────┐ │
│   │                     Deployment                             │ │
│   │                (Name: nginx-deployment)                    │ │
│   │                     replicas: 3                            │ │
│   └────────────────────────────────────────────────────────────┘ │
│                             │                                    │
│                             │ creates & manages                  │
│                             ▼                                    │
│   ┌────────────────────────────────────────────────────────────┐ │
│   │                     ReplicaSet                             │ │
│   │          (Ensures 3 pods are always running)               | │
│   └────────────────────────────────────────────────────────────┘ │
│                             │                                    │
│                             │ creates                            │
│                             ▼                                    │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐            │
│   │    Pod 1    │   │    Pod 2    │   │    Pod 3    │            │
│   │   (nginx)   │   │   (nginx)   │   │   (nginx)   │            │
│   └─────────────┘   └─────────────┘   └─────────────┘            │
│                                                                  │
│   If Pod 2 crashes → ReplicaSet creates new Pod automatically!   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3                    # I want 3 copies
  selector:
    matchLabels:
      app: nginx
  template:                      # Template for each Pod
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

### Service - Network Access to Pods

**Simple Explanation**: A Service provides a stable network endpoint to reach Pods. Pods come and go, but the Service IP stays the same.

**Real-life Analogy**: A Service is like a phone number for a business. Employees (Pods) may change, but customers always call the same number (Service).

```
┌──────────────────────────────────────────────────────────────────┐
│                        Service Types                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. ClusterIP (Default) - Internal only                          │
│  ─────────────────────────────────────                           │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │   Service: my-service                                    │    │
│  │   ClusterIP: 10.96.50.100:80                             │    │
│  │                                                          │    │
│  │   Only accessible INSIDE the cluster                     │    │
│  │   Other pods reach: my-service:80 or 10.96.50.100:80     │    │
│  └──────────────────────────────────────────────────────────┘    │
│                             │                                    │
│             ┌───────────────┼───────────────┐                    │
│             ▼               ▼               ▼                    │
│        ┌────────┐      ┌────────┐      ┌────────┐                │
│        │ Pod 1  │      │ Pod 2  │      │ Pod 3  │                │
│        │10.x.x.1│      │10.x.x.2│      │10.x.x.3│                │
│        └────────┘      └────────┘      └────────┘                │
│                                                                  │
│  2. NodePort - Exposed on each node                              │
│  ─────────────────────────────────                               │
│  Accessible from OUTSIDE at: <any-node-ip>:30080                 │
│                                                                  │
│  3. LoadBalancer - Cloud provider load balancer                  │
│  ─────────────────────────────────────────────                   │
│  Cloud creates external load balancer (AWS ELB, GCP LB, etc.)    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-service
spec:
  selector:
    app: nginx           # Find pods with label app=nginx
  ports:
  - port: 80             # Service port
    targetPort: 80       # Pod port
  type: ClusterIP        # Only internal access
```

### ConfigMap and Secret

**Simple Explanation**: 
- **ConfigMap**: Store non-sensitive configuration
- **Secret**: Store sensitive data (passwords, tokens)

```yaml
# ConfigMap example
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  database_host: "postgres.default.svc"
  log_level: "info"

---
# Secret example
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:
  database_password: "mysecretpassword"
```

---

## 3.4 RBAC - Role-Based Access Control

### What is RBAC?

**Simple Explanation**: RBAC controls WHO can do WHAT to WHICH resources.

**Real-life Analogy**: In a hotel, housekeeping can access all rooms but not the safe. The manager can access everything. Guests can only access their own room.

```
┌──────────────────────────────────────────────────────────────────┐
│                       RBAC Components                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   WHO                  WHAT                   WHERE              │
│   ───                  ────                   ─────              │
│   Subject      →→→    Role/ClusterRole  →→→  Namespace/Cluster   │
│                       (permissions)                              │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│   WHO (Subjects):                                                │
│   • ServiceAccount (SA) = Identity for pods                      │
│   • User                = Human identity                         │
│   • Group               = Collection of users                    │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│   WHAT (Roles):                                                  │
│   • Role        = Permissions in ONE namespace                   │
│   • ClusterRole = Permissions CLUSTER-WIDE                       │
│                                                                  |
├──────────────────────────────────────────────────────────────────┤
│   BINDING (Connects WHO to WHAT):                                │
│   • RoleBinding        = Subject → Role (namespace-scoped)       │
│   • ClusterRoleBinding = Subject → ClusterRole (cluster-wide)    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3.5 Storage in Kubernetes

```
┌──────────────────────────────────────────────────────────────────┐
│                    Storage Architecture                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Pod requests storage via PVC (PersistentVolumeClaim)    │    │
│  └──────────────────────────────────────────────────────────┘    │
│                             │                                    │
│                             ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  PVC binds to PV (PersistentVolume)                      │    │
│  └──────────────────────────────────────────────────────────┘    │
│                             │                                    │
│                             ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  PV provisioned by StorageClass (dynamic) or admin       │    │
│  └──────────────────────────────────────────────────────────┘    │
│                             │                                    │
│                             ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  CSI Driver → Actual Storage (AWS EBS, NFS, Ceph, etc.)  │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Access Modes

| Mode | Abbreviation | Meaning | Use Case |
|------|--------------|---------|----------|
| ReadWriteOnce | RWO | Single node read-write | Database (PostgreSQL) |
| ReadOnlyMany | ROX | Multiple nodes read-only | Shared config files |
| ReadWriteMany | RWX | Multiple nodes read-write | Shared uploads folder |

---

## 3.6 Essential kubectl Commands

```bash
# Basic commands
kubectl get pods                    # List pods
kubectl get pods -o wide            # More details
kubectl describe pod <name>         # Full details
kubectl logs <pod>                  # View logs
kubectl logs <pod> -f               # Follow logs
kubectl exec -it <pod> -- bash      # Enter pod

# Working with resources
kubectl apply -f file.yaml          # Create/update
kubectl delete -f file.yaml         # Delete
kubectl delete pod <name>           # Delete by name

# Debugging
kubectl get events --sort-by='.lastTimestamp'
kubectl describe pod <name> | grep -A 10 "Events:"

# Documentation
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy
```

---

## 3.7 Kubernetes Quiz

### Questions

1. **What is the smallest deployable unit in Kubernetes?**
   - A) Container  B) Pod  C) Deployment  D) Node

2. **What stores all cluster state?**
   - A) API Server  B) etcd  C) Scheduler  D) kubelet

3. **What provides a stable network endpoint for Pods?**
   - A) ConfigMap  B) Secret  C) Service  D) PVC

4. **What does PVC stand for?**
   - A) Pod Volume Container  B) Persistent Volume Claim  C) Private Virtual Cloud

### Answers
1. B, 2. B, 3. C, 4. B

---

# Practice Labs

## Lab 1: Linux Exploration
```bash
ps aux | head -20
pstree | head -30
ls -la /proc/self/ns/
ss -tulpn
```

## Lab 2: Container Basics
```bash
podman pull nginx:latest
podman run -d --name web -p 8080:80 nginx
podman ps
podman exec -it web /bin/bash
podman stop web && podman rm web
```

## Lab 3: Kubernetes Basics
```bash
kubectl create namespace lab-test
kubectl create deployment nginx --image=nginx -n lab-test
kubectl expose deployment nginx --port=80 -n lab-test
kubectl get all -n lab-test
kubectl delete namespace lab-test
```

---

# Quick Reference Cheat Sheets

## Linux Commands
| Command | Purpose |
|---------|---------|
| `ps aux` | List all processes |
| `ss -tulpn` | Show listening ports |
| `df -h` | Disk space |
| `free -h` | Memory usage |

## Container Commands
| Command | Purpose |
|---------|---------|
| `podman run -d <image>` | Run container |
| `podman ps` | List containers |
| `podman logs <container>` | View logs |
| `podman exec -it <container> bash` | Enter container |

## Kubernetes Commands
| Command | Purpose |
|---------|---------|
| `kubectl get <resource>` | List resources |
| `kubectl describe <resource>` | Detailed info |
| `kubectl apply -f <file>` | Create/update |
| `kubectl logs <pod>` | View logs |
| `kubectl explain <resource>` | Documentation |

---

**Next Step**: Phase 2 - OpenShift Core Concepts

---

*Phase 1 Complete! You now understand the foundations.*
