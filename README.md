# IceBot Infrastructure

Infrastructure, deployment manifests, production snapshots, and operational guidance for the IceBot platform.

This repository is intended to serve as the **infrastructure source of truth** for the IceBot cloud deployment and as shared context for Backend, Edge, DevOps, and AI agents working on the project.

> **Important:** Files under `k3s/` represent maintained/desired infrastructure configuration. Files under `snapshots/production/` represent observed production state and must not be treated as deployable source-of-truth manifests.

---

## 1. Scope

This repository tracks the infrastructure required to run the IceBot cloud backend, including:

- Single-node K3s deployment
- ASP.NET Core IceBot Backend deployment
- PostgreSQL
- MinIO object storage
- EF Core database migration jobs
- Traefik ingress and TCP routing
- Cloudflare Tunnel
- NetBird private networking context
- Production networking snapshots
- Edge mTLS integration planning
- Operational and troubleshooting documentation

Application source code remains in the dedicated Backend repository.

---

## 2. Current Production Platform

The current production environment is a single VPS running K3s.

### Cloud host

- Linux VPS
- Single-node K3s cluster
- K3s namespace: `icebot`
- Traefik installed as the K3s ingress controller
- K3s ServiceLB enabled
- Flannel/K3s pod networking
- NetBird installed on the VPS for private Edge connectivity

### Main workloads

- `icebot-backend`
- `postgres`
- `minio`
- `cloudflared`
- database migration Jobs

### Backend runtime

The IceBot Backend is an ASP.NET Core application.

Current listener model:

```text
8080
HTTP
Existing public/backend traffic
```

Planned Full Edge listener:

```text
8443
HTTPS + mTLS
Private Full Edge transport
```

---

## 3. Architecture

### 3.1 Public API

The existing public API path is:

```text
Internet
    |
    v
api.icebot.io.vn
    |
    v
Cloudflare
    |
    v
Cloudflare Tunnel
    |
    v
cloudflared pod
    |
    v
icebot-backend:8080
    |
    v
ASP.NET Core / Kestrel
```

Cloudflare terminates public TLS.

The public API path should continue using the Backend HTTP listener on port `8080`.

---

### 3.2 Private Edge transport

The target Full Edge architecture is:

```text
Windows Edge
    |
    | NetBird
    v
Private IceBot hostname
    |
    | TCP 443
    v
Traefik
    |
    | TCP routing only
    | TLS PASSTHROUGH
    v
icebot-backend:8443
    |
    v
Kestrel HTTPS / mTLS
    |
    +--> server certificate
    |
    +--> Edge client certificate
    |
    +--> SHA-256 fingerprint validation
```

The planned private hostname is:

```text
api.internal.icebot.io.vn
```

The Edge-side DNS entry should resolve this hostname to the VPS NetBird address.

The TLS session must terminate at Kestrel so the Backend can receive the original Edge client certificate.

See:

```text
docs/EDGE_MTLS_PLAN.md
```

for the detailed mTLS integration plan.

---

## 4. Repository Layout

```text
IceBot-Infrastructure/
├── README.md
│
├── docs/
│   ├── AGENT_CONTEXT.md
│   └── EDGE_MTLS_PLAN.md
│
├── k3s/
│   ├── backend/
│   │   ├── deployment.yaml
│   │   ├── keyring-pvc.yaml
│   │   └── service.yaml
│   │
│   ├── cloudflared/
│   │   └── deployment.yaml
│   │
│   ├── ingress/
│   │   └── backend-private-ingress.yaml
│   │
│   ├── migration/
│   │   └── job.yaml
│   │
│   ├── minio/
│   │   ├── service.yaml
│   │   └── statefulset.yaml
│   │
│   └── postgres/
│       ├── service.yaml
│       └── statefulset.yaml
│
└── snapshots/
    └── production/
        ├── live/
        └── system/
```

Future infrastructure may add:

```text
k3s/ingress/backend-edge-mtls-tcp.yaml
scripts/snapshot-production.sh
docs/NETWORKING.md
docs/OPERATIONS.md
```

---

## 5. Desired State vs Observed State

This distinction is critical.

### `k3s/`

This directory contains the infrastructure that is intentionally maintained.

Treat this directory as:

```text
DESIRED STATE
```

Changes intended for production should normally be made here first, reviewed, committed, and then applied to the cluster.

Examples:

```text
k3s/backend/deployment.yaml
k3s/backend/service.yaml
k3s/ingress/*
```

---

### `snapshots/production/live/`

This directory contains Kubernetes resources exported from the running cluster.

Examples include:

- Deployments
- Services
- Ingresses
- Traefik resources
- ServiceLB pods

These files may contain runtime-generated metadata such as:

- UIDs
- resource versions
- generated annotations
- creation timestamps
- status blocks

They are provided for debugging, review, and agent context.

Do **not** blindly apply snapshot YAML back to the cluster.

---

### `snapshots/production/system/`

This directory contains non-Kubernetes production observations such as:

- NetBird status
- Linux network interface state
- iptables state
- K3s configuration
- Service descriptions
- Pod summaries
- EndpointSlice summaries
- deployed Backend image information

These files exist primarily for diagnostics and architecture review.

---

## 6. Core Dependencies

### 6.1 K3s

K3s provides the Kubernetes control plane and runtime for the VPS.

Responsibilities:

- workload orchestration
- Services
- Deployments
- StatefulSets
- Jobs
- Secrets
- storage integration
- Traefik integration
- ServiceLB

---

### 6.2 Traefik

Traefik is the K3s ingress controller.

Current responsibilities:

- Kubernetes HTTP ingress
- public/private HTTP routing where applicable
- `websecure` entrypoint on TCP 443

Planned responsibility for Full Edge:

```text
TCP 443
-> HostSNI(private hostname)
-> TLS passthrough
-> icebot-backend:8443
```

For Full Edge mTLS, Traefik must **not terminate TLS**.

---

### 6.3 Cloudflare Tunnel

Cloudflare Tunnel provides public connectivity to the Backend without exposing the Backend service directly.

Current route:

```text
api.icebot.io.vn
-> Cloudflare Tunnel
-> cloudflared
-> icebot-backend:8080
```

Cloudflare Tunnel is independent from the private Full Edge mTLS path.

---

### 6.4 NetBird

NetBird provides the private network between Edge machines and the IceBot cloud VPS.

The expected external policy for the preferred Full Edge architecture is:

```text
icebot-edge
    ->
icebot-cloud

Protocol: TCP
Port: 443
```

Port `8443` is intended to remain internal to Kubernetes when using Traefik TCP passthrough.

---

### 6.5 PostgreSQL

PostgreSQL stores IceBot application state.

It must remain private to the cluster.

Do not expose PostgreSQL directly to the public Internet.

---

### 6.6 MinIO

MinIO provides object storage used by Backend workflows such as robot/Lua artifacts.

It must remain private unless a specific controlled download architecture is intentionally introduced.

---

### 6.7 ASP.NET Core / Kestrel

The Backend currently uses Kestrel on port `8080`.

The planned Full Edge mTLS listener uses:

```text
https://+:8443
```

Expected production configuration includes:

```text
ExecutionEndpointTransport__MutualTlsListener__Required=true
Kestrel__Endpoints__EdgeMtls__Url=https://+:8443
Kestrel__Endpoints__EdgeMtls__Certificate__Path=/https/edge-api.pfx
Kestrel__Endpoints__EdgeMtls__Certificate__Password=<secret>
```

The actual secret value must never be committed.

---

## 7. Reverse Proxy and Forwarded Headers

The public Backend path terminates TLS before requests reach Kestrel.

ASP.NET Core therefore needs trusted reverse-proxy configuration for:

```text
X-Forwarded-For
X-Forwarded-Proto
```

Production must only trust explicitly configured reverse proxy networks.

The current default trusted K3s network is:

```text
10.42.0.0/16
```

This can be overridden through application configuration where required.

Forwarded headers restore the original HTTPS scheme for proxied HTTP requests.

They do **not** preserve the original TLS client certificate.

This is why Full Edge mTLS requires TLS passthrough to Kestrel.

---

## 8. Edge mTLS Design

Full Edge mTLS requires two different certificates.

### Backend server certificate

Purpose:

```text
Edge verifies that it is communicating with the IceBot Backend.
```

Requirements:

- private key present
- valid and unexpired
- Server Authentication EKU
- SAN matches the private Edge hostname
- certificate chain trusted by Windows Edge machines

Target hostname:

```text
api.internal.icebot.io.vn
```

---

### Edge client certificate

Purpose:

```text
Backend verifies the identity of the provisioned Execution Endpoint.
```

The Backend must be able to call:

```csharp
await context.Connection.GetClientCertificateAsync()
```

and compare the SHA-256 fingerprint against the fingerprint provisioned for the Execution Endpoint.

The client certificate must therefore reach Kestrel inside the original TLS session.

See `docs/EDGE_MTLS_PLAN.md` for the implementation plan and current decisions.

---

## 9. Known NetBird / K3s Forwarding Issue

Private connectivity testing identified an interaction between NetBird and K3s ServiceLB.

Observed path:

```text
Edge
-> NetBird
-> VPS wt0
-> K3s ServiceLB / DNAT
-> Linux FORWARD chain
```

NetBird currently installs a DROP rule for unmatched forwarded traffic from `wt0`.

During troubleshooting, a temporary FORWARD ACCEPT rule was introduced and successfully proved that the remaining path to Traefik works.

That rule is **diagnostic only**.

It must not be treated as the final production solution.

The permanent solution for:

```text
NetBird -> K3s forwarding
```

must be finalized separately from the Kestrel mTLS listener.

---

## 10. Security Guidelines

### Never commit secrets

The following must never be committed:

```text
.env
.env.*
*.pfx
*.p12
*.pem
*.key
private certificates
private keys
Kubernetes Secret objects containing real values
kubeconfig
k3s.yaml containing credentials
Cloudflare Tunnel tokens
NetBird setup keys
SSH private keys
database passwords
MinIO passwords
PayOS credentials
email/SMTP passwords
certificate passwords
GitHub tokens
```

Use Kubernetes Secrets or external secret management instead.

---

### Secret references are allowed

References such as:

```yaml
env:
  - name: SOME_PASSWORD
    valueFrom:
      secretKeyRef:
        name: icebot-env
        key: SOME_PASSWORD
```

are safe to commit because no secret value is embedded.

---

### Do not hardcode production credentials

Bad:

```yaml
env:
  - name: PASSWORD
    value: actual-secret
```

Good:

```yaml
env:
  - name: PASSWORD
    valueFrom:
      secretKeyRef:
        name: icebot-env
        key: PASSWORD
```

---

### Certificates

PFX files and private keys must remain outside Git.

The repository may contain:

- certificate requirements
- placeholder manifests
- example Secret references
- certificate automation configuration without private material

---

## 11. Infrastructure Change Guidelines

### Before making changes

1. Review `docs/AGENT_CONTEXT.md`.
2. Review `docs/EDGE_MTLS_PLAN.md` for Edge/mTLS work.
3. Review the maintained manifest under `k3s/`.
4. Compare against the latest production snapshot if the change affects a running service.
5. Check whether the change affects the public API, private Edge path, or both.

---

### Prefer declarative changes

Production changes should preferably be represented in Git before they become permanent.

Preferred workflow:

```text
Edit maintained manifest
-> review diff
-> commit
-> apply to K3s
-> verify
-> refresh production snapshot
```

Avoid long-term infrastructure changes that exist only as ad-hoc shell commands.

---

### Temporary diagnostic commands

Temporary commands are acceptable during troubleshooting, but must be documented and removed or replaced by declarative configuration afterward.

Examples:

- temporary iptables ACCEPT rules
- temporary port forwarding
- debug-only Kubernetes resources
- `curl -k` certificate bypasses

Do not silently treat diagnostic state as production configuration.

---

## 12. Kubernetes Guidelines

### Namespace

Application workloads should remain in:

```text
icebot
```

K3s system components such as Traefik remain in:

```text
kube-system
```

---

### Services

Prefer private `ClusterIP` Services for internal components.

Do not expose:

```text
PostgreSQL 5432
MinIO 9000/9001
Backend 8080
Backend 8443
K3s API 6443
```

directly to the public Internet unless explicitly required and reviewed.

---

### Backend ports

Target Backend Service model:

```text
8080 -> normal HTTP Backend traffic
8443 -> private HTTPS/mTLS Edge traffic
```

`8443` should remain internal to the cluster in the preferred architecture.

---

### Secrets

Use `secretKeyRef`, Secret-backed volumes, or equivalent Kubernetes Secret mechanisms.

Do not place secret values directly in committed manifests.

---

### Persistent data

PostgreSQL, MinIO, and Backend key material requiring persistence must use persistent storage where appropriate.

Do not delete PVCs or StatefulSets casually.

---

## 13. Operational Guidelines

### Verify cluster state

Useful checks include:

```text
kubectl get pods -n icebot
kubectl get svc -n icebot
kubectl get ingress -n icebot
kubectl get pods -n kube-system
kubectl get svc -n kube-system
```

### Verify Backend logs

```text
kubectl logs -f deployment/icebot-backend -n icebot
```

### Verify Traefik

```text
kubectl logs -f deployment/traefik -n kube-system
```

### Verify NetBird

```text
netbird status -d
```

### Verify current network policy behavior

Inspect:

```text
iptables INPUT
iptables FORWARD
NETBIRD-ACL-INPUT
```

Do not permanently edit NetBird-managed chains without understanding the impact.

---

## 14. Production Snapshot Guidelines

Production snapshots should be refreshed when meaningful infrastructure changes are completed.

Snapshot data should capture enough state for:

- debugging
- review
- agent context
- post-change comparison

Snapshots should **not** contain real secret values.

Before committing refreshed snapshots, scan for:

```text
password
token
secret
api key
private key
connection string
certificate material
```

Generated Kubernetes metadata is acceptable in snapshots because snapshots are observational, not desired-state manifests.

---

## 15. Git Workflow

Infrastructure changes should be committed with clear intent.

Suggested commit styles:

```text
Add Edge mTLS infrastructure plan
Add Backend mTLS service port
Add Traefik Edge TLS passthrough route
Document NetBird K3s forwarding behavior
Update production infrastructure snapshot
```

Avoid commits that mix unrelated infrastructure changes.

Before committing:

```text
git status
git diff
git diff --cached
```

Review all files for accidental credentials.

---

## 16. Guidance for AI Agents

Agents reviewing this repository must follow these rules.

### Source-of-truth hierarchy

Use:

```text
1. k3s/
2. docs/
3. snapshots/production/
```

for different purposes.

Interpretation:

```text
k3s/
= intended infrastructure configuration

docs/
= architecture decisions, constraints, and implementation plans

snapshots/production/
= observed runtime state
```

Do not infer that a generated snapshot is the intended desired state.

---

### Do not assume Docker Compose

The production Backend runs in K3s.

Docker-style examples such as:

```text
8443:8443
host-path:/https/file.pfx
```

must be translated into Kubernetes concepts:

- container ports
- Services
- Secrets
- volumes
- Traefik routing
- Kubernetes networking

---

### Do not redesign working public infrastructure unnecessarily

The existing public path:

```text
Cloudflare Tunnel
-> backend:8080
```

is intentional and should remain unless a separate requirement justifies changing it.

Full Edge mTLS is a parallel private transport path.

---

### Treat mTLS and network forwarding as separate concerns

There are two independent problems:

```text
A. NetBird -> K3s forwarding
B. Edge -> Kestrel end-to-end mTLS
```

Do not assume solving one automatically solves the other.

---

### Validate assumptions against the manifests

Before proposing changes, agents should inspect:

```text
k3s/backend/
k3s/ingress/
snapshots/production/live/
snapshots/production/system/
docs/EDGE_MTLS_PLAN.md
```

and identify mismatches between desired state and production observations.

---

## 17. Planned Infrastructure Work

Current planned work includes:

- Add Backend Kestrel HTTPS/mTLS listener on `8443`
- Add Backend Service port `8443`
- Provision Backend server certificate through Kubernetes Secrets
- Add Traefik `IngressRouteTCP`
- Enable TLS passthrough for Full Edge
- Configure private Edge DNS
- Finalize permanent NetBird-to-K3s forwarding
- Verify Edge client certificate reaches Kestrel
- Verify provisioned SHA-256 certificate fingerprint
- Verify Edge heartbeat and command pull
- Test Configuration Deployment and Lua artifact download
- Add repeatable production snapshot script

---

## 18. Lua / Robot Artifact Flow

Lua deployment is downstream from transport/authentication.

Do not begin by debugging artifact deployment while mTLS is still failing.

Expected order:

```text
mTLS connectivity
-> Edge identity validation
-> heartbeat
-> command pull
-> DeployConfiguration
-> artifact metadata
-> Lua download
-> checksum verification
-> activation
```

The Backend must eventually provide:

- valid Configuration Release
- artifact stored in object storage
- checksum
- content length
- deployment command
- download path reachable by Edge

---

## 19. Documentation

Important documents:

### `docs/AGENT_CONTEXT.md`

Describes the observed production environment and provides shared context for reviewers and agents.

### `docs/EDGE_MTLS_PLAN.md`

Defines the current Cloud/Infrastructure interpretation of the Full Edge mTLS requirements and the target architecture.

Both documents should be reviewed before making Edge transport changes.

---

## 20. Core Principles

The Infrastructure repository follows these principles:

1. **Git tracks infrastructure intent.**
2. **Snapshots document production reality.**
3. **Secrets never belong in Git.**
4. **Public and private transport paths remain separated.**
5. **Full Edge mTLS terminates at Kestrel.**
6. **Traefik only performs TCP TLS passthrough for the Full Edge route.**
7. **Port 8080 remains available for the existing public Backend path.**
8. **Port 8443 is reserved for private Kestrel HTTPS/mTLS.**
9. **NetBird is the private Edge network boundary.**
10. **Temporary troubleshooting rules are not production architecture.**
11. **Infrastructure changes should become declarative and reviewable.**
12. **Agents must validate against actual manifests instead of assuming topology.**

---

## 21. Related Repository

Backend application repository:

```text
SU26SE092-IceCream-arm-Robot/IceBot-Backend
```

This infrastructure repository should be reviewed together with the Backend implementation when changes involve:

- Kestrel configuration
- reverse proxy trust
- mTLS
- Execution Endpoint authentication
- artifact delivery
- database migrations
- application environment variables

---

## 22. Status

The public Backend deployment is operational.

The Full Edge private transport is currently being evolved toward:

```text
NetBird
-> Traefik TCP 443
-> TLS passthrough
-> Kestrel HTTPS/mTLS 8443
```

The mTLS architecture is planned but should not be considered complete until:

- Kestrel `8443` is active
- the server certificate is trusted by Edge
- the Edge client certificate reaches Kestrel
- fingerprint validation succeeds
- NetBird/K3s forwarding is production-safe
- heartbeat and command pull succeed
