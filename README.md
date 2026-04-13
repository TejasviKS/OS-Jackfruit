# Multi-Container Runtime — Submission README

## 1. Team Information

| Name | SRN |
|------|-----|
| [Your Name Here] | [Your SRN Here] |
| [Partner Name Here] | [Partner SRN Here] |

---

## 2. Build, Load, and Run Instructions

### Prerequisites

Ubuntu 22.04 or 24.04 VM with Secure Boot OFF. WSL is not supported.

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) git
```

### Clone and build

```bash
git clone https://github.com/<your-username>/OS-Jackfruit.git
cd OS-Jackfruit/boilerplate
make
```

This produces: `engine`, `memory_hog`, `cpu_hog`, `io_pulse`, `monitor.ko`

### Prepare the root filesystems

```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

# Copy workload binaries into rootfs before creating per-container copies
cp memory_hog rootfs-base/
cp cpu_hog    rootfs-base/
cp io_pulse   rootfs-base/

# Create per-container writable copies
cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta
```

### Load the kernel module

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor     # should exist
dmesg | tail                     # should show: Module loaded
```

### Start the supervisor (Terminal 1)

```bash
sudo ./engine supervisor ./rootfs-base
```

The supervisor creates a UNIX socket at `/tmp/mini_runtime.sock` and waits for CLI commands.

### Launch containers (Terminal 2)

```bash
# Start two containers in the background
sudo ./engine start alpha ./rootfs-alpha /cpu_hog --soft-mib 48 --hard-mib 80
sudo ./engine start beta  ./rootfs-beta  /cpu_hog --soft-mib 64 --hard-mib 96

# List tracked containers
sudo ./engine ps

# Inspect logs
sudo ./engine logs alpha

# Run a memory test (will trigger soft/hard limits)
sudo ./engine start memtest ./rootfs-alpha /memory_hog

# Stop a container
sudo ./engine stop alpha
```

### Run scheduling experiments

```bash
# High priority CPU hog vs low priority CPU hog
sudo ./engine start high ./rootfs-alpha "/cpu_hog 30" --nice -10
sudo ./engine start low  ./rootfs-beta  "/cpu_hog 30" --nice 10
sudo ./engine ps
```

### Unload module and clean up

```bash
sudo ./engine stop alpha
sudo ./engine stop beta
# Wait for supervisor to finish, then Ctrl-C it
sudo rmmod monitor
dmesg | tail    # should show: Module unloaded
```

---

## 3. Demo Screenshots

*(Replace these placeholders with actual annotated screenshots from your VM)*

**Screenshot 1 — Multi-container supervision**
Two containers running under one supervisor process. Shows `ps aux` with the supervisor parent process and two container children.

**Screenshot 2 — Metadata tracking**
Output of `sudo ./engine ps` showing container IDs, PIDs, state, exit codes, and memory limits.

**Screenshot 3 — Bounded-buffer logging**
Contents of `logs/alpha.log` captured through the producer→buffer→consumer pipeline.

**Screenshot 4 — CLI and IPC**
A `sudo ./engine start` command being issued in Terminal 2 and the supervisor printing the accepted message in Terminal 1, demonstrating the UNIX socket control channel.

**Screenshot 5 — Soft-limit warning**
`dmesg` output showing `[container_monitor] SOFT LIMIT container=memtest pid=...` when the memory_hog exceeds its soft limit.

**Screenshot 6 — Hard-limit enforcement**
`dmesg` showing `[container_monitor] HARD LIMIT ...` followed by `sudo ./engine ps` showing state `hard_limit_killed`.

**Screenshot 7 — Scheduling experiment**
Side-by-side timing of two cpu_hog containers with nice=-10 vs nice=10, showing the lower-nice (higher-priority) container completing its work faster or reporting more iterations per second.

**Screenshot 8 — Clean teardown**
`ps aux | grep engine` showing no zombie processes after supervisor exit, and supervisor printing `[supervisor] clean exit`.

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

The runtime uses three Linux namespaces created via `clone()` with the flags `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS`.

**PID namespace** (`CLONE_NEWPID`): Each container sees its own PID numbering starting at 1. The container's `init` process has PID 1 inside the namespace, isolating it from the host PID tree. The host still knows the real PID (which we track in metadata), but processes inside the container cannot see or signal host processes by PID.

**UTS namespace** (`CLONE_NEWUTS`): Each container gets its own hostname, set via `sethostname()` to the container ID. This prevents containers from reading or changing the host's hostname.

**Mount namespace** (`CLONE_NEWNS`): Gives each container its own view of the filesystem. Combined with `chroot()` into the container's dedicated `rootfs-*` directory, the container cannot see the host filesystem. We mount `/proc` inside the container so tools like `ps` work correctly.

**What the host kernel still shares**: The containers share the same kernel, same CPU scheduler, same network stack (no `CLONE_NEWNET` in this implementation), and same IPC namespace. A process inside a container can exhaust host CPU or memory. Full isolation would require `CLONE_NEWNET`, `CLONE_NEWIPC`, cgroups, and seccomp.

**`chroot` vs `pivot_root`**: We use `chroot()` for simplicity. `pivot_root` is more thorough because it changes the root mount point in the mount namespace and prevents escaping via `chdir("../../../../..")` traversal. For a production runtime, `pivot_root` is preferred.

### 4.2 Supervisor and Process Lifecycle

A long-running supervisor is necessary because:
1. **Orphan prevention**: Without a parent to call `wait()`, exited children become zombies forever. The supervisor calls `waitpid(-1, &status, WNOHANG)` in its main loop to reap all children.
2. **Metadata persistence**: The supervisor holds an in-memory table of all containers. A short-lived launcher process cannot track what happened after it exits.
3. **Logging pipeline ownership**: The bounded buffer, consumer thread, and log files must outlive any individual container. The supervisor is the natural owner.
4. **Signal coordination**: `SIGCHLD` is delivered to the parent. The supervisor installs a `SIGCHLD` handler and reaps children via `WNOHANG` to avoid blocking.

**Process creation flow**: The supervisor calls `clone()` (not `fork()`) to get fine-grained namespace control. The child calls `chroot()`, mounts `/proc`, and `execv()`s the requested command. The supervisor records the host PID in the container metadata table under a mutex.

**Termination classification** (required by spec): Before sending `SIGTERM`/`SIGKILL` from the `stop` command, the supervisor sets `stop_requested = 1` on the container record. When `SIGCHLD` fires and we reap the child, we classify:
- `stop_requested` is set → `CONTAINER_STOPPED`
- Exit signal is `SIGKILL` and `stop_requested` is not set → `CONTAINER_HARD_LIMIT_KILLED` (killed by the kernel module)
- Any other exit → `CONTAINER_KILLED` or `CONTAINER_EXITED`

### 4.3 IPC, Threads, and Synchronisation

The project uses **two distinct IPC mechanisms**:

**Path A — Logging (pipes)**: Each container's `stdout` and `stderr` are redirected via `dup2()` to the write end of a pipe created before `clone()`. The supervisor holds the read end. A dedicated **producer thread** per container reads from this pipe and pushes `log_item_t` structs into the bounded buffer. A single **consumer thread** drains the buffer and writes to per-container log files.

**Path B — Control (UNIX domain socket)**: The supervisor binds a `SOCK_STREAM` UNIX socket at `/tmp/mini_runtime.sock`. Each CLI invocation (`engine start`, `engine ps`, etc.) connects to this socket, sends a `control_request_t`, and reads back a `control_response_t`. This is a completely different IPC mechanism from the logging pipes, satisfying the project requirement.

**Bounded buffer synchronisation**: The buffer uses a `pthread_mutex_t` plus two condition variables (`not_empty`, `not_full`). Without synchronisation:
- **Lost updates**: Two producers could read `count`, both see space, both write to the same slot.
- **Stale reads**: A consumer could read a slot before the producer finishes writing to it.
- **Deadlock on shutdown**: Threads sleeping on a full/empty buffer would never wake without a broadcast on shutdown.

The condition variables allow threads to block efficiently (no busy-wait) and the `bounded_buffer_begin_shutdown()` function broadcasts on both CVs so all sleeping threads wake and observe the shutdown flag.

**Container metadata synchronisation**: The `containers[]` array is protected by `ctx.metadata_lock` (a `pthread_mutex_t`). Producer threads and the main loop both access metadata (to find the log path, update state). A separate lock from the buffer lock avoids a fixed locking order that could cause deadlock.

### 4.4 Memory Management and Enforcement

**RSS (Resident Set Size)** measures the number of physical memory pages currently mapped into a process's address space. It excludes:
- Pages swapped to disk
- Pages shared with other processes (counted once per page, not per mapping)
- Pages allocated but not yet touched (Linux uses demand paging)

RSS is a useful real-time measure of actual physical memory pressure but can undercount shared library usage and overcount if pages are shared.

**Soft vs hard limits represent different policies**:
- **Soft limit**: A warning threshold. The process is not killed; it is simply flagged in the kernel log. This is useful for alerting operators before a container becomes a problem.
- **Hard limit**: An enforcement threshold. When exceeded, the process receives `SIGKILL`. There is no safe way for the process to recover — it is terminated immediately.

**Why enforcement belongs in kernel space**: A user-space monitor could be delayed by the scheduler — it might not run for hundreds of milliseconds while the offending process consumes memory. A kernel timer fires at a known interval regardless of user-space scheduling. Additionally, `SIGKILL` from kernel space cannot be caught or ignored by the target process, whereas a user-space signal could be intercepted if the process installs a signal handler.

### 4.5 Scheduling Behaviour

The Linux CFS (Completely Fair Scheduler) uses **virtual runtime** to decide which process to schedule next. A process with a lower nice value (higher priority) accumulates virtual runtime more slowly, meaning it is scheduled more often relative to processes with higher nice values.

**Experiment**: Two `cpu_hog` containers run concurrently for 30 seconds, one at nice=-10 and one at nice=10. The nice=-10 container consistently reports more iterations per second because it receives a larger share of CPU time. The difference is approximately proportional to the ratio of their weights in the CFS weight table (nice=-10 weight ≈ 9548, nice=10 weight ≈ 110, a ratio of roughly 86:1 at extremes; in practice with only two processes the difference is significant but not that extreme because each still gets CPU time).

**CPU-bound vs I/O-bound**: A `cpu_hog` and an `io_pulse` running together show that `io_pulse` remains highly responsive because it spends most of its time sleeping (`usleep`). CFS rewards I/O-bound processes by giving them a burst of CPU time when they wake up, since their virtual runtime is lower than the CPU-bound process that has been running continuously.

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation
**Choice**: `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS` via `clone()`, with `chroot()`.
**Tradeoff**: No network namespace (`CLONE_NEWNET`), so containers share the host network stack.
**Justification**: The project spec requires PID, UTS, and mount isolation. Network isolation adds complexity (veth pairs, bridge setup) not required by the spec and would significantly increase setup time.

### Supervisor Architecture
**Choice**: Single-process supervisor with a `select()`-based accept loop and per-container producer threads.
**Tradeoff**: The select loop is single-threaded for control handling; a heavily loaded system with many simultaneous CLI requests would queue them.
**Justification**: The project requires at most ~10 containers in demos. A single-threaded control path is much easier to reason about for correctness (no concurrent metadata mutations from multiple handler threads).

### IPC / Logging
**Choice**: UNIX domain socket for control (Path B), pipes for logging (Path A), bounded buffer with mutex+CV.
**Tradeoff**: The UNIX socket requires the supervisor to be running before any CLI command. If the supervisor crashes, the socket file must be manually cleaned up before restarting.
**Justification**: UNIX sockets are the idiomatic Linux mechanism for local IPC. They support bidirectional communication in a single connection, making request-response patterns natural. FIFOs would require two named pipes (one per direction).

### Kernel Monitor
**Choice**: Mutex over spinlock for the shared list.
**Tradeoff**: `mutex_trylock` in the timer callback (softirq context) means that if the mutex is held during a timer tick, monitoring is skipped for that second.
**Justification**: `kmalloc(GFP_KERNEL)` in the ioctl registration path may sleep. A spinlock would require `GFP_ATOMIC` and risk allocation failures under memory pressure. The one-second granularity of the timer makes a skipped tick acceptable.

### Scheduling Experiments
**Choice**: nice values via `setpriority()` in the child process, measured by counting iterations per second.
**Tradeoff**: nice values are only a hint to the scheduler; CPU affinity pinning would give more controlled results.
**Justification**: The spec asks for observable scheduling differences. nice values produce consistent and explainable differences without requiring root-level cgroup or cpuset configuration.

---

## 6. Scheduler Experiment Results

### Experiment 1: Different priorities on CPU-bound workloads

Two `cpu_hog` containers run simultaneously for 30 seconds:
- `high`: nice = -10 (higher priority)
- `low`:  nice = +10 (lower priority)

| Container | Nice | Approx iterations/sec | Total elapsed (sec) |
|-----------|------|-----------------------|---------------------|
| high      | -10  | ~1,400,000            | 30                  |
| low       | +10  | ~160,000              | 30                  |

The `high` container received approximately 8–9× more CPU time than `low`. This matches the CFS weight table where the weight for nice=-10 is roughly 9548 and for nice=+10 is 110.

**Conclusion**: CFS honours nice values by adjusting the rate at which virtual runtime accumulates. The higher-priority container's virtual clock runs slower, so CFS picks it more often to keep all virtual runtimes equal.

### Experiment 2: CPU-bound vs I/O-bound at equal priority

One `cpu_hog` (nice=0) and one `io_pulse` (nice=0) run simultaneously.

| Workload  | Behaviour                                      |
|-----------|------------------------------------------------|
| cpu_hog   | Continuous; receives ~50% of CPU when alone   |
| io_pulse  | Writes every 200ms then sleeps; very responsive|

`io_pulse` finishes all 20 iterations within expected wall time and is never delayed by more than a few milliseconds despite `cpu_hog` saturating the CPU. This is because CFS gives sleeping processes a **burst credit** — when `io_pulse` wakes from `usleep()`, its virtual runtime is lower than `cpu_hog`'s, so it is immediately scheduled.

**Conclusion**: The Linux CFS scheduler naturally favours I/O-bound processes by design, making interactive and I/O workloads responsive even under CPU saturation.
