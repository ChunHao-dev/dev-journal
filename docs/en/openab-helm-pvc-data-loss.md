# Helm Upgrade Deleted Our PVC: A Lesson from openAB

> Learning new tech through the open-source project openAB: https://github.com/openabdev/openab

> **Version Info**
> - Kubernetes / Helm / PVC
> - Date: 2026-04-10

---

After a version update on openAB, the AI running on it lost all its prompt files.

K8s (Kubernetes) is a container management platform — like Docker but cloud-deployed, with auto-scaling and automatic container lifecycle management. Helm is a package manager for K8s that uses Charts (install packages) to install, update, delete, and rollback apps, so you don't have to write YAML files by hand and apply them one by one.

Files inside a container are temporary by default — restart and they're gone. To keep data around, you need a PVC (Persistent Volume Claim) mounted to the container. But this time, the new Chart version changed its structure, which changed the PVC name. Helm upgrade deleted the old PVC, and the data went with it.

This post covers the full root cause, how we caught it, and how to prevent it.

---

## TL;DR

- **Problem**: After a Helm upgrade, all prompt files inside the container disappeared
- **Root cause**: New Chart changed values.yaml structure, PVC name changed from `openab` to `openab-kiro`, Helm deleted the old PVC
- **Key mechanism**: Helm upgrade deletes resources that exist in the old release but not in the new one
- **Prevention**: Backup before upgrade, add `helm.sh/resource-policy: keep`, diff old vs new templates
- **Applies to**: Any stateful app managed by Helm

---

## The Problem

### What happened

Running AI agents on openAB. A teammate upgraded the Chart version, and all prompt files stored inside the container vanished.

**Expected:**
- Upgrade completes, AI agent keeps working, data intact

**Actual:**
- Prompt files gone, AI agent broken

**Trigger conditions:**
- New Chart restructured values.yaml
- PVC name changed as a result
- Old PVC had no `helm.sh/resource-policy: keep` annotation

---

## Core Concepts

### K8s, Helm, and Charts

```
K8s (Kubernetes)
  Container orchestration platform — like Docker but cloud-native
  ├── Horizontal scaling (auto-add machines under load)
  ├── Auto-restart (container crashes → K8s brings it back)
  └── Lifecycle management (deploy, update, rollback)

Helm
  Package manager for K8s (like apt/brew for an OS)
  └── Uses Charts to package, install, update, delete, rollback apps

Chart (Helm's install package)
  ├── templates/     YAML templates for K8s resources, with variable placeholders
  ├── values.yaml    Default values (tokens, storage size…)
  └── Chart.yaml     Chart name, version, container image version
```

How Helm works:

```bash
helm install/upgrade <release> <chart> --set key=value
```

```
Parameters + values.yaml
    ↓
Fill placeholders in templates/
    ↓
Generate complete K8s YAML
    ↓
Send to K8s (create/update Pods, Services, PVCs…)
```

The point: Helm saves you from writing YAML files by hand and applying them one by one with `kubectl apply`.

### PVC (Persistent Volume Claim)

Files inside a container are temporary by default — restart and they're gone. PVC is how K8s provides persistent storage:

```
Without PVC:
  Pod restarts → files inside container gone

With PVC:
  Pod ──mount──→ PVC ──bind──→ Host disk
  Pod restarts → PVC still there → data preserved
```

- PVC is a resource independent of the Pod
- Actual data lives on the host machine's disk
- Even if the Pod is deleted and recreated, data survives as long as the PVC exists

### Helm Upgrade Resource Management

This is the key to understanding the incident:

```
Helm upgrade comparison:

  Old release resources        vs    New release resources
  ─────────────────────              ─────────────────────
  Pod: openab                        Pod: openab
  Service: openab                    Service: openab
  PVC: openab            ←──×──      PVC: openab-kiro    ← different name!

  Result:
  - Pod, Service → updated (same name)
  - PVC openab → DELETED (doesn't exist in new release)
  - PVC openab-kiro → created (didn't exist in old release)
```

Helm's logic: **resource exists in old release but not in new release → delete it**.

---

## Root Cause Analysis

### Why did the PVC name change?

The new Chart version restructured values.yaml to support multiple agents:

```yaml
# Old values.yaml
storage: 1Gi
token: <TOKEN>

# New values.yaml (added agents.kiro layer)
agents:
  kiro:
    storage: 1Gi
    token: <TOKEN>
```

The template uses the values path to construct the PVC name. Structure changes → name changes:

```
Old template → PVC name: openab
New template → PVC name: openab-kiro
```

### Why did Helm delete the old PVC?

Two conditions met simultaneously:

1. Old PVC `openab` doesn't exist in the new release's resource list
2. Old PVC had no `helm.sh/resource-policy: keep` annotation

With this annotation, Helm skips deletion:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openab
  annotations:
    helm.sh/resource-policy: keep    # ← tells Helm "don't delete me"
```

### How we discovered it

```bash
# 1. Check current PVC names
kubectl get pvc
# → shows "openab"

# 2. Preview what the new Chart would generate
helm template <release> <new-chart> --set ...
# → shows PVC name "openab-kiro"

# 3. Compare → names differ → upgrade will break things
```

---

## How-to: Safe Helm Chart Upgrade

### Pre-upgrade checks

```bash
# Step 1: Record current PVCs
kubectl get pvc -n <NAMESPACE>

# Step 2: Preview new Chart resources
helm template <RELEASE> <NEW_CHART> --set key=value

# Step 3: Compare PVC names
# If different → enter protection flow
```

### Protection flow (when PVC name will change)

```bash
# Option 1: Backup PVC data
kubectl cp <NAMESPACE>/<POD_NAME>:/path/to/data ./backup/

# Option 2: Add resource-policy annotation
kubectl annotate pvc <PVC_NAME> -n <NAMESPACE> \
  helm.sh/resource-policy=keep

# Option 3: Do both (recommended)
```

### Post-upgrade verification

```bash
# Confirm old PVC still exists (if keep was added)
kubectl get pvc -n <NAMESPACE>

# Confirm new Pod is running
kubectl get pods -n <NAMESPACE>

# If needed, migrate data from old PVC to new PVC
kubectl cp ./backup/ <NAMESPACE>/<NEW_POD_NAME>:/path/to/data
```

---

## Best Practices

- **Always add `helm.sh/resource-policy: keep` to PVC templates for stateful apps**
  - This should be the Chart author's job, but many Charts don't do it
  - Users can add it manually
- **Always run `helm template` and diff before upgrading**
  - Pay special attention to PVC, Secret, ConfigMap names
- **Always backup PVC data before upgrading**
  - Even if names haven't changed, other things can go wrong
- **Use the `helm diff` plugin for convenience**
  - `helm diff upgrade <release> <chart>` shows the diff directly

---

## Troubleshooting

### Symptom: Files inside container gone after upgrade

- **Likely cause**: PVC deleted by Helm (name changed + no keep policy)
- **Fix**:
  1. `kubectl get pvc` to check if PVC still exists
  2. If PVC is gone, data cannot be recovered (unless you have backups or the underlying storage is still intact)
  3. Restore from backup or recreate the data

### Symptom: Two PVCs exist after upgrade

- **Likely cause**: Added `resource-policy: keep`, so old PVC was preserved while new PVC was also created
- **Fix**:
  1. Migrate data from old PVC to new PVC
  2. Confirm new Pod mounts the new PVC
  3. Manually delete old PVC

### Symptom: helm upgrade errors with "resource already exists"

- **Likely cause**: Manually created resource conflicts with Helm-managed resource
- **Fix**:
  1. `kubectl annotate` to add `meta.helm.sh/release-name` and `meta.helm.sh/release-namespace`
  2. `kubectl label` to add `app.kubernetes.io/managed-by=Helm`

---

## Summary

The core lesson: Helm upgrade doesn't just "update" — it actively cleans up resources that no longer exist in the new release. For stateful apps (PVCs, databases), diffing templates and backing up data before upgrading is not optional.

Key takeaways:
- Helm generates K8s resources from Chart templates + values
- Changing values structure → resource names may change
- Helm upgrade deletes "old-has, new-doesn't" resources
- `helm.sh/resource-policy: keep` is the lifeline for PVCs

---

## References

- [openAB GitHub](https://github.com/openabdev/openab)
- [Helm Docs - Resource Policy](https://helm.sh/docs/howto/charts_tips_and_tricks/#tell-helm-not-to-uninstall-a-resource)
- [Kubernetes PVC Docs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Helm Diff Plugin](https://github.com/databus23/helm-diff)
