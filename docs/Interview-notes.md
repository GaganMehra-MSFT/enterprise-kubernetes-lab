Q1: Why does every node need containerd or a container runtime?

Ans: Because every node that runs Pods requires a container runtime to pull images and start containers. kubelet manages Pods but delegates container creation to the container runtime.

Q2 : How a Pod is born ?

Answer: Suppose you run:

**kubectl apply -f postgres.yaml**

Here's what happens:

1. You submit postgres.yaml
            │
            ▼
2. API Server stores it in etcd
            │
            ▼
3. Scheduler chooses Worker-1
            │
            ▼
4. kubelet on Worker-1 says:
   "I need a PostgreSQL Pod."
            │
            ▼
5. containerd starts creating the container
            │
            ▼
6. Linux provides:
      ✔ Namespace   (Isolation)
      ✔ cgroups     (CPU/RAM limits)
      ✔ OverlayFS   (Shared image)
            │
            ▼
8. PostgreSQL starts running

# Interview Questions

### Q1. What is kubeadm?

kubeadm is a Kubernetes bootstrapping tool used to initialize a control plane and join worker nodes to a cluster.

---

### Q2. Is kubeadm a Kubernetes component?

No.

It is an installation tool.

Once the cluster has been created, Kubernetes is managed by components such as the API Server, kubelet and containerd.

---

### Q3. What does kubeadm init do?

It creates the Kubernetes control plane by configuring certificates, etcd, the API Server, Scheduler and Controller Manager.

---

### Q4. Why is admin.conf important?

It contains the credentials and cluster information required for kubectl to securely communicate with the Kubernetes API Server.

---
