# QNX Operating System Services

## Overview

Since QNX is a **microkernel**, most system services are delivered by **processes**, not built into the kernel.

### Key Principle

| If You... | Then... |
|-----------|---------|
| **Want** a capability/service | Run the process |
| **Don't need** a capability/service | Don't run the process |

**Benefits:**
- No code/data overhead for unused services
- Services can be **dynamically added or removed**
- System is fully configurable at runtime

---

## Example: The Pipe Service

### Demonstration

```bash
# List processes and threads
pidin

# Output scrolls off screen, so pipe to 'less'
pidin | less
```

### Pipe is NOT Built Into QNX

The pipe capability (`|`) is provided by a **process** called `pipe`.

```bash
# View running processes - pipe is process ID 167949
pidin | less

# Kill the pipe process
kill 167949

# Now piping fails!
pidin | less
# Error: can't create a pipe
```

### Restoring the Service

```bash
# Run the pipe program again
pipe

# Piping works again!
pidin | less

# Note: pipe now has a NEW process ID (1523714)
```

---

## Services as Processes

Most things you think of as "part of the OS" are actually **separate processes** in QNX:

| Service | Process | Provides |
|---------|---------|----------|
| **Random numbers** | `random` | `/dev/random`, `/dev/urandom` |
| **Pipes** | `pipe` | Pipe capability (`\|`) |
| **Message queues** | `mqueue` | POSIX message queues |
| **File systems** | `devb-*`, `fs-*` | Disk, flash, NAND, NOR access |
| **Networking** | `io-sock` | TCP/IP stack |
| **Logging** | `slogger2` | System logging |
| **PCI bus** | `pci-server` | PCI bus management |
| **USB** | `io-usb-otg` | USB bus management |
| **Core dumps** | `dumper` | Core dump creation |

---

## Practical Implications

### Embedded Systems

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Embedded System (No Command Line)                          │
│                                                             │
│  DON'T need:                                                │
│  ├── pipe (no shell piping)                                 │
│  ├── dumper (no core dumps needed)                          │
│  └── slogger2 (maybe no logging)                            │
│                                                             │
│  Result: Smaller memory footprint, faster boot              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Development/Debugging

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Development System (Full Debugging)                        │
│                                                             │
│  RUN these services:                                        │
│  ├── pipe (command line convenience)                        │
│  ├── dumper (core dumps for debugging)                      │
│  ├── slogger2 (logging for diagnostics)                     │
│  └── qconn (IDE connection)                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Safety-Critical Systems

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Safety-Critical System                                     │
│                                                             │
│  DON'T run dumper because:                                  │
│  - Slows down process termination                           │
│  - May violate timing requirements                          │
│  - Not needed in production                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Dynamic Configuration

Services can be added/removed **while the system is running**:

```bash
# System running without pipe capability
# Need to debug something...

# Add pipe capability
pipe &

# Now piping works
pidin | less

# Done debugging, remove it
kill $(pidin -p pipe -F %a)

# Pipe capability removed
```

---

## pidin - System Information Tool

`pidin` is the QNX tool to list processes and threads:

```bash
# List all processes
pidin

# List specific process
pidin -p pipe

# Show memory usage
pidin mem

# Show file descriptors
pidin fd
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Microkernel design** | Services are processes, not kernel components |
| **Optional services** | Only run what you need |
| **No overhead** | Don't pay for unused services |
| **Dynamic** | Add/remove services at runtime |
| **Flexible** | Configure system for specific needs |

---

## Key Takeaways

1. **Services = Processes** — Most OS capabilities are separate processes
2. **Run what you need** — No overhead for unused features
3. **Dynamic configuration** — Add/remove services anytime
4. **Embedded-friendly** — Minimize footprint by excluding unneeded services
5. **Safety-aware** — Exclude services that violate timing requirements

---

> *In QNX, the operating system is what you make it — run only the services you need.*