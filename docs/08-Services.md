# Kubernetes Services

## 1. Why Do We Need a Service?

Before understanding a Kubernetes Service, first remember one important fact:

> **Pods are temporary.**

A Pod gets its own IP address.

For example:

```text
website-pod
IP: 192.168.186.21
```

Another application could technically communicate directly with this Pod using its IP address.

But this creates a problem.

Suppose we have three website Pods:

```text
Website Deployment

    ├── Pod-A → 192.168.1.10
    ├── Pod-B → 192.168.1.11
    └── Pod-C → 192.168.1.12
```

Now imagine Pod-A dies.

The Deployment creates a replacement Pod:

```text
Old Pod-A
192.168.1.10
     ❌

      ↓

New Pod
192.168.1.47
     ✅
```

The new Pod can have a completely different IP address.

If another application was configured to communicate directly with:

```text
192.168.1.10
```

that connection would now fail.

We therefore need something that stays stable even when the Pods behind it change.

That is one of the main jobs of a **Service**.

---

# 2. What Is a Kubernetes Service?

A Kubernetes Service provides a **stable network endpoint for a group of Pods**.

The easiest way to remember it is:

> [!IMPORTANT]
> **Service = a permanent phone number/front door for a group of temporary Pods.**

Instead of clients communicating directly with individual Pod IP addresses:

```text
Client
  ↓
Pod IP
```

they communicate with a Service:

```text
                  Client
                     │
                     ▼
              ┌─────────────┐
              │   SERVICE   │
              │ Stable IP   │
              └──────┬──────┘
                     │
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
        Pod-A      Pod-B      Pod-C
```

Pods can be destroyed and recreated.

The Service remains.

---

# 3. Simple Real-World Analogy

Imagine a hotel has three employees:

```text
John
Mary
Sam
```

Customers do not normally care which employee answers the phone.

The hotel publishes one phone number:

```text
HOTEL

555-1000
   │
   ├── John
   ├── Mary
   └── Sam
```

Customers call:

```text
555-1000
```

Someone answers.

The customer does not need John's, Mary's, or Sam's personal phone number.

A Kubernetes Service works similarly.

```text
Application
     │
     ▼
 Service
     │
     ├── Pod-A
     ├── Pod-B
     └── Pod-C
```

The application talks to the Service rather than keeping track of individual Pod IP addresses.

---

# 4. Service vs Pod

Think of the difference this way:

```text
POD
=
The actual application instance.

SERVICE
=
A stable way of reaching one or more application Pods.
```

For example:

```text
Service: website-service

          │
          ▼

Pods running NGINX:

website-A
website-B
website-C
```

The Service does NOT run NGINX.

The Pods run NGINX.

The Service provides network access to those Pods.

---

# 5. How Does a Service Know Which Pods Belong to It?

Kubernetes uses:

```text
Labels
+
Selectors
```

Suppose our Pods have:

```yaml
labels:
  app: website
```

The Service can contain:

```yaml
selector:
  app: website
```

The Service is essentially saying:

> Find the Pods whose label is `app=website`.

Conceptually:

```text
SERVICE

selector:
app=website

      │
      │ finds matching labels
      ▼

┌──────────────────────────┐
│ Pod-A                    │
│ app=website              │
└──────────────────────────┘

┌──────────────────────────┐
│ Pod-B                    │
│ app=website              │
└──────────────────────────┘

┌──────────────────────────┐
│ Pod-C                    │
│ app=website              │
└──────────────────────────┘
```

A Pod with:

```text
app=database
```

would NOT be selected by this Service.

---

# 6. Example Service YAML

A simple Service could look like:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: website-service

spec:
  selector:
    app: website

  ports:
    - port: 80
      targetPort: 80

  type: ClusterIP
```

Let's understand the important parts.

---

## `selector`

```yaml
selector:
  app: website
```

Means:

> Send traffic to Pods with the label `app=website`.

---

## `port`

```yaml
port: 80
```

This is the port exposed by the **Service**.

Think:

```text
Client
   ↓
Service:80
```

---

## `targetPort`

```yaml
targetPort: 80
```

This is the port on the **Pod/container** where the application is listening.

So:

```text
Client
   ↓
Service:80
   ↓
Pod:80
   ↓
NGINX
```

---

# 7. ClusterIP Service

`ClusterIP` is the default Kubernetes Service type.

Example:

```yaml
type: ClusterIP
```

Kubernetes gives the Service its own virtual IP address.

Example:

```text
website-service
10.96.10.50
```

Now applications inside the Kubernetes cluster can communicate with:

```text
website-service
```

rather than remembering individual Pod IP addresses.

Conceptually:

```text
                   Client Pod
                       │
                       ▼
              website-service
                 10.96.10.50
                       │
                       │
            ┌──────────┼──────────┐
            ▼          ▼          ▼
          Pod-A      Pod-B      Pod-C
```

The Service provides one stable front door.

---

# 8. What Happens When a Pod Dies?

Suppose initially:

```text
Service
10.96.10.50
     │
     ├── Pod-A → 192.168.1.10
     ├── Pod-B → 192.168.1.11
     └── Pod-C → 192.168.1.12
```

Pod-A dies.

Deployment creates another Pod:

```text
Pod-A
192.168.1.10
     ❌

New Pod-D
192.168.1.47
     ✅
```

The Service remains:

```text
Service
10.96.10.50
```

Now the Service can direct traffic to the current matching Pods:

```text
Service
10.96.10.50
     │
     ├── Pod-B → 192.168.1.11
     ├── Pod-C → 192.168.1.12
     └── Pod-D → 192.168.1.47
```

The client does not need to know that Pod-A disappeared.

This is one of the major benefits of a Service.

---

# 9. The Important Mental Model

Remember:

> [!IMPORTANT]
> **Pods change. Service stays.**

Or:

```text
Pods
=
Temporary workers

Service
=
Permanent front desk
```

---

# 10. Common Kubernetes Service Types

The main Service types are:

```text
ClusterIP
NodePort
LoadBalancer
```

There is also a special Service configuration called a:

```text
Headless Service
```

For now, think:

```text
ClusterIP
=
Reach the application through a stable Service IP
from inside the cluster.

NodePort
=
Expose the Service through a port on the Kubernetes nodes.

LoadBalancer
=
Expose the Service using an external load balancer,
commonly in cloud environments.

Headless Service
=
Do not give me one normal Service IP.
Allow individual backend Pods to be discovered.
```

---

# 11. Why Do We Need a Headless Service?

To understand Headless Service, first remember how a normal Service works.

Suppose we have:

```text
Pod-A
Pod-B
Pod-C
```

A normal Service gives us one front door:

```text
                 SERVICE
                10.96.5.20
                    │
                    │
           "Give me a backend"
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
      Pod-A       Pod-B       Pod-C
```

The client normally does not care which individual Pod handles the request.

This works extremely well for many stateless applications.

But sometimes the identity of an individual Pod matters.

---

# 12. Headless Service Analogy

Return to our hotel example.

Normal hotel phone number:

```text
555-1000
```

Calling it means:

> I need someone at the hotel.

You don't care whether John, Mary, or Sam answers.

That is like a **normal Service**.

But imagine you specifically need to speak to Mary.

Calling:

```text
555-1000
```

does not guarantee Mary will answer.

Instead, imagine the hotel provides a directory:

```text
HOTEL DIRECTORY

John → extension 101
Mary → extension 102
Sam  → extension 103
```

Now individual employees can be discovered.

Think:

> [!IMPORTANT]
> **Normal Service = one common phone number/front door for the group.**
>
> **Headless Service = a directory that lets you discover the individual members.**

---

# 13. What Makes a Service Headless?

A Headless Service contains:

```yaml
clusterIP: None
```

Example:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: stateful-demo

spec:
  clusterIP: None

  selector:
    app: stateful-demo

  ports:
    - port: 80
```

The important line is:

```yaml
clusterIP: None
```

This tells Kubernetes:

> Do not assign the normal virtual ClusterIP to this Service.

---

# 14. Why Is It Called "Headless"?

Think of the normal Service IP as the common **head/front door**.

Normal Service:

```text
                   HEAD
              Service ClusterIP
                10.96.5.20
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
        Pod-A      Pod-B      Pod-C
```

A Headless Service removes that normal ClusterIP:

```text
             NO SERVICE CLUSTER IP

                clusterIP: None

          Pod-A      Pod-B      Pod-C
            ▲          ▲          ▲
            │          │          │
            └──── DNS discovery ──┘
```

Therefore:

```text
HEAD-LESS
=
No normal ClusterIP front door
```

---

# 15. What Does a Headless Service Actually Give Us?

Instead of using the Service as one load-balanced virtual IP, Kubernetes DNS can expose information about the individual backend Pods/endpoints.

This becomes particularly useful when individual application instances need to discover one another.

For example:

```text
database-0
database-1
database-2
```

Sometimes:

```text
database-1
```

needs to communicate specifically with:

```text
database-0
```

rather than:

```text
"Give me any database Pod."
```

A Headless Service supports this type of discovery.

---

# 16. Why Headless Service Is Common with StatefulSet

Remember our StatefulSet definition:

> [!IMPORTANT]
> **StatefulSet = like a Deployment, but with stable Pod identities.**

Deployment:

```text
"I need 3 interchangeable Pods."
```

StatefulSet:

```text
"I need 3 Pods with stable individual identities."

stateful-demo-0
stateful-demo-1
stateful-demo-2
```

Now the networking requirement becomes obvious.

If Kubernetes gives Pods stable identities:

```text
stateful-demo-0
stateful-demo-1
stateful-demo-2
```

we may also want applications to discover those individual identities over the network.

That is where the Headless Service fits.

---

# 17. StatefulSet + Headless Service

Think:

```text
STATEFULSET

"Give my Pods stable identities."

             │
             ▼

stateful-demo-0
stateful-demo-1
stateful-demo-2


HEADLESS SERVICE

"Let those individual Pods be
discoverable through DNS."

             │
             ▼

stateful-demo-0
stateful-demo-1
stateful-demo-2
```

Together:

```text
                StatefulSet
          Stable Pod identities
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
    demo-0        demo-1       demo-2
       ▲            ▲            ▲
       │            │            │
       └────── Headless ─────────┘
                Service
                   │
                   ▼
             DNS discovery
```

---

# 18. Example Stable DNS Identity

Suppose:

```text
StatefulSet name:
stateful-demo

Headless Service:
stateful-demo

Namespace:
project1-website
```

A StatefulSet Pod can have a predictable DNS identity such as:

```text
stateful-demo-0.stateful-demo.project1-website.svc.cluster.local
```

Break it down:

```text
stateful-demo-0
      │
      └── Pod identity

stateful-demo
      │
      └── Headless Service

project1-website
      │
      └── Namespace

svc.cluster.local
      │
      └── Kubernetes Service DNS domain
```

Conceptually:

```text
Application
     │
     │ "I need stateful-demo-0"
     ▼
Kubernetes DNS
     │
     ▼
stateful-demo-0
```

---

# 19. Normal Service vs Headless Service

| Normal Service | Headless Service |
|---|---|
| Has a ClusterIP | `clusterIP: None` |
| Provides one stable front door | No normal single Service IP front door |
| Client generally talks to the Service | Client can discover backend endpoints/Pods through DNS |
| Useful when backend Pods are interchangeable | Useful when individual backend identities matter |
| Common for stateless applications | Common with StatefulSets and clustered applications |

Easy memory trick:

```text
NORMAL SERVICE

"Take me to ANY appropriate Pod."


HEADLESS SERVICE

"Help me DISCOVER the individual Pods."
```

---

# 20. Deployment, StatefulSet, Service and Headless Service

These concepts should NOT be mixed together.

## Deployment

Answers:

> What Pods should Kubernetes maintain?

Think:

```text
Deployment
=
Give me N interchangeable Pods.
```

---

## StatefulSet

Also answers:

> What Pods should Kubernetes maintain?

But:

```text
StatefulSet
=
Like Deployment,
but give the Pods stable individual identities.
```

---

## Service

Answers a NETWORKING question:

> How do applications reliably reach a changing group of Pods?

Think:

```text
Service
=
Stable front door for a group of Pods.
```

---

## Headless Service

Also answers a NETWORKING question:

> What if I need to discover the individual Pods instead of hiding them behind one Service IP?

Think:

```text
Headless Service
=
No normal front-door ClusterIP.
Use DNS to discover the individual backend Pods.
```

---

# 21. Add Storage to the Picture

Now our StatefulSet lab makes much more sense.

```text
                         STATEFULSET

                 "Like Deployment,
                but stable identities"

                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          demo-0         demo-1        demo-2


        STORAGE                         NETWORK
           │                               │
           ▼                               ▼

        PVC-0                        Headless Service
        PVC-1                               │
        PVC-2                               ▼
           │                         DNS discovery
           ▼                               │
        PV-0                     Individual Pod identities
        PV-1
        PV-2
```

These are separate responsibilities:

```text
StatefulSet
=
Manage Pods and their stable identities.

PV/PVC
=
Persistent storage.

Headless Service
=
Network discovery of individual Pods.
```

---

# 22. Interview Explanation

If asked:

### "What is a Kubernetes Service?"

A simple answer:

> A Kubernetes Service provides a stable network endpoint for a set of Pods. Because Pods are ephemeral and their IP addresses can change, clients communicate with the Service instead of directly tracking individual Pod IPs. The Service identifies its backend Pods using label selectors.

---

### "What is a Headless Service?"

A simple answer:

> A Headless Service is a Service configured with `clusterIP: None`. Instead of providing the normal single virtual ClusterIP for load-balanced access, it allows Kubernetes DNS to expose the backend endpoints directly. It is commonly used with StatefulSets when individual Pod identities need to be discoverable.

---

### "Why use a Headless Service with StatefulSet?"

A simple answer:

> StatefulSet gives Pods stable identities such as `database-0`, `database-1`, and `database-2`. A Headless Service complements this by allowing those individual Pods to be discovered through stable DNS identities rather than hiding all of them behind one Service ClusterIP.

---

# 23. Commands to Remember

View Services:

```bash
kubectl get services
```

Short form:

```bash
kubectl get svc
```

Get more information:

```bash
kubectl get svc -o wide
```

Describe a Service:

```bash
kubectl describe service <service-name>
```

Example:

```bash
kubectl describe service stateful-demo
```

Check Pods and their IP addresses:

```bash
kubectl get pods -o wide
```

Check the endpoints selected by Services:

```bash
kubectl get endpoints
```

For newer Kubernetes environments, EndpointSlices can also be inspected:

```bash
kubectl get endpointslices
```

---

# 24. Final Mental Model

Do not memorize all the YAML first.

Remember these four sentences:

> [!IMPORTANT]
> **Deployment = Give me N interchangeable Pods.**
>
> **StatefulSet = Like Deployment, but preserve each Pod's stable identity.**
>
> **Service = Give me one stable front door for a group of changing Pods.**
>
> **Headless Service = Remove the normal single front door and let DNS help clients discover the individual Pods.**

Then connect storage separately:

```text
StatefulSet
     │
     ├── Identity
     │
     │    demo-0
     │    demo-1
     │    demo-2
     │
     ├── Networking
     │       ↓
     │   Headless Service
     │       ↓
     │   DNS discovery
     │
     └── Storage
             ↓
      volumeClaimTemplates
             ↓
          PVC per Pod
             ↓
          PV per Pod
```

That is the complete mental model.
