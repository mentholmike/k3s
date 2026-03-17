---
layout: default
title: Windows VM
parent: Home Lab Kubernetes
---

# Windows 11 VM

**Windows 11 virtual machine** via KubeVirt with KVM passthrough.

---

## Overview

Runs Windows 11 in a K8s pod using the `dockurr/windows` image. Features:

- KVM hardware virtualization (near-native performance)
- Longhorn storage (75Gi replicated PVC)
- RDP + VNC + Web UI access
- NodeSelector pinning to KVM-capable node

---

## Architecture

```yaml
Namespace: windows-space
Replicas: 1
Image: dockurr/windows
Port: 8006 (Web), 3389 (RDP), 5900 (VNC)
Storage: 75Gi Longhorn PVC
Node: talos-ckf-wwf (KVM-capable)
```

### Why KubeVirt?

KubeVirt enables **VMs in K8s** — useful for:
- Windows workloads (can't containerize)
- Legacy apps requiring full OS
- GPU passthrough (future expansion)

---

## Manifest Structure

```
Apps/windows/base/
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
  name: windows-storage-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 75Gi
  storageClassName: longhorn
```

**deployment.yaml:**
```yaml
nodeSelector:
  kvm: "true"
  kubernetes.io/hostname: talos-ckf-wwf
containers:
- name: windows
  image: dockurr/windows
  env:
  - name: VERSION
    value: "10"
  - name: RAM_SIZE
    value: "6G"
  - name: CPU_CORES
    value: "2"
  resources:
    requests:
      memory: "2Gi"
      cpu: "500m"
    limits:
      memory: "10Gi"
      cpu: "4"
  securityContext:
    privileged: true
    capabilities:
      add: ["NET_ADMIN"]
  volumeMounts:
  - name: windows-storage
    mountPath: /storage
  - name: dev-kvm
    mountPath: /dev/kvm
```

---

## Deployment

```bash
# Apply via kubectl
kubectl apply -k Apps/windows/base/

# Watch rollout (takes ~5 minutes for Windows boot)
kubectl rollout status deployment/windows -n windows-space
```

### Access

Connect via:
- **Web UI:** `http://<node-ip>:30007` (noval novnc console)
- **RDP:** `<node-ip>:30008` (Windows Remote Desktop)
- **VNC:** `<node-ip>:30009` (alternative console)

---

## Why This Design?

### Decisions

| Decision | Rationale |
|----------|-----------|
| **Longhorn PVC** | HA storage — VM survives node failure |
| **NodeSelector** | Pins to KVM-capable node (not all nodes have KVM) |
| **Recreate strategy** | RWO volumes can't do RollingUpdate |
| **Privileged container** | Required for KVM device access |

### Trade-offs

- **Pro:** Full Windows in K8s, HA storage, multiple access methods
- **Con:** Privileged container (elevated risk), node-specific

---

## Lessons Learned

**What worked:** The `dockurr/windows` image is brilliant — auto-installs Windows, handles licensing, provides web console.

**What I'd improve:**
- Add **GPU passthrough** for graphics workloads
- Use **VirtIO drivers** for better disk/network performance

---

## Resource Tuning

Default is conservative. Adjust for your workload:

```yaml
resources:
  requests:
    memory: "4Gi"   # Minimum for smooth Win11
    cpu: "1000m"
  limits:
    memory: "16Gi"  # Heavy workloads (video editing, etc.)
    cpu: "8"
```

---

## Related

- [Longhorn](longhorn.md) — Storage backend for Windows PVC
- [PairDrop](pairdrop.md) — Much lighter workload (contrast)
