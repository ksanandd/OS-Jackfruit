# OS-Jackfruit: Multi-Container Runtime from Scratch

## Project Overview

- OS-Jackfruit is a lightweight container runtime built from scratch in C that demonstrates core operating system concepts:
  - Process isolation using Linux namespaces (PID, UTS, Mount)
  - Memory enforcement via a custom kernel module with soft/hard limits
  - Inter-process communication using UNIX sockets and pipes
  - Thread synchronization with mutexes and condition variables
  - Linux CFS scheduler behavior analysis
The project runs multiple isolated containers, each with its own root filesystem (Alpine Linux), and provides a CLI to manage them.

---


## Team Information
Name	SRN
Anand	PES1UG24AM341
Basawaraj Panchal	PES1UG24AM350

---

### Architecture
  
  text

┌─────────────────────────────────────────────────────────────┐
│                         USER SPACE                          │
├─────────────────────────────────────────────────────────────┤
│  CLI (engine) ──UNIX Socket──► SUPERVISOR (engine)          │
│                                    │                         │
│                              ┌─────┴─────┐                   │
│                              ▼           ▼                   │
│                         Container    Container               │
│                           alpha         beta                 │
│                              │           │                   │
│                           (stdout/stderr via pipes)          │
│                                    │                         │
│                              ▼     ▼     ▼                   │
│                        Bounded Buffer + Logger Thread       │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ ioctl
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                         KERNEL SPACE                         │
├─────────────────────────────────────────────────────────────┤
│              monitor.ko (LKM)                                │
│              - Timer checks RSS every 2 seconds             │
│              - Soft limit → warning in dmesg                 │
│              - Hard limit → SIGKILL                          │
└─────────────────────────────────────────────────────────────┘

---

### Prerequisites

- Ubuntu 22.04 or 24.04 (native VM, not WSL)
- sudo access
- Internet connection (to download Alpine rootfs)

---

### Step-by-Step Setup and Execution Guide
  - Step 1: Install Dependencies
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) wget

Step 2: Clone the Repository
bash

git clone https://github.com/shreeranganathsaravade78/OS-Jackfruite.git
cd OS-Jackfruite/Codes

Step 3: Run Environment Check
bash

chmod +x environment-check.sh
sudo ./environment-check.sh

Expected output: All checks should show [OK]
Step 4: Build the Project
bash

make clean
make

What gets built:

    engine - Supervisor + CLI binary

    cpu_hog - CPU-bound test workload

    io_pulse - I/O-bound test workload

    memory_hog - Memory allocation test workload

    monitor.ko - Kernel module for memory enforcement

Step 5: Download Alpine Linux Root Filesystem
bash

mkdir -p rootfs-alpha rootfs-beta rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-alpha
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-beta
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

Step 6: Copy Workloads to Root Filesystems
bash

cp cpu_hog memory_hog io_pulse rootfs-alpha/
cp cpu_hog memory_hog io_pulse rootfs-beta/
cp cpu_hog memory_hog io_pulse rootfs-base/

Step 7: Load the Kernel Module
bash

sudo insmod monitor.ko

Verify the module is loaded:
bash

lsmod | grep monitor

Create the device file:
bash

sudo mknod /dev/container_monitor c $(grep container_monitor /proc/devices | awk '{print $1}') 0
sudo chmod 666 /dev/container_monitor

Verify device file:
bash

ls -l /dev/container_monitor

Expected output:
text

crw-rw-rw- 1 root root 239, 0 Apr 16 14:53 /dev/container_monitor

Step 8: Start the Supervisor

Open Terminal 1:
bash

sudo ./engine supervisor ./rootfs-base

Expected output:
text

[supervisor] ready on /tmp/engine_supervisor.sock

⚠️ Keep this terminal running – the supervisor must stay alive.
Step 9: CLI Commands (Open Terminal 2)
9.1 List containers (initially empty)
bash

sudo ./engine ps

9.2 Start two containers
bash

sudo ./engine start alpha ./rootfs-alpha /bin/sh --soft-mib 48 --hard-mib 80
sudo ./engine start beta ./rootfs-beta /bin/sh --soft-mib 64 --hard-mib 96

9.3 List running containers
bash

sudo ./engine ps

Expected output:
text

NAME         PID      STATE        UPTIME(s)  SOFT(MiB) HARD(MiB)
----------------------------------------------------------------------
alpha        8005     running      10         48       80      
beta         8012     running      10         64       96      

9.4 View container logs
bash

sudo ./engine logs alpha

9.5 Start a CPU-intensive workload
bash

sudo ./engine start hog1 ./rootfs-alpha /cpu_hog --soft-mib 48 --hard-mib 80
sleep 5
sudo ./engine logs hog1 | tail -10

Expected output:
text

[cpu_hog] starting CPU burn
[cpu_hog] elapsed=5s iterations=500
[cpu_hog] elapsed=5s iterations=1000
...

9.6 Stop a container
bash

sudo ./engine stop hog1
sudo ./engine ps

Expected output: hog1 state changes to killed.
Step 10: Memory Limit Enforcement Test

Open Terminal 3:
bash

sudo dmesg -w | grep container_monitor

In Terminal 2, start the memory hog:
bash

sudo ./engine start memhog ./rootfs-alpha /memory_hog --soft-mib 32 --hard-mib 64

Watch Terminal 3 – after 30-60 seconds, you'll see:
text

container_monitor: [SOFT LIMIT] container 'memhog' (pid=12027) using 33356 KB, soft limit is 32768 KB
container_monitor: [HARD LIMIT] container 'memhog' (pid=12027) using 67148 KB, hard limit is 65536 KB — killing

Verify the container was killed:
bash

sudo ./engine ps | grep memhog

Expected output: State shows killed with PID 0.

Stop dmesg: Press Ctrl+C in Terminal 3.
Step 11: Scheduler Experiments
Experiment 1: Equal Priority (CFS Fairness)
bash

sudo ./engine start hog4 ./rootfs-alpha /cpu_hog --soft-mib 48 --hard-mib 80
sudo ./engine start hog5 ./rootfs-beta /cpu_hog --soft-mib 48 --hard-mib 80
sleep 30
echo "=== hog4 ===" && sudo ./engine logs hog4 | tail -3
echo "=== hog5 ===" && sudo ./engine logs hog5 | tail -3
sudo ./engine stop hog4
sudo ./engine stop hog5

Expected result: Both containers show similar iteration counts (CFS fairness).
Experiment 2: Different Nice Values
bash

sudo nice -n -5 ./engine start highprio ./rootfs-alpha /cpu_hog --soft-mib 48 --hard-mib 80
sudo nice -n +10 ./engine start lowprio ./rootfs-beta /cpu_hog --soft-mib 48 --hard-mib 80
sleep 30
echo "=== highprio (nice -5) ===" && sudo ./engine logs highprio | tail -3
echo "=== lowprio (nice +10) ===" && sudo ./engine logs lowprio | tail -3
sudo ./engine stop highprio
sudo ./engine stop lowprio

Experiment 3: CPU-bound vs I/O-bound
bash

sudo ./engine start cpuwork ./rootfs-alpha /cpu_hog --soft-mib 48 --hard-mib 80
sudo ./engine start iowork ./rootfs-beta /io_pulse --soft-mib 48 --hard-mib 80
sleep 30
echo "=== CPU-bound (cpu_hog) ===" && sudo ./engine logs cpuwork | tail -3
echo "=== I/O-bound (io_pulse) ===" && sudo ./engine logs iowork | tail -3
sudo ./engine stop cpuwork
sudo ./engine stop iowork

Step 12: Clean Shutdown

In Terminal 1: Press Ctrl+C to stop the supervisor.

In Terminal 2:
bash

sudo rmmod monitor
ps aux | grep defunct

Expected output: No zombie processes (only the grep command itself).
📊 CLI Command Reference
Command	Description
sudo ./engine supervisor <rootfs>	Start the supervisor daemon
sudo ./engine start <name> <rootfs> <cmd> [--soft-mib N] [--hard-mib N]	Start a container in background
sudo ./engine run <name> <rootfs> <cmd> [--soft-mib N] [--hard-mib N]	Start a container in foreground (blocks until exit)
sudo ./engine ps	List all containers with their status
sudo ./engine logs <name>	Show container logs
sudo ./engine stop <name>	Stop a running container
Memory Limit Flags
Flag	Description	Default
--soft-mib N	Soft memory limit in MiB (warning only)	48
--hard-mib N	Hard memory limit in MiB (enforced with SIGKILL)	96
🔧 Troubleshooting
Problem	Solution
insmod: ERROR: could not insert module monitor.ko: File exists	Module already loaded – run sudo rmmod monitor first
/dev/container_monitor: No such file or directory	Create device file: sudo mknod /dev/container_monitor c 239 0 && sudo chmod 666 /dev/container_monitor
exec: No such file or directory inside container	Recompile with -static flag: gcc -static -o cpu_hog cpu_hog.c
ERROR: container already exists	Restart supervisor to clear stale entries
Address already in use (supervisor)	Run sudo pkill -f "engine supervisor"
Zombie processes after shutdown	Ensure SIGCHLD handler calls waitpid() with WNOHANG
🧪 Test Workloads
Workload	Description	Behavior
cpu_hog	Busy spin loop	Burns 100% CPU, prints iterations every ~1 second
io_pulse	Repeated file writes/reads	I/O-bound, sleeps 10ms between cycles
memory_hog	Allocates 1 MiB every second	Tests soft/hard memory limits
📁 Project Structure
text

Codes/
├── engine.c              # Supervisor + CLI (800+ lines)
├── monitor.c             # Kernel module for memory enforcement (300+ lines)
├── monitor_ioctl.h       # ioctl definitions for kernel-user communication
├── cpu_hog.c             # CPU-bound test workload
├── io_pulse.c            # I/O-bound test workload
├── memory_hog.c          # Memory allocation test workload
├── Makefile              # Build configuration
├── environment-check.sh  # System verification script
├── README.md             # This file
├── rootfs-alpha/         # Alpine Linux rootfs for container alpha
├── rootfs-beta/          # Alpine Linux rootfs for container beta
├── rootfs-base/          # Base rootfs (unused, for reference)
└── Screenshots/          # Demo screenshots

🎯 Key OS Concepts Demonstrated
Concept	Implementation
Namespaces	unshare(CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS)
chroot isolation	chroot(rootfs) – container sees only its filesystem
IPC - UNIX Socket	CLI ↔ supervisor at /tmp/engine_supervisor.sock
IPC - Pipes	Container stdout/stderr → supervisor
Threads	One pipe reader thread per container + one log consumer thread
Synchronization	pthread_mutex_t + pthread_cond_t for bounded buffer
Kernel Module	monitor.ko – registers containers, checks RSS every 2 seconds
Memory Enforcement	Soft limit (warning) + Hard limit (SIGKILL)
Process Reaping	SIGCHLD handler with waitpid(-1, &status, WNOHANG)
CFS Scheduler	Fair CPU distribution, nice values affect weight
📸 Screenshots

All demo screenshots are available in the Screenshots/ directory:

    Environment check

    Successful build

    Kernel module loaded + device file

    Supervisor running

    ps output with running containers

    Container logs

    Container stopped (killed state)

    dmesg showing soft/hard limit messages

    Memory hog killed state

    Clean shutdown (no zombies)
    11-13. Scheduler experiment results

👨‍💻 Contributors

    Shreeranganath M Saravade - PES1UG24CS622

    Suhas Bajantri - PES1UG24CS631

📝 License

This project was developed as part of the Operating Systems course at PES University.
🙏 Acknowledgments

    Professor for the project skeleton and guidance

    Linux kernel documentation for namespaces and module programming

    Alpine Linux for the minimal root filesystemlisted in the project guide.

Your fork's `README.md` should be replaced with your own project documentation as described in the submission package section of the project guide. (As in get rid of all the above content and replace with your README.md)
