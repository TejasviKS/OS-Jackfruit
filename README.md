# Multi-Container Runtime

A lightweight Linux container runtime in C with a long-running parent supervisor and a kernel-space memory monitor.

---

## 1. Team Information

| Name | SRN |
|------|-----|
| Tejasvi K S | PES1UG24CS499 |
| Tanmay T A  | PES1UG24CS492 |

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

This produces: `engine`, `memory_hog`, `cpu_hog`, `io_pulse`, and `monitor.ko`

### Prepare root filesystems

```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

# Copy workload binaries into base rootfs before creating per-container copies
cp memory_hog cpu_hog io_pulse rootfs-base/

# Create per-container writable copies
cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta
```

### Load the kernel module

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor    # should exist
dmesg | tail                    # should show: Module loaded
```

### Start the supervisor (Terminal 1)

```bash
sudo ./engine supervisor ./rootfs-base
```

The supervisor binds a UNIX socket at `/tmp/mini_runtime.sock` and enters its accept loop.

### Use the CLI (Terminal 2)

```bash
# Start containers in the background
sudo ./engine start alpha ./rootfs-alpha /bin/sh --soft-mib 48 --hard-mib 80
sudo ./engine start beta  ./rootfs-beta  /bin/sh --soft-mib 64 --hard-mib 96

# List tracked containers and metadata
sudo ./engine ps

# Run a container in the foreground and wait for it to finish
sudo ./engine run logtest ./rootfs-alpha "/cpu_hog 5"

# Inspect captured log output
sudo ./engine logs alpha

# Stop a running container
sudo ./engine stop alpha
sudo ./engine stop beta
```

### Run memory limit tests

```bash
sudo ./engine start memtest ./rootfs-alpha "/memory_hog" --soft-mib 40 --hard-mib 60
sudo ./engine ps          # state will change to hard_limit_killed
dmesg | tail              # shows SOFT LIMIT then HARD LIMIT events
```

### Run scheduling experiments

```bash
# Experiment 1: Different nice values
sudo ./engine start c1 ./rootfs-alpha "/cpu_hog 10" --nice 0
sudo ./engine start c2 ./rootfs-beta  "/cpu_hog 10" --nice 10

# Experiment 2: CPU-bound vs I/O-bound
sudo ./engine start cpu ./rootfs-alpha "/cpu_hog 10"
sudo ./engine start io  ./rootfs-beta  "/io_pulse 20 200"
```

### Unload the module and clean up

```bash
# Stop all containers, then Ctrl-C the supervisor
sudo rmmod monitor
dmesg | tail    # shows: Module unloaded
```

---

## 3. Demo with Screenshots

---

### Screenshot 1 — Multi-container supervision

![Supervisor Started](screenshots/task1_supervisor_running.png)

**The supervisor process starts**, binds the UNIX socket at `/tmp/mini_runtime.sock`, and enters its event loop waiting for CLI connections.

![Two Containers Running](screenshots/task1_multiple_containers.png)

**Two containers running under one supervisor.** `alpha` (PID 3299) and `beta` (PID 3307) are both launched and tracked concurrently. `engine ps` confirms both are in `running` state with their respective soft (40 MiB) and hard (64 MiB) memory limits recorded in the supervisor's metadata table.

![Namespace Isolation](screenshots/task1_namespace_isolation.png)

**PID and UTS namespace isolation confirmed.** `nsenter` enters container `alpha`'s namespaces. Inside, `ps` shows only the container's own processes (isolated PID namespace) and `hostname` returns `alpha` (isolated UTS namespace) — completely separate from the host.

![Filesystem Isolation](screenshots/task1_filesystem_isolation.png)

**Filesystem isolation per container.** A file `hello` written to `rootfs-alpha/` is visible there but absent from `rootfs-beta/`, confirming each container has its own independent writable rootfs copy.

---

### Screenshot 2 — Metadata tracking

![Container Lifecycle](screenshots/task1_container_lifecycle.png)

**`engine ps` showing live metadata tracking.** After `engine stop alpha`, the supervisor updates `alpha`'s state to `stopped` while `beta` and `test` remain `running`. The table shows all tracked fields: container ID, host PID, state, exit code, and configured memory limits — all maintained under a mutex-protected metadata table inside the supervisor.

---

### Screenshot 3 — Bounded-buffer logging

![Logging Output via run](screenshots/task3_logging_output.png)

**Log pipeline in action via `engine run`.** `engine run logtest ./rootfs-alpha "/cpu_hog 5"` launches a container and the supervisor's pipeline captures all stdout through the producer thread → bounded buffer → consumer thread path. `engine logs logtest` retrieves all 6 lines of captured output from `logs/logtest.log`, confirming the full pipeline is working.

![Concurrent Logging](screenshots/task3_concurrent_logging.png)

**Concurrent per-container logging.** Two containers (`c1` and `c2`) each ran `cpu_hog` simultaneously. Their stdout was routed through separate producer threads into the shared bounded buffer and written to independent log files. `engine logs c1` and `engine logs c2` show complete, uninterleaved output for each container — no data loss or corruption.

![Log Files Directory](screenshots/task3_log_files.png)

**Persistent per-container log files.** The `logs/` directory contains a separate `.log` file for every container that ran (`alpha.log`, `fg.log`, `logtest.log`, `test1.log`, `test.log`), each written exclusively by the consumer thread routing on container ID.

---

### Screenshot 4 — CLI and IPC

![CLI Commands](screenshots/task2_cli_commands.png)

**All CLI commands exercised over the UNIX socket control channel (Path B IPC).** In sequence: `start` launches `alpha` and returns immediately; `ps` lists metadata; `run` launches `test1` with `/bin/echo hello`, blocks until exit, and returns the final state; `logs test1` retrieves the captured output (`hello`) streamed back through the socket; `stop beta` terminates the container. Every command is a short-lived client process that connects to `/tmp/mini_runtime.sock`, sends a `control_request_t`, and receives a `control_response_t` — a completely separate IPC mechanism from the logging pipes.

![Supervisor Shutdown](screenshots/task2_supervisor_shutdown.png)

**Supervisor receiving and responding to CLI commands, then shutting down cleanly.** The supervisor prints each accepted command's result in real time. On Ctrl-C (SIGINT), it prints `[supervisor] shutting down...` and then `[supervisor] clean exit`, confirming the control channel, signal handling, and orderly teardown all work correctly.

---

### Screenshot 5 — Soft-limit warning

![Soft and Hard Limits dmesg](screenshots/task4_soft_hard_limits.png)

**`dmesg` showing the soft-limit warning event.** Container `memtest` (PID 4082) is registered with soft=41943040 bytes (40 MiB) and hard=62914560 bytes (60 MiB). When the kernel module's 1-second timer detects RSS=42569728 bytes exceeding the soft limit, it logs `SOFT LIMIT container=memtest pid=4082` — a one-time warning emitted by `log_soft_limit_event()` in the kernel module.

---

### Screenshot 6 — Hard-limit enforcement

![Hard Limit dmesg](screenshots/task4_soft_hard_limits.png)

**`dmesg` showing the hard-limit kill event** (same screen as Screenshot 5, lower line). When the timer next detects RSS=67710976 bytes exceeding the hard limit of 62914560, it calls `kill_process()` which sends SIGKILL via `send_sig()` in kernel space.

![Hard Limit Killed State](screenshots/task4_hard_limit_killed.png)

**Supervisor metadata reflecting the hard-limit kill.** After the kernel module sends SIGKILL, the supervisor receives SIGCHLD, reaps the child, and classifies the termination as `hard_limit_killed` — because the exit signal was SIGKILL and `stop_requested` was not set (it was not stopped via `engine stop`). `engine ps` confirms state=`hard_limit_killed` for `memtest`.

![Device Creation](screenshots/task4_device_creation.png)

**`/dev/container_monitor` created on module load.** `sudo insmod monitor.ko` creates the character device (major 235). The supervisor opens this device and uses `ioctl(MONITOR_REGISTER)` to hand each container's PID and memory limits to the kernel module at launch time.

---

### Screenshot 7 — Scheduling experiment

![CPU Priority Experiment](screenshots/task5_cpu_priority.png)

**Experiment 1: Two CPU-bound containers at different nice values.** Container `c1` runs with `--nice 0` (default CFS weight ≈1024) and `c2` runs with `--nice 10` (CFS weight ≈110). Both run the same `cpu_hog` workload for 10 seconds. The logs retrieved afterwards show `c1` accumulating far more iterations per second than `c2`, reflecting the scheduler allocating CPU proportional to weight: c1 gets ≈90%, c2 gets ≈10% of CPU time.

![CPU vs IO Experiment](screenshots/task5_cpu_vs_io.png)

**Experiment 2: CPU-bound vs I/O-bound at equal priority.** `cpu_hog` (CPU-bound) and `io_pulse` (I/O-bound, 200ms sleep between writes) run concurrently at nice=0. Despite `cpu_hog` saturating the CPU, `io_pulse` completes all 20 iterations on schedule without any delay. CFS immediately schedules `io_pulse` when it wakes from sleep because its virtual runtime is far below `cpu_hog`'s — demonstrating CFS's built-in responsiveness for I/O-bound workloads.

---

### Screenshot 8 — Clean teardown

![No Zombies](screenshots/task6_no_zombies.png)

**No zombie processes after container exit.** `ps aux | grep defunct` returns only the grep process itself — zero `[defunct]` entries. The supervisor calls `waitpid(-1, &status, WNOHANG)` in its main loop on every iteration and on every SIGCHLD, reaping all children promptly.

![Supervisor Clean Exit](screenshots/task6_supervisor_cleanup.png)

**All containers reaped and threads joined on supervisor shutdown.** The supervisor prints exit notifications for every container (`c1`, `c2`, `cpu`, `io`) as they are reaped. Then `[supervisor] shutting down...` is followed by `[supervisor] clean exit` — confirming all producer threads were joined, the bounded buffer was drained, and the consumer thread exited cleanly.

![Kernel Module Cleanup](screenshots/task6_kernel_cleanup.png)

**Kernel module frees all list entries on unload.** `dmesg` shows each container being registered, then either auto-removed when its process exited or explicitly unregistered via `MONITOR_UNREGISTER ioctl`. The final line `Module unloaded.` confirms `module_exit` iterated the linked list and freed every remaining `monitored_entry` node with no memory leaks.

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

The runtime uses `clone()` with `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS` to create isolated containers.

**PID namespace** gives each container its own PID space starting at 1 — processes inside cannot see or signal host processes. **UTS namespace** gives each container its own hostname, set to the container ID via `sethostname()`. **Mount namespace** combined with `chroot()` into a dedicated `rootfs-*` directory means the container can only see its own filesystem; `/proc` is mounted inside so tools like `ps` work.

**What the host still shares:** the kernel itself, the network stack (no `CLONE_NEWNET`), the IPC namespace, and the CPU scheduler. A container can still exhaust host CPU or memory. Full isolation would also need `CLONE_NEWNET`, cgroups, and seccomp. We use `chroot()` over `pivot_root` for simplicity — `pivot_root` is more secure as it makes the old root completely unreachable, but is not required by the spec.

---

### 4.2 Supervisor and Process Lifecycle

A long-running supervisor is necessary because: (1) **zombie prevention** — `SIGCHLD` is delivered only to the direct parent, and the supervisor calls `waitpid(-1, WNOHANG)` in its loop to reap all children immediately; (2) **metadata persistence** — container state, PIDs, and log paths must outlive any individual container; (3) **logging ownership** — the bounded buffer and consumer thread must live for the full session; (4) **signal coordination** — only the direct parent receives `SIGCHLD`.

The supervisor uses `clone()` (not `fork()`) for namespace control. The child calls `chroot()`, mounts `/proc`, and `execv()`s the command. **Termination classification:** if `stop_requested` was set before signalling → `CONTAINER_STOPPED`; if exit signal is SIGKILL without `stop_requested` → `CONTAINER_HARD_LIMIT_KILLED`; otherwise → `CONTAINER_EXITED`.

---

### 4.3 IPC, Threads, and Synchronisation

**Path A (logging — pipes):** Each container's stdout/stderr is redirected via `dup2()` to a pipe write end. A producer thread per container reads from the pipe and pushes `log_item_t` entries into the bounded buffer. One consumer thread drains the buffer and writes to per-container log files.

**Path B (control — UNIX socket):** The supervisor binds a `SOCK_STREAM` socket at `/tmp/mini_runtime.sock`. Each CLI call connects, sends a `control_request_t`, and receives a `control_response_t` — a completely separate mechanism from the logging pipes.

**Bounded buffer:** 64-slot circular array protected by a `pthread_mutex_t` and two condition variables (`not_empty`, `not_full`). Without synchronisation: two producers could write to the same slot simultaneously; a consumer could read a partially-written entry; threads could sleep forever on shutdown. Condition variables are chosen so threads block efficiently (no busy-wait) and `pthread_cond_broadcast()` on shutdown wakes all waiters cleanly. A separate `metadata_lock` protects the container table to avoid holding both locks at once.

---

### 4.4 Memory Management and Enforcement

**RSS** (Resident Set Size) counts physical RAM pages currently mapped for a process (`get_mm_rss(mm) * PAGE_SIZE`). It excludes swapped pages, shared library pages, and `malloc`'d-but-untouched pages (demand paging). **Soft limit** = warning only — the module logs `KERN_WARNING` once when RSS first exceeds it, letting operators investigate without killing the workload. **Hard limit** = enforcement — the module calls `send_sig(SIGKILL, ...)` unconditionally when RSS exceeds it.

Enforcement belongs in kernel space because a user-space monitor can be delayed by the scheduler for hundreds of milliseconds, while a kernel timer fires at a guaranteed interval. `send_sig()` from kernel space also cannot be caught or blocked by the target process.

---

### 4.5 Scheduling Behaviour

Linux CFS tracks **virtual runtime** per process — the process with the lowest virtual runtime runs next. A lower nice value → higher CFS weight → virtual runtime grows more slowly → scheduled more often.

**Experiment 1 (nice=0 vs nice=10):** CFS weight for nice=0 is ≈1024, for nice=10 is ≈110. The nice=0 container received ≈90% of CPU time (1024/1134), accumulating far more iterations per second — demonstrating **fairness** via proportional weight-based allocation.

**Experiment 2 (CPU-bound vs I/O-bound):** While `io_pulse` slept in `usleep()`, it accumulated no virtual runtime. On wakeup its virtual runtime was far below `cpu_hog`'s, so CFS immediately preempted `cpu_hog` — demonstrating **responsiveness** for I/O-bound workloads even under full CPU saturation.

---

## 5. Design Decisions and Tradeoffs

| Subsystem | Choice | Tradeoff | Justification |
|-----------|--------|----------|---------------|
| Namespace isolation | `CLONE_NEWPID\|NEWUTS\|NEWNS` + `chroot()` | No network isolation (`CLONE_NEWNET`), containers share host network stack | Spec requires only PID, UTS, mount — network isolation needs veth/bridge setup outside scope |
| Supervisor architecture | Single process, `select()` event loop, per-container producer threads, one consumer | CLI requests are serialised — concurrent `engine start` calls queue up | Single-threaded control avoids concurrent metadata mutation bugs; adequate for demo scale |
| IPC and logging | UNIX socket (control) + pipes (logging) + mutex+CV bounded buffer | Stale socket file if supervisor crashes; mitigated by `unlink()` before `bind()` | UNIX sockets give full-duplex request-response naturally; mutex+CV gives efficient blocking and clean broadcast shutdown |
| Kernel monitor | `DEFINE_MUTEX` over spinlock | `mutex_trylock` in softirq context skips a tick if contested | `kmalloc(GFP_KERNEL)` in ioctl path may sleep — forbidden under spinlock; skipped tick is acceptable at 1s granularity |
| Scheduling experiments | `nice()` for priority, log iteration counts as measurement | Nice values are hints, not hard guarantees; host load adds noise | No root cgroup config needed; 10-second runs average out noise and give clear, explainable results |

---

## 6. Scheduler Experiment Results

### Experiment 1: CPU-bound workloads at different priorities

Two containers ran `cpu_hog` simultaneously for 10 seconds — `c1` at nice=0, `c2` at nice=10.

![CPU Priority Experiment](screenshots/task5_cpu_priority.png)

**What it shows:** `c1` (nice=0) and `c2` (nice=10) are started. `c1` finishes its full 10-second workload significantly ahead in accumulated iterations because CFS allocated it ≈90% of CPU time.

| Container | Nice | CFS Weight | Expected CPU share |
|-----------|------|------------|-------------------|
| c1        | 0    | ≈1024      | 1024/1134 ≈ 90%   |
| c2        | 10   | ≈110       | 110/1134 ≈ 10%    |

The logs retrieved with `engine logs c1` and `engine logs c2` confirm this — `c1`'s accumulator values grow much faster per elapsed second than `c2`'s.

---

### Experiment 2: CPU-bound vs I/O-bound at equal priority

`cpu_hog` (CPU-bound) and `io_pulse` (I/O-bound, 200ms sleep between writes) ran simultaneously at nice=0.

![CPU vs IO Experiment](screenshots/task5_cpu_vs_io.png)

**What it shows:** Despite `cpu_hog` running 9 continuous seconds and saturating the CPU, `io_pulse` completed all 20 iterations fully on schedule with no missed or delayed writes. The `engine logs cpu` output shows `cpu_hog` ran uninterrupted for the full 10s. The `engine logs io` output shows `io_pulse` wrote all 20 iterations cleanly.

| Workload | Type | Nice | Duration | Completed on time? |
|----------|------|------|----------|--------------------|
| cpu_hog  | CPU-bound  | 0 | 10s | Yes — used all available CPU |
| io_pulse | I/O-bound  | 0 | ~4s | Yes — all 20 iterations, no delay |

**Why:** While sleeping in `usleep(200ms)`, `io_pulse` accumulated zero virtual runtime. Each time it woke, its virtual runtime was far below `cpu_hog`'s, so CFS immediately preempted `cpu_hog` to run `io_pulse`. This is CFS's built-in responsiveness guarantee for I/O-bound workloads.

**Conclusion:** Experiment 1 demonstrates **fairness** — CPU time is divided proportionally to priority weight. Experiment 2 demonstrates **responsiveness** — sleeping processes wake and run immediately regardless of CPU load. Together they show CFS achieves fairness, responsiveness, and throughput simultaneously.
