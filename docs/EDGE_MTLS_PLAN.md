# IceBot Edge mTLS Integration Plan

## Status

This document summarizes the Cloud/Infrastructure-side interpretation of the Edge team's initial mTLS requirements and defines the intended integration direction for the current IceBot K3s production environment.

It should be read together with:

- `README.md`
- `docs/AGENT_CONTEXT.md`
- `k3s/`
- `snapshots/production/`

The manifests under `k3s/` represent the maintained infrastructure configuration, while `snapshots/production/` represents observed production state.

---

## 1. Edge Team Requirements Reviewed

The Edge implementation expects a dedicated HTTPS/mTLS listener on the Backend.

The proposed production configuration is conceptually:

```text
ExecutionEndpointTransport__MutualTlsListener__Required=true
Kestrel__Endpoints__EdgeMtls__Url=https://+:8443
Kestrel__Endpoints__EdgeMtls__Certificate__Path=/https/edge-api.pfx
Kestrel__Endpoints__EdgeMtls__Certificate__Password=<secret>
```

The server certificate must:

- contain a private key;
- be valid and unexpired;
- support Server Authentication;
- contain a SAN matching the hostname used by Edge;
- chain to a CA trusted by the Windows Edge machine.

The Edge client certificate must reach ASP.NET Core/Kestrel so that the Backend can obtain it through:

```csharp
await context.Connection.GetClientCertificateAsync()
```

The Backend then validates the SHA-256 certificate fingerprint against the credential provisioned for the Execution Endpoint.

These requirements are accepted as the basis for the Cloud-side mTLS design.

---

## 2. Current Production Architecture

The current production Backend runs inside a single-node K3s cluster.

The existing public API path is:

```text
Internet
    |
    v
api.icebot.io.vn
    |
    v
Cloudflare Tunnel
    |
    v
cloudflared
    |
    v
icebot-backend:8080
    |
    v
ASP.NET Core / Kestrel HTTP
```

This path should remain unchanged.

Port `8080` therefore remains the normal HTTP listener used by the existing Cloudflare/public API path.

---

## 3. Current Private Edge Path Limitation

The currently tested private path is:

```text
Edge
    |
    | NetBird
    v
VPS NetBird interface
100.81.247.172:443
    |
    v
K3s ServiceLB
    |
    v
Traefik websecure
    |
    | TLS terminated by Traefik
    v
HTTP Ingress
    |
    v
icebot-backend:8080
```

This path is sufficient for normal HTTPS proxying but is not suitable for the Full Edge mTLS contract.

When Traefik terminates TLS, the original Edge client certificate does not reach Kestrel.

Forwarded headers can restore the original HTTPS scheme, but they do not preserve the original TLS client certificate.

Therefore:

```text
X-Forwarded-Proto
```

solves reverse-proxy HTTPS awareness, but does not solve mTLS client-certificate propagation.

---

## 4. Target Architecture

The preferred architecture is to keep the public API and Full Edge transport as two separate paths.

### Public API

```text
Internet
    |
    v
api.icebot.io.vn
    |
    v
Cloudflare Tunnel
    |
    v
icebot-backend:8080
```

### Private Full Edge mTLS

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
    +--> Server certificate
    |
    +--> GetClientCertificateAsync()
    |
    +--> SHA-256 fingerprint validation
```

Traefik must not terminate TLS on this route.

TLS must terminate at Kestrel.

---

## 5. Backend Listener Model

The Backend should expose two listeners inside the pod:

```text
8080
HTTP
Existing public/backend traffic

8443
HTTPS + mTLS
Full Edge transport
```

Conceptually:

```text
IceBot Backend Pod
|
+-- :8080  HTTP
|          Public API / Cloudflare path
|
+-- :8443  HTTPS
           Full Edge mTLS path
```

Port `8080` must not be removed when introducing `8443`.

---

## 6. Kubernetes Changes Expected

The Docker-specific instructions from the Edge note must be adapted to K3s.

The intended K3s changes are:

### Backend Deployment

Add:

- container port `8443`;
- Kestrel Edge mTLS configuration;
- server certificate volume mount;
- certificate password from a Kubernetes Secret.

The PFX and its password must never be committed to this repository.

### Backend Service

Keep the Service internal and add an additional port:

```text
8080 -> backend HTTP
8443 -> backend HTTPS/mTLS
```

The Backend Service should remain private to the cluster.

### Traefik

The current private HTTP Ingress must not be used as the final Full Edge mTLS route.

The target route should use a Traefik TCP router / `IngressRouteTCP` with TLS passthrough:

```text
TCP :443
HostSNI(private-hostname)
        |
        v
tls.passthrough = true
        |
        v
icebot-backend:8443
```

The Edge client certificate must remain inside the original TLS session until it reaches Kestrel.

---

## 7. Private Hostname and DNS

The preferred private hostname is:

```text
api.internal.icebot.io.vn
```

The intended separation is:

```text
api.icebot.io.vn
    -> public API

api.internal.icebot.io.vn
    -> private Edge API over NetBird
```

For Edge clients, the private hostname should resolve to:

```text
100.81.247.172
```

The Kestrel server certificate must contain:

```text
DNS:api.internal.icebot.io.vn
```

in its SAN.

The certificate chain must be trusted by Windows Edge clients.

The private hostname should not route Edge traffic through Cloudflare.

---

## 8. Server and Client Certificates

Two different certificates participate in mTLS.

### Backend Server Certificate

Owned by the Backend/Kestrel side.

Purpose:

```text
Edge verifies that it is talking to the real IceBot Backend.
```

Expected hostname:

```text
api.internal.icebot.io.vn
```

### Edge Client Certificate

Owned by each Full Edge runtime.

Purpose:

```text
Backend verifies the identity of the Execution Endpoint.
```

The Backend must receive this certificate directly from the TLS connection and compare its SHA-256 fingerprint with the fingerprint stored for the provisioned Execution Endpoint.

Traefik must not replace this certificate or terminate this TLS session.

---

## 9. NetBird and K3s Networking Issue

A separate networking issue has already been identified during private connectivity testing.

Traffic successfully travels:

```text
Edge
-> NetBird
-> VPS wt0
```

However, K3s ServiceLB causes incoming traffic to enter the Linux `FORWARD` path.

NetBird currently installs a default DROP rule for unmatched forwarded traffic from `wt0`.

During testing, a temporary FORWARD ACCEPT rule was introduced and successfully proved that the remaining TCP path to Traefik works.

That rule is diagnostic only.

It must not be considered the production solution.

The permanent NetBird-to-K3s forwarding design still needs to be finalized independently of the mTLS listener work.

Therefore there are two separate infrastructure concerns:

```text
A. NetBird -> K3s forwarding
B. End-to-end Edge -> Kestrel mTLS
```

Kestrel `8443` and Traefik TLS passthrough solve concern B, but do not automatically solve concern A.

---

## 10. NetBird Port Exposure

With the preferred architecture:

```text
Edge
-> NetBird
-> VPS TCP 443
-> Traefik
-> backend:8443
```

the external NetBird access policy only needs to expose:

```text
icebot-edge
    ->
icebot-cloud
TCP 443
```

Port `8443` remains internal to Kubernetes between Traefik and the Backend Service.

Direct Edge access to VPS port `8443` is therefore not required by the preferred architecture.

---

## 11. Lua Deployment

Lua deployment should be tested only after mTLS connectivity is operational.

After authentication succeeds, the Backend is expected to provide:

- a valid Configuration Release;
- the Lua artifact in object storage;
- correct checksum and content length;
- a `DeployConfiguration` command for the target Execution Endpoint;
- an artifact/download URL reachable by Edge.

Expected high-level sequence:

```text
mTLS connection
    |
    v
Edge authentication
    |
    v
Heartbeat / command pull
    |
    v
DeployConfiguration
    |
    v
Artifact metadata
    |
    v
Lua download
    |
    v
Checksum verification
    |
    v
Release activation
```

Lua/artifact issues should not be debugged before the transport and authentication layers are confirmed operational.

---

## 12. Implementation Order

The intended implementation order is:

```text
1. Confirm Backend mTLS implementation and configuration contract
2. Add Kestrel HTTPS/mTLS listener on 8443
3. Add Backend Service port 8443
4. Provision the Backend server certificate through Kubernetes Secrets
5. Add Traefik TCP TLS-passthrough route
6. Configure private DNS / hostname
7. Finalize the NetBird-to-K3s forwarding solution
8. Verify TLS server certificate from Edge
9. Verify Edge client certificate reaches Kestrel
10. Verify provisioned SHA-256 fingerprint validation
11. Verify Edge heartbeat / command pull
12. Test DeployConfiguration and Lua artifact download
```

---

## 13. Decisions

The current Cloud/Infrastructure direction is therefore:

- Keep the existing Cloudflare + Backend `8080` public path.
- Add a dedicated Kestrel HTTPS/mTLS listener on `8443`.
- Keep `8443` internal to the K3s cluster where possible.
- Route private Edge traffic through NetBird.
- Use Traefik TCP TLS passthrough instead of HTTP TLS termination.
- Use a private hostname separate from the public API hostname.
- Ensure the Edge client certificate reaches Kestrel unchanged.
- Store certificate material and passwords only in Kubernetes Secrets.
- Do not commit PFX files, private keys, or passwords.
- Do not treat the temporary NetBird FORWARD rule as a production solution.
- Do not begin Lua deployment debugging until mTLS transport is verified.

This plan may be refined after the Backend and Edge agents review the actual infrastructure manifests and current production snapshots in this repository.
