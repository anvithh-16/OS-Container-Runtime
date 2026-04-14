# OS Jackfruit — Supervised Multi-Container Runtime

## Team Members

| Name | SRN |
|------|-----|
| Anvith Vegi | PES1UG24AM047 |
| Anirudh N | PES1UG24AM041 |

---

## Build, Load, and Run Instructions

### Prerequisites

- Ubuntu 24.04 (aarch64)
- Kernel headers installed
- Secure Boot off (or not enforced)

### 1. Download Alpine Linux root filesystem (aarch64)

```bash
cd ~/OS-Container-Runtime
mkdir rootfs
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/aarch64/alpine-minirootfs-3.20.3-aarch64.tar.gz
sudo tar -xzf alpine-minirootfs-3.20.3-aarch64.tar.gz -C rootfs
```

### 2. Build everything

```bash
cd boilerplate
sudo make
```

This produces: `engine` (user-space binary), `monitor.ko` (kernel module), and workload binaries (`cpu_hog`, `io_pulse`, `memory_hog`).

### 3. Copy workloads into rootfs

```bash
sudo cp cpu_hog ../rootfs/
sudo cp io_pulse ../rootfs/
sudo cp memory_hog ../rootfs/
```

### 4. Load the kernel module

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor   # should exist
sudo dmesg | tail -5           # should show "Module loaded"
```

### 5. Start the supervisor (Terminal 1)

```bash
sudo ./engine supervisor ../rootfs
```

### 6. Issue CLI commands (Terminal 2)

```bash
sudo ./engine start alpha ../rootfs /bin/busybox
sudo ./engine start beta  ../rootfs /bin/busybox
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine stop alpha
sudo ./engine stop beta
```

### 7. Verify clean shutdown

```bash
sudo dmesg | tail -20      # shows register/unregister events
ps aux | grep Z            # should show no zombie processes
```

### 8. Unload the kernel module

```bash
sudo rmmod monitor
```

---

## Screenshots

### Screenshot 1 — Two containers running under one supervisor & `./engine ps` showing full metadata

> `./engine ps` output showing both `alpha` and `beta` containers with `running` state under a single supervisor process.
> Output showing container ID, host PID, state, soft limit (MB), and hard limit (MB) for all containers.

<img width="1379" height="707" alt="Screenshot 2026-04-14 at 7 36 15 AM" src="https://github.com/user-attachments/assets/73ddefa4-3f3c-4397-8c10-f3a2d2777b8b" />


### Screenshot 2 — Log file contents and producer/consumer activity

> `./engine logs alpha` showing BusyBox output captured through the pipe → bounded buffer → log file pipeline.

<img width="1393" height="628" alt="Screenshot 2026-04-14 at 7 36 37 AM" src="https://github.com/user-attachments/assets/24bafd70-c45b-4ecd-91f9-d3b63f18c94d" />


### Screenshot 3 — CLI command issued and supervisor responding

> Terminal showing `./engine start alpha` being issued and the supervisor responding with `Started container 'alpha' pid=XXXX`.

<img width="1310" height="79" alt="Screenshot 2026-04-14 at 7 49 58 AM" src="https://github.com/user-attachments/assets/9ff64f16-0921-43c5-a2b9-cd42a1a92860" />


### Screenshot 4 — dmesg showing soft-limit warning

> `dmesg` output showing `[container_monitor] SOFT LIMIT container=... rss=... limit=...` when a container exceeds its soft memory limit.

<img width="1396" height="581" alt="Screenshot 2026-04-14 at 7 51 32 AM" src="https://github.com/user-attachments/assets/2146359e-94c3-4c78-bb09-b4095f602e64" />


### Screenshot 6 — dmesg showing hard-limit kill

> `dmesg` output showing `[container_monitor] HARD LIMIT container=... rss=... limit=...` and `ps` showing the container transitioning to `killed` state.

<img width="1396" height="581" alt="Screenshot 2026-04-14 at 7 51 32 AM" src="https://github.com/user-attachments/assets/67aeb7a7-06db-4393-b131-aa33ae33f781" />


### Screenshot 7 — Scheduling experiment output

> Terminal output from Experiment A (priority) and Experiment B (CPU vs I/O) with measurable timing differences.

<img width="1215" height="671" alt="Screenshot 2026-04-14 at 7 52 42 AM" src="https://github.com/user-attachments/assets/563d64e7-1d02-4b16-a41b-aab964000271" />

### Screenshot 8 — No zombies after clean shutdown

> `ps aux | grep Z` returning no zombie processes after all containers have been stopped and supervisor has shut down.

<img width="1255" height="102" alt="Screenshot 2026-04-14 at 7 53 24 AM" src="https://github.com/user-attachments/assets/7a60cc69-7607-40a5-8550-13af76dbc565" />


---

## Engineering Analysis

### 1. Container Isolation

Each container is launched using `clone()` with three namespace flags:

- `CLONE_NEWPID` — gives the container its own PID namespace, so the first process inside sees itself as PID 1.
- `CLONE_NEWUTS` — gives the container its own hostname, set to the container ID via `sethostname()`.
- `CLONE_NEWNS` — gives the container its own mount namespace, preventing mount/unmount operations from affecting the host.

After `clone()`, the child calls `chroot()` into the Alpine rootfs and mounts `/proc` so process utilities work correctly inside the container. This approach is simpler than `pivot_root` and sufficient for this project's isolation requirements.

### 2. Supervisor Lifecycle

The supervisor is a long-running process that:

1. Opens a UNIX domain socket at `/tmp/mini_runtime.sock` and listens for CLI connections.
2. Initializes the bounded buffer and spawns the logger (consumer) thread.
3. Opens `/dev/container_monitor` to register containers with the kernel module.
4. Enters a `select()`-based event loop, accepting one client connection at a time.
5. On `SIGINT`/`SIGTERM`, sets a `should_stop` flag, stops all running containers, joins the logger thread, frees all memory, and exits cleanly.

`SIGCHLD` is handled with `waitpid(WNOHANG)` in a loop to reap all exited children immediately, preventing zombie accumulation.

### 3. IPC and Synchronization

Two separate IPC channels are used:

- **Control plane**: UNIX domain socket (`SOCK_STREAM`) between CLI client and supervisor. Each CLI invocation connects, sends a `control_request_t` struct, reads a `control_response_t`, and disconnects.
- **Logging pipeline**: A `pipe()` per container. The container's `stdout`/`stderr` are redirected into the write end via `dup2()`. A producer thread reads from the read end and pushes chunks into the bounded circular buffer. The consumer (logger) thread pops chunks and writes them to per-container log files.

**Why mutex + condvar over semaphores:**
Semaphores only signal counts, not conditions. With a bounded buffer we need to express two distinct conditions: "buffer is not full" (producer waits) and "buffer is not empty" (consumer waits). Using `pthread_cond_t` with a single `pthread_mutex_t` makes both conditions explicit and readable. Semaphores would require two semaphores plus additional locking to protect the buffer indices, making the code more error-prone.

**Race conditions without synchronization:**
Without the mutex, a producer and consumer could simultaneously read and write `head`, `tail`, and `count`, corrupting the circular buffer state. Without `cond_wait`, both threads would busy-spin, wasting CPU. The `shutting_down` flag is set under the mutex and broadcast to both condition variables so threads wake up and drain cleanly on shutdown.

### 4. Memory Management (Kernel Module)

The kernel module (`monitor.c`) maintains a linked list of `monitored_entry` nodes, one per registered container. Each node stores the PID, container ID, soft limit, hard limit, and a `soft_warned` flag.

A kernel timer fires every second and iterates the list:

- If `get_rss_bytes()` returns -1, the process has exited and the entry is removed.
- If RSS exceeds the hard limit, `SIGKILL` is sent and the entry is removed.
- If RSS exceeds the soft limit for the first time, a `printk(KERN_WARNING)` is emitted and `soft_warned` is set to prevent repeated warnings.

**Mutex vs spinlock choice:** A `mutex` was chosen over a `spinlock` because the timer callback and ioctl handler can both sleep (memory allocation in ioctl, list iteration in timer). Spinlocks cannot be held across sleepable operations, making mutex the correct choice here.

On `rmmod`, `module_exit()` calls `del_timer_sync()` to stop the timer, then iterates the list under the mutex to free all remaining entries, preventing kernel memory leaks.

### 5. Scheduling

Linux uses the Completely Fair Scheduler (CFS), which assigns CPU time proportional to each task's weight. Weight is derived from the `nice` value: lower nice = higher weight = more CPU time.

#### Experiment A — Priority Comparison

| Container | nice value | Wall time |
|-----------|-----------|-----------|
| high_prio | 0 | 9.14s |
| low_prio | 19 | 9.99s |

The high-priority container finished ~0.85 seconds faster. CFS gave it a larger share of CPU time due to its higher weight. The difference is modest because the machine was not heavily loaded; under contention the gap would be larger.

#### Experiment B — CPU-bound vs I/O-bound

| Container | Workload | Wall time |
|-----------|----------|-----------|
| cpu_task | CPU-bound (cpu_hog) | 9.99s |
| io_task | I/O-bound (io_pulse) | 14.11s |

The I/O-bound container took longer because it repeatedly blocked waiting for disk I/O, accumulating less CPU time. CFS does not starve I/O-bound tasks — it actually gives them a small bonus when they wake up from I/O, but the inherent blocking nature of I/O work means wall time is longer. The CPU-bound task ran continuously without blocking and completed sooner.

---

## Design Decisions and Tradeoffs

| Subsystem | Decision | Tradeoff |
|-----------|----------|----------|
| Namespace isolation | `chroot` over `pivot_root` | Simpler implementation; `pivot_root` is more secure but requires more setup |
| Control IPC | UNIX domain socket | More flexible than FIFO; supports bidirectional communication per connection |
| Buffer sync | mutex + condvar | More explicit than semaphores; slightly more verbose but clearer semantics |
| Kernel lock | mutex | Safe for sleepable paths; spinlock would be faster but cannot sleep |
| Log routing | per-container log files | Simple and persistent; a single multiplexed log would be harder to query per container |
| Architecture | aarch64 Alpine rootfs | Required to match host CPU; x86-64 rootfs causes `Exec format error` on aarch64 host |
