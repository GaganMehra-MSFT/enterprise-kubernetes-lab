# Kubernetes StatefulSets

## 1. What Is a StatefulSet?

A StatefulSet is a Kubernetes workload controller used to manage Pods that need a **stable identity**.

The easiest way to understand it is to compare it with a Deployment.

> [!IMPORTANT]
> **StatefulSet = like a Deployment, but the Pods have stable individual identities.**

A Deployment might create:

```text
website-64dc95559f-abc12
website-64dc95559f-xzy45
website-64dc95559f-pqr78
```

These Pods are treated as interchangeable copies.

A StatefulSet creates predictable names:

```text
stateful-demo-0
stateful-demo-1
stateful-demo-2
```

These identities are important to Kubernetes.

---

# 2. Deployment vs StatefulSet

Start with a Deployment.

Suppose we tell Kubernetes:

```text
Give me 3 copies of my website.
```

Deployment creates:

```text
                 Deployment
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
        Pod-A      Pod-B      Pod-C
```

For most stateless applications:

```text
Pod-A = Pod-B = Pod-C
```

We don't care which Pod handles a request.

If Pod-B dies:

```text
Pod-B ❌
```

Deployment creates another Pod.

We only care that:

```text
Desired replicas = 3
Actual replicas  = 3
```

Think:

> **Deployment = Give me N interchangeable Pods.**

---

# 3. StatefulSet Changes This Model

Now imagine that the Pods need individual identities.

Instead of:

```text
Pod-A
Pod-B
Pod-C
```

StatefulSet creates:

```text
stateful-demo-0
stateful-demo-1
stateful-demo-2
```

Think:

> **StatefulSet = Give me N Pods, but preserve their individual identities.**

This is the foundation of StatefulSet.

---

# 4. What Does "Stable Identity" Mean?

Suppose:

```text
stateful-demo-1
```

dies.

Kubernetes does not simply decide:

```text
"I need another random Pod."
```

The StatefulSet controller knows that this specific identity should exist:

```text
stateful-demo-1
```

So it recreates:

```text
stateful-demo-1
```

Think of the number as the Pod's assigned identity:

```text
stateful-demo-0
              ↑
              ordinal

stateful-demo-1
              ↑
              ordinal

stateful-demo-2
              ↑
              ordinal
```

These numbers are called **ordinals**.

---

# 5. Simple Analogy

Think about a Deployment as three cashiers.

```text
Cashier
Cashier
Cashier
```

You don't care which cashier serves you.

If one leaves, another cashier can take the position.

That is similar to:

```text
Deployment
=
interchangeable workers
```

Now imagine three hotel rooms:

```text
Room 0
Room 1
Room 2
```

Room 1 has its own:

```text
Room number
Storage
Identity
```

If the door to Room 1 is replaced, it is still:

```text
Room 1
```

It does not suddenly become Room 7.

That is closer to a StatefulSet.

```text
StatefulSet
=
individual Pods have identities
```

---

# 6. Why Would an Application Need Stable Pod Identity?

Many web applications don't care about individual Pod identity.

For example:

```text
NGINX
Web API
Frontend
Stateless microservice
```

Any replica can usually handle the request.

But some distributed/stateful applications need individual members to have predictable identities.

Examples include:

```text
Database clusters
Kafka brokers
ZooKeeper
Elasticsearch clusters
Distributed storage systems
```

For example:

```text
database-0
database-1
database-2
```

Individual members may need to know about other specific members.

---

# 7. StatefulSet Provides Several Important Behaviors

StatefulSet commonly provides:

```text
1. Stable Pod names

2. Ordered Pod identities

3. Ordered creation/scaling behavior

4. Stable network identity when used with
   the appropriate Service/DNS setup

5. Persistent storage can be associated
   with each individual Pod
```

These behaviors make StatefulSet useful for applications where individual replicas are not simply anonymous copies.

---

# 8. Ordered Pod Creation

This is something we actually observed in our lab.

We configured:

```yaml
replicas: 3
```

Kubernetes did not simply start all three StatefulSet Pods without considering order.

The default StatefulSet behavior follows the ordinal sequence.

Conceptually:

```text
Create stateful-demo-0
          │
          ▼
Wait for it to become Ready
          │
          ▼
Create stateful-demo-1
          │
          ▼
Wait for it to become Ready
          │
          ▼
Create stateful-demo-2
```

In our lab we actually saw:

```text
stateful-demo-0    Running
stateful-demo-1    Pending
```

After storage became available for `stateful-demo-1`, it started.

Then Kubernetes proceeded with:

```text
stateful-demo-2
```

This demonstrated StatefulSet's ordered behavior.

---

# 9. StatefulSet and Storage

This is one of the most important reasons StatefulSets are used.

A StatefulSet Pod can have its **own persistent storage**.

Our final lab architecture looked like:

```text
StatefulSet
│
├── stateful-demo-0
│        │
│        ▼
│   data-stateful-demo-0
│          PVC
│        │
│        ▼
│   pv-stateful-demo-0
│        │
│        ▼
│   Storage for demo-0
│
├── stateful-demo-1
│        │
│        ▼
│   data-stateful-demo-1
│          PVC
│        │
│        ▼
│   pv-stateful-demo-1
│        │
│        ▼
│   Storage for demo-1
│
└── stateful-demo-2
         │
         ▼
    data-stateful-demo-2
           PVC
         │
         ▼
    pv-stateful-demo-2
         │
         ▼
    Storage for demo-2
```

The important idea is:

> **Each StatefulSet Pod can have its own PVC and therefore its own persistent storage.**

---

# 10. Deployment Storage vs StatefulSet Storage

Our original website Deployment used one PVC:

```text
Website Deployment
       │
       ▼
website Pod
       │
       ▼
project1-pvc
       │
       ▼
project1-pv
```

Our StatefulSet behaves differently:

```text
stateful-demo-0 → PVC-0 → PV-0

stateful-demo-1 → PVC-1 → PV-1

stateful-demo-2 → PVC-2 → PV-2
```

This allows each StatefulSet member to maintain its own data.

---

# 11. volumeClaimTemplates

How did Kubernetes automatically create:

```text
data-stateful-demo-0
data-stateful-demo-1
data-stateful-demo-2
```

?

We did NOT manually create three PVC YAML files.

The StatefulSet contained:

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
        - ReadWriteOnce

      storageClassName: project1-hostpath

      resources:
        requests:
          storage: 1Gi
```

Think:

> [!IMPORTANT]
> **volumeClaimTemplates = "Create a separate PVC from this template for each StatefulSet Pod."**

---

# 12. How PVC Names Were Generated

Our template was called:

```yaml
metadata:
  name: data
```

Our StatefulSet was called:

```text
stateful-demo
```

The Pods were:

```text
stateful-demo-0
stateful-demo-1
stateful-demo-2
```

Kubernetes therefore created PVC names:

```text
data-stateful-demo-0
data-stateful-demo-1
data-stateful-demo-2
```

Think of the pattern as:

```text
<claim-template-name>-<statefulset-name>-<ordinal>
```

For our lab:

```text
data + stateful-demo + 0

        ↓

data-stateful-demo-0
```

---

# 13. How the StatefulSet Pod Uses Its PVC

Inside the StatefulSet container configuration we have a volume mount.

Conceptually:

```yaml
volumeMounts:
  - name: data
    mountPath: /data
```

This means:

> Mount this Pod's `data` volume inside the container at `/data`.

So:

```text
stateful-demo-0
       │
       ▼
data-stateful-demo-0
       │
       ▼
pv-stateful-demo-0
       │
       ▼
Worker storage
       │
       ▼
Mounted inside container
as /data
```

The container sees:

```text
/data
```

It does not need to know the physical Linux path behind the PV.

---

# 14. StorageClass in Our StatefulSet

The StatefulSet PVC template requested:

```yaml
storageClassName: project1-hostpath
```

Our PVs also advertised:

```yaml
storageClassName: project1-hostpath
```

Therefore Kubernetes could match:

```text
PV

"I HAVE:
project1-hostpath storage"

             ↕ MATCH

PVC

"I NEED:
project1-hostpath storage"
```

Once the other requirements matched:

```text
PV + PVC
   ↓
BOUND
```

---

# 15. Important Naming Note About Our StorageClass

Our StorageClass is named:

```text
project1-hostpath
```

This is simply the name we originally chose.

For our StatefulSet PVs we later changed the actual volume type from:

```yaml
hostPath:
```

to:

```yaml
local:
```

Therefore the name:

```text
project1-hostpath
```

can now be slightly misleading.

The StorageClass name itself does NOT determine the Kubernetes volume type.

For example, we could have instead called it:

```text
project1-local-storage
```

as long as the PV and PVC both referenced the same StorageClass name.

---

# 16. Why Our StatefulSet Pods Initially Stayed Pending

This was an important troubleshooting exercise.

The StatefulSet automatically created:

```text
data-stateful-demo-0
```

The PVC requested:

```text
1Gi
RWO
project1-hostpath
```

But our StorageClass uses:

```yaml
provisioner: kubernetes.io/no-provisioner
```

Therefore Kubernetes did NOT automatically create storage.

The flow became:

```text
StatefulSet
      │
      ▼
Creates Pod
stateful-demo-0
      │
      ▼
Creates PVC
data-stateful-demo-0
      │
      ▼
"I need 1Gi storage"
      │
      ▼
No matching PV
      │
      ▼
PVC Pending
      │
      ▼
Pod Pending
```

This was expected because we were doing **static provisioning**.

---

# 17. What We Had to Do Manually

On Worker-1 we created physical directories:

```text
/opt/k8s-storage/stateful-demo-0

/opt/k8s-storage/stateful-demo-1

/opt/k8s-storage/stateful-demo-2
```

Then we created three PVs:

```text
pv-stateful-demo-0
pv-stateful-demo-1
pv-stateful-demo-2
```

Each PV represented one storage location.

Eventually:

```text
PV-0 → PVC-0 → stateful-demo-0

PV-1 → PVC-1 → stateful-demo-1

PV-2 → PVC-2 → stateful-demo-2
```

---

# 18. Why We Used Local PersistentVolumes

Initially we tried:

```yaml
hostPath:
  path: /opt/k8s-storage/stateful-demo-0
```

We later changed the StatefulSet PVs to:

```yaml
local:
  path: /opt/k8s-storage/stateful-demo-0
```

The physical directory did NOT change.

It was still:

```text
Worker-1:
/opt/k8s-storage/stateful-demo-0
```

What changed was how the PV represented that storage to Kubernetes.

For the StatefulSet lab, we used a local PersistentVolume together with node affinity.

---

# 19. Why nodeAffinity Was Required

Our local storage physically exists on:

```text
kubeadm-worker-1
```

Therefore Kubernetes must know:

> This storage belongs to Worker-1.

Our PV contained:

```yaml
nodeAffinity:
  required:
    nodeSelectorTerms:
      - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
              - kubeadm-worker-1
```

Think:

```text
PV
│
├── Storage:
│   /opt/k8s-storage/stateful-demo-0
│
└── Location:
    kubeadm-worker-1
```

Now Kubernetes can understand:

```text
Pod needs this PV
       │
       ▼
PV belongs to Worker-1
       │
       ▼
Schedule Pod appropriately
```

---

# 20. StatefulSet Networking

Storage is only one side of StatefulSet.

There is also networking.

StatefulSet gives us stable Pod identities:

```text
stateful-demo-0
stateful-demo-1
stateful-demo-2
```

Sometimes applications need to discover individual members.

For that reason StatefulSets are commonly paired with a **Headless Service**.

---

# 21. Headless Service — Simple Connection

Remember:

> **Normal Service = one stable front door for a group of Pods.**

Think:

```text
              Normal Service
                    │
           "Give me a Pod"
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
     Pod-0        Pod-1        Pod-2
```

A Headless Service is different.

It uses:

```yaml
clusterIP: None
```

Think:

> **Headless Service = don't give me one normal front-door ClusterIP; let DNS expose/discover the individual Pods.**

```text
              Headless Service
                     │
                DNS discovery
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
     Pod-0         Pod-1         Pod-2
```

Detailed Service concepts belong in:

```text
docs/08-Services.md
```

---

# 22. Why StatefulSet + Headless Service Fit Together

This relationship is easy to remember:

```text
STATEFULSET
=
Give Pods stable identities.

HEADLESS SERVICE
=
Make those individual Pods discoverable
through DNS.
```

Together:

```text
                  StatefulSet
                       │
              Stable identities
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
     demo-0          demo-1         demo-2
        ▲              ▲              ▲
        │              │              │
        └──────── Headless ───────────┘
                    Service
                       │
                       ▼
                 DNS discovery
```

---

# 23. StatefulSet Does NOT Automatically Create Database Replication

This distinction is important.

StatefulSet can provide:

```text
Stable Pod names
Stable identities
Ordered Pods
Persistent storage relationships
Stable network identities
```

But StatefulSet does NOT automatically decide:

```text
Primary database
Replica database
Leader
Follower
Replication
Database failover
Data synchronization
```

Those responsibilities belong to:

```text
Database software
Database clustering configuration
or
Kubernetes Operators
```

For example:

```text
StatefulSet
     ↓
Creates:
mysql-0
mysql-1
mysql-2
```

But MySQL configuration/operator logic determines which instance is primary and how replication works.

---

# 24. What Happens If a StatefulSet Pod Dies?

Suppose:

```text
stateful-demo-1
```

is deleted.

The StatefulSet controller sees:

```text
Desired replicas = 3

But only:
stateful-demo-0
stateful-demo-2
```

It recreates:

```text
stateful-demo-1
```

The important point is that the identity returns.

Its persistent storage relationship can also remain associated with that ordinal through its PVC.

Conceptually:

```text
stateful-demo-1 ❌
       │
       ▼
StatefulSet notices
       │
       ▼
stateful-demo-1 recreated
       │
       ▼
Uses its existing PVC
       │
       ▼
Its persistent data remains
```

---

# 25. What If the Worker Node Dies?

This is extremely important for our lab.

Our storage currently physically lives on:

```text
kubeadm-worker-1
```

For example:

```text
/opt/k8s-storage/stateful-demo-0
```

Therefore:

```text
Worker-1 dies
      │
      ▼
Local storage becomes unavailable
      │
      ▼
Pod cannot simply move to Worker-2
and continue using that directory
```

Why?

Because Worker-2 does NOT have:

```text
Worker-1's local disk
```

This is a limitation of our current lab design.

---

# 26. Local Storage Is NOT High Availability

This is an important distinction.

We achieved:

```text
Pod deletion survival
```

but we have NOT achieved:

```text
Node failure survival
```

Think:

```text
Pod dies
   ↓
New Pod on correct node
   ↓
Same local storage
   ↓
Data survives
   ✅


Worker node dies
   ↓
Worker's local disk unavailable
   ↓
Storage unavailable
   ❌
```

For real high availability, applications commonly use networked/distributed/cloud storage rather than relying only on one worker's local disk.

---

# 27. StatefulSet Is Not the Storage

Do not mix these concepts.

```text
StatefulSet
=
Manages Pods with stable identities


PVC/PV
=
Provides persistent storage


Headless Service
=
Provides network discovery for
individual Pods
```

They work together, but they are separate Kubernetes concepts.

---

# 28. Complete Mental Model

This is the most important diagram in this document:

```text
                         STATEFULSET

                  "Like Deployment,
                 but stable identities"

                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          demo-0         demo-1        demo-2
             │             │             │
             │             │             │
     ┌───────┴─────────────┴─────────────┴───────┐
     │                                           │
     │                                           │
 STORAGE                                     NETWORK
     │                                           │
     ▼                                           ▼
 PVC per Pod                              Headless Service
     │                                           │
     ▼                                           ▼
 PV per Pod                                 DNS discovery
     │                                           │
     ▼                                           ▼
Persistent data                         Individual Pod names
```

---

# 29. Three Questions to Keep Kubernetes Concepts Separate

Whenever Kubernetes starts becoming confusing, ask:

### Who manages my Pods?

```text
Deployment
or
StatefulSet
```

### How do I reach my Pods?

```text
Service
```

### Where does my data live?

```text
PVC
 ↓
PV
 ↓
Storage
```

This keeps workload, networking, and storage concepts separate.

---

# 30. Commands Used in Our Lab

View StatefulSets:

```bash
kubectl get statefulset
```

Short form:

```bash
kubectl get sts
```

View Pods:

```bash
kubectl get pods
```

See which worker runs each Pod:

```bash
kubectl get pods -o wide
```

View PVCs:

```bash
kubectl get pvc
```

View PVs:

```bash
kubectl get pv
```

View StorageClasses:

```bash
kubectl get storageclass
```

Describe a StatefulSet:

```bash
kubectl describe statefulset stateful-demo
```

Describe a Pod:

```bash
kubectl describe pod stateful-demo-0
```

Enter a Pod:

```bash
kubectl exec -it stateful-demo-0 -- /bin/bash
```

If Bash is unavailable:

```bash
kubectl exec -it stateful-demo-0 -- /bin/sh
```

---

# 31. Troubleshooting Lesson From Our Lab

When our Pod showed:

```text
Pending
```

we did not immediately assume the container was broken.

We followed the dependency chain:

```text
Pod Pending
     │
     ▼
kubectl describe pod
     │
     ▼
Scheduling problem
     │
     ▼
Check PVC
     │
     ▼
PVC Pending
     │
     ▼
Check PV
     │
     ▼
Is matching storage available?
```

Useful commands:

```bash
kubectl describe pod stateful-demo-0
kubectl describe pvc data-stateful-demo-0
kubectl describe pv pv-stateful-demo-0
```

This is an important Kubernetes troubleshooting habit:

> **Start with the symptom and follow the dependencies instead of randomly changing YAML.**

---

# 32. Interview Explanation

If asked:

### "What is a StatefulSet?"

A simple answer:

> A StatefulSet is similar to a Deployment, but it is designed for workloads whose Pods require stable identities. StatefulSet Pods receive predictable ordinal names such as `database-0`, `database-1`, and `database-2`. StatefulSets also support ordered deployment and can provide each Pod with its own persistent storage through `volumeClaimTemplates`.

If asked:

### "Deployment vs StatefulSet?"

Answer:

> Deployment Pods are generally treated as interchangeable replicas, while StatefulSet Pods maintain stable individual identities and can maintain their own persistent storage relationships.

If asked:

### "Why does StatefulSet use a Headless Service?"

Answer:

> A StatefulSet provides stable Pod identities, while a Headless Service allows the individual Pods to be discovered through DNS instead of hiding all of them behind one normal ClusterIP.

---

# 33. Final Memory Rules

> [!IMPORTANT]
> **Deployment = Give me N interchangeable Pods.**

> [!IMPORTANT]
> **StatefulSet = Like Deployment, but preserve each Pod's identity.**

> [!IMPORTANT]
> **Service = Stable network front door for a group of Pods.**

> [!IMPORTANT]
> **Headless Service = No normal single front door; use DNS to discover individual Pods.**

> [!IMPORTANT]
> **PV = "I HAVE storage."**

> [!IMPORTANT]
> **PVC = "I NEED storage."**

> [!IMPORTANT]
> **volumeClaimTemplates = Create one PVC for each StatefulSet Pod.**

And finally:

```text
StatefulSet
   │
   ├── WHO are my Pods?
   │       ↓
   │   stable identity
   │
   ├── HOW are they discovered?
   │       ↓
   │   Headless Service
   │
   └── WHERE is their data?
           ↓
       PVC → PV → Storage
```
