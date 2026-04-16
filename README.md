
# Multi-Container Runtime

## 1. Team Information
- Riyana K  & Rhea Menon
- SRN: PES1UG24AM226 & PES1UG24AM222

---

## 2. Build, Load, and Run Instructions

### Environment
- Ubuntu 22.04 / 24.04 (UTM VM)
- Secure Boot OFF

Install dependencies:
```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

---

### Step 1 — Build
```bash
make
```

---

### Step 2 — Prepare Root Filesystem (ARM)
```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/aarch64/alpine-minirootfs-3.20.3-aarch64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-aarch64.tar.gz -C rootfs-base
```

Create container copies:
```bash
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```

---

### Step 3 — Load Kernel Module
```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor
```

---

### Step 4 — Start Supervisor (Terminal 1)
```bash
sudo ./engine supervisor ./rootfs-base
```

---

### Step 5 — CLI Commands (Terminal 2)
```bash
sudo ./engine start s1 ./rootfs-alpha "sleep 100"
sudo ./engine start s2 ./rootfs-beta "sleep 100"
sudo ./engine ps
sudo ./engine logs s1
sudo ./engine stop s1
sudo ./engine stop s2
```

---

### Step 6 — Memory Experiments
Soft limit:
```bash
sudo ./engine start soft1 ./rootfs-alpha "/memory_hog" --soft-mib 10 --hard-mib 200
sudo dmesg | tail -20
```

Hard limit:
```bash
sudo ./engine start hard1 ./rootfs-beta "/memory_hog" --soft-mib 10 --hard-mib 20
sudo dmesg | tail -20
```

---

### Step 7 — Scheduling Experiments

CPU vs CPU:
```bash
sudo ./engine start cpu0 ./rootfs-alpha "/cpu_hog" --nice 0
sudo ./engine start cpu10 ./rootfs-beta "/cpu_hog" --nice 10
```

CPU vs IO:
```bash
sudo ./engine start cpuA ./rootfs-alpha "/cpu_hog"
sudo ./engine start ioA ./rootfs-beta "/io_pulse"
```

---

### Step 8 — Cleanup
```bash
sudo ./engine stop s1
sudo ./engine stop s2
ps -ef | grep Z
sudo rmmod monitor
```

---

## 3. Demo with Screenshots

### 1. Multi-container supervision
![1](screenshots/1.png)  
Two containers running under a single supervisor.

### 2. Metadata tracking
![2](screenshots/2.png)  
`engine ps` output showing container metadata.

### 3. Bounded-buffer logging
![3](screenshots/3.png)  
Logging pipeline capturing container output.

### 4. CLI and IPC
![4](screenshots/4.png)  
CLI interacting with supervisor via socket IPC.

### 5. Soft-limit warning
![5](screenshots/5.png)  
Kernel soft memory warning event.

### 6. Hard-limit enforcement
![6](screenshots/6.png)  
Container killed after exceeding memory limit.

### 7. Scheduling experiment (CPU vs CPU)
![7](screenshots/7.png)  
Different nice values affecting CPU scheduling.

### 8. Scheduling experiment (CPU vs IO)
![8](screenshots/8.png)  
CPU-bound vs IO-bound workload behavior.

### 9. Clean teardown
![9](screenshots/9.png)  
Containers cleaned up, no zombies remain.

---

## 4. Engineering Analysis

### Isolation Mechanisms
Containers use PID, UTS, and mount namespaces along with chroot to isolate processes and filesystems. All containers share the same kernel but operate in separate execution environments.

### Supervisor and Process Lifecycle
A persistent supervisor process manages all containers. It spawns children using clone(), tracks metadata, and reaps processes using waitpid() to prevent zombies.

### IPC, Threads, and Synchronization
Two IPC mechanisms are used:
- Pipes for logging (container → supervisor)
- UNIX domain socket for CLI (client → supervisor)

A bounded buffer with mutex and condition variables ensures safe concurrent logging.

### Memory Management and Enforcement
RSS measures actual physical memory usage. Soft limits generate warnings while hard limits enforce termination. Enforcement is done in kernel space for reliability.

### Scheduling Behavior
Experiments show that:
- Lower nice values receive more CPU time
- IO-bound processes yield CPU, improving responsiveness
- Linux scheduler balances fairness and throughput

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation
- Choice: chroot + namespaces  
- Tradeoff: less secure than pivot_root  
- Justification: simpler and sufficient for project scope  

### Supervisor Architecture
- Choice: single supervisor  
- Tradeoff: central bottleneck  
- Justification: easier lifecycle management  

### IPC and Logging
- Choice: pipes + bounded buffer  
- Tradeoff: complexity  
- Justification: prevents data loss  

### Kernel Monitor
- Choice: timer-based RSS monitoring  
- Tradeoff: not real-time precise  
- Justification: efficient and simple  

### Scheduling Experiments
- Choice: nice values  
- Tradeoff: limited control  
- Justification: demonstrates real scheduler behavior  

---

## 6. Scheduler Experiment Results

| Experiment | Observation |
|-----------|------------|
| CPU vs CPU | Lower nice value dominates CPU |
| CPU vs IO  | IO workload remains responsive |

Conclusion:
Linux scheduler prioritizes fairness and responsiveness, dynamically allocating CPU time.

---

## Notes
- Each container uses a unique writable rootfs
- Logging ensures no data loss
- Kernel module enforces memory constraints
