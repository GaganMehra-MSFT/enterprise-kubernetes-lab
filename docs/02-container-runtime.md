# 02 - Container Runtime

## Objective

Install and configure the container runtime required by Kubernetes to create and manage containers.

---

# What is a Container Runtime?

A container runtime is software responsible for downloading container images, creating containers, starting them, stopping them and managing their lifecycle.

In this lab, we are using **containerd** as the container runtime.

---

# Why is containerd needed?

Kubernetes is an orchestration platform.

It decides:

- What application should run
- How many instances should run
- Which worker node should run them

However, Kubernetes **does not know how to create a container**.

Instead, Kubernetes asks the container runtime (containerd) to:

- Pull the image
- Create the container
- Start the container
- Stop the container

---

# Why is containerd installed on every node?

Every Kubernetes node that runs Pods requires a container runtime.

This includes:

- Control Plane
- Worker-1
- Worker-2

The control plane also runs Kubernetes system Pods such as:

- kube-apiserver
- etcd
- kube-scheduler
- kube-controller-manager

These Pods also require containerd.

---

# Installation

## Install containerd

```bash
sudo apt update
sudo apt install -y containerd
```

---

## Verify installation

```bash
containerd --version
systemctl status containerd
```

Expected:

```
Active: active (running)
```

---

## Generate the default configuration

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

### Why?

Generate the default configuration so it can be customized for Kubernetes.

---

## Configure Systemd cgroups

Edit:

```bash
sudo nano /etc/containerd/config.toml
```

Find:

```toml
SystemdCgroup = false
```

Change to:

```toml
SystemdCgroup = true
```

### Why?

Allows containerd and kubelet to use the same Linux cgroup manager for CPU and memory management.

---

## Restart containerd

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

# Verification

```bash
systemctl status containerd
```

Expected:

```
Active: active (running)
```

---

# Kubernetes Pod Creation Flow

```
kubectl apply -f postgres.yaml
            │
            ▼
API Server
            │
            ▼
etcd stores desired state
            │
            ▼
Scheduler selects Worker Node
            │
            ▼
kubelet receives instruction
            │
            ▼
containerd creates container
            │
            ▼
Linux Kernel provides:
    • Namespace
    • cgroups
    • OverlayFS
            │
            ▼
PostgreSQL starts
```

---

# Interview Questions

### Q1. What is containerd?

Containerd is the container runtime responsible for pulling images, creating containers and managing their lifecycle.

---

### Q2. Why is containerd installed on every node?

Because every node that runs Pods requires a container runtime. This includes both worker nodes and the control plane.

---

### Q3. Who creates the containers?

✅ containerd

---

### Q4. If containerd crashes, what happens?

- Existing containers generally continue running.
- New Pods cannot start.
- Crashed Pods cannot be restarted until containerd is available again.

---

# Key Takeaways

- Kubernetes is an orchestrator.
- containerd creates and manages containers.
- Linux provides Namespaces, cgroups and OverlayFS.
- Every node that runs Pods requires a container runtime.
