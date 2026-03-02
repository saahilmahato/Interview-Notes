# 🗺️ Service Discovery & Load Balancing — Complete Cheatsheet

---

## 📌 What is Service Discovery?

In a **microservices architecture**, services need to find and communicate with each other. But services are dynamic — they scale up/down, crash, restart, and get new IP addresses constantly.

**Service Discovery** is the mechanism by which services automatically detect and locate each other **without hardcoding IPs or hostnames**.

> Think of it like a **phone book** that's always up to date. Instead of memorizing a friend's number, you look them up by name.

---

## 🔍 Why Do We Need It?

In traditional monoliths, services talk to known IPs. In microservices:

- Instances are **ephemeral** (containers, VMs that come and go)
- **IP addresses change** on every restart
- Services **scale horizontally** (10 instances today, 3 tomorrow)
- **Cloud-native** environments assign dynamic addresses

Without service discovery → you'd need to manually update configs every time a service moves. That's unmanageable.

---

## 🏗️ Core Components

| Component | Role |
|---|---|
| **Service Registry** | A database/store that holds the list of available service instances and their locations |
| **Service Provider** | The service that registers itself into the registry |
| **Service Consumer** | The service that queries the registry to find a provider |
| **Health Checker** | Periodically pings services to remove dead ones from the registry |

---

## 📐 Types of Service Discovery

### 1. 🟦 Client-Side Discovery

The **client** is responsible for:
1. Querying the service registry
2. Choosing an instance (applying load balancing logic itself)
3. Making the request directly

```
Client → Query Registry → Get list of instances → Pick one → Call it
```

**Examples:** Netflix Eureka + Ribbon, Consul (client mode)

**Pros:**
- Simple architecture, no extra hop
- Client has full control over load balancing strategy

**Cons:**
- Every client must implement discovery logic
- Tight coupling between client and registry
- Must write discovery code in every language/service

---

### 2. 🟩 Server-Side Discovery

The **client** makes a request to a **load balancer or router**, which queries the registry and forwards the request.

```
Client → Load Balancer → Query Registry → Pick instance → Forward request
```

**Examples:** AWS ALB + ECS, Kubernetes Service + kube-proxy, NGINX, HAProxy

**Pros:**
- Client is completely unaware of discovery logic
- Centralized, language-agnostic
- Easy to update routing logic without touching clients

**Cons:**
- Load balancer is an extra network hop
- Load balancer can become a bottleneck/single point of failure

---

## 📋 Service Registration Patterns

### 🔵 Self-Registration
The service itself registers and deregisters from the registry on startup/shutdown.

```
Service starts → Registers itself → Service stops → Deregisters
```

- Simple to implement
- But service is responsible for managing its own registration (error-prone on crashes)

### 🟠 Third-Party Registration
A separate system (orchestrator/sidecar) watches for service changes and handles registration.

```
Orchestrator detects new container → Registers it → Container dies → Deregisters it
```

- Used in **Kubernetes** (kubelet handles registration)
- Service doesn't need discovery logic at all

---

## 🗄️ Service Registry Tools

| Tool | Description |
|---|---|
| **Consul** (HashiCorp) | Full-featured: service registry, health checks, KV store, DNS |
| **Eureka** (Netflix/Spring) | Designed for AWS; AP system (available over consistent) |
| **etcd** | Distributed key-value store; used by Kubernetes internally |
| **ZooKeeper** | Coordination service; CP system |
| **Kubernetes DNS** | Built-in DNS-based discovery for K8s services |

---

## ⚖️ What is Load Balancing?

**Load Balancing** is the process of **distributing incoming traffic across multiple server instances** to:
- Prevent any single server from being overwhelmed
- Improve availability and fault tolerance
- Maximize throughput and minimize latency

> Think of it like a **traffic cop** at a busy intersection directing cars to different lanes so no single lane gets jammed.

---

## 📐 Load Balancing Algorithms

### 1. 🔄 Round Robin
Requests are distributed **sequentially** across all instances.

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A (cycles back)
```
- Simple, works well when all servers have equal capacity
- Doesn't consider server load

---

### 2. ⚖️ Weighted Round Robin
Servers are assigned **weights** based on capacity.

```
Server A (weight 3): gets 3 requests
Server B (weight 1): gets 1 request
```
- Good when servers have different hardware specs

---

### 3. 🔗 Least Connections
New request goes to the server with the **fewest active connections**.

```
Server A: 10 active connections
Server B: 2 active connections  ← request goes here
Server C: 7 active connections
```
- Better than Round Robin for long-lived connections (WebSockets, DB connections)

---

### 4. 🌐 IP Hash / Sticky Sessions
The client's IP is hashed to always route to the **same server**.

```
Client IP: 192.168.1.5 → always maps to Server B
```
- Useful when session state is stored locally on a server
- Problem: uneven distribution if many users share an IP (NAT)

---

### 5. 📊 Least Response Time
Routes to the server with the **lowest average response time** and fewest connections.
- More sophisticated, used in advanced proxies like HAProxy

---

### 6. 🎲 Random
Picks a server at **random**.
- Simple, surprisingly effective at scale (power of two random choices is even better)

---

### 7. 🏆 Power of Two Random Choices (P2C)
Pick **2 random servers**, route to the one with fewer connections.
- Better distribution than pure random
- Used by Nginx, Envoy, AWS

---

## 🏛️ Layers of Load Balancing

### L4 — Transport Layer (TCP/UDP)
- Works at the **network level**
- Routes based on IP + port only
- Does **not** inspect packet content
- **Very fast**, low overhead
- Examples: AWS NLB, HAProxy (TCP mode)

### L7 — Application Layer (HTTP/HTTPS)
- Works at the **application level**
- Can inspect HTTP headers, URLs, cookies
- Can make **smart routing decisions** (route `/api/*` to API servers, `/static/*` to CDN)
- Examples: AWS ALB, NGINX, Envoy, Traefik

---

## 🌍 Types of Load Balancers

| Type | Description | Example |
|---|---|---|
| **Hardware** | Physical appliance, very expensive, high performance | F5, Citrix NetScaler |
| **Software** | Runs on commodity hardware | NGINX, HAProxy |
| **Cloud-managed** | Fully managed by cloud provider | AWS ALB/NLB, GCP Load Balancer |
| **DNS-based** | Uses DNS TTL to distribute traffic across IPs | AWS Route 53 |
| **Client-side** | Logic lives inside client (service mesh/library) | Netflix Ribbon, Envoy sidecar |

---

## 🔗 Service Discovery + Load Balancing Together

They work hand-in-hand:

```
New service instance starts
       ↓
Registers itself in Service Registry (Consul/Eureka/K8s)
       ↓
Load Balancer queries Registry for healthy instances
       ↓
Client request arrives at Load Balancer
       ↓
LB picks best instance (Round Robin / Least Conn etc.)
       ↓
Request forwarded to that instance
       ↓
Instance crashes → Registry marks it unhealthy → LB stops sending traffic
```

---

## ☸️ How Kubernetes Handles This

Kubernetes has **built-in** service discovery and load balancing:

- **Pod** = a running instance of a service
- **Service** = a stable virtual IP (ClusterIP) that represents a group of pods
- **kube-proxy** = watches API server, updates iptables/IPVS to route traffic to healthy pods
- **CoreDNS** = resolves service names to ClusterIPs (`my-service.default.svc.cluster.local`)
- **Ingress Controller** = L7 LB (NGINX Ingress, Traefik) for external traffic

```
Client calls "payment-service"
       ↓
CoreDNS resolves to ClusterIP (10.96.0.5)
       ↓
kube-proxy routes to one of the healthy pods
       ↓
Pod handles request
```

---

## 🛡️ Health Checks

Load balancers and registries need to know if an instance is alive:

| Type | How it works |
|---|---|
| **TCP Check** | Can we open a connection to the port? |
| **HTTP Check** | Does `GET /health` return `200 OK`? |
| **gRPC Check** | Does the gRPC health probe respond? |
| **Custom Script** | Run a script and check exit code |

**In Kubernetes:**
- `livenessProbe` → restart container if it fails
- `readinessProbe` → stop sending traffic if it fails
- `startupProbe` → give slow-starting containers extra time

---

## 🌐 Service Mesh (Advanced)

A **Service Mesh** takes service discovery + load balancing further by injecting a **sidecar proxy** (like Envoy) next to every service.

```
Service A → Envoy Sidecar → [mTLS] → Envoy Sidecar → Service B
```

**Features:**
- Automatic service discovery
- Client-side load balancing
- mTLS (mutual TLS) for security
- Observability (traces, metrics, logs)
- Circuit breaking, retries, timeouts

**Examples:** Istio, Linkerd, Consul Connect, AWS App Mesh

---

## ⚡ Circuit Breaker Pattern (related concept)

Prevents cascading failures when a downstream service is slow or down.

```
States: CLOSED → OPEN → HALF-OPEN

CLOSED: Normal operation, requests flow through
OPEN: Too many failures, requests fail immediately (fast fail)
HALF-OPEN: Test a few requests to see if service recovered
```

Works alongside load balancers — if a service keeps failing, circuit breaks so the LB doesn't keep routing to it.

---

## 🧠 Key Tradeoffs to Know

| Topic | CAP Consideration |
|---|---|
| **Eureka** | AP — prefers availability over consistency (may serve stale data) |
| **Consul** | CP (default) — consistent but may be unavailable during partition |
| **etcd / ZooKeeper** | CP — strongly consistent |

> In service discovery, most systems prefer **AP** — it's better to route to a slightly stale list of instances than to fail completely during a network partition.

---

---

# 💼 Common Interview Questions & Answers

---

### Q1. What is service discovery and why is it needed?

**Answer:**
Service discovery is a mechanism that allows services in a distributed system to automatically find and communicate with each other without hardcoded addresses. It's needed because in cloud-native/microservices environments, service instances are ephemeral — they get new IPs when they restart, scale dynamically, and are deployed across multiple hosts. Manual configuration is not feasible at this scale.

---

### Q2. What is the difference between client-side and server-side service discovery?

**Answer:**
- **Client-side:** The client queries the service registry directly and applies load balancing logic to pick an instance (e.g., Netflix Eureka + Ribbon). More control for the client, but every client must implement discovery logic.
- **Server-side:** The client sends the request to a load balancer/router, which queries the registry and forwards the request (e.g., AWS ALB, Kubernetes Services). Simpler for clients, centralized control, but adds a network hop.

---

### Q3. What is the difference between L4 and L7 load balancing?

**Answer:**
- **L4 (Transport Layer):** Routes traffic based on IP address and TCP/UDP port only. Does not inspect the content of packets. It's very fast and efficient but has no awareness of HTTP, headers, or URLs. Used when you need raw performance (e.g., AWS NLB).
- **L7 (Application Layer):** Inspects the HTTP request — headers, URLs, cookies. Can make intelligent routing decisions like routing `/api` to one server group and `/static` to another. More flexible but slightly more overhead (e.g., AWS ALB, NGINX).

---

### Q4. Explain Round Robin vs Least Connections. When would you use each?

**Answer:**
- **Round Robin** distributes requests sequentially. Best when all requests have similar processing time and all servers have equal capacity.
- **Least Connections** routes to the server with the fewest active connections. Best for workloads with variable request durations (e.g., long-lived WebSocket connections, database queries) — otherwise, one server could end up holding 100 slow requests while Round Robin keeps adding to it.

---

### Q5. How does Kubernetes handle service discovery?

**Answer:**
Kubernetes uses a combination of:
1. **CoreDNS** — every Service gets a DNS name (e.g., `my-service.default.svc.cluster.local`), which resolves to a stable ClusterIP.
2. **kube-proxy** — runs on every node and maintains iptables/IPVS rules that forward traffic from the ClusterIP to actual healthy pod IPs.
3. **Endpoints/EndpointSlices** — Kubernetes keeps a live list of healthy pod IPs behind each service. When a pod dies or fails its readiness probe, it's removed from this list automatically.

---

### Q6. What is a service registry? Name some examples.

**Answer:**
A service registry is a database that stores the network locations (IP, port) of service instances. Services register themselves on startup and deregister on shutdown. Health checks remove crashed instances automatically.

**Examples:**
- **Consul** — feature-rich, supports DNS and HTTP API, health checks, KV store
- **Eureka** — Netflix OSS, designed for AWS, AP system
- **etcd** — distributed KV store, backbone of Kubernetes
- **ZooKeeper** — CP system, used in older Kafka/Hadoop setups

---

### Q7. What is a service mesh and how does it relate to service discovery?

**Answer:**
A service mesh is an infrastructure layer that handles service-to-service communication by injecting a **sidecar proxy** (e.g., Envoy) next to each service. The sidecar handles service discovery, load balancing, retries, timeouts, circuit breaking, and mTLS — all transparently without the application knowing.

It extends service discovery by making it fully automated and adding observability and security on top. Popular implementations: **Istio**, **Linkerd**, **Consul Connect**.

---

### Q8. What happens when a load balancer sends requests to an unhealthy instance?

**Answer:**
Without health checks, the load balancer will keep routing traffic to the dead instance, causing request failures for end users. This is why **health checks** are critical:
- The LB periodically polls instances (e.g., `GET /health`)
- If an instance fails N consecutive checks, it's **removed from the rotation**
- When it recovers, it passes health checks and is **added back**

In Kubernetes, the `readinessProbe` specifically controls whether a pod receives traffic — a pod that's alive but not ready (e.g., still warming up) won't get traffic.

---

### Q9. What is sticky session / session affinity? What are its downsides?

**Answer:**
Sticky sessions (or session affinity) ensure that all requests from the **same client** always go to the **same server instance** — typically using IP hashing or a session cookie. This is useful when session state is stored locally on the server.

**Downsides:**
- Defeats the purpose of load balancing — one server can get overloaded if it has many active users
- If that server goes down, all its sessions are lost
- Doesn't work well behind NAT (many users share one IP)

**Better approach:** Store session state in a shared store (Redis, DynamoDB) and make services stateless — then any instance can serve any request.

---

### Q10. How would you design a highly available service discovery system?

**Answer:**
Key principles:
1. **Replicate the registry** — run 3 or 5 nodes with consensus (Raft/Paxos) to tolerate node failures (Consul cluster, etcd cluster)
2. **Cache registry data on clients** — so clients can still operate even if the registry is temporarily unreachable (Eureka clients cache the instance list)
3. **Health checks with appropriate intervals** — detect failures quickly without overwhelming the registry
4. **Prefer AP over CP** for service discovery — stale data is better than total unavailability
5. **Graceful deregistration** — services should deregister on shutdown; use TTL-based leases as a fallback
6. **Multiple availability zones** — spread registry nodes across AZs to survive zone failures

---

### Q11. What is the difference between self-registration and third-party registration?

**Answer:**
- **Self-registration:** The service instance registers/deregisters itself. Simpler but the service must include discovery logic, and if it crashes without deregistering, stale entries remain until TTL expires.
- **Third-party registration:** An external system (orchestrator, sidecar, or platform like Kubernetes) manages registration. The service itself has no discovery code, making it cleaner and more robust — the orchestrator detects instance lifecycle and handles registration/deregistration automatically.

---

### Q12. Explain the Circuit Breaker pattern and its relationship to load balancing.

**Answer:**
A circuit breaker wraps calls to downstream services and tracks failure rates. When failures exceed a threshold, the circuit "opens" and subsequent calls fail immediately (fast fail) without actually hitting the downstream service. After a timeout, it moves to "half-open" to test recovery.

Relationship to load balancing: Both protect against cascading failures. The load balancer removes instances from rotation based on health checks (reactive). The circuit breaker stops sending requests to a struggling service entirely when it detects a pattern of failures (proactive). Together they form a defense-in-depth approach to resilience.

---

> ✅ **Summary Mental Model:**
> - **Service Registry** = The phonebook
> - **Service Discovery** = Looking up the phonebook
> - **Load Balancer** = Traffic cop distributing load across found instances
> - **Health Checks** = Ensuring dead entries are removed from the phonebook
> - **Service Mesh** = Making all of the above completely automatic and invisible to your code