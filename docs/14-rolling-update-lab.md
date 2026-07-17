
# 09 - Kubernetes Rolling Update Lab

## Objective

Learn how a Kubernetes Deployment performs a rolling update by gradually replacing Pods from an old ReplicaSet with Pods from a new ReplicaSet.

---

## Important Concepts

### Scaling

Scaling changes only the number of Pods.

```text
Same image
Same ReplicaSet
More or fewer Pods
```

### Rolling Update

A rolling update happens when the Deployment's Pod template changes, such as changing the container image.

```text
Old image
    ↓
New ReplicaSet created
    ↓
Old Pods gradually replaced
    ↓
New Pods gradually started
```

---

## Lab Environment

Run all commands from:

```text
kubeadm-controlplane
```

The `kubectl` configuration on this machine communicates with the Kubernetes API Server.

---

## Step 1 - Verify the Deployment

```bash
kubectl get deployment nginxalpine
```

Verify the existing Pods:

```bash
kubectl get pods -o wide
```

Check the current image:

```bash
kubectl describe deployment nginxalpine
```

Expected current image:

```text
nginx:alpine
```

---

## Step 2 - Scale the Deployment to Three Replicas

```bash
kubectl scale deployment nginxalpine --replicas=3
```

Verify:

```bash
kubectl get deployment nginxalpine
kubectl get rs
kubectl get pods -o wide
```

### Expected Result

- One Deployment
- One active ReplicaSet
- Three running Pods
- All Pods use the same image

This is called **horizontal scaling**, not a rolling update.

---

## Step 3 - Open Three Terminals

Open three SSH sessions to the control-plane node.

### Terminal 1 - Watch ReplicaSets

```bash
kubectl get rs -w
```

### Terminal 2 - Watch Pods

```bash
kubectl get pods -o wide -w
```

### Terminal 3 - Perform the Image Update

Before updating, confirm the container name:

```bash
kubectl get deployment nginxalpine \
  -o jsonpath='{.spec.template.spec.containers[*].name}'
```

Expected:

```text
nginx
```

Update the image:

```bash
kubectl set image deployment/nginxalpine \
  nginx=nginx:1.27-alpine
```

---

## Step 4 - Observe the Rolling Update

Watch Terminal 1.

You should see:

```text
Old ReplicaSet: 3 → 2 → 1 → 0
New ReplicaSet: 0 → 1 → 2 → 3
```

Watch Terminal 2.

You should see:

```text
New Pods created
Old Pods terminated
New Pods become Running
```

The Deployment keeps the application available while gradually replacing old Pods.

Press:

```text
Ctrl+C
```

to stop each watch command.

---

## Step 5 - Check Rollout Status

```bash
kubectl rollout status deployment/nginxalpine
```

Expected:

```text
deployment "nginxalpine" successfully rolled out
```

---

## Step 6 - Verify the New Image

```bash
kubectl get deployment nginxalpine -o wide
```

Verify the image directly:

```bash
kubectl get deployment nginxalpine \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
```

Expected:

```text
nginx:1.27-alpine
```

---

## Step 7 - View Both ReplicaSets

```bash
kubectl get rs
```

Expected:

```text
Old ReplicaSet   0 Pods
New ReplicaSet   3 Pods
```

The old ReplicaSet remains so Kubernetes can use it for rollback.

---

## Step 8 - View Rollout History

```bash
kubectl rollout history deployment/nginxalpine
```

This shows the Deployment revisions.

---

## Step 9 - Roll Back the Deployment

Undo the image update:

```bash
kubectl rollout undo deployment/nginxalpine
```

Watch the rollback:

```bash
kubectl rollout status deployment/nginxalpine
```

Verify the image:

```bash
kubectl get deployment nginxalpine \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
```

The previous ReplicaSet should scale up again while the newer ReplicaSet scales down.

---

## Architecture Story

```text
Deployment
    │
    ├── Old ReplicaSet
    │      └── Old Pods
    │
    └── New ReplicaSet
           └── New Pods
```

During a rolling update:

```text
Old ReplicaSet decreases
New ReplicaSet increases
```

The Deployment coordinates the process.

---

## Key Takeaways

- Scaling changes the number of Pods but keeps the same ReplicaSet.
- Changing the Pod template creates a new ReplicaSet.
- A rolling update gradually replaces old Pods with new Pods.
- Old ReplicaSets are retained for rollback.
- The Deployment manages ReplicaSets.
- ReplicaSets maintain the desired number of Pods.
- Pods run containers.

---

## Useful Commands

```bash
kubectl get deployment
kubectl get rs
kubectl get pods -o wide
kubectl scale deployment nginxalpine --replicas=3
kubectl set image deployment/nginxalpine nginx=nginx:1.27-alpine
kubectl rollout status deployment/nginxalpine
kubectl rollout history deployment/nginxalpine
kubectl rollout undo deployment/nginxalpine
```
