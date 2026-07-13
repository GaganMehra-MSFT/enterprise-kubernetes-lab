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
