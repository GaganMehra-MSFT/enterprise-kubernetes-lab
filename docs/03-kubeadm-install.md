# Step 7 – Install containerd

Run on all three nodes:

```bash
sudo apt install -y containerd
```

Verify installation:

```bash
containerd --version
systemctl status containerd
```

Expected:

```
Active: active (running)
```

### Why?

containerd is the Container Runtime Interface (CRI).

Its responsibilities include:

- Pulling container images
- Creating containers
- Starting and stopping containers

---

# Step 8 – Configure containerd

Generate the default configuration:

```bash
sudo mkdir -p /etc/containerd

sudo containerd config default | sudo tee /etc/containerd/config.toml
```

Edit:

```bash
sudo nano /etc/containerd/config.toml
```

Change:

```
SystemdCgroup = false
```

to

```
SystemdCgroup = true
```

Restart and enable containerd:

```bash
sudo systemctl restart containerd

sudo systemctl enable containerd
```

### Why?

Kubernetes manages resources using **systemd cgroups**.

Setting `SystemdCgroup = true` ensures containerd and kubelet use the same cgroup driver, preventing resource management issues.

---

# Step 9 – Install Required HTTPS and Package Verification Tools

Run on all three nodes:

```bash
sudo apt install -y apt-transport-https ca-certificates curl gpg
```

### Why?

| Package | Purpose |
|---------|----------|
| apt-transport-https | Allows APT to download packages over HTTPS |
| ca-certificates | Trusts public Certificate Authorities |
| curl | Downloads files from the internet |
| gpg | Verifies package signatures |

---

# Step 10 – Add the Kubernetes Package Repository

Download the repository signing key:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
| sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the Kubernetes repository:

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Verify:

```bash
cat /etc/apt/sources.list.d/kubernetes.list
```

Expected:

```
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /
```

Refresh package information:

```bash
sudo apt update
```

---

# Step 11 – Install Kubernetes Components

Run on all three nodes:

```bash
sudo apt install -y kubelet kubeadm kubectl
```

Prevent automatic upgrades:

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

Verify:

```bash
kubeadm version

kubelet --version

kubectl version --client
```

Installed version:

```
Kubernetes v1.34.9
```

### What is kubeadm?

kubeadm is a Kubernetes **bootstrapping tool**.

It creates new Kubernetes clusters using `kubeadm init` and joins worker nodes using `kubeadm join`.

After the cluster is created, kubeadm's work is complete.

The cluster is then managed by:

- kube-apiserver
- kubelet
- kube-controller-manager
- kube-scheduler
- containerd

### Analogy

| SQL Server | Kubernetes |
|------------|------------|
| SQLSetup.exe | kubeadm |
| SQL Server Service | API Server, kubelet, etc. |

---

# Step 12 – Initialize the Kubernetes Control Plane

Run only on the control-plane node:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.235.134 \
  --pod-network-cidr=192.168.0.0/16
```

### Parameters

**--apiserver-advertise-address**

IP address that worker nodes use to communicate with the Kubernetes API Server.

**--pod-network-cidr**

The IP range assigned to Pods.

This must match the CNI plugin (Calico).

---

### Issue Encountered

The first initialization failed with:

```
/proc/sys/net/ipv4/ip_forward contents are not set to 1
```

### Temporary Fix

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

### Permanent Fix

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-kubernetes.conf

sudo sysctl --system
```

After fixing IP forwarding, rerun:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.235.134 \
  --pod-network-cidr=192.168.0.0/16
```

The initialization completed successfully.

---

# Step 13 – Configure kubectl

Run:

```bash
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Why?

This configures `kubectl` to communicate securely with the Kubernetes API Server using the cluster administrator credentials.

Verify:

```bash
kubectl get nodes
```

Initially the control-plane node may show **NotReady** until the Calico CNI plugin is installed.
