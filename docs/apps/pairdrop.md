---
layout: default
title: PairDrop
parent: Home Lab Kubernetes
---

# PairDrop

**Local file sharing** — AirDrop alternative for cross-platform networks.

---

## Overview

PairDrop is a self-hosted Progressive Web App (PWA) for local file sharing. It uses WebRTC for peer-to-peer transfers and works across:

- Windows, macOS, Linux
- iOS, Android
- Any device with a browser

**No accounts, no cloud uploads** — files transfer directly between devices on your LAN.

---

## Architecture

```yaml
Namespace: pairdrop-space
Replicas: 1
Image: lscr.io/linuxserver/pairdrop:latest
Port: 3000 (NodePort 30000)
Storage: None (stateless)
```

### Security Context
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
```

Runs as non-root user (UID 1000) — minimal attack surface.

---

## Manifest Structure

```
Apps/pairdrop/base/
├── kustomization.yaml
├── namespace.yaml
├── deployment.yaml
└── service.yaml
```

### Key Files

**deployment.yaml:**
```yaml
containers:
- name: pairdrop
  image: lscr.io/linuxserver/pairdrop:latest
  env:
  - name: PUID
    value: "1000"
  - name: PGID
    value: "1000"
  - name: TZ
    value: "America/New_York"
  ports:
  - containerPort: 3000
```

**service.yaml:**
```yaml
type: NodePort
ports:
- port: 3000
  nodePort: 30000
```

---

## Deployment

```bash
# Apply via kubectl
kubectl apply -k Apps/pairdrop/base/

# Or via ArgoCD
argocd app sync pairdrop
```

### Access

After deploy, access at:
- `http://<node-ip>:30000`

All devices on your LAN can visit this URL to share files.

---

## Why This Design?

### Decisions

| Decision | Rationale |
|----------|-----------|
| **NodePort** | Simple, no ingress needed for LAN-only service |
| **Stateless** | No PVC — app is ephemeral, no data to persist |
| **Non-root** | Security best practice, reduces container escape risk |
| **LinuxServer image** | Actively maintained, handles permissions automatically |

### Trade-offs

- **Pro:** Zero maintenance, just works
- **Con:** NodePort exposes to entire network (acceptable for LAN)

---

## Lessons Learned

**What worked:** PairDrop is production-ready — minimal config, reliable transfers.

**What I'd improve:** Add ingress with TLS for HTTPS (currently HTTP only).

---

## Related

- [Minecraft](minecraft.md) — Another NodePort service
- [nginx-proxy-manager](../apps/nginx.md) — Add HTTPS termination
