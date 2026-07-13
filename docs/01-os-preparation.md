# Documentation

## Phase 1

- 01 OS Preparation

  # 01 - Operating System Preparation

  ## Objective

  Prepare Ubuntu Linux machines so they are ready for Kubernetes installation using kubeadm.
    ---
  ## Lab Machines

| Hostname | Role | IP Address |
|----------|------|------------|
| control-plane | Kubernetes Control Plane | 192.168.xxx.xxx |
| worker-1 | Kubernetes Worker | 192.168.xxx.xxx |
| worker-2 | Kubernetes Worker | 192.168.xxx.xxx |

---

  ## Step 1 - Update the Operating System

  ### Why?
    Before installing Kubernetes, ensure all packages are up to date.
  ### Commands
    sudo apt update
    sudo apt upgrade -y
  ## Step 2 - Disable Swap
  ### Why?
    Kubernetes expects memory management to be predictable. Swap can interfere with scheduling decisions, so kubeadm requires swap to be disabled.
  ### Check swap

```bash
free -h
```

### Disable swap

```bash
sudo swapoff -a
```

### Make it permanent

Edit:

```bash
sudo nano /etc/fstab
```
Comment the swap entry.
Example:

```text
#/swap.img none swap sw 0 0
```

### Verify

```bash
free -h
```

Expected:

```text
Swap: 0B
```

---

## Step 3 - Configure Hostname Resolution

### Why?

Allow the Kubernetes nodes to communicate using hostnames instead of IP addresses.

Edit:

```bash
sudo nano /etc/hosts
```

Contents:

```text
192.168.xxx.xxx control-plane
192.168.xxx.xxx worker-1
192.168.xxx.xxx worker-2
```

- 02 Container Runtime
- 03 kubeadm Installation

## Phase 2

- 04 Control Plane
- 05 Worker Nodes

## Phase 3

- 06 Networking
- 07 Storage

## Phase 4

- 08 Troubleshooting

## Reference

- Architecture
