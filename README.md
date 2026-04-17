# OS-Jackfruit — Lightweight Linux Container Runtime

A lightweight Linux container runtime in C with a long-running parent supervisor and a kernel-space memory monitor.
---
## Team Information
| Name | SRN |
|------|-----|
| G Ganikhasri| PES1UG24AM098 |
| Disha Lokesh | PES1UG24AM093 |
---
## Build, Load, and Run Instructions
### 1. Install Dependencies
```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```
### 2. Build Everything
```bash
make
```
### 3. Prepare Root Filesystems
```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```
### 4. Load the Kernel Module
```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor
```
### 5. Run the Supervisor and Containers
```bash
# Terminal 1 — start the supervisor
sudo ./engine supervisor ./rootfs-base

# Terminal 2 — launch containers
sudo ./engine start alpha ./rootfs-alpha /bin/sh --soft-mib 48 --hard-mib 80
sudo ./engine start beta  ./rootfs-beta  /bin/sh --soft-mib 64 --hard-mib 96

# List tracked containers
sudo ./engine ps
# Inspect a container log
sudo ./engine logs alpha
# Stop a container
sudo ./engine stop alpha
# Run and wait for exit
sudo ./engine run alpha ./rootfs-alpha /bin/echo hello
```
### 6. Run Workloads Inside a Container
```bash
cp memory_hog ./rootfs-alpha/
cp cpu_hog    ./rootfs-alpha/
sudo ./engine start alpha ./rootfs-alpha /memory_hog
```
### 7. Inspect Kernel Logs
```bash
dmesg | tail
```
### 8. Unload and Clean Up
```bash
sudo ./engine stop alpha
sudo ./engine stop beta
# Ctrl+C the supervisor
sudo rmmod monitor
sudo rm -rf /tmp/engine-logs/
sudo rm -f /tmp/engine.sock
```
### CI-Safe Build
```bash
make -C boilerplate ci
```
---
## Demo Screenshots

### 1 — Multi-container supervision
**Caption:** Two containers (alpha and beta) running concurrently under one supervisor process.
<img width="940" height="379" alt="image" src="https://github.com/user-attachments/assets/831988ab-91a7-4a5f-af47-23cbadc180df" /> <br>

### 2 — Metadata tracking
**Caption:** Output of `engine ps` showing container ID, host PID, start time, state, soft/hard memory limits, and log path.
<img width="940" height="678" alt="image" src="https://github.com/user-attachments/assets/6b72e625-dd4f-4d49-b58d-d711356967bc" />

### 3 — Bounded-buffer logging
**Caption:** Supervisor terminal showing `[producer]` and `[consumer]` thread activity; `engine logs` output showing captured log contents.
<img width="940" height="464" alt="image" src="https://github.com/user-attachments/assets/8a18f8c9-d8ba-4605-b0e9-dd3e7e603995" />

### 4 — CLI and IPC
**Caption:** CLI terminal issuing `engine start` with flags; supervisor terminal showing `[supervisor] received command: start` and responding with `OK:` over the UNIX domain socket.<br>
<img width="487" height="75" alt="image" src="https://github.com/user-attachments/assets/3ec72e50-75df-4afc-bd4e-ef10eba1e835" />
<img width="1408" height="179" alt="image" src="https://github.com/user-attachments/assets/2f8cfc26-54b3-4ce9-a679-5d6b7a740c3b" />

### 5 — Soft-limit warning
**Caption:** `dmesg` output showing a soft-limit warning event for a container exceeding its soft memory limit.
<img width="987" height="155" alt="image" src="https://github.com/user-attachments/assets/b8c1e3ea-1d63-4555-9b8d-55a39aefda3a" />

### 6 — Hard-limit enforcement
**Caption:** `dmesg` output showing a container killed after exceeding its hard limit; `engine ps` showing state as `hard_limit_killed`.
<img width="494" height="95" alt="image" src="https://github.com/user-attachments/assets/72143014-4d28-4430-a0c3-42078110a33f" />

### 7 — Scheduling experiment
**Caption:** Terminal output showing measurable differences between two scheduling configurations.
<img width="935" height="258" alt="image" src="https://github.com/user-attachments/assets/f33d5c8a-4903-4af4-9042-c0922557436b" />


### 8 — Clean teardown
**Caption:** Supervisor printing `[supervisor] shutting down... done.`; `ps aux` showing no zombie processes.
<img width="1600" height="332" alt="image" src="https://github.com/user-attachments/assets/79adebab-e1e8-475e-b2f0-2c12f933ac9a" />


## Engineering Analysis

### 1. Isolation Mechanisms

The runtime achieves isolation using Linux namespaces and filesystem isolation.

Each container is created with:

* **PID namespace** → processes inside the container see their own PID space
* **UTS namespace** → each container can have its own hostname
* **Mount namespace** → isolated filesystem view

Additionally, `chroot()` is used to restrict the container to its own root filesystem (`rootfs-alpha`, `rootfs-beta`). This ensures that processes inside the container cannot access files outside their assigned directory.

Inside each container, `/proc` is mounted so that commands like `ps` work correctly.

However, all containers share the **same host kernel**, meaning:

* System calls are handled by the host
* CPU scheduling is shared
* Physical memory is shared

Thus, this implementation provides **process-level isolation, not full virtualization**.

---

### 2. Supervisor and Process Lifecycle

A long-running supervisor process is used to manage all containers.

From the outputs:

* The supervisor starts once:

  ```
  Supervisor started...
  ```
* It receives commands:

  ```
  Received command: start alpha ...
  ```
* It launches containers and tracks them:

  ```
  Container alpha started with PID 3454
  ```

The supervisor:

* Creates containers using `fork()/clone()`
* Maintains metadata (ID, PID, state)
* Tracks lifecycle states (`RUNNING`, `EXITED`)
* Handles multiple containers concurrently

Example from output:

```
ID: alpha | PID: 3454 | STATE: EXITED
ID: beta  | PID: 3474 | STATE: RUNNING
```

When a container finishes (e.g., `sleep 3`), the supervisor:

* Detects exit using `SIGCHLD`
* Reaps the process using `wait()`
* Updates metadata

This prevents zombie processes and ensures proper lifecycle management.

---

### 3. IPC, Threads, and Synchronization

The project uses two IPC paths:

#### Control Path (CLI → Supervisor)

Commands like:

```
./engine start alpha ...
./engine ps
./engine stop alpha
```

are sent to the supervisor, which processes them and responds.

#### Logging Path (Container → Supervisor)

Container output is captured using **pipes** and stored in log files.

Example:

```
cat logs/alpha.log
hello_log
hello_world
```

This shows that:

* Container stdout is redirected
* Logs are persisted per container

#### Synchronization

Potential issues:

* Multiple containers writing logs simultaneously
* Concurrent access to shared metadata
* Race conditions during insert/remove

Solution:

* **Mutex locks** protect shared structures
* Producer-consumer model ensures safe logging
* Safe list iteration avoids crashes during deletion

This ensures:

* No data corruption
* No lost logs
* No race conditions

---

### 4. Memory Management and Enforcement

Memory usage is tracked using **RSS (Resident Set Size)**.

RSS measures:

* Physical memory currently used by a process

RSS does NOT include:

* Swapped-out memory
* Non-resident pages

From your outputs:

```
Soft limit warning for container alpha
Hard limit exceeded. Killing container beta
```

This shows:

#### Soft Limit

* Only generates a warning
* Triggered once per container
* Helps detect excessive usage early

#### Hard Limit

* Strict enforcement
* Process is killed using `SIGKILL`

Why kernel space?

* Direct access to process memory (`task_struct`, `mm_struct`)
* Faster and more reliable than user-space polling
* Immediate enforcement without delay

This ensures **accurate and real-time memory control**.

---

### 5. Scheduling Behavior

Scheduling behavior was observed using:

* Multiple containers (`alpha`, `beta`)
* Workloads like `busybox` and `sleep`
* Monitoring using `top`

Example output:

```
PID   USER   %CPU   COMMAND
1511  gani   86.7   gnome-shell
4683  root   66.7   busybox
4707  root   53.3   busybox
```

Observations:

* CPU-bound processes (busybox) consume high CPU
* Multiple processes share CPU based on scheduler decisions
* System load increases when multiple containers run

Linux scheduler balances:

* **Fairness** → distributes CPU among processes
* **Responsiveness** → interactive processes remain smooth
* **Throughput** → maximize total work

---

## 6. Scheduler Experiment Results

### Experiment Setup

Two containers were started simultaneously:

```
sudo ./engine start alpha ./rootfs-alpha /bin/busybox yes
sudo ./engine start beta  ./rootfs-beta  /bin/busybox yes
```

System performance was monitored using:

```
top
```

---

### Observed Results

| Container | Process | CPU Usage (%) | Behavior         |
| --------- | ------- | ------------- | ---------------- |
| Alpha     | busybox | ~60–70%       | CPU intensive    |
| Beta      | busybox | ~50–60%       | Competes for CPU |

System stats:

```
load average: ~1.4
Tasks: multiple running
```

---

### Analysis

* Both containers compete for CPU resources
* CPU is shared dynamically between processes
* No single container gets 100% CPU
* Scheduler distributes CPU time across active processes

Key conclusions:

* Linux scheduler ensures **fair CPU distribution**
* CPU-bound workloads dominate CPU usage
* Multiple containers increase system load
* Scheduler dynamically adjusts based on demand

This demonstrates how Linux scheduling achieves:

* Fairness between processes
* Efficient CPU utilization
* Balanced system performance

---
