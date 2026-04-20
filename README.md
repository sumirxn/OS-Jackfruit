# Multi-Container Runtime

## Overview

This project implements a lightweight Linux container runtime in C with support for process isolation, resource monitoring, and container lifecycle management.

Key features include:

* Namespace-based isolation (PID, UTS, mount)
* A long-running supervisor for managing containers
* A kernel module for memory monitoring and enforcement
* A CLI interface for container control

The project demonstrates core Operating Systems concepts such as process management, inter-process communication (IPC), scheduling, and kernel-user interaction.

---

## 1. Team Information

| Name                    | SRN           |
| ----------------------- | ------------- |
| **Sumiran Vuppuluri**   | PES2UG24CS536 |
| **Suraj Acharya**       | PES2UG24CS539 |

---

## 2. Build, Load, and Run Instructions

### Prerequisites

* Ubuntu 22.04 / 24.04
* Secure Boot **disabled**

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

---

### Clone and Build

```bash
git clone https://github.com/YOUR-USERNAME/OS-Jackfruit.git
cd OS-Jackfruit/boilerplate
sudo make clean
make
```

> Replace `YOUR-USERNAME` with your GitHub username.

---

### Environment Check

```bash
chmod +x environment-check.sh
sudo ./environment-check.sh
```

---

### Root Filesystem Setup

```bash
cd ~/OS-Jackfruit
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
```

#### Create container filesystems

```bash
cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta
```

#### Copy workload binaries

```bash
cp boilerplate/cpu_hog rootfs-alpha/
cp boilerplate/cpu_hog rootfs-beta/
cp boilerplate/memory_hog rootfs-alpha/
cp boilerplate/io_pulse rootfs-beta/
```

---

### Load Kernel Module

```bash
cd boilerplate
sudo insmod monitor.ko
ls -l /dev/container_monitor
```

> Ensure that `/dev/container_monitor` is created successfully.

---

### Start Supervisor (Terminal 1)

```bash
mkdir -p logs
sudo ./engine supervisor ../rootfs-base
```

Wait until the supervisor prints **"Ready"**.

---

### Launch Containers (Terminal 2)

```bash
sudo ./engine start alpha ../rootfs-alpha /cpu_hog
sudo ./engine start beta ../rootfs-beta /cpu_hog
```

#### List running containers

```bash
sudo ./engine ps
```

#### View logs

```bash
sudo ./engine logs alpha
```

#### Stop containers

```bash
sudo ./engine stop alpha
sudo ./engine stop beta
```

---

### Memory Limit Test

```bash
sudo ./engine start mem1 ../rootfs-alpha /memory_hog --soft-mib 30 --hard-mib 50
sleep 5
sudo dmesg | grep container_monitor
sudo ./engine ps
```

**Expected Result:** Container `mem1` should be in the **killed** state.

---

### Scheduling Experiment

```bash
cp -a rootfs-base rootfs-low
cp -a rootfs-base rootfs-high

cp boilerplate/cpu_hog rootfs-low/
cp boilerplate/cpu_hog rootfs-high/

sudo ./engine start low ../rootfs-low /cpu_hog --nice 0
sudo ./engine start high ../rootfs-high /cpu_hog --nice 15
```

#### View logs

```bash
sudo ./engine logs low
sudo ./engine logs high
```

---

### Cleanup

```bash
sudo ./engine stop alpha
sudo ./engine stop beta
sudo rmmod monitor
```

---

## 3. Screenshots

### 1 — Multi-Container Supervision
<img width="906" height="944" alt="image" src="https://github.com/user-attachments/assets/cbf35121-5ff3-4f2a-ac55-b33962dbeec5" /> <br>
<img width="906" height="944" alt="image" src="https://github.com/user-attachments/assets/686ab2ff-b78d-4070-8f1c-aedb97a67576" /> <br>
<img width="634" height="246" alt="image" src="https://github.com/user-attachments/assets/437774e4-cd51-4069-a799-829870f8a926" /> <br>
<img width="1110" height="127" alt="image" src="https://github.com/user-attachments/assets/7a9f3b73-806c-42bd-b5bf-443030224b95" /> <br>
Caption: Two containers running simultaneously under a single supervisor.

---

### 2 — Metadata Tracking
<img width="1157" height="769" alt="image" src="https://github.com/user-attachments/assets/c092e92d-5407-417c-9893-024bf309eb4d" />
<br>
Caption: Container metadata tracking and kernel memory monitoring output

---

### 3 — Bounded-Buffer Logging
<img width="675" height="264" alt="image" src="https://github.com/user-attachments/assets/c811ef16-2c32-4f2d-acd1-5a379e6f2e1d" />
<br>
Caption: Output of `./engine logs alpha` showing container stdout captured by the logging system

---

### 4 — CLI and IPC
<img width="670" height="99" alt="image" src="https://github.com/user-attachments/assets/cf992f5d-6432-44eb-ab01-4652b7863215" />
<br>
Caption: Stopping containers using the CLI and updating their state in the supervisor

---

### 5 — Soft-Limit Warning
<img width="1129" height="269" alt="image" src="https://github.com/user-attachments/assets/7e35b1a9-ad93-4d8e-a623-630b72f97039" />
<br>
Caption: Kernel log output showing soft memory limit warning for a container

---

### 6 — Hard-Limit Enforcement
<img width="1129" height="269" alt="image" src="https://github.com/user-attachments/assets/e39eb6f1-607a-4fb7-8ffd-1ea430066889" />
<br>
Caption: Hard memory limit enforcement with process termination and corresponding state update in ps output

---

### 7 — Scheduling Experiment
<img width="509" height="808" alt="image" src="https://github.com/user-attachments/assets/c8c59c07-f021-41ae-8d32-296ed8a8a074" />
<br>
Caption: Logs from two CPU-bound containers showing the impact of scheduling priority

---

### 8 — Clean Teardown
<img width="1517" height="218" alt="Screenshot from 2026-04-20 20-14-20" src="https://github.com/user-attachments/assets/c874b826-78bf-47be-a0b2-38b132c6abda" />
<br>
Caption: Cleanup sequence showing supervisor termination and kernel module unloading with container unregister events

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

Containers are isolated using:

* **PID namespace** → separate process ID space
* **UTS namespace** → independent hostname
* **Mount namespace** → isolated filesystem view
* **chroot** → restricted root directory

**Limitations:**

* No network namespace
* No user namespace
* Some resources remain shared with the host

---

### 4.2 Supervisor and Process Lifecycle

* Acts as the parent process for all containers
* Uses `waitpid()` to prevent zombie processes
* Maintains container metadata
* Differentiates between normal exit, user stop, and forced termination

---

### 4.3 IPC, Threads, and Logging

* Pipes capture container stdout/stderr
* Bounded buffer prevents uncontrolled memory growth
* Producer-consumer threads handle logging
* UNIX domain socket enables CLI communication

---

### 4.4 Memory Management and Enforcement

* Memory usage tracked using **RSS**
* Soft limit → warning
* Hard limit → enforced using `SIGKILL`

Kernel module ensures:

* Accurate monitoring
* Safe enforcement without race conditions

---

### 4.5 Scheduling Behavior

* Based on Linux **Completely Fair Scheduler (CFS)**
* Lower nice value → higher priority
* CPU-bound vs I/O-bound behavior observed

---

## 5. Design Decisions and Tradeoffs

* Simple namespace-based isolation for reduced complexity
* Supervisor-based architecture for centralized control
* UNIX sockets for reliable IPC
* Kernel module for precise memory enforcement

**Tradeoff:**
Simplicity is prioritized over full isolation and security.

---

## 6. Results

| Container | Nice Value | Observation      |
| --------- | ---------- | ---------------- |
| low       | 0          | Higher CPU usage |
| high      | 15         | Lower CPU usage  |

### Conclusion

* CFS provides **proportional fairness**
* Higher-priority processes receive more CPU time
* Lower-priority processes are not starved

---
