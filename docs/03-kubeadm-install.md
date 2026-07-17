# 03 – kubeadm Installation

## Objective

Build a Kubernetes cluster using kubeadm.

My lab consists of:

| Node | Role |
|------|------|
| kubeadm-controlplane | Control Plane |
| worker-1 | Worker Node |
| worker-2 | Worker Node |

---

## What is kubeadm?

kubeadm is a cluster bootstrap tool.

It installs and configures the Kubernetes control plane and prepares worker nodes to join the cluster.

It is **not** used to manage applications.

---

## Components installed

Every node:

- kubeadm
- kubelet
- containerd

Control plane only:

- kubectl

---

## Cluster initialization

The control plane was initialized using:

```bash
kubeadm init \
--apiserver-advertise-address=<control-plane-ip> \
--pod-network-cidr=192.168.0.0/16
```

After initialization:

- API Server started
- etcd started
- Scheduler started
- Controller Manager started

---

## Configure kubectl

```bash
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
```

This allows kubectl to communicate with the API Server.

---

## Install Pod Network

Install Calico.

Without a CNI plugin, nodes remain **NotReady**.

---

## Join worker nodes

Run the generated join command on each worker.

```bash
kubeadm join ...
```

Workers register with the API Server.

---

## Verify

```bash
kubectl get nodes
kubectl get pods -n kube-system
kubectl cluster-info
```

---

## Architecture

```
Windows
      │
      ▼
Control Plane
├── API Server
├── Scheduler
├── Controller Manager
├── etcd
├── kubelet
└── containerd

Worker-1
├── kubelet
└── containerd

Worker-2
├── kubelet
└── containerd
```

---

## Key Learning

- kubeadm builds the cluster.
- kubelet runs on every node.
- containerd runs containers.
- kubectl communicates with the API Server.
- Workers join using `kubeadm join`.
