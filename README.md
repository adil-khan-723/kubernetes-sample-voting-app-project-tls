# Kubernetes Multi-Service Voting Application

A learning-focused Kubernetes project designed to understand how real distributed systems operate in a cluster environment. This isn't a production system - it's a deliberate exploration of Kubernetes primitives, networking, service discovery, and operational patterns.

## Project Goal

The goal was not to "just deploy containers," but to build a complete mental model of how Kubernetes:
- Routes traffic between services
- Handles service discovery and DNS
- Manages stateful vs stateless workloads
- Exposes applications externally through Ingress
- Terminates TLS at the edge
- Injects configuration and secrets at runtime
- Recovers from failures automatically

This project simulates distributed architecture patterns to understand the platform, not just the syntax.

## What This Application Does

A voting system where users can vote between two options. Votes are queued, processed asynchronously by a worker service, stored in a database, and displayed in real-time on a results page.

The application has five independent services:
- **Voting Frontend** - Web UI where users cast votes (stateless)
- **Results Frontend** - Web UI where users view aggregated results (stateless)
- **Redis** - Acts as a message queue between frontend and worker
- **PostgreSQL** - Persistent storage for vote results
- **Worker Service** - Background processor that consumes from Redis and writes to PostgreSQL

Each service runs in its own container and communicates through Kubernetes networking primitives.

## Why This Architecture

This mirrors real microservices patterns:
- Frontend services are stateless and horizontally scalable
- Data services are isolated with clear persistence boundaries
- Communication happens through stable service abstractions, not pod IPs
- Asynchronous processing decouples vote submission from vote storage
- External traffic enters through a single controlled ingress point

Kubernetes handles orchestration, self-healing, DNS resolution, and load balancing across all components.

## Architecture Flow

```
External User
      ↓
  (HTTPS)
      ↓
Ingress Controller (TLS Termination)
      ↓
  (HTTP - Internal)
      ↓
┌─────────────────────────────────────────────┐
│           Kubernetes Cluster                │
│                                             │
│  Voting Frontend → Redis                    │
│                      ↓                      │
│                   Worker                    │
│                      ↓                      │
│  Results Frontend ← PostgreSQL              │
│                                             │
└─────────────────────────────────────────────┘
```

**Traffic Flow:**
1. User submits vote via `vote.app.local` (HTTPS)
2. Ingress Controller terminates TLS and routes to Voting Service (HTTP)
3. Voting frontend pushes vote to Redis queue
4. Worker polls Redis, processes vote, writes to PostgreSQL
5. User views results via `result.app.local` (HTTPS)
6. Ingress Controller terminates TLS and routes to Result Service (HTTP)
7. Results frontend reads aggregated data from PostgreSQL

All internal communication uses service discovery via Kubernetes DNS. No pod IPs are used anywhere.

## Kubernetes Components Used

### Deployments

Used for all stateless workloads:
- Voting frontend
- Results frontend  
- Redis (for this learning project)
- Worker service
- PostgreSQL (initially, before exploring StatefulSets)

**Why Deployments:**
- Manage replica counts declaratively
- Enable rolling updates with zero downtime
- Provide self-healing through automatic pod recreation
- Support horizontal scaling

**Key Learning:**
Deployments are designed for stateless workloads where any pod replica is interchangeable. Pod identity doesn't matter. Kubernetes can kill and reschedule pods freely.

### StatefulSets

Explored separately to understand how Kubernetes handles stateful workloads differently.

**What StatefulSets Provide:**
- Stable, persistent pod identities (pod-0, pod-1, pod-2)
- Ordered, sequential startup and shutdown
- Stable network identifiers through headless services
- Integration with persistent volume claims for durable storage

**Key Learning:**
StatefulSets enforce ordering and identity. Pods are created in order and each pod gets its own persistent volume. DNS entries are stable and predictable. This matters for databases, distributed systems, and anything that requires stable identity or storage.

PostgreSQL was redeployed as a StatefulSet to observe the difference in pod lifecycle, DNS behavior, and volume attachment compared to a Deployment.

### Services

**ClusterIP Services** provide stable internal networking:
- `redis` - Internal access to Redis queue
- `db` - Internal access to PostgreSQL database
- `voting-service` - Backend target for voting Ingress
- `result-service` - Backend target for results Ingress

**Why Services Matter:**
Pods are ephemeral. Their IPs change constantly. Services provide a stable DNS name and virtual IP that never changes. All internal communication uses service names, which Kubernetes DNS resolves automatically.

**Key Learning:**
- ClusterIP is the default and right choice for internal communication
- Services load-balance traffic across pod replicas automatically
- Service discovery through DNS is namespace-scoped by default
- Fully qualified service names: `<service-name>.<namespace>.svc.cluster.local`

### Ingress

**Ingress Resource** defines HTTP routing rules:
- `vote.app.local` → routes to `voting-service`
- `result.app.local` → routes to `result-service`

**Ingress Controller** (Nginx) processes the rules:
- Runs as a pod inside the cluster
- Acts as a reverse proxy
- Dynamically reconfigures based on Ingress resources
- Terminates TLS at the edge
- Forwards HTTP traffic to backend Services

**Key Learning:**
Ingress resources are configuration only. The Ingress Controller does the actual work. Without a controller installed, Ingress rules do nothing. This separation of concerns is intentional - you define what you want, the controller implements it.

Multi-host routing allows multiple applications to share a single load balancer and entry point while maintaining logical separation.

### ConfigMaps

Used to inject non-sensitive configuration into containers at runtime.

**Implementation:**
- Mounted as files using volumes (not environment variables)
- Contains application settings, feature flags, environment-specific config
- Updates propagate to running pods automatically when mounted as volumes

**Key Learning:**
ConfigMaps mounted as volumes update live without pod restarts. The container sees the new file content immediately. This is different from environment variables, which are static at pod creation time.

File-based mounting is preferred over env vars because it supports live updates and doesn't leak config through process inspection or logs.

### Secrets

Used to inject sensitive data (database credentials, TLS certificates) into containers.

**Implementation:**
- Mounted as files using volumes (not environment variables)
- TLS certificates for Ingress stored as type `kubernetes.io/tls`
- Database credentials stored as Opaque secrets
- Access controlled through RBAC at service account level

**Key Learning:**
Kubernetes Secrets are not encryption at rest by default. They are a distribution and access control mechanism. The security model depends on:
- RBAC policies limiting which service accounts can read which secrets
- File-based mounting with filesystem permissions
- Avoiding environment variable injection which leaks through logs and exec sessions

Secrets must exist before pods that reference them can start. Kubernetes does not create secrets automatically.

### Init Containers

Used to enforce startup dependencies and fail fast with clear errors.

**Implementation:**
- Worker pod uses Init Container to validate database Secret exists
- Init Container checks RBAC access before main container starts
- If Secret is missing or inaccessible, Init Container fails and pod never proceeds
- Main container only starts after Init Container completes successfully

**Key Learning:**
Kubernetes schedules pods optimistically. It doesn't validate that Secrets exist or volumes can mount before starting pods. Init Containers provide a way to enforce preconditions and fail immediately with context instead of letting application containers crash in obscure ways.

Init Containers run sequentially to completion before main containers start. They're the right place for dependency validation, schema migrations, config preprocessing, or any one-time setup task.

**Key Learning:**
By default, pods run as the `default` service account which may have broad access. Production patterns require explicit service accounts with scoped permissions.

RBAC prevents pods from reading arbitrary secrets. A compromised frontend container cannot access database credentials if RBAC is configured correctly.

## What I Actually Learned

### Service Discovery is DNS-Based
Services resolve through Kubernetes DNS automatically. Pods find each other using service names like `redis` or `db`. No IP addresses are hardcoded anywhere. DNS is namespace-scoped - services in the same namespace are accessible by short name, cross-namespace requires FQDN.

### Pod IPs are Meaningless
Pods die and get recreated constantly. Their IPs change. Using pod IPs directly breaks immediately. Services abstract this away and provide stable endpoints.

### Ingress Requires Two Things
An Ingress resource (configuration) and an Ingress Controller (implementation). The resource alone does nothing. Many beginners create Ingress rules and wonder why traffic doesn't flow - they forgot to install the controller.

### TLS Terminates at the Edge
The Ingress Controller handles HTTPS. Backend services communicate over plain HTTP internally. You decrypt once, not at every hop. This is both a performance and operational simplicity decision.

### ConfigMaps and Secrets are Not the Same Thing
ConfigMaps are for non-sensitive config. Secrets are for sensitive data and have stricter RBAC implications. Both can be mounted as files or env vars, but file mounting is strongly preferred because:
- Files can be updated live without pod restarts (for ConfigMaps)
- Files don't leak through process inspection or container logs
- Filesystem permissions provide an additional access boundary

### Init Containers Enforce Dependencies
Kubernetes doesn't validate your dependencies before starting pods. Init Containers let you fail fast with clear errors instead of obscure application crashes.

### StatefulSets are for Identity and Order
Deployments treat all pods as interchangeable. StatefulSets give each pod a stable identity and persistent storage. Use StatefulSets for databases, distributed systems, or anything that requires stable network identity.

### Debugging Requires Multiple Layers
When something breaks, the problem could be:
- Application layer (code bug, wrong config)
- Network layer (service not resolving, wrong port)
- Platform layer (RBAC blocking access, secret missing)
- Infrastructure layer (node resources, scheduling constraints)

You need to think across all layers simultaneously.

## Technical Stack

- **Kubernetes** - Container orchestration platform
- **Docker** - Container runtime
- **Nginx Ingress Controller** - Reverse proxy for HTTP routing and TLS termination
- **Redis** - In-memory data store used as message queue
- **PostgreSQL** - Relational database for persistent storage
- **Python/Node.js** - Application languages for frontend and worker services

## Prerequisites

- Kubernetes cluster (minikube, kind, k3s, or cloud-managed cluster)
- kubectl installed and configured
- Ingress controller installed in the cluster (nginx-ingress recommended)
- Basic understanding of containers and YAML

## Project Structure

```
.
├── README.md
├── namespace.yaml              # Namespace definition
├── configMap.yaml              # ConfigMap for app configuration
├── secrets.yaml                # Secrets for sensitive data
├── tls-secret.yaml            # TLS certificates for HTTPS
├── oggy.crt                   # TLS certificate file
├── oggy.key                   # TLS private key file
├── deployment-postgres.yaml   # PostgreSQL Deployment
├── deployment-redis.yaml      # Redis Deployment
├── deployment-result.yaml     # Results Frontend Deployment
├── deployment-voting.yaml     # Voting Frontend Deployment
├── deployment-worker.yaml     # Worker Deployment
├── service-postgres.yaml      # PostgreSQL Service
├── service-redis.yaml         # Redis Service
├── service-results.yaml       # Results Service
├── service-voting.yaml        # Voting Service
└── ingress.yaml               # Ingress with TLS configuration
```

## Setup Instructions

### 1. Clone the Repository

```bash
git clone https://github.com/yourusername/kubernetes-voting-app.git
cd kubernetes-voting-app
```

### 2. Verify Cluster Access

```bash
kubectl cluster-info
kubectl get nodes
```

### 3. Install Ingress Controller (if not already installed)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```

Wait for the Ingress Controller to be ready:

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

### 4. Create Secrets and ConfigMaps

```bash
kubectl apply -f app-secrets.yaml
kubectl apply -f app-config.yaml
```

### 5. Deploy All Services

```bash
kubectl apply -f service-db.yaml
kubectl apply -f service-redis.yaml
kubectl apply -f service-voting.yaml
kubectl apply -f service-result.yaml
```

### 6. Deploy All Applications

```bash
kubectl apply -f deployment-db.yaml
kubectl apply -f deployment-redis.yaml
kubectl apply -f deployment-worker.yaml
kubectl apply -f deployment-voting.yaml
kubectl apply -f deployment-result.yaml
```

### 7. Configure Ingress

```bash
kubectl apply -f ingress-rules.yaml
```

### 8. Verify Deployments

```bash
kubectl get pods
kubectl get services
kubectl get ingress
```

All pods should be in `Running` state. This may take a few minutes.

## Local Access

For local Kubernetes clusters, you'll need to expose the Ingress Controller and configure hostname resolution.

### Expose Ingress Controller

```bash
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80 8443:443
```

### Configure Hostname Resolution

Add entries to `/etc/hosts` (Linux/Mac) or `C:\Windows\System32\drivers\etc\hosts` (Windows):

```
127.0.0.1 vote.app.local
127.0.0.1 result.app.local
```

### Access the Application

- **Voting Interface:** http://vote.app.local:8080 or https://vote.app.local:8443
- **Results Interface:** http://result.app.local:8080 or https://result.app.local:8443

## Debugging Commands

### Check Pod Status and Logs

```bash
# List all pods
kubectl get pods

# Describe a pod for detailed status
kubectl describe pod <pod-name>

# View logs
kubectl logs <pod-name>

# View logs from Init Container
kubectl logs <pod-name> -c <init-container-name>

# Follow logs in real-time
kubectl logs -f <pod-name>
```

### Test Service Discovery

```bash
# Exec into a pod
kubectl exec -it <pod-name> -- /bin/sh

# Test DNS resolution
nslookup redis
nslookup db

# Test connectivity
curl http://voting-service
```

### Check Ingress Configuration

```bash
# Describe ingress
kubectl describe ingress

# Check Ingress Controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

## Common Issues and Solutions

### Pods Stuck in Pending
**Problem:** Pods show `Pending` status  
**Solution:** Check node resources, scheduling constraints, and pod events with `kubectl describe pod <pod-name>`

### Ingress Not Working
**Problem:** Ingress created but traffic not reaching services  
**Solution:** Verify Ingress Controller is installed and running. Check controller logs for errors.

### Init Container Failures
**Problem:** Pod stuck in `Init:Error` or `Init:CrashLoopBackOff`  
**Solution:** Check Init Container logs with `kubectl logs <pod-name> -c <init-container-name>`. Usually means a Secret or dependency is missing.

### Service Name Not Resolving
**Problem:** Pods cannot reach other services by name  
**Solution:** Verify services exist in the same namespace. Use FQDN for cross-namespace access: `<service>.<namespace>.svc.cluster.local`

### Secrets Not Found
**Problem:** Pods fail to start with secret-related errors  
**Solution:** Secrets must exist before pods that reference them. Create secrets first: `kubectl apply -f app-secrets.yaml`

### TLS Certificate Issues
**Problem:** HTTPS not working or certificate errors  
**Solution:** Verify TLS secret is created and referenced correctly in Ingress. Check Ingress Controller logs for TLS-related errors.

## Cleanup

Remove all resources:

```bash
kubectl delete -f .
```

Remove Ingress Controller (if desired):

```bash
kubectl delete namespace ingress-nginx
```

## Key Takeaways

1. **Kubernetes is primitives, not magic** - It gives you building blocks. You connect them intentionally.

2. **Services are the networking abstraction** - Never use pod IPs. Always use service names.

3. **Ingress needs a controller** - The resource alone does nothing. Install the controller first.

4. **File-based config beats environment variables** - Especially for secrets. Env vars leak everywhere.

5. **Init Containers enforce dependencies** - Use them to fail fast with context instead of obscure crashes.

6. **StatefulSets for state, Deployments for stateless** - They have different lifecycle guarantees.

7. **RBAC is manual** - Kubernetes doesn't enforce least privilege by default. You configure it explicitly.

8. **TLS terminates at ingress** - Internal traffic stays HTTP for simplicity.

9. **Debugging is multi-layered** - Application, network, platform, and infrastructure all matter.

10. **Learning by breaking is faster than learning by reading** - Deploy it, break it, fix it. That's where understanding comes from.

## Related

Full writeup on dev.to: [I Built a Multi-Service Kubernetes App and Here's What Actually Broke](https://dev.to/adil-khan-723/i-built-a-multi-service-kubernetes-app-and-heres-what-actually-broke-4f99)
