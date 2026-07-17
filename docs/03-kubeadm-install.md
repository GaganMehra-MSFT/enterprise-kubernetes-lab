# 03 - Kubernetes Installation using kubeadm

## Objective

Install the Kubernetes components, initialize the control plane and prepare the cluster for worker nodes to join.

---

# Prerequisites

Complete the following labs before continuing:

- 01 - Operating System Preparation
- 02 - Container Runtime

These steps ensure:

- Ubuntu is prepared for Kubernetes.
- Required Linux kernel modules are loaded.
- Kubernetes networking settings are configured.
- containerd is installed and configured.

---

# Enable Linux Kernel Features

Run on all three nodes.

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Persist the modules after reboot.

```bash
sudo nano /etc/modules-load.d/k8s.conf
```

Add:

```text
overlay
br_netfilter
```

### Why?

- **overlay** enables OverlayFS used by container image layers.
- **br_netfilter** allows bridged network traffic to be processed by Linux networking rules, which Kubernetes networking plugins such as Calico require.

---

# Cube Notes

## What is modprobe?

Think of the Linux kernel as Windows.

Windows has drivers such as:

- NTFS driver
- USB driver
- Printer driver
- Wi-Fi driver

Linux also has drivers, but they are called **kernel modules**.

Running:

```bash
sudo modprobe overlay
```

tells Linux:

> Load the OverlayFS kernel module into memory.

Running:

```bash
sudo modprobe br_netfilter
```

tells Linux:

> Load the bridge packet filtering module into memory.

Nothing changes visually. These commands simply enable Linux capabilities required by Kubernetes.

---

# Configure Kubernetes Networking Settings

Create:

```bash
sudo nano /etc/sysctl.d/k8s.conf
```

Add:

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

Apply the configuration.

```bash
sudo sysctl --system
```

Verify.

```bash
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

### Why?

| Setting | Purpose |
|----------|---------|
| net.bridge.bridge-nf-call-iptables | Allows Kubernetes firewall rules to inspect bridged traffic |
| net.bridge.bridge-nf-call-ip6tables | Same for IPv6 traffic |
| net.ipv4.ip_forward | Allows Linux to forward packets between Pods on different nodes |

---

# Install Required HTTPS and Package Verification Tools

Run on all three nodes.

```bash
sudo apt install -y apt-transport-https ca-certificates curl gpg
```

### Why?

| Package | Purpose |
|----------|---------|
| apt-transport-https | Allows APT to download packages over HTTPS |
| ca-certificates | Trusts public Certificate Authorities |
| curl | Downloads repository files |
| gpg | Verifies repository signatures |

---

# Add the Kubernetes Repository

Download the Kubernetes signing key.

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
| sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the Kubernetes repository.

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Verify.

```bash
cat /etc/apt/sources.list.d/kubernetes.list
```

Expected:

```text
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /
```

Refresh package information.

```bash
sudo apt update
```

---

# Install Kubernetes Components

Run on all three nodes.

```bash
sudo apt install -y kubelet kubeadm kubectl
```

Prevent automatic upgrades.

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

---

# Verify Installation

```bash
kubeadm version
kubelet --version
kubectl version --client
```

Installed version:

```text
Kubernetes v1.34.9
```

---

# What is kubeadm?

kubeadm is a Kubernetes bootstrapping tool used to create and join Kubernetes clusters.

It is **not** a long-running Kubernetes component.

Its responsibilities are:

- Initialize the control plane using **kubeadm init**
- Join worker nodes using **kubeadm join**

Once the cluster is built, kubeadm's job is complete.

The cluster is then maintained by Kubernetes components such as:

- kube-apiserver
- kube-scheduler
- kube-controller-manager
- kubelet
- containerd

---

# Analogy

| SQL Server | Kubernetes |
|------------|------------|
| SQLSetup.exe | kubeadm |
| SQL Server Service | API Server, kubelet, etc. |

---

# Initialize the Control Plane

Run only on **kubeadm-controlplane**.

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.235.134 \
  --pod-network-cidr=192.168.0.0/16
```

### What do these parameters mean?

**--apiserver-advertise-address**

Specifies the IP address that worker nodes will use to communicate with the Kubernetes API Server.

**--pod-network-cidr**

Specifies the IP address range that will be assigned to Pods.

This CIDR must match the networking plugin (Calico) that will be installed later.

---

# Troubleshooting

The first initialization failed with:

```text
/proc/sys/net/ipv4/ip_forward contents are not set to 1
```

Temporary fix:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Permanent fix:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-kubernetes.conf

sudo sysctl --system
```

Rerun the same `kubeadm init` command after applying the fix.

---

# Configure kubectl

Run:

```bash
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

# Cube Notes

When `kubeadm init` completes, it creates several Kubernetes configuration files.

One of the most important is:

```text
/etc/kubernetes/admin.conf
```

Think of this file as your **Administrator Identity Card** for the Kubernetes cluster.

It contains:

- Cluster endpoint (API Server)
- Client certificate
- Client private key
- Certificate Authority (CA)
- Cluster name

Without this file, `kubectl` would not know:

- Which cluster to connect to
- How to authenticate
- Whether the API Server can be trusted

Copying this file into your home directory allows `kubectl` to securely communicate with the Kubernetes API Server.

---

# Verification

Verify the control plane.

```bash
kubectl get nodes
```

Initially, the control-plane node will show:

```text
NotReady
```

This is expected because the networking plugin has not yet been installed.

The node will become **Ready** after installing Calico in the next lab.

---

# Key Takeaways

- kubeadm is used to build a Kubernetes cluster.
- kubeadm init creates the control plane.
- kubeadm join adds worker nodes.
- admin.conf allows kubectl to communicate securely with the cluster.
- A CNI plugin such as Calico is required before the cluster becomes fully operational.
