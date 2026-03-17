---
layout: default
title: Minecraft
parent: Home Lab Kubernetes
---

# Minecraft Server

**Kubernetes-managed Minecraft server** with persistent storage.

---

## Overview

Self-hosted Minecraft server using the `itzg/minecraft-server` image. Runs in K8s with:

- Persistent world data (10Gi PVC)
- NodePort exposure for game clients
- EULA auto-acceptance

---

## Architecture

```yaml
Namespace: minecraft-space
Replicas: 1
Image: itzg/minecraft-server:latest
Port: 25565 (NodePort 30001)
Storage: 10Gi hostPath PVC
```

### Storage Strategy

Uses **hostPath PV** (not Longhorn) for:
- Direct disk access (better I/O performance)
- Node-local storage (Minecraft doesn't need HA)
- Simpler backup (tar the host directory)

---

## Manifest Structure

```
Apps/minecraft/base/
├── kustomization.yaml
├── namespace.yaml
├── pvc.yaml
├── deployment.yaml
└── service.yaml
```

### Key Files

**pvc.yaml:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minecraft-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  volumeName: minecraft-data-pv
```

**deployment.yaml:**
```yaml
containers:
- name: minecraft-server
  image: itzg/minecraft-server:latest
  env:
  - name: EULA
    value: "TRUE"
  volumeMounts:
  - name: minecraft-data
    mountPath: /data
```

---

## Deployment

```bash
# Apply via kubectl
kubectl apply -k Apps/minecraft/base/

# Watch rollout
kubectl rollout status deployment/minecraft-server -n minecraft-space
```

### Access

Connect via:
- `<node-ip>:30001` (standard Minecraft port 25565)

---

## Why This Design?

### Decisions

| Decision | Rationale |
|----------|-----------|
| **hostPath PV** | Minecraft is single-node, doesn't need HA storage |
| **NodePort** | Direct game client access, no proxy needed |
| **itzg image** | Industry standard, actively maintained |
| **EULA env var** | Auto-accepts Mojang EULA on first boot |

### Trade-offs

- **Pro:** Simple, performant, easy backup
- **Con:** Tied to single node (can't migrate without copying data)

---

## Lessons Learned

**What worked:** The itzg image handles everything — version upgrades, world management, modpacks.

**What I'd improve:**
- Add **resource limits** (Minecraft can spike on chunk generation)
- Use **Longhorn** if you need HA (but hostPath is fine for casual servers)

---

## Backup

```bash
# Tar the world data
ssh talos-326-d4w "tar -czf /var/minecraft-backup-$(date +%F).tar.gz /var/minecraft-data"

# Restore
tar -xzf /var/minecraft-backup-2026-03-17.tar.gz -C /var/minecraft-data
```

---

## Related

- [PairDrop](pairdrop.md) — Another NodePort service
- [Windows VM](windows.md) — Also uses hostPath for storage
