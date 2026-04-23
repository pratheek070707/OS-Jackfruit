
# Multi-Container Runtime (OS-Jackfruit)

## 1. Team Information

- Team Member 1: AMIT(PES2UG24CS624)
- Team Member 2: SANGAMESH (PES2UG24CS640)

## 2. Build, Load, and Run Instructions

These steps are written so the project can be reproduced on a fresh Ubuntu 22.04/24.04 VM.

### 2.1 Prerequisites

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### 2.2 Build

```bash
cd boilerplate
make
```

CI-safe compile check:

```bash
make -C boilerplate ci
```

### 2.3 Load Kernel Module

```bash
cd boilerplate
sudo insmod monitor.ko
ls -l /dev/container_monitor
```

### 2.4 Prepare Root Filesystems

```bash
cd boilerplate
mkdir -p rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```

Copy helper binaries into container rootfs before launch (if needed):

```bash
cp ./cpu_hog ./rootfs-alpha/
cp ./io_pulse ./rootfs-beta/
cp ./memory_hog ./rootfs-alpha/
```

### 2.5 Start Supervisor (Terminal 1)

```bash
cd boilerplate
sudo ./engine supervisor ./rootfs-base
```

### 2.6 Use CLI (Terminal 2)

```bash
cd boilerplate

# Start containers (background)
sudo ./engine start alpha ./rootfs-alpha "/bin/sh" --soft-mib 48 --hard-mib 80
sudo ./engine start beta ./rootfs-beta "/bin/sh" --soft-mib 64 --hard-mib 96

# Required commands
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine stop alpha
sudo ./engine stop beta

# Foreground run example
sudo ./engine run gamma ./rootfs-alpha "./cpu_hog" --nice 10
```

### 2.7 Cleanup

```bash
cd boilerplate
sudo ./engine stop alpha
sudo ./engine stop beta
dmesg | tail -n 100
sudo rmmod monitor
sudo make clean
```

## 3. Demo with Screenshots

Caption: Two containers are running under one supervisor process, proving concurrent supervision.
### 3.1 Multi-container supervision

Caption: Two containers are running under one supervisor process, proving concurrent supervision.


### 3.2 Metadata tracking (`ps` output)

Caption: `engine ps` displays container metadata and state transitions maintained by the supervisor.

### 3.3 Bounded-buffer logging

Caption: `logs/alpha.log` contains captured stdout/stderr, proving log-path producer-consumer operation.

### 3.4 CLI and IPC

Caption: CLI command is sent over control IPC and supervisor returns response.

### 3.5 Soft-limit warning

Caption: `dmesg` shows soft-limit warning when RSS crosses configured soft threshold.

### 3.6 Hard-limit enforcement

Caption: `dmesg` shows hard-limit enforcement (SIGKILL) and supervisor metadata reflects forced termination.

### 3.7 Scheduling experiment

Caption: Different scheduler configurations (`nice`) show measurable performance difference.
Each screenshot includes a short caption describing what it proves.


### 3.2 Metadata tracking (`ps` output)


Conclusion from demo: all required functional goals were demonstrated end-to-end: concurrent supervision, metadata/CLI control path, bounded logging, kernel memory policy, scheduling behavior, and cleanup correctness.

## 4. Engineering Analysis

### 4.1 Isolation mechanisms

The runtime isolates containers using PID, UTS, and mount namespaces. PID namespace hides host process numbering. UTS namespace allows per-container hostname. Mount namespace lets each container mount its own `/proc` without changing host mount view. Filesystem visibility is restricted by `chroot` into per-container rootfs. Containers still share the host kernel and scheduler.

### 4.2 Supervisor and process lifecycle

A long-running parent supervisor simplifies lifecycle control. It creates containers, tracks metadata, handles signals, and reaps child exits through `waitpid` to avoid zombies. Keeping this centralized makes state transitions (`running`, `stopped`, `killed`, `exited`) deterministic.

### 4.3 IPC, threads, and synchronization

Two IPC mechanisms are used:

- Logging path: container `stdout`/`stderr` via pipe FDs.
- Control path: CLI to supervisor via UNIX domain socket.

Bounded-buffer logging uses mutex + condition variables (`not_full`, `not_empty`) to avoid races and busy waiting. Shared container metadata has a separate lock. Without synchronization, races can corrupt metadata, lose logs, or deadlock producers/consumers.

### 4.4 Memory management and enforcement

RSS measures resident physical memory currently used by a process. It does not represent all reserved virtual memory. Soft and hard limits are intentionally different policies: soft limit is an observation warning, while hard limit is an enforcement boundary. Enforcement is done in kernel space so user workloads cannot bypass policy.

### 4.5 Scheduling behavior

Linux CFS balances fairness with priority hints (`nice`). CPU-bound tasks with lower nice values obtain larger CPU share, reducing completion time. I/O-bound tasks usually remain responsive because they sleep and wake frequently, allowing CFS to reassign CPU time efficiently.

Conclusion from engineering analysis: this project demonstrates core OS mechanisms working together in a realistic system: isolation, lifecycle management, IPC synchronization, kernel-enforced memory policy, and scheduler-driven performance tradeoffs.

## 5. Design Decisions and Tradeoffs

### 5.1 Namespace isolation

- Design choice: PID/UTS/mount namespaces with `chroot` and per-container rootfs copies.
- Tradeoff: `chroot` is simpler but less strict than `pivot_root` in hardened setups.
- Justification: lower implementation complexity with sufficient isolation for this project scope.

### 5.2 Supervisor architecture

- Design choice: one long-lived supervisor process for all containers.
- Tradeoff: a single control loop may become bottleneck under heavy command load.
- Justification: easier correctness, centralized state, and straightforward cleanup/reaping.

### 5.3 IPC and logging

- Design choice: UNIX socket for control channel + pipe with bounded buffer for logs.
- Tradeoff: more synchronization code compared to direct logging.
- Justification: clear separation of control and data paths, and safe concurrent logging.

### 5.4 Kernel monitor

- Design choice: linked-list tracking with lock protection and periodic RSS checks.
- Tradeoff: sampling interval can miss very brief spikes between checks.
- Justification: robust, explainable policy with manageable module complexity.

### 5.5 Scheduling experiments

- Design choice: compare CPU-bound and I/O-bound workloads across different nice values.
- Tradeoff: values may vary across VMs/host load.
- Justification: directly observable behavior that maps to CFS policy.

Conclusion from design tradeoffs: selected designs prioritize correctness, clarity, and demonstrable OS concepts over maximum feature depth.

## 6. Scheduler Experiment Results

### 6.1 Raw Measurements

| Experiment | Configuration A | Configuration B | Metric | Result |
| --- | --- | --- | --- | --- |
| CPU-bound (`cpu_hog`) | `nice=19` | `nice=-10` | Completion time | ~5.088s vs ~0.025s |
| Mixed (`cpu_hog` + `io_pulse`) | Same nice | Different nice | Throughput/latency | CPU share shifts while I/O remains responsive |

### 6.2 Comparison and Interpretation

- Lower nice value (`-10`) completed significantly faster than higher nice (`19`), showing explicit scheduler priority effect.
- Under mixed load, CPU-bound throughput changed with nice adjustments while I/O-bound responsiveness stayed comparatively stable.
- This indicates CFS behavior aligns with expected goals: fairness baseline with tunable priority weighting.

Final scheduler conclusion: Linux scheduling is fair by default but not uniform; priority hints strongly affect CPU-bound completion time, while I/O behavior is influenced by sleep/wake dynamics.

## 7. Source Files Included

- `boilerplate/engine.c`
- `boilerplate/monitor.c`
- `boilerplate/monitor_ioctl.h`
- `boilerplate/cpu_hog.c`
- `boilerplate/io_pulse.c`
- `boilerplate/memory_hog.c`
- `boilerplate/Makefile`

