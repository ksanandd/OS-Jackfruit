# OS-Jackfruit: Multi-Container Runtime from Scratch

## Project Overview

OS-Jackfruit is a lightweight container runtime built from scratch in C that demonstrates core operating system concepts:

* Process isolation using Linux namespaces (PID, UTS, Mount)
* Memory enforcement via a custom kernel module with soft/hard limits
* Inter-process communication using UNIX sockets and pipes
* Thread synchronization with mutexes and condition variables
* Linux CFS scheduler behavior analysis

The project runs multiple isolated containers, each with its own root filesystem (Alpine Linux), and provides a CLI to manage them.

---

## Team Information

| Name              | SRN           |
| ----------------- | ------------- |
| Anand             | PES1UG24AM341 |
| Basawaraj Panchal | PES1UG24AM350 |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         USER SPACE                          │
├─────────────────────────────────────────────────────────────┤
│  CLI (engine) ──UNIX Socket──► SUPERVISOR (engine)          │
│                                    │                        │
│                              ┌─────┴─────┐                  │
│                              ▼           ▼                  │
│                         Container    Container              │
│                           alpha         beta                │
│                              │           │                  │
│                           (stdout/stderr via pipes)         │
│                                    │                        │
│                              ▼     ▼     ▼                  │
│                        Bounded Buffer + Logger Thread       │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ ioctl
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                         KERNEL SPACE                        │
├─────────────────────────────────────────────────────────────┤
│              monitor.ko (LKM)                               │
│              - Timer checks RSS every 2 seconds             │
│              - Soft limit → warning in dmesg                │
│              - Hard limit → SIGKILL                         │
└─────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

* Ubuntu 22.04 or 24.04 (native VM, not WSL)
* sudo access
* Internet connection (to download Alpine rootfs)

---

## Step-by-Step Setup and Execution Guide

### Step 1: Install Dependencies

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) wget
```

---

### Step 2: Clone the Repository

```bash
git clone https://github.com/ksanandd/OS-Jackfruit.git
cd OS-Jackfruite/boilerplate
```

---

### Step 3: Run Environment Check

```bash
chmod +x environment-check.sh
sudo ./environment-check.sh
```

**Expected Output:** All checks should show `[OK]`

---

### Step 4: Build the Project

```bash
make clean
make
```

**Build Output:**

* `engine` – Supervisor + CLI
* `cpu_hog` – CPU workload
* `io_pulse` – I/O workload
* `memory_hog` – Memory workload
* `monitor.ko` – Kernel module

---

### Step 5: Download Alpine RootFS

```bash
mkdir -p rootfs-alpha rootfs-beta rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz

tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-alpha
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-beta
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
```

---

### Step 6: Copy Workloads

```bash
cp cpu_hog memory_hog io_pulse rootfs-alpha/
cp cpu_hog memory_hog io_pulse rootfs-beta/
cp cpu_hog memory_hog io_pulse rootfs-base/
```

---

### Step 7: Load Kernel Module

```bash
sudo insmod monitor.ko
lsmod | grep monitor
```

Create device:

```bash
sudo mknod /dev/container_monitor c $(grep container_monitor /proc/devices | awk '{print $1}') 0
sudo chmod 666 /dev/container_monitor
```

---

### Step 8: Start Supervisor

```bash
sudo ./engine supervisor ./rootfs-base
```

**Expected Output:**

```
[supervisor] ready on /tmp/engine_supervisor.sock
```

---

### Step 9: CLI Commands

#### List containers

```bash
sudo ./engine ps
```

#### Start containers

```bash
sudo ./engine start alpha ./rootfs-alpha /bin/sh --soft-mib 48 --hard-mib 80
sudo ./engine start beta ./rootfs-beta /bin/sh --soft-mib 64 --hard-mib 96
```

#### Logs

```bash
sudo ./engine logs alpha
```

#### Stop container

```bash
sudo ./engine stop hog1
```

---

### Step 10: Memory Limit Test

```bash
sudo dmesg -w | grep container_monitor
sudo ./engine start memhog ./rootfs-alpha /memory_hog --soft-mib 32 --hard-mib 64
```

---

### Step 11: Scheduler Experiments

* Equal priority → fair CPU distribution
* Different nice values → priority impact
* CPU vs I/O workloads

---

### Step 12: Clean Shutdown

```bash
sudo rmmod monitor
```

---

## CLI Command Reference

| Command    | Description       |
| ---------- | ----------------- |
| supervisor | Start daemon      |
| start      | Start container   |
| run        | Run in foreground |
| ps         | List containers   |
| logs       | Show logs         |
| stop       | Stop container    |

---

## Key OS Concepts

| Concept       | Implementation      |
| ------------- | ------------------- |
| Namespaces    | `unshare()`         |
| Isolation     | `chroot()`          |
| IPC           | UNIX sockets, pipes |
| Threads       | pthreads            |
| Kernel Module | monitor.ko          |
| Memory        | Soft + Hard limits  |
| Scheduling    | Linux CFS           |

---

## Project Structure

```
boilerplate/
├── engine.c
├── monitor.c
├── cpu_hog.c
├── io_pulse.c
├── memory_hog.c
├── Makefile
├── environment-check.sh
├── README.md
```

---

## Screenshots

Available in `Screenshots/` directory:

* Environment check
* Build output
* Supervisor running
* Container logs
* Memory limit enforcement
* Scheduler experiments

---

## Contributors

* Anand
* Basawaraj Panchal

---

## License

Developed as part of the Operating Systems course at PES University.

---

## Acknowledgments

* Course instructor
* Linux kernel documentation
* Alpine Linux

---
