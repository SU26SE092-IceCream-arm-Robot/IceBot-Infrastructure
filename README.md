# IceBot Infrastructure

Infrastructure and deployment configuration for the IceBot platform.

## Scope

This repository tracks:

- K3s deployment manifests
- Traefik ingress configuration
- Cloudflare Tunnel deployment
- PostgreSQL and MinIO manifests
- Backend deployment and migration jobs
- NetBird/K3s networking context
- Production deployment snapshots
- Private Edge mTLS architecture

## Architecture

### Public API

Internet
→ api.icebot.io.vn
→ Cloudflare Tunnel
→ cloudflared
→ icebot-backend:8080

### Private Edge

Target architecture:

Edge
→ NetBird
→ private IceBot hostname
→ Traefik TCP TLS passthrough
→ icebot-backend:8443
→ Kestrel mTLS

## Repository layout

- `k3s/` — maintained deployment manifests
- `snapshots/production/` — exported production state for debugging and review
- `docs/` — architecture and operational documentation
- `scripts/` — infrastructure utility scripts

## Security

Secrets, private keys, PFX files, kubeconfig files, and environment files must never be committed.

Kubernetes manifests should reference Kubernetes Secrets instead of embedding secret values.
