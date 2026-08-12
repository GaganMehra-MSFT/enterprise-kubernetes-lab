# 12 - Kubernetes Storage

## Objective

Understand how Kubernetes provides persistent storage to applications using:

* StorageClass
* PersistentVolume (PV)
* PersistentVolumeClaim (PVC)
* Volumes and Volume Mounts
* `hostPath`

The first storage lab uses storage physically located on **kubeadm-worker-1**.

---

# Kubernetes Storage Architecture

For this lab:

```text
kubeadm-worker-1
│
└── /opt/k8s-storage/project1
             │
             ▼
      PersistentVolume
             │
             ▼
   PersistentVolumeClaim
             │
             ▼
            Pod
```

The actual data is stored on the Linux filesystem of `kubeadm-worker-1`.

---

# Why Do We Need Persistent Storage?

Containers are temporary.

If a container is deleted and recreated, data stored only inside the container can be lost.

For applications such as:

* Databases
* File servers
* Stateful applications

we need storage whose lifecycle is independent of the container.

Kubernetes provides this abstraction through PersistentVolumes and PersistentVolumeClaims.

Think:

```text
Pod can disappear
        ↓
New Pod can be created
        ↓
Persistent storage remains
        ↓
New Pod reconnects to storage
```

---

# Kubernetes Storage Components

## PersistentVolume (PV)

A PersistentVolume represents storage that is available to Kubernetes.

Think:

> **PV = "I have storage available."**

Example:

```text
5 GiB storage available
on kubeadm-worker-1
at /opt/k8s-storage/project1
```

---

## PersistentVolumeClaim (PVC)

A PVC is a request for storage from an application.

Think:

> **PVC = "I need storage."**

For example:

```text
Application:
"I need 2 GiB of ReadWriteOnce storage."
```

Kubernetes looks for a compatible PV and binds the PVC to it.

---

## Pod

The Pod normally references the **PVC**, not the underlying physical storage directly.

Therefore:

```text
Pod
 │
 ▼
PVC
 │
 ▼
PV
 │
 ▼
Actual Storage
```

This abstraction becomes important when the storage backend changes.

For example:

```text
Today:

Pod → PVC → PV → hostPath


Later:

Pod → PVC → PV → NFS


Enterprise:

Pod → PVC → Storage → NetApp / Nutanix / Cloud Disk
```

The application does not need to understand the physical storage implementation.

---

# What is hostPath?

`hostPath` allows Kubernetes to use a directory from the filesystem of the node running the workload.

For this lab:

```text
Node:
kubeadm-worker-1

Storage directory:
/opt/k8s-storage/project1
```

Architecture:

```text
kubeadm-worker-1
│
├── Linux filesystem
│
│   └── /opt/k8s-storage/project1
│
└── Kubernetes Pod
        │
        ▼
       PVC
        │
        ▼
       PV
        │
        ▼
/opt/k8s-storage/project1
```

---

# Important Limitation of hostPath

The storage exists only on the node where the directory exists.

For example:

```text
Worker-1
/opt/k8s-storage/project1
        ↑
        Data exists here


Worker-2
/opt/k8s-storage/project1
        ↑
        Different local filesystem
```

Worker-2 cannot automatically access the data stored on Worker-1.

This is one reason `hostPath` is useful for learning and testing but is generally not appropriate as shared enterprise storage.

Later, this lab can be extended using a dedicated NFS server to demonstrate shared storage.

---

# Lab Environment

Kubernetes nodes:

```text
kubeadm-controlplane
192.168.235.134

kubeadm-worker-1
192.168.235.135

kubeadm-worker-2
192.168.235.136
```

Storage for this lab will reside on:

```text
kubeadm-worker-1
192.168.235.135
```

---

# Step 1 - Label the Storage Node

The worker node was labeled so that Kubernetes configuration can identify the node associated with the local storage.

Run from the control plane:

```bash
kubectl label node kubeadm-worker-1 storage=project1
```

Verify:

```bash
kubectl get node kubeadm-worker-1 --show-labels
```

Expected label:

```text
storage=project1
```

## Why Add a Node Label?

A label is metadata attached to a Kubernetes object.

In this case:

```text
kubeadm-worker-1
        │
        └── storage=project1
```

The label does **not create storage**.

It simply gives us a way to identify Worker-1 as the node being used for the project storage.

---

# Step 2 - Prepare Storage on Worker-1

SSH from the control plane to Worker-1:

```bash
ssh root1@192.168.235.135
```

Create the storage directory:

```bash
sudo mkdir -p /opt/k8s-storage/project1
```

For this learning lab, permissions were configured as:

```bash
sudo chmod 777 /opt/k8s-storage/project1
```

Verify:

```bash
ls -ld /opt/k8s-storage/project1
```

Lab result:

```text
drwxrwxrwx root root /opt/k8s-storage/project1
```

> `777` permissions are being used only to simplify this learning lab. This is not a recommended production security configuration.

Return to the control plane:

```bash
exit
```

---

# Step 3 - Create a StorageClass

## What is a StorageClass?

A StorageClass describes a **class or type of storage** available to Kubernetes.

Think:

```text
StorageClass
      =
"What kind of storage is this?"
```

Examples in real environments might represent:

```text
fast-ssd
standard-disk
premium-disk
nfs-storage
netapp-storage
```

For this lab we created:

```text
project1-hostpath
```

---

# StorageClass YAML

File:

```text
manifests/website/storageclass.yaml
```

Contents:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass

metadata:
  name: project1-hostpath

provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
```

---

# Understanding the StorageClass YAML

## apiVersion

```yaml
apiVersion: storage.k8s.io/v1
```

StorageClass belongs to the Kubernetes storage API group.

---

## kind

```yaml
kind: StorageClass
```

We are asking Kubernetes to create a StorageClass object.

---

## metadata.name

```yaml
metadata:
  name: project1-hostpath
```

The name of our StorageClass is:

```text
project1-hostpath
```

PVCs and PVs can reference this name when they need to use this storage class.

---

# Why `kubernetes.io/no-provisioner`?

```yaml
provisioner: kubernetes.io/no-provisioner
```

This means Kubernetes will **not automatically create storage** for us.

We are manually providing the storage.

The actual directory already exists:

```text
kubeadm-worker-1

/opt/k8s-storage/project1
```

We will manually create a PersistentVolume that points to this storage.

Therefore:

```text
StorageClass
project1-hostpath
        │
        │ no-provisioner
        ▼
Kubernetes does NOT create a disk
        │
        ▼
Administrator manually creates PV
```

This is called **static provisioning**.

---

# Static vs Dynamic Provisioning

## Static Provisioning

The administrator creates the storage and PV manually.

```text
Administrator
      │
      ▼
Creates storage
      │
      ▼
Creates PV
      │
      ▼
Application creates PVC
```

This is what we are doing in this lab.

---

## Dynamic Provisioning

In a production environment, a StorageClass can use a storage provisioner or CSI driver.

For example:

```text
Application
     │
     ▼
PVC
     │
     ▼
StorageClass
     │
     ▼
CSI / Storage Provisioner
     │
     ▼
Storage automatically created
```

Examples of storage backends include:

* Azure Disk
* AWS EBS
* NetApp
* Nutanix
* NFS provisioners

We will study this after understanding static provisioning.

---

# Why `WaitForFirstConsumer`?

```yaml
volumeBindingMode: WaitForFirstConsumer
```

This tells Kubernetes:

> Do not finalize the storage binding until a Pod actually needs the PVC.

Why is this useful?

Our storage exists only on:

```text
kubeadm-worker-1
```

Kubernetes eventually needs to consider two things together:

```text
Where is the storage?
        +
Where should the Pod run?
```

For node-local storage, these decisions are related.

Conceptually:

```text
PVC requests storage
        │
        ▼
Wait
        │
        ▼
Pod requests PVC
        │
        ▼
Kubernetes considers Pod placement
and storage location
        │
        ▼
Storage binding / scheduling proceeds
```

---

# Why `Retain`?

```yaml
reclaimPolicy: Retain
```

`Retain` tells Kubernetes that the underlying storage should not automatically be discarded when its claim is released.

Think:

```text
Application uses PVC
        │
        ▼
Data written to storage
        │
        ▼
PVC eventually deleted
        │
        ▼
PV may become Released
        │
        ▼
Underlying data is retained
```

For our lab, the data resides at:

```text
/opt/k8s-storage/project1
```

The goal is to preserve the data rather than automatically remove it.

This is especially important when dealing with valuable application or database data.

---

# Is StorageClass Namespaced?

No.

StorageClass is a **cluster-scoped object**.

Therefore, the StorageClass YAML does not contain:

```yaml
namespace: project1-website
```

Compare:

```text
OBJECT                    SCOPE

StorageClass              Cluster
PersistentVolume          Cluster

PersistentVolumeClaim     Namespace
Deployment                Namespace
Pod                       Namespace
Service                   Namespace
ConfigMap                 Namespace
Secret                    Namespace
```

This distinction is important.

The StorageClass can potentially be used by applications from multiple namespaces.

---

# Step 4 - Validate StorageClass

Before applying the manifest:

```bash
kubectl apply --dry-run=server -f storageclass.yaml
```

If validation succeeds:

```bash
kubectl apply -f storageclass.yaml
```

Verify:

```bash
kubectl get storageclass
```

Expected:

```text
project1-hostpath
```

More detailed information:

```bash
kubectl describe storageclass project1-hostpath
```

---

# Current Lab Status

Completed:

```text
[✓] Kubernetes cluster running
[✓] project1-website namespace
[✓] Worker-1 labeled storage=project1
[✓] /opt/k8s-storage/project1 created
[✓] StorageClass created
```

Next:

```text
[ ] PersistentVolume (PV)
[ ] PersistentVolumeClaim (PVC)
[ ] Attach PVC to workload
[ ] Write data
[ ] Delete Pod
[ ] Recreate Pod
[ ] Verify data survives
```

---

# Architecture So Far

At this stage:

```text
Kubernetes Cluster
│
├── StorageClass
│     project1-hostpath
│
└── kubeadm-worker-1
      │
      └── /opt/k8s-storage/project1
```

These two pieces exist, but they are **not connected yet**.

The next object will create that connection:

```text
StorageClass
project1-hostpath
        │
        ▼
PersistentVolume       ← NEXT
        │
        ▼
/opt/k8s-storage/project1
on kubeadm-worker-1
```

---

# Cube Notes

## StorageClass

Defines the type/class of storage available to Kubernetes.

## PersistentVolume

Represents actual storage available to Kubernetes.

Think:

> **PV = I have storage.**

## PersistentVolumeClaim

Represents an application's request for storage.

Think:

> **PVC = I need storage.**

## Pod

Consumes storage through the PVC.

The basic relationship is:

```text
Pod
 ↓
PVC
 ↓
PV
 ↓
Physical Storage
```

For this lab:

```text
Pod
 ↓
PVC
 ↓
PV
 ↓
hostPath
 ↓
kubeadm-worker-1
 ↓
/opt/k8s-storage/project1
```

---

# Key Takeaways

* Container filesystem storage is not the same as persistent application storage.
* PV represents storage available to Kubernetes.
* PVC represents an application's request for storage.
* Pods normally consume persistent storage through PVCs.
* StorageClass describes a category/type of storage.
* `no-provisioner` means the PV must be created manually.
* `WaitForFirstConsumer` delays binding until a workload needs the storage.
* `Retain` preserves underlying storage after the claim is released.
* `hostPath` uses storage local to a Kubernetes node.
* `hostPath` is useful for learning but does not provide shared enterprise storage.
* StorageClass and PV are cluster-scoped.
* PVCs and application workloads are namespaced.

---

# Next Step

Create the PersistentVolume that connects Kubernetes to:

```text
kubeadm-worker-1
/opt/k8s-storage/project1
```

The next YAML file will be:

```text
manifests/website/persistentvolume.yaml
```
