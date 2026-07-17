# 04 - Kubernetes Control Plane

## Objective

Understand the Kubernetes Control Plane, the components it contains and how they work together to manage the entire Kubernetes cluster.

---

# What is the Kubernetes Control Plane?

The Control Plane is the **brain** of a Kubernetes cluster.

It receives every request, stores the desired state of the cluster, decides where applications should run and continuously ensures the cluster matches the desired state.

Without the Control Plane, Kubernetes cannot manage workloads.

---

# Responsibilities of the Control Plane

The Control Plane is responsible for:

- Receiving requests from users and applications
- Storing the cluster configuration
- Scheduling Pods onto worker nodes
- Monitoring the health of the cluster
- Maintaining the desired state of the cluster

---

# Control Plane Components

After running `kubeadm init`, Kubernetes creates the following Control Plane components.

```
kube-apiserver
etcd
kube-scheduler
kube-controller-manager
```

Each component runs as a **Static Pod** on the Control Plane node.

---

# kube-apiserver

The API Server is the **front door** of Kubernetes.

Every request to the cluster goes through the API Server.

Examples include:

- kubectl commands
- Dashboard
- CI/CD pipelines
- Kubernetes Controllers

The API Server validates every request before updating the cluster.

---

## Responsibilities

- Accepts all Kubernetes API requests
- Authenticates and authorizes users
- Validates requests
- Stores cluster state in etcd
- Returns cluster information

---

# etcd

etcd is a distributed key-value database.

It is the **single source of truth** for the Kubernetes cluster.

Everything about the cluster is stored inside etcd.

Examples include:

- Nodes
- Pods
- Deployments
- Services
- Secrets
- ConfigMaps
- Namespaces

If etcd is lost, the cluster loses its configuration.

---

# kube-scheduler

The Scheduler decides **where new Pods should run**.

It watches for Pods that do not yet have a node assigned.

When a new Pod is created, the Scheduler evaluates every worker node and selects the most appropriate one.

---

## Scheduler Decision Factors

The Scheduler considers:

- Available CPU
- Available memory
- Resource requests and limits
- Node labels
- Node affinity
- Taints and tolerations
- Pod affinity and anti-affinity

The Scheduler **does not run containers**.

It only assigns Pods to nodes.

---

# kube-controller-manager

The Controller Manager continuously compares the cluster's **Desired State** with the **Actual State**.

If they differ, it takes corrective action.

Example:

Desired State:

```
3 Pods
```

Actual State:

```
2 Pods
```

Controller Manager automatically creates one additional Pod.

---

## Common Controllers

Examples include:

- Deployment Controller
- ReplicaSet Controller
- Node Controller
- Job Controller
- Service Account Controller

Each controller is responsible for a specific Kubernetes resource.

---

# cloud-controller-manager

The Cloud Controller Manager integrates Kubernetes with cloud providers.

Examples include:

- Azure Kubernetes Service (AKS)
- Amazon EKS
- Google Kubernetes Engine (GKE)

It manages cloud resources such as:

- Load Balancers
- Cloud Routes
- Persistent Volumes

Since this lab is running on-premises, the Cloud Controller Manager is **not used**.

---

# Static Pods

The Control Plane components created by kubeadm are **Static Pods**.

Their manifest files are stored here:

```text
/etc/kubernetes/manifests
```

Example:

```text
kube-apiserver.yaml
etcd.yaml
kube-scheduler.yaml
kube-controller-manager.yaml
```

The kubelet continuously watches this directory.

If a Static Pod stops or its manifest changes, kubelet automatically recreates it.

---

# Control Plane Architecture

```
                kubectl
                    │
                    ▼
            kube-apiserver
                    │
      ┌─────────────┼─────────────┐
      ▼             ▼             ▼
    etcd      kube-scheduler   kube-controller-manager
                                      │
                                      ▼
                              Worker Nodes
```

---

# Request Flow

Example:

```
kubectl apply -f nginx.yaml
            │
            ▼
      kube-apiserver
            │
            ▼
        Request Validated
            │
            ▼
     Desired State Stored in etcd
            │
            ▼
      kube-scheduler
            │
            ▼
     Worker Node Selected
            │
            ▼
          kubelet
            │
            ▼
        containerd
            │
            ▼
        Pod Created
```

---

# Verification

View the Control Plane Pods.

```bash
kubectl get pods -n kube-system
```

Expected output includes Pods similar to:

```
etcd-kubeadm-controlplane

kube-apiserver-kubeadm-controlplane

kube-controller-manager-kubeadm-controlplane

kube-scheduler-kubeadm-controlplane
```

---

Verify the cluster.

```bash
kubectl cluster-info
```

Example output:

```
Kubernetes control plane is running at https://192.168.xxx.xxx:6443
```

---

Verify the nodes.

```bash
kubectl get nodes
```

---

# Cube Notes

## Why are the Control Plane components Pods instead of Linux services?

Kubernetes manages everything as Pods.

Running the Control Plane as Static Pods allows kubelet to monitor them continuously.

If one crashes, kubelet automatically recreates it.

This makes the Control Plane self-healing.

---

# Interview Questions

### Q1. What is the Kubernetes Control Plane?

The Control Plane is the brain of Kubernetes. It manages the cluster, stores its configuration, schedules Pods and maintains the desired state.

---

### Q2. What is the role of the API Server?

The API Server is the front door of Kubernetes. Every request passes through it before being processed.

---

### Q3. What does etcd store?

etcd stores the complete state of the Kubernetes cluster, including Pods, Nodes, Deployments, Services, Secrets, ConfigMaps and Namespaces.

---

### Q4. What does the Scheduler do?

The Scheduler selects the most appropriate worker node for newly created Pods.

---

### Q5. What is the Controller Manager responsible for?

The Controller Manager continuously compares the desired state with the actual state and takes corrective actions whenever they differ.

---

### Q6. What are Static Pods?

Static Pods are Pods managed directly by kubelet using manifest files stored under:

```text
/etc/kubernetes/manifests
```

The Control Plane components created by kubeadm are Static Pods.

---

# Key Takeaways

- The Control Plane is the brain of the Kubernetes cluster.
- The API Server is the entry point for all Kubernetes operations.
- etcd stores the complete cluster state.
- The Scheduler assigns Pods to worker nodes.
- The Controller Manager maintains the desired state.
- kubeadm creates the Control Plane components as Static Pods.
- kubelet monitors Static Pods and recreates them automatically if they fail.
