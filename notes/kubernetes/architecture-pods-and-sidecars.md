# Kubernetes — Learning Notes
### Session 1: The WHY, The Architecture, Pods & Sidecars
> *Built through reasoning, not rote learning. Every concept earned its place.*

---

## 0. The Mindset — Why This Way

Before a single concept, the right question to ask about anything in Kubernetes is:
> **"What problem existed before this, and why wasn't the previous solution good enough?"**

A technician follows steps. An engineer understands decisions. These notes are written for the engineer.

---

## 1. The Problem Kubernetes Solves — The Trilogy of Pain

Imagine a Spring Boot JAR running on a single EC2. It works. Until it doesn't.

### Pain Point 1 — Single Point of Failure (Fault Tolerance)
One instance, one process, one machine. If that process dies — for any reason — your entire operation collapses. Customers are impacted. Revenue is lost. Goodwill, the hardest thing to rebuild, takes the hit.

**What we needed:** Something that *automatically* detects failure and replaces the instance — without a human in the loop.

### Pain Point 2 — Operational Overhead
Even if you run two EC2s, someone has to watch them. That means a person, on-call, possibly at 3am, doing work that is entirely mechanical and repetitive. That doesn't scale — and it's an appalling use of human intelligence.

**What we needed:** The watching, the responding, the replacing — all automated. Infrastructure that manages itself.

### Pain Point 3 — Dynamic Scaling
Traffic is not a straight line. A sale goes live, traffic spikes 10x in twenty minutes. By the time a human notices, calculates how many instances to add, and provisions them — the damage is done. Worse, once the spike subsides, those extra machines sit idle, burning money.

**What we needed:** Infrastructure that *breathes* with demand. Expanding intelligently when needed, contracting when not — without human intervention and without guesswork.

### The Compounding Problem — Microservices
None of these three pains are unique to one service. A real company runs dozens — payment service, inventory, notifications, user auth, each with its own instances, failure modes, and traffic patterns. Do you solve this independently for each service, fifteen times over?

> **No. You want one intelligent governing system that manages all workloads uniformly. That system is Kubernetes.**

---

## 2. The Core Architecture

### Cluster
The entire collective of machines that Kubernetes governs — treated as **one unified pool of compute resources**. You don't think about individual machines when deploying. You think about the cluster.

*Why not just individual machines?* Because managing machines individually defeats the purpose. The cluster abstraction lets Kubernetes make intelligent, global scheduling decisions — placing workloads where resources are available, not where you manually point them.

### Node
An individual machine — physical or virtual — within the cluster. This is the raw material Kubernetes works with. It provides CPU, RAM, and storage.

**Two kinds of Nodes:**

| Type | Role |
|---|---|
| **Control Plane** (formerly "Master") | The brain. Makes scheduling decisions, watches for failures, maintains desired state. |
| **Worker Nodes** | The muscle. Run your actual workloads. Execute instructions, no opinions. |

*What if the Control Plane goes down?* A single Control Plane is itself a single point of failure. Production clusters run **multiple Control Plane nodes** for resilience — typically three, so they can reach consensus even if one fails. This is called a **highly available control plane**.

### The Control Plane — What's Inside It

The Control Plane isn't just one process. It's a set of components, each with a specific responsibility:

| Component | Job |
|---|---|
| **API Server** | The front door. Every instruction — from you, from internal components — goes through here. |
| **etcd** | The cluster's memory. A distributed key-value store that holds the *entire desired state* of the cluster. If etcd is lost, the cluster loses its mind. |
| **Scheduler** | Decides *which Node* a new Pod should run on, based on available resources and constraints. |
| **Controller Manager** | Watches actual state vs desired state. If something drifts — a Pod dies, a Node goes offline — it acts to reconcile. |

---

## 3. Pod — The Atomic Unit of Kubernetes

### What is a Container? (Precisely)
A container is **not** a small VM. A VM virtualises the entire hardware, including its own kernel. A container is far lighter — it shares the host machine's OS kernel but gets its own isolated slice of everything else: filesystem, process space, network interface, environment variables.

Docker is the most popular tool for building and running containers — but not the only one. Others include containerd, CRI-O, and podman. Each works slightly differently internally.

**The CRI — Container Runtime Interface**
Kubernetes doesn't hardcode Docker. It defines a standard interface — the CRI — that any container runtime must implement to work with Kubernetes. Think of it like UPI in India: PhonePe, GPay, and Paytm all look different, but they all converge on the same standardised payment interface. The CRI is that interface for container runtimes.

### Why Pod and Not Just Container?
Docker already manages containers. Kubernetes could have made the container its smallest unit. It didn't — and for good reason.

Some processes are so tightly coupled that separating them would be artificial and painful. Consider: your Spring Boot app writes logs to a file, and a log-shipping agent reads those logs and forwards them to Elasticsearch. These are two separate processes — two separate containers. But they need to:
- Share the **same network namespace** — same IP address, same `localhost`
- Share the **same storage volumes** — same filesystem access

If they're in separate Pods, this is impossible. A Pod solves this by providing **shared context** — an envelope in which tightly coupled containers coexist as if they're processes on the same machine.

> **A Pod is not a container. It is an envelope that provides shared network and storage context to one or more containers that logically belong together.**

### The Full Hierarchy

```
Cluster
  └── Node (many)
        └── Pod (many per node)
              └── Container (usually one, sometimes more)
                    └── Your JAR / Application
```

### Important Reality Check
In *most* real-world cases, a Pod runs exactly **one container**. The multi-container Pod is the exception, not the rule. But Kubernetes designed for that exception deliberately — because when you need it, you *really* need it.

---

## 4. The Sidecar Pattern

A **sidecar** is a secondary container within a Pod that serves, enhances, or proxies the main container — without being the main act itself. It only makes sense *because* the main container exists.

### What Makes Something a Sidecar Candidate?
- It is **not** your core application logic
- It handles a **cross-cutting concern** — something that every service needs, not unique to one
- It benefits from sharing the Pod's **network or filesystem**
- Keeping it separate keeps your application code **clean and focused**

### Classic Sidecar Example 1 — Log Shipper
Your Spring Boot app writes logs to a file on disk. With 20 Pods across 5 Nodes, you'd have to SSH into each container to read logs — chaos.

A lightweight sidecar (e.g., Fluentd, Filebeat) runs alongside your app, reads the shared log volume, and ships everything to a central store like Elasticsearch. Your app doesn't know or care.

```
Pod
 ├── spring-boot-container  →  writes logs to /var/log/app.log
 └── fluentd-sidecar        →  reads /var/log/app.log, ships to Elasticsearch
         (shared volume)
```

### Classic Sidecar Example 2 — Proxy / Service Mesh Agent
Your Spring Boot service calls the Payment service. You want encryption in transit, automatic retries, circuit breaking, and traffic metrics — but you don't want to write that logic inside every single service.

A proxy sidecar (e.g., **Envoy**) runs alongside every Pod. All traffic in and out flows through this proxy, which handles the cross-cutting concerns transparently. Your app code stays clean.

At scale, this pattern across an entire cluster is called a **Service Mesh**. Tools like **Istio** and **Linkerd** are built entirely on this sidecar proxy idea.

### What is NOT a Sidecar — and Why
A database like MySQL is **not** a sidecar. Consider why:

If your Spring Boot app runs 3 Pods for resilience, and each Pod has its own MySQL sidecar — you now have 3 independent databases, each with a different copy of your data. Catastrophic.

Databases are **stateful** — they cannot be casually duplicated or destroyed the way stateless app containers can. Kubernetes handles stateful workloads through a completely different construct: the **StatefulSet** *(covered in a later session)*.

> The rule of thumb: sidecars are **stateless helpers**. Anything with its own data, its own identity, its own persistence — is not a sidecar.

---

## 5. Key Vocabulary Earned So Far

| Term | Definition |
|---|---|
| **Fault Tolerance** | A system's capacity to absorb failure without collapsing |
| **Orchestration** | Coordination of complex, multi-component systems toward a unified outcome — borrowed from music |
| **CRI** | Container Runtime Interface — the standard Kubernetes uses to talk to any container runtime |
| **Control Plane** | The brain of the cluster — schedules, watches, reconciles |
| **etcd** | The cluster's distributed memory — holds desired state |
| **Sidecar** | A helper container in the same Pod, handling cross-cutting concerns |
| **Service Mesh** | A cluster-wide pattern of sidecar proxies managing inter-service communication |
| **Idempotent** | An operation that produces the same result regardless of how many times it is executed — Kubernetes is built on this principle |
| **Resilience** | The capacity of a system to absorb disruption and return to function — not mere survival, but recovery with continuity |

---

## 6. The Question We Haven't Answered Yet

We know what a Pod is. But:

> Who tells Kubernetes that a Pod should exist in the first place? How many of them? What happens when one dies — does someone manually request a replacement?

If the answer were "yes, manually" — we've solved nothing. The answer involves **Controllers** — specifically, the **Deployment** and **ReplicaSet**.

*That's where Session 2 begins.*

---

*Notes compiled from first-principles reasoning — Session 1*
*Next: Controllers, Deployments, ReplicaSets, and StatefulSets*
