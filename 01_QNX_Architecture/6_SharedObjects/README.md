# QNX Shared Objects

## Overview

**Shared objects** are code that is brought into a process at **runtime** rather than being linked at compilation time.

---

## Types of Shared Objects

| Type | Description | How It's Loaded |
|------|-------------|-----------------|
| **Shared Library** | Library linked against at compile time | Automatically at process launch |
| **DLL (Dynamic Link Library)** | Explicitly loaded by program code | Via `dlopen()`, `dlsym()`, etc. |

### Examples

- `libc.so` — C library
- `lib-tcpip.so` — TCP/IP library

---

## How Shared Objects Work

### At Compile Time (Shared Library)

```
Program links against libc.so
         │
         ▼
Information stored in program file
         │
         ▼
System loader sees dependency
         │
         ▼
At process launch: loads code + data into process
```

### At Runtime (DLL)

```c
// Program explicitly loads shared object
void *handle = dlopen("mylib.so", RTLD_NOW);
void *func = dlsym(handle, "my_function");
```

---

## Memory Efficiency

When multiple programs use the same shared object:

```
┌─────────────────────────────────────────────────────────┐
│                   Physical Memory                       │
│                                                         │
│     ┌─────────────────────────────┐                     │
│     │     libc.so (ONE copy)      │                     │
│     │     (read-only)             │                     │
│     └─────────────────────────────┘                     │
│              │         │         │                      │
│              ▼         ▼         ▼                      │
│         Process A  Process B  Process C                 │
│         (mapped)   (mapped)   (mapped)                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Key Points:**
- Only **one copy** in system memory
- **Mapped** into all processes
- **Read-only** — processes can see but not change

---

## Global Data in Shared Objects

### Important: Global Data is Process Private

If a shared object contains global data:

| What You Might Expect | What Actually Happens |
|-----------------------|-----------------------|
| Data shared between processes | Data is **private to each process** |

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Shared Object with global variable "int counter = 0"   │
│                                                         │
│  Process A: counter = 5    (A's private copy)           │
│  Process B: counter = 10   (B's private copy)           │
│  Process C: counter = 0    (C's private copy)           │
│                                                         │
│  NO automatic sharing between processes!                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### If You Need to Share Data

You must use **explicit shared memory**:

```c
// Create shared memory explicitly
int fd = shm_open("/my_shared_data", O_CREAT | O_RDWR, 0666);
ftruncate(fd, sizeof(shared_data_t));
shared_data_t *data = mmap(NULL, sizeof(shared_data_t), 
                           PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
```

---

## Comparison with Other Systems

| Aspect | QNX | Linux/Unix |
|--------|-----|------------|
| Shared object model | Very similar | Very similar |
| Code sharing | One copy, read-only mapped | One copy, read-only mapped |
| Global data | Process private | Process private |
| Data sharing | Explicit shared memory needed | Explicit shared memory needed |

---

## Summary

| Concept | Description |
|---------|-------------|
| **Shared Object** | Code loaded at runtime |
| **Shared Library** | Linked at compile time, loaded at launch |
| **DLL** | Loaded explicitly via `dlopen()` |
| **Code** | One copy shared (read-only) |
| **Global Data** | Process private (NOT shared) |
| **Data Sharing** | Requires explicit shared memory |

---

> *Shared objects save memory by keeping one copy of code for multiple processes, but global data remains private to each process.*