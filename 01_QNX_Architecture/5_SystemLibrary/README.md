# QNX System Library

## Overview

The QNX system library contains two types of functions:

| Type | Description | Example |
|------|-------------|---------|
| **QNX-specific functions** | Unique to QNX | `MsgSend()`, `TimerSettime()` |
| **Standard/Portable functions** | Common POSIX/Unix calls | `timer_settime()`, `read()`, `open()` |

---

## How Library Functions Work

### Built on Kernel Calls

Many standard functions are built on top of kernel calls:

- Usually a **thin layer** that changes argument format
- Sometimes **multiple standard calls** funnel to one kernel call

**Example:**

```
timer_settime()  ──────►  TimerSettime()
   (POSIX)                  (Kernel Call)

- Converts seconds/nanoseconds (POSIX format)
- To 64-bit nanoseconds (kernel format)
- Then calls the kernel
```

### Recognizing Kernel Calls

Kernel calls use **CamelCase** (mixed capitals):

| Type | Naming Style | Example |
|------|--------------|---------|
| POSIX/Standard | snake_case | `timer_settime()` |
| Kernel Call | CamelCase | `TimerSettime()` |

---

## Microkernel Message Passing

Since QNX is a microkernel, many operations that would be kernel calls in traditional OSes become **messages to servers**.

### How It Works

```
Traditional OS:
  read() ──────► Kernel handles directly

QNX Microkernel:
  read() ──────► Build message ──────► MsgSend() ──────► Resource Manager
```

### Examples

| Function | What Happens |
|----------|--------------|
| `read()` | Builds message → Sends to resource manager |
| `open()` | Builds message → Sends to process manager + resource manager |
| `fork()` | Builds message → Sends to process manager |
| `posix_spawn()` | Builds message → Sends to process manager |

The cover functions (`read()`, `fork()`, etc.):
1. Build the message in QNX format
2. Call `MsgSend()` or a variant
3. Send to appropriate server

---

## Best Practice: Use Standard Calls

### Recommendation

When you have a choice between:
- Direct kernel call (`TimerSettime()`)
- Standard POSIX function (`timer_settime()`)

**Always use the standard POSIX function.**

### Why Use Standard Calls?

| Reason | Benefit |
|--------|---------|
| **Portability** | Code can run on Linux for testing, then QNX for production |
| **Familiarity** | Other developers recognize standard calls |
| **Maintenance** | Easier for others to read and maintain your code |
| **Code Review** | Reviewers don't need to look up QNX-specific functions |

### Same for Messages

When you have a choice between:
- Calling `read()`, `fork()`, etc.
- Directly building and sending messages with `MsgSend()`

**Always use the cover functions. Don't custom-build messages.**

---

## Summary

```
┌─────────────────────────────────────────────────────────────┐
│                  QNX System Library                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Standard POSIX Functions (USE THESE)                       │
│  ├── timer_settime()                                        │
│  ├── read()                                                 │
│  ├── open()                                                 │
│  ├── fork()                                                 │
│  └── posix_spawn()                                          │
│           │                                                 │
│           ▼                                                 │
│  Thin Layer (argument conversion)                           │
│           │                                                 │
│           ▼                                                 │
│  Kernel Calls OR Message Passing                            │
│  ├── TimerSettime() (kernel call)                           │
│  └── MsgSend() to servers (microkernel)                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Use standard POSIX calls** — Not QNX-specific kernel calls
2. **Use cover functions** — Don't build messages manually
3. **Portability matters** — Even if you never port, others will read your code
4. **Kernel calls are CamelCase** — `TimerSettime()`, `MsgSend()`
5. **Standard calls are snake_case** — `timer_settime()`, `read()`

---

> *Prefer portable, standard functions over QNX-specific calls. This improves code readability, maintainability, and portability.*