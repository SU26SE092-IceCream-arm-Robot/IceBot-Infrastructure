# IceBot Production VPS Deployment Context

This snapshot represents the currently running IceBot production deployment.

## Platform

- Single-node K3s cluster
- Namespace: icebot
- Traefik is the K3s ingress controller
- Backend currently exposes HTTP port 8080
- PostgreSQL and MinIO are private Kubernetes services
- NetBird provides the private network between Edge machines and the VPS

## Current Public Backend Path

Internet
-> api.icebot.io.vn
-> Cloudflare Tunnel
-> cloudflared pod
-> icebot-backend:8080
-> ASP.NET Core / Kestrel

Cloudflare terminates public TLS.

## Current Private Edge Path

Edge
-> NetBird
-> VPS NetBird IP
-> TCP 443
-> K3s ServiceLB
-> Traefik websecure
-> HTTP Ingress
-> icebot-backend:8080

This path is not suitable for end-to-end mTLS because Traefik terminates TLS before Kestrel.

## Known NetBird/K3s Interaction

NetBird peer ACL permits Edge peers to reach the cloud peer.

K3s ServiceLB DNATs incoming traffic and causes it to enter the Linux FORWARD path.

NetBird currently drops unmatched forwarded traffic from wt0.

A temporary FORWARD ACCEPT rule was used only to prove connectivity and must not be treated as the production solution.

## Planned Full Edge mTLS Architecture

Edge
-> NetBird
-> private IceBot hostname
-> TCP 443
-> Traefik TCP TLS passthrough
-> icebot-backend:8443
-> Kestrel HTTPS/mTLS listener

Backend should retain:
- 8080 HTTP for the existing public Cloudflare path
- 8443 HTTPS/mTLS for Full Edge transport

The Edge client certificate must reach Kestrel unchanged so the backend can use GetClientCertificateAsync() and validate the provisioned SHA-256 fingerprint.

## Important

No secrets are intentionally included in this snapshot.
Do not infer secret values from Secret references.
