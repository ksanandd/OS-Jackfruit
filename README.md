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

### Kernel Module Loaded & Device File Created

<img width="1217" height="95" alt="SS 1" src="https://github.com/user-attachments/assets/0b46b8db-a675-4328-8e99-ba103b0ce134" />

Shows successful loading of the kernel module and creation of /dev/container_monitor device.

---

### Starting Containers & Listing (ps)

<img width="1207" height="246" alt="SS 2" src="https://github.com/user-attachments/assets/d1a88ace-9a0b-4962-979b-407fae2780cb" />

Displays running containers (alpha, beta) with PID, state, and memory limits.

---

### Workload Execution (cpu_hog logs)

<img width="1216" height="714" alt="SS 3" src="https://github.com/user-attachments/assets/1409e065-eae2-4c2b-9a7a-19d764931c87" />

CPU-intensive workload starting execution inside container.

<img width="1216" height="714" alt="SS 4" src="https://github.com/user-attachments/assets/3687a7ad-9336-4be9-8ea9-d71c5122c599" />

Shows continuous CPU usage with increasing iteration count.

<img width="1216" height="714" alt="SS 5" src="https://github.com/user-attachments/assets/fa4dab8f-eed1-404e-ab49-42249139b63b" />

Demonstrates sustained CPU consumption over time.

<img width="1216" height="714" alt="SS 6" src="https://github.com/user-attachments/assets/b8f4f660-6e8a-4ecb-96ac-49e00f424e34" />

Confirms container stability under heavy CPU load.

<img width="1213" height="489" alt="SS 7" src="https://github.com/user-attachments/assets/600465e1-d463-48c5-931f-b9b6d065afd0" />

---

### Container Stop (Killed State)

<img width="1208" height="135" alt="SS 8" src="https://github.com/user-attachments/assets/892763d6-099e-4fcf-8bbc-c08aa6affc0f" />

Shows container termination and status change to killed.

---

### Memory Limit Enforcement (memhog)

<img width="1204" height="160" alt="SS 9" src="https://github.com/user-attachments/assets/0021ed77-051a-416d-8d70-9142e1e15962" />

Memory-intensive workload exceeding limits and triggering enforcement.

---

### Kernel Logs (Soft + Hard Limit)

<img width="1204" height="160" alt="SS 10 T3" src="https://github.com/user-attachments/assets/4bed9275-4920-46b7-b866-dac321c20b6a" />

Displays soft memory limit warning generated by kernel module.

<img width="1212" height="453" alt="SS 11 T3" src="https://github.com/user-attachments/assets/e1969de7-0f83-4a46-a581-3ef0d4b70cb9" />

Shows hard limit breach leading to process termination (SIGKILL).

---

### Clean Shutdown (No Zombie Processes)

<img width="1218" height="77" alt="SS 12 Zombie Defunt" src="https://github.com/user-attachments/assets/56fda943-9e96-4c27-a622-fef95a1c68a4" />

Confirms proper process cleanup with no zombie processes remaining.

---

## Scheduler Experiments

### Experiment 1: Equal Priority (CFS Fairness)

<img width="1217" height="491" alt="SS 13 Exp1" src="https://github.com/user-attachments/assets/5e7be89c-52a9-477c-b128-f0899db4aad6" />

Two CPU-bound containers running with equal priority.

<img width="1217" height="491" alt="SS 14 Exp1(2)" src="https://github.com/user-attachments/assets/938829b3-b53d-4a42-81b4-d2e91c255240" />

Both containers receive nearly equal CPU time (fair scheduling).

---

### Experiment 2: Different Nice Values

<img width="1207" height="443" alt="SS 15 Exp2" src="https://github.com/user-attachments/assets/5754a832-3018-484e-b142-bd182b30c865" />

Containers started with different nice values (priority levels).

`<img width="1207" height="465" alt="SS 16 Exp2(2)" src="https://github.com/user-attachments/assets/745007cb-281c-4537-91a8-63f5c43c2620" />

Higher priority container receives more CPU time than lower priority one.

---

### Experiment 3: CPU-bound vs I/O-bound

<img width="1202" height="443" alt="SS 17 Exp3" src="https://github.com/user-attachments/assets/49699be9-e278-4dba-8f32-bedc85f7b070" />

Running CPU-bound and I/O-bound workloads simultaneously.

<img width="1216" height="474" alt="SS 18 Exp3(2)" src="https://github.com/user-attachments/assets/f9678fd0-eec1-46e1-826e-9cbd08df7173" />

I/O-bound process yields CPU frequently, allowing better scheduling balance.

---

## Engineering Analysis

This project demonstrates several core operating system concepts through practical implementation:

* **Namespaces (PID, UTS, Mount):**
  Containers are isolated using `unshare()`, which creates separate process IDs, hostnames, and filesystem views. This ensures each container runs independently without interfering with others.

* **chroot Isolation:**
  The `chroot()` system call restricts the container’s root directory to its own filesystem, preventing access to host files.

* **Inter-Process Communication (IPC):**
  Communication between CLI and supervisor is implemented using UNIX domain sockets, while pipes are used to capture container stdout/stderr.

* **Threading and Synchronization:**
  Multiple threads handle container output using a bounded buffer. Synchronization is achieved using `pthread_mutex` and `pthread_cond` to avoid race conditions.

* **Kernel Module & Memory Monitoring:**
  A custom kernel module (`monitor.ko`) periodically checks the Resident Set Size (RSS) of each container process.

* **Memory Enforcement:**

  * Soft limit → Warning logged in `dmesg`
  * Hard limit → Process terminated using `SIGKILL`

* **Scheduler Behavior (CFS):**
  Experiments show fair CPU sharing between processes and how `nice` values influence scheduling priority.

---

## Design Decisions

* **UNIX Sockets for IPC:**
  Chosen for efficient and reliable communication between CLI and supervisor within the same system.

* **Pipes for Logging:**
  Used to capture container output in real-time and forward it to the logging system.

* **Multithreading:**
  Each container has a dedicated thread for handling logs, improving responsiveness and concurrency.

* **Kernel Module for Memory Control:**
  User-space programs cannot enforce strict memory limits reliably, so a kernel module is used for accurate monitoring and enforcement.

* **Alpine Linux RootFS:**
  Selected due to its lightweight nature, making it ideal for container environments.

---

## Results & Observations

* Multiple containers run independently with proper isolation.
* CPU-bound processes share CPU fairly under the CFS scheduler.
* Higher priority (`nice -5`) processes receive more CPU time than lower priority (`nice +10`).
* I/O-bound processes yield CPU frequently, allowing better scheduling balance.
* Memory limits are effectively enforced:

  * Soft limit generates warnings
  * Hard limit terminates the process

---

## Conclusion

This project successfully demonstrates the working principles behind container runtimes by implementing process isolation, resource management, and scheduling analysis from scratch.

It provides a clear understanding of how operating system concepts such as namespaces, memory management, inter-process communication, and scheduling are applied in real-world systems like Docker.

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
