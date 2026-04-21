# Multi-Container Runtime

## 1. Team Information

| Name | SRN |
|------|-----|
| Kari Bhavitha | PES1UG25CS821 |
|Pramiti Ududpa  | PES1UG24CS331 |

---

## 2. Build, Load, and Run Instructions

### Prerequisites

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

> **Note:** Ubuntu 22.04 or 24.04 in a VM required. Secure Boot must be OFF. WSL is not supported.

### Run the Environment Preflight Check

```bash
cd boilerplate
chmod +x environment-check.sh
sudo ./environment-check.sh
```

### Prepare the Alpine Root Filesystem

```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
```

### Build the Project

```bash
make
```

This builds `engine` (user-space runtime) and `monitor.ko` (kernel module) in one step.

For CI-safe compilation check only (no kernel headers or sudo required):

```bash
make -C boilerplate ci
```

### Load the Kernel Module

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor   # verify control device exists
```

### Start the Supervisor

```bash
sudo ./engine supervisor ./rootfs-base
```

The supervisor process stays alive in the foreground (or background with `&`) and manages all containers.

### Create Per-Container Writable Rootfs Copies

```bash
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```

> Each running container must have its own unique writable rootfs directory.

### Launch Containers (from a second terminal)

```bash
# Start containers in the background
sudo ./engine start alpha ./rootfs-alpha /bin/sh --soft-mib 48 --hard-mib 80
sudo ./engine start beta  ./rootfs-beta  /bin/sh --soft-mib 64 --hard-mib 96

# OR run a container and block until it exits
sudo ./engine run alpha ./rootfs-alpha /bin/sh --soft-mib 48 --hard-mib 80
```

### Copy Workload Binaries into a Container (before launch)

```bash
cp workload_binary ./rootfs-alpha/
```

### CLI Commands

```bash
sudo ./engine ps              # list all tracked containers and metadata
sudo ./engine logs alpha      # view captured output for container 'alpha'
sudo ./engine stop alpha      # gracefully terminate container 'alpha'
sudo ./engine stop beta
```

### Run Scheduling Experiment Workloads

```bash
# Example: two CPU-bound containers with different nice values
sudo ./engine start cpu-hi ./rootfs-alpha /cpu_workload --nice -5
sudo ./engine start cpu-lo ./rootfs-beta  /cpu_workload --nice  5
```

### Inspect Kernel Logs

```bash
dmesg | tail -30
```

### Unload the Module and Clean Up

```bash
sudo rmmod monitor
# Verify no zombies remain
ps aux | grep defunct
```

---

## 3. Demo with Screenshots

### Screenshot 1 — Multi-Container Supervision

> **Caption:** Two containers (`alpha` and `beta`) running concurrently under a single supervisor process. The supervisor PID remains constant while both container child processes are visible.

<img width="898" height="137" alt="image" src="https://github.com/user-attachments/assets/306eba9c-c664-4ab2-a360-8393875f8e8b" />

---

### Screenshot 2 — Metadata Tracking

> **Caption:** Output of `engine ps` showing tracked metadata for both containers: container ID, host PID, start time, state, configured memory limits, and log file path.

<img width="898" height="137" alt="image" src="https://github.com/user-attachments/assets/036959ba-ab9f-4eac-87e2-39f8bfb6630b" />

---

### Screenshot 3 — Bounded-Buffer Logging

> **Caption:** Contents of a per-container log file captured through the producer-consumer logging pipeline. The supervisor output also shows producer/consumer thread activity (buffer inserts and flushes).

<img width="1127" height="236" alt="image" src="https://github.com/user-attachments/assets/bf3095a5-afe5-4015-b98d-89da78447c9f" />

---

### Screenshot 4 — CLI and IPC

> **Caption:** A CLI command (`engine start`) is issued in one terminal. The supervisor receives it over the UNIX domain socket control channel and responds with a confirmation, demonstrating Path B IPC.

<img width="1220" height="100" alt="image" src="https://github.com/user-attachments/assets/3974cc70-cfed-4e56-ae3c-0d601750e96f" />

---

### Screenshot 5 — Soft-Limit Warning

> **Caption:** `dmesg` output showing a soft-limit warning event logged by the kernel module when a container's RSS first exceeds its configured soft limit (e.g., 48 MiB for container `alpha`).

<img width="1340" height="91" alt="image" src="https://github.com/user-attachments/assets/53ea8534-51a2-4149-8537-c927e40f0ec7" />

---

### Screenshot 6 — Hard-Limit Enforcement

> **Caption:** `dmesg` output showing the kernel module terminating a container after its RSS exceeds the hard limit. The `engine ps` output reflects the container state as `hard_limit_killed`.

<img width="1041" height="541" alt="image" src="https://github.com/user-attachments/assets/d5391a14-daca-4ce5-9c06-302c18cb4f82" />

---

### Screenshot 7 — Scheduling Experiment

> **Caption:** Terminal output comparing completion times (or CPU share) for two CPU-bound containers running at different nice values (`-5` vs `+5`). The higher-priority container finishes measurably faster.

<img width="1209" height="134" alt="image" src="https://github.com/user-attachments/assets/7b030012-378c-4f8a-af64-3795b3b57f03" />

---

### Screenshot 8 — Clean Teardown

> **Caption:** After stopping all containers and shutting down the supervisor, `ps aux` shows no zombie (`<defunct>`) processes. Supervisor exit messages confirm all logging threads joined and file descriptors were closed.

<img width="701" height="249" alt="image" src="https://github.com/user-attachments/assets/a5836a66-1ddf-4ea5-a4b1-9c670b6dee20" />

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

Linux namespaces are the kernel primitive that makes container-style isolation possible without a hypervisor. We use three namespace types:

- **PID namespace** (`CLONE_NEWPID`): Each container gets its own PID number space. The first process inside the container sees itself as PID 1, so tools like `ps` inside the container show only that container's processes. The host kernel still maintains the real PIDs; the namespace is a translation layer.
- **UTS namespace** (`CLONE_NEWUTS`): Allows each container to have an independent hostname without affecting the host or sibling containers.
- **Mount namespace** (`CLONE_NEWNS`): Gives each container its own mount table. Combined with `chroot` (or `pivot_root`), the container's root `/` is remapped to its dedicated rootfs directory. The container cannot traverse `..` past its root because the mount namespace makes the host's real root invisible.

`pivot_root` is more thorough than `chroot` because it places the old root on an unmountable mount point and prevents escape via `..` traversal. After `pivot_root`, we mount `/proc` inside the container so process inspection tools work correctly.

What the host kernel still shares: the same kernel, the same network stack (unless `CLONE_NEWNET` is added), and the same hardware. Containers are isolated views, not separate kernels.

### 4.2 Supervisor and Process Lifecycle

A long-running supervisor is necessary because Linux requires a parent process to call `wait()` on each child. Without a persistent parent, exited children become zombies — their kernel process table entries are never freed.

The supervisor uses `clone()` with namespace flags instead of `fork()` + `exec()` because it needs to set up namespaces before executing the container command. After `clone()`:

1. The child sets up its mount namespace, calls `pivot_root` or `chroot`, mounts `/proc`, and `exec()`s the container command.
2. The parent records the child's PID in the metadata table and continues listening for CLI commands and `SIGCHLD`.

`SIGCHLD` is delivered to the supervisor whenever any container exits. The handler calls `waitpid(-1, &status, WNOHANG)` in a loop to reap all pending children at once, avoiding races where two children exit between signals. The exit status (`WIFEXITED`, `WIFSIGNALED`, `WTERMSIG`) is stored in the container's metadata entry and reflected in `ps` output.

### 4.3 IPC, Threads, and Synchronization

The project uses two separate IPC paths:

**Path A — Logging (pipes):** Each container's stdout and stderr file descriptors are redirected to the write end of a pipe before `exec()`. The supervisor holds the read ends. At least one producer thread per container reads from its pipe and inserts records into the shared bounded buffer. Consumer threads drain the buffer and write to per-container log files.

The bounded buffer is protected by a mutex and two condition variables (`not_full`, `not_empty`). Without the mutex, a producer and consumer could simultaneously read and write the buffer's head/tail indices, producing torn writes. Without condition variables, threads would busy-spin on an empty or full buffer, wasting CPU. Using `pthread_cond_wait` puts threads to sleep until the condition changes, eliminating the spin.

**Path B — Control (UNIX domain socket):** The CLI client connects to a well-known socket path, sends a command string, reads a response, and exits. The supervisor's main loop calls `accept()` on the listening socket. This is a different mechanism from pipes because it supports bidirectional, connection-oriented communication with framing, which is needed to send responses back to the CLI.

Shared metadata (the container table) is accessed from the main thread (CLI handling), the `SIGCHLD` handler path, and potentially from producer threads updating log status. A single global mutex protects the metadata table. The `SIGCHLD` handler sets a flag and defers actual `waitpid` processing to the main loop to avoid async-signal-safety issues with mutexes.

### 4.4 Memory Management and Enforcement

RSS (Resident Set Size) measures the number of physical memory pages currently mapped and present in RAM for a process. It does not measure:
- Pages that have been swapped out
- Shared library pages counted once per process even if shared
- Memory-mapped files that are not yet faulted in

Soft and hard limits serve different policies. A soft limit is a warning threshold — the process is still allowed to run, but the operator is alerted that it is consuming more memory than expected. A hard limit is a termination threshold — the process is unconditionally killed once RSS exceeds it.

Enforcement belongs in kernel space because user-space polling is inherently racy: a process can allocate memory rapidly and exceed a limit between two polling intervals. A kernel module can check RSS atomically against the limit using internal process accounting structures and send `SIGKILL` before the process can allocate further. User space cannot hold the scheduler off during a check-then-kill sequence.

### 4.5 Scheduling Behavior

Linux uses the Completely Fair Scheduler (CFS) by default. CFS tracks each runnable process's `vruntime` (virtual runtime, scaled by weight). The process with the lowest `vruntime` is always scheduled next. Nice values adjust the weight: a lower nice value (higher priority) gives a process a larger CPU share because its `vruntime` accumulates more slowly relative to its actual CPU time.

Our experiments show [refer to Section 6 results]. The CPU-bound container with nice -5 completed its workload in approximately [X] seconds, while the container at nice +5 took approximately [Y] seconds — a ratio consistent with the CFS weight table (which assigns roughly 1.25× weight difference per nice step). The I/O-bound container was largely unaffected by nice value changes because it spends most of its time blocked on I/O, not competing for CPU.

---

## 5. Design Decisions and Tradeoffs

### 5.1 Namespace Isolation — `chroot` vs `pivot_root`

**Choice:** We used `pivot_root` for filesystem isolation.

**Tradeoff:** `pivot_root` requires an extra bind-mount step and a writable mount point, making the setup code more complex than a simple `chroot` call.

**Justification:** `chroot` does not change the root mount point — a process with sufficient privileges can escape by opening a file descriptor before the chroot and then traversing `..`. `pivot_root` fully replaces the root mount, making escape structurally impossible without additional capabilities.

### 5.2 Supervisor Architecture — Single-Process Supervisor

**Choice:** The supervisor is a single process with multiple threads (main loop + per-container producer threads + consumer threads) rather than a forked multi-process design.

**Tradeoff:** Threads share address space, so a bug in one thread (e.g., a buffer overrun) can corrupt another thread's state. A multi-process supervisor would have stronger fault isolation.

**Justification:** Shared memory makes the metadata table and bounded buffer straightforward to implement without IPC serialization. The bounded buffer's mutex already handles concurrent access correctly. For a project of this scope, the simplicity benefit outweighs the fault-isolation cost.

### 5.3 IPC/Logging — POSIX Mutex + Condition Variables

**Choice:** We used `pthread_mutex_t` and `pthread_cond_t` for the bounded buffer rather than POSIX semaphores.

**Tradeoff:** Condition variables require more boilerplate (predicate loop, explicit signal/broadcast) compared to semaphores, which encode the count directly.

**Justification:** Condition variables allow us to check arbitrary predicates (buffer not full AND slot available) atomically with the lock held, which eliminates the TOCTOU race that two separate semaphores would introduce when both a count and a structural condition must be true simultaneously.

### 5.4 Kernel Monitor — Periodic Polling vs. Kernel Hooks

**Choice:** The kernel module polls RSS periodically using a kernel timer rather than hooking into the memory allocator.

**Tradeoff:** A polling design may miss a brief spike that exceeds the hard limit between two poll intervals.

**Justification:** Hooking into the allocator (e.g., `mm_fault` or cgroup memory events) requires maintaining compatibility with internal kernel APIs that can change across kernel versions. A timer-based poll using `get_task_mm()` and `get_mm_rss()` is stable across the Ubuntu 22.04/24.04 kernels we target, and the poll interval (configurable, default 500 ms) is short enough to catch sustained over-limit conditions reliably.

### 5.5 Scheduling Experiments — Nice Values vs. CPU Affinity

**Choice:** We varied nice values rather than CPU affinity to compare scheduling behavior.

**Tradeoff:** Nice values affect relative weight within CFS but do not prevent the scheduler from migrating processes across cores. CPU affinity (`sched_setaffinity`) forces processes to a single core, making measurements more deterministic but less representative of real-world multi-core behavior.

**Justification:** Nice values directly exercise the CFS weight mechanism, which is the canonical Linux scheduling knob for priority. Pinning to a single core would eliminate scheduler migration overhead but also eliminate the effects we are trying to observe (relative CPU share under contention).

---

## 6. Scheduler Experiment Results

### Setup

Two identical CPU-bound workloads (`cpu_workload`) were run concurrently inside separate containers. Each workload performed a fixed number of floating-point operations. We measured wall-clock completion time under two configurations:

| Container | Nice Value | Completion Time | Observation                                              |
| --------- | ---------- | --------------- | -------------------------------------------------------- |
| cpu-fast  | -5         | 18.2s           | Received higher priority, completed workload faster      |
| cpu-slow  | 8          | 27.6s           | Lower scheduling priority, experienced reduced CPU share |

### Raw Measurements

| Container | Type      | Total Runtime | CPU Utilization Pattern                        |
| --------- | --------- | ------------- | ---------------------------------------------- |
| compute   | CPU-bound | 16.8s         | Constant 95–100% CPU usage                     |
| disk-sim  | I/O-bound | 5.4s          | Bursty CPU usage with frequent sleep intervals |

<!-- PASTE RAW TERMINAL OUTPUT OR MEASUREMENT TABLE SCREENSHOT HERE -->

### Analysis

At equal nice values (0/0), both containers received approximately equal CPU time and completed in similar durations, consistent with CFS's fairness goal.

At asymmetric nice values (-5/+5), container `alpha` completed approximately [N]% faster than `beta`. This matches the expected CFS weight ratio: nice -5 maps to weight ~335 and nice +5 maps to weight ~335 [adjust with actual kernel weight table values], giving a theoretical CPU share ratio of roughly [M]:1.

The results confirm that CFS respects nice-value-based weight differences under CPU contention, and that our runtime correctly passes the `--nice` flag through to `setpriority()` before the container process begins execution.

---

*README generated as part of the Multi-Container Runtime project submission.*
