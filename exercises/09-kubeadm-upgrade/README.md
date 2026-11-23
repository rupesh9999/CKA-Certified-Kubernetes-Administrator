# Exercise 09 — kubeadm Cluster Upgrade

> Related: [README — Cluster Architecture](../../README.md#domain-4--cluster-architecture-installation--configuration-25)

Upgrade a cluster from one minor version to the next using kubeadm. The CKA frequently asks this — you need to know the exact sequence.

## Tasks

1. Check the current cluster version (`k get nodes`, `kubeadm version`)
2. Upgrade the control plane node:
   a. Update the kubeadm package to the target version
   b. Run `kubeadm upgrade plan` to see available upgrades
   c. Run `kubeadm upgrade apply v1.35.x`
   d. Upgrade kubelet and kubectl packages
   e. Restart kubelet
3. Upgrade a worker node:
   a. Drain the worker node
   b. Update kubeadm, kubelet, kubectl packages on the worker
   c. Run `kubeadm upgrade node`
   d. Restart kubelet
   e. Uncordon the worker
4. Verify all nodes show the new version

## Hints

<details>
<summary>Stuck? Click to reveal hints</summary>

- Always upgrade control plane first, then workers one at a time
- The package manager commands depend on your OS (apt for Ubuntu, yum for CentOS)
- `kubeadm upgrade plan` shows you what versions are available
- On workers, it's `kubeadm upgrade node` (NOT `kubeadm upgrade apply`)
- Don't forget to drain before upgrading a worker

</details>

## What tripped me up

> After upgrading kubelet, it didn't start. I ran `systemctl restart kubelet` but forgot `systemctl daemon-reload` first. The systemd unit file changed but systemd didn't know about it. Always `daemon-reload` then `restart`. In that order. I lost 4 minutes on this during practice.
>
> On the worker node, I ran `kubeadm upgrade apply` instead of `kubeadm upgrade node`. Apply is for the control plane only. On workers it's `kubeadm upgrade node`. The error message is confusing — it doesn't immediately say "wrong command." It tries to run and fails partway through.
>
> Also: drain the worker BEFORE upgrading it. I upgraded first, then drained, and some pods got killed mid-upgrade when the node restarted. The safe order is: drain → upgrade → restart kubelet → uncordon.

## Verify

```bash
# All nodes should show the new version
k get nodes

# Components should match
kubeadm version
kubelet --version
kubectl version --client
```

## Cleanup

No cleanup needed — this is a one-way upgrade.

<details>
<summary>Solution</summary>

```bash
# === CONTROL PLANE NODE ===

# Check current version
k get nodes
kubeadm version

# Update kubeadm (Ubuntu/Debian example)
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=1.35.0-1.1
sudo apt-mark hold kubeadm

# Plan
sudo kubeadm upgrade plan

# Apply
sudo kubeadm upgrade apply v1.35.0

# Update kubelet + kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.35.0-1.1 kubectl=1.35.0-1.1
sudo apt-mark hold kubelet kubectl

# Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Verify control plane
k get nodes

# === WORKER NODE (run from control plane, then SSH to worker) ===

# From control plane: drain worker
k drain worker-1 --ignore-daemonsets --delete-emptydir-data

# SSH to worker node
ssh worker-1

# Update packages on worker
sudo apt-mark unhold kubeadm kubelet kubectl
sudo apt-get update
sudo apt-get install -y kubeadm=1.35.0-1.1 kubelet=1.35.0-1.1 kubectl=1.35.0-1.1
sudo apt-mark hold kubeadm kubelet kubectl

# Upgrade node config
sudo kubeadm upgrade node

# Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Exit back to control plane
exit

# Uncordon worker
k uncordon worker-1

# Verify
k get nodes
```

Remember:
- Control plane: `kubeadm upgrade apply`
- Worker: `kubeadm upgrade node`
- Always drain workers before upgrading them
- Always uncordon after

</details>


---

# Comprehensive Kubernetes Upgrade Guide — Definitions, Scenarios & Production Practices

> This section extends the exercise above with deep-dive definitions, YAML manifests, production-grade upgrade procedures for kubeadm, EKS, AKS, GKE, and every common upgrade scenario you will encounter in real-world operations and CKA/CKS exams.

---

## Table of Contents

- [Key Definitions & Terminology](#key-definitions--terminology)
- [Kubernetes Version Skew Policy](#kubernetes-version-skew-policy)
- [Pre-Upgrade Checklist (Universal)](#pre-upgrade-checklist-universal)
- [Scenario 1 — kubeadm Single Control Plane Upgrade](#scenario-1--kubeadm-single-control-plane-upgrade)
- [Scenario 2 — kubeadm HA Multi-Control-Plane Upgrade](#scenario-2--kubeadm-ha-multi-control-plane-upgrade)
- [Scenario 3 — etcd Backup & Restore During Upgrade](#scenario-3--etcd-backup--restore-during-upgrade)
- [Scenario 4 — PodDisruptionBudgets for Safe Upgrades](#scenario-4--poddisruptionbudgets-for-safe-upgrades)
- [Scenario 5 — Amazon EKS Cluster Upgrade](#scenario-5--amazon-eks-cluster-upgrade)
- [Scenario 6 — Azure AKS Cluster Upgrade](#scenario-6--azure-aks-cluster-upgrade)
- [Scenario 7 — Google GKE Cluster Upgrade](#scenario-7--google-gke-cluster-upgrade)
- [Scenario 8 — Rolling vs Blue-Green vs Canary Upgrade Strategies](#scenario-8--rolling-vs-blue-green-vs-canary-upgrade-strategies)
- [Scenario 9 — Deprecated API Migration](#scenario-9--deprecated-api-migration)
- [Scenario 10 — CNI & Add-on Upgrades](#scenario-10--cni--add-on-upgrades)
- [Scenario 11 — Upgrading Across Multiple Minor Versions](#scenario-11--upgrading-across-multiple-minor-versions)
- [Scenario 12 — Terraform/IaC-Driven Cluster Upgrades](#scenario-12--terraformiac-driven-cluster-upgrades)
- [Production Upgrade Runbook Template](#production-upgrade-runbook-template)
- [Troubleshooting Common Upgrade Failures](#troubleshooting-common-upgrade-failures)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Key Definitions & Terminology

| Term | Definition |
|------|-----------|
| **Minor Version** | The `Y` in `vX.Y.Z`. Kubernetes releases a new minor version roughly every 4 months (e.g., 1.31 → 1.32). Contains new features and API changes. |
| **Patch Version** | The `Z` in `vX.Y.Z`. Contains bug fixes and security patches only. No API or feature changes (e.g., 1.32.0 → 1.32.3). |
| **kubeadm** | The official Kubernetes tool to bootstrap, upgrade, and manage the lifecycle of a cluster's control plane and node configurations. |
| **Control Plane** | The set of components (kube-apiserver, kube-controller-manager, kube-scheduler, etcd) that manage the cluster state. Upgraded first during cluster upgrades. |
| **Data Plane** | The worker nodes running kubelet and kube-proxy that execute application workloads. Upgraded after the control plane. |
| **kubelet** | The agent running on every node that manages pod lifecycle, container creation, and node health reporting to the API server. |
| **kube-proxy** | Runs on each node; maintains network rules for Service connectivity (iptables/IPVS). |
| **etcd** | The distributed key-value store that serves as Kubernetes' backing store for all cluster data. Must be backed up before every upgrade. |
| **Static Pods** | Pods managed directly by kubelet on each node (e.g., kube-apiserver, etcd). Their manifests live in `/etc/kubernetes/manifests/`. |
| **Cordon** | Marks a node as unschedulable. New pods will not be placed on it, but existing pods continue running. Command: `kubectl cordon <node>`. |
| **Drain** | Safely evicts all pods from a node (respecting PDBs and graceful termination). Automatically cordons the node first. Command: `kubectl drain <node>`. |
| **Uncordon** | Marks a cordoned node as schedulable again. Command: `kubectl uncordon <node>`. |
| **PodDisruptionBudget (PDB)** | A policy object that limits the number of pods of a replicated application that can be simultaneously disrupted during voluntary disruptions like upgrades. |
| **Version Skew** | The allowed difference in versions between cluster components. Kubernetes enforces strict skew policies between apiserver, kubelet, kube-proxy, and kubectl. |
| **Rolling Update** | An upgrade strategy that replaces nodes/pods one (or a few) at a time, maintaining availability throughout the process. |
| **Blue-Green Upgrade** | An upgrade strategy that provisions an entirely new environment (green) alongside the old one (blue), then switches traffic. Enables instant rollback. |
| **Canary Upgrade** | An upgrade strategy that applies changes to a small subset first, validates, then rolls out to the rest of the fleet. |
| **Surge Upgrade** | Temporarily over-provisions extra nodes during upgrade to maintain capacity while old nodes are drained and replaced. |
| **Release Channel** | A managed Kubernetes version stream (e.g., Rapid, Regular, Stable) used by cloud providers to control the pace of automatic upgrades. |
| **Maintenance Window** | A defined time period when automated upgrades/changes are permitted. Used to avoid disruption during peak traffic. |
| **Maintenance Exclusion** | A blackout period during which no automated upgrades are allowed (e.g., Black Friday for e-commerce). |
| **In-place Upgrade** | Upgrading the existing cluster infrastructure directly without spinning up parallel environments. |
| **Node Pool** | A group of nodes with the same configuration (instance type, OS image, Kubernetes version). Cloud providers manage upgrades at the node pool level. |
| **AMI / OS Image** | The base machine image for nodes. Upgrades often require rolling to a new AMI/image with the updated kubelet version baked in. |
| **Feature Gate** | A mechanism to enable/disable experimental or alpha features. Some feature gates change defaults between versions and can break workloads if not audited. |

---

## Kubernetes Version Skew Policy

Kubernetes enforces strict version compatibility between components. Violating these rules can cause cluster instability.

```
┌─────────────────────────────────────────────────────────────────┐
│                    VERSION SKEW RULES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  kube-apiserver        → Must be the NEWEST component           │
│                                                                 │
│  kube-controller-mgr   → Same minor or N-1 of apiserver        │
│  kube-scheduler        → Same minor or N-1 of apiserver        │
│                                                                 │
│  kubelet               → Same minor, N-1, N-2, or N-3 of       │
│                           apiserver (but N-1 recommended)       │
│                                                                 │
│  kube-proxy            → Same minor as kubelet                  │
│                                                                 │
│  kubectl               → N+1, same, or N-1 of apiserver        │
│                                                                 │
│  etcd                  → 3.5.x supported (check release notes)  │
│                                                                 │
│  RULE: Never skip minor versions during upgrade!                │
│  1.30 → 1.31 → 1.32 ✓    1.30 → 1.32 ✗                       │
└─────────────────────────────────────────────────────────────────┘
```

**Example:** If kube-apiserver is at v1.33, then:
- kubelet can be at v1.33, v1.32, v1.31, or v1.30 (N to N-3)
- kube-controller-manager can be at v1.33 or v1.32 (N or N-1)
- kubectl can be at v1.34, v1.33, or v1.32 (N+1, N, or N-1)

---

## Pre-Upgrade Checklist (Universal)

This checklist applies to **every** Kubernetes upgrade regardless of platform.

```bash
# ──────────────────────────────────────
# PRE-UPGRADE CHECKLIST
# ──────────────────────────────────────

# 1. Verify cluster health
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed
kubectl get componentstatuses  # deprecated but still useful
kubectl cluster-info

# 2. Check current versions
kubectl version --short
kubeadm version  # if kubeadm cluster
kubelet --version

# 3. Review release notes for the target version
#    - Check for removed APIs
#    - Check for changed feature gate defaults
#    - Check for breaking changes
# URL pattern: https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.XX.md

# 4. Scan for deprecated APIs
# Install pluto (Fairwinds)
kubectl api-resources --verbs=list -o name | \
  xargs -I {} kubectl get {} -A -o yaml 2>/dev/null | \
  pluto detect -

# Or use kubent (Kube No Trouble)
kubent

# 5. Back up etcd (kubeadm clusters)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-pre-upgrade-$(date +%F).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 6. Verify backup integrity
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-pre-upgrade-$(date +%F).db -w table

# 7. Back up cluster resource manifests
kubectl get all -A -o yaml > /backup/all-resources-$(date +%F).yaml

# 8. Verify PodDisruptionBudgets exist for critical workloads
kubectl get pdb -A

# 9. Check node capacity and resource headroom
kubectl top nodes
kubectl describe nodes | grep -A5 "Allocated resources"

# 10. Notify stakeholders and open a maintenance window
```

---

## Scenario 1 — kubeadm Single Control Plane Upgrade

> **Use case:** Self-managed cluster with 1 control plane + N workers. CKA exam scenario.
> **Example:** Upgrading from v1.32.0 to v1.33.0 on Ubuntu/Debian

### Architecture Before Upgrade

```
┌──────────────────────────────────────────────┐
│              BEFORE UPGRADE                   │
│                                               │
│   control-plane (v1.32.0)                     │
│   ├── kube-apiserver v1.32.0                  │
│   ├── kube-controller-manager v1.32.0         │
│   ├── kube-scheduler v1.32.0                  │
│   ├── etcd 3.5.x                              │
│   ├── kubelet v1.32.0                         │
│   └── kubectl v1.32.0                         │
│                                               │
│   worker-1 (v1.32.0)                          │
│   ├── kubelet v1.32.0                         │
│   └── kube-proxy v1.32.0                      │
│                                               │
│   worker-2 (v1.32.0)                          │
│   ├── kubelet v1.32.0                         │
│   └── kube-proxy v1.32.0                      │
└──────────────────────────────────────────────┘
```

### Step-by-Step Procedure

```bash
# ══════════════════════════════════════════════
# PHASE 1: UPGRADE CONTROL PLANE
# ══════════════════════════════════════════════

# --- Step 1.1: Update package repository to target version ---
# Change the Kubernetes apt repo to point to v1.33
# Edit /etc/apt/sources.list.d/kubernetes.list to use v1.33 repo
# Example: deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
#   https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /

# --- Step 1.2: Upgrade kubeadm ---
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=1.33.0-1.1
sudo apt-mark hold kubeadm

# Verify kubeadm version
kubeadm version

# --- Step 1.3: Run upgrade plan ---
sudo kubeadm upgrade plan
# This shows:
#   - Current cluster version
#   - Target versions available
#   - Components that will be upgraded
#   - Any manual actions required

# --- Step 1.4: Apply the upgrade ---
sudo kubeadm upgrade apply v1.33.0
# This upgrades:
#   - kube-apiserver
#   - kube-controller-manager
#   - kube-scheduler
#   - kube-proxy (DaemonSet)
#   - CoreDNS
#   - etcd (if managed by kubeadm)

# --- Step 1.5: Drain the control plane (optional but recommended) ---
kubectl drain $(hostname) --ignore-daemonsets --delete-emptydir-data

# --- Step 1.6: Upgrade kubelet and kubectl ---
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.33.0-1.1 kubectl=1.33.0-1.1
sudo apt-mark hold kubelet kubectl

# --- Step 1.7: Restart kubelet ---
# ⚠️ CRITICAL: daemon-reload BEFORE restart!
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# --- Step 1.8: Uncordon the control plane ---
kubectl uncordon $(hostname)

# --- Step 1.9: Verify control plane upgrade ---
kubectl get nodes
# control-plane should show v1.33.0

# ══════════════════════════════════════════════
# PHASE 2: UPGRADE EACH WORKER NODE (one at a time)
# ══════════════════════════════════════════════

# --- Step 2.1: Drain the worker (from control plane) ---
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data

# --- Step 2.2: SSH to the worker ---
ssh worker-1

# --- Step 2.3: Upgrade kubeadm on the worker ---
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=1.33.0-1.1
sudo apt-mark hold kubeadm

# --- Step 2.4: Upgrade node configuration ---
# ⚠️ On workers: use "kubeadm upgrade node" (NOT "kubeadm upgrade apply")
sudo kubeadm upgrade node

# --- Step 2.5: Upgrade kubelet and kubectl ---
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.33.0-1.1 kubectl=1.33.0-1.1
sudo apt-mark hold kubelet kubectl

# --- Step 2.6: Restart kubelet ---
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# --- Step 2.7: Exit and uncordon (from control plane) ---
exit
kubectl uncordon worker-1

# --- Repeat Steps 2.1–2.7 for every remaining worker ---
```

### CentOS/RHEL Variant

```bash
# Package manager commands for CentOS/RHEL-based systems

# Upgrade kubeadm
sudo yum install -y kubeadm-1.33.0-150500.1.1 --disableexcludes=kubernetes

# Upgrade kubelet and kubectl
sudo yum install -y kubelet-1.33.0-150500.1.1 kubectl-1.33.0-150500.1.1 \
  --disableexcludes=kubernetes

# Restart
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

---

## Scenario 2 — kubeadm HA Multi-Control-Plane Upgrade

> **Use case:** Production cluster with 3 control plane nodes (HA) + N workers.
> The critical difference: only the FIRST control plane runs `kubeadm upgrade apply`. All others run `kubeadm upgrade node`.

### Architecture

```
┌──────────────────────────────────────────────────────────┐
│                 HA CONTROL PLANE (3 nodes)                │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   cp-1       │  │   cp-2       │  │   cp-3       │    │
│  │  (PRIMARY)   │  │ (SECONDARY)  │  │ (SECONDARY)  │    │
│  │             │  │             │  │             │       │
│  │  apiserver  │  │  apiserver  │  │  apiserver  │       │
│  │  controller │  │  controller │  │  controller │       │
│  │  scheduler  │  │  scheduler  │  │  scheduler  │       │
│  │  etcd       │  │  etcd       │  │  etcd       │       │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘       │
│         │               │               │               │
│         └───────────────┼───────────────┘               │
│                         │                                │
│                   Load Balancer                          │
│                         │                                │
│         ┌───────────────┼───────────────┐               │
│  ┌──────┴──────┐  ┌─────┴──────┐  ┌────┴───────┐       │
│  │  worker-1   │  │  worker-2  │  │  worker-3  │       │
│  └─────────────┘  └────────────┘  └────────────┘       │
└──────────────────────────────────────────────────────────┘
```

### Upgrade Order

```
Step 1: Back up etcd (on cp-1)
Step 2: Upgrade cp-1 → kubeadm upgrade apply v1.33.0
Step 3: Upgrade cp-2 → kubeadm upgrade node
Step 4: Upgrade cp-3 → kubeadm upgrade node
Step 5: Upgrade worker-1 (drain → upgrade → uncordon)
Step 6: Upgrade worker-2 (drain → upgrade → uncordon)
Step 7: Upgrade worker-3 (drain → upgrade → uncordon)
Step 8: Verify all nodes
```

```bash
# ══════════════════════════════════════════════
# HA CLUSTER UPGRADE — DETAILED PROCEDURE
# ══════════════════════════════════════════════

# === STEP 1: BACKUP etcd (on cp-1) ===
sudo ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%F).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
sudo ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-$(date +%F).db -w table

# === STEP 2: UPGRADE FIRST CONTROL PLANE (cp-1) ===
# On cp-1:
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.33.0-1.1
sudo apt-mark hold kubeadm

sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.33.0  # ← Only on the FIRST CP!

kubectl drain cp-1 --ignore-daemonsets --delete-emptydir-data

sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.33.0-1.1 kubectl=1.33.0-1.1
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon cp-1

# === STEP 3: UPGRADE cp-2 ===
# SSH to cp-2:
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.33.0-1.1
sudo apt-mark hold kubeadm

sudo kubeadm upgrade node  # ← NOT "apply"!

sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.33.0-1.1 kubectl=1.33.0-1.1
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

# === STEP 4: UPGRADE cp-3 (same as cp-2) ===
# (Repeat Step 3 on cp-3)

# === STEP 5-7: UPGRADE WORKERS (one at a time) ===
# (Same procedure as Scenario 1, Phase 2)

# === STEP 8: FINAL VERIFICATION ===
kubectl get nodes -o wide
kubectl get pods -n kube-system
kubectl get cs  # component status
```

> **⚠️ Production Tip:** In an HA cluster, the API server remains available on other control plane nodes while you upgrade one at a time. This is a zero-downtime control plane upgrade if done correctly.

---

## Scenario 3 — etcd Backup & Restore During Upgrade

> **Use case:** You upgrade the cluster and something goes catastrophically wrong. You need to restore etcd from a snapshot to roll back to the pre-upgrade state.

### Pre-Upgrade Backup

```bash
# ──────────────────────────────────────
# AUTOMATED etcd BACKUP SCRIPT
# Save as: /usr/local/bin/etcd-backup.sh
# ──────────────────────────────────────
#!/bin/bash
set -euo pipefail

BACKUP_DIR="/opt/etcd-backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
SNAPSHOT_FILE="${BACKUP_DIR}/etcd-snapshot-${TIMESTAMP}.db"

ENDPOINTS="https://127.0.0.1:2379"
CACERT="/etc/kubernetes/pki/etcd/ca.crt"
CERT="/etc/kubernetes/pki/etcd/server.crt"
KEY="/etc/kubernetes/pki/etcd/server.key"

mkdir -p "${BACKUP_DIR}"

echo "Taking etcd snapshot..."
ETCDCTL_API=3 etcdctl snapshot save "${SNAPSHOT_FILE}" \
  --endpoints="${ENDPOINTS}" \
  --cacert="${CACERT}" \
  --cert="${CERT}" \
  --key="${KEY}"

echo "Verifying snapshot..."
ETCDCTL_API=3 etcdctl snapshot status "${SNAPSHOT_FILE}" -w table

echo "Snapshot saved to: ${SNAPSHOT_FILE}"

# Retain only last 10 backups
ls -t ${BACKUP_DIR}/etcd-snapshot-*.db | tail -n +11 | xargs -r rm -f
echo "Cleanup complete. Retained 10 most recent backups."
```

### Emergency Restore Procedure

```bash
# ══════════════════════════════════════════════
# EMERGENCY etcd RESTORE — Post Failed Upgrade
# ══════════════════════════════════════════════
# Run on the CONTROL PLANE node

# --- Step 1: Stop the control plane ---
# Move static pod manifests to stop apiserver and etcd
sudo mkdir -p /tmp/k8s-manifests-backup
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/k8s-manifests-backup/
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/k8s-manifests-backup/

# Wait for containers to stop
sleep 20
sudo crictl ps | grep -E "etcd|apiserver"  # Should show nothing

# --- Step 2: Restore from snapshot ---
SNAPSHOT="/opt/etcd-backups/etcd-snapshot-XXXXXXXX_XXXXXX.db"
RESTORE_DIR="/var/lib/etcd-restored"

sudo ETCDCTL_API=3 etcdctl snapshot restore "${SNAPSHOT}" \
  --data-dir="${RESTORE_DIR}"

# --- Step 3: Update etcd manifest to use restored data ---
# Edit the etcd.yaml to point to the new data directory
sudo cp /tmp/k8s-manifests-backup/etcd.yaml /tmp/k8s-manifests-backup/etcd.yaml.orig

# Replace the data directory path in the manifest
sudo sed -i "s|path: /var/lib/etcd|path: ${RESTORE_DIR}|g" \
  /tmp/k8s-manifests-backup/etcd.yaml

# --- Step 4: Fix ownership ---
sudo chown -R root:root "${RESTORE_DIR}"

# --- Step 5: Restore static pod manifests ---
sudo mv /tmp/k8s-manifests-backup/etcd.yaml /etc/kubernetes/manifests/
sudo mv /tmp/k8s-manifests-backup/kube-apiserver.yaml /etc/kubernetes/manifests/

# --- Step 6: Wait and verify ---
sleep 30

# Check etcd health
sudo ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify cluster
kubectl get nodes
kubectl get pods -A
```

### etcd Static Pod Manifest (Reference)

```yaml
# /etc/kubernetes/manifests/etcd.yaml (key sections)
apiVersion: v1
kind: Pod
metadata:
  name: etcd
  namespace: kube-system
  labels:
    component: etcd
    tier: control-plane
spec:
  hostNetwork: true
  containers:
  - name: etcd
    image: registry.k8s.io/etcd:3.5.15-0
    command:
    - etcd
    - --advertise-client-urls=https://10.0.0.10:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd                # ← This path changes during restore
    - --initial-advertise-peer-urls=https://10.0.0.10:2380
    - --initial-cluster=cp-1=https://10.0.0.10:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://10.0.0.10:2379
    - --listen-peer-urls=https://10.0.0.10:2380
    - --name=cp-1
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    volumeMounts:
    - mountPath: /var/lib/etcd
      name: etcd-data
    - mountPath: /etc/kubernetes/pki/etcd
      name: etcd-certs
  volumes:
  - hostPath:
      path: /var/lib/etcd               # ← Change to /var/lib/etcd-restored during restore
      type: DirectoryOrCreate
    name: etcd-data
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
```

### Automated etcd Backup CronJob (Production)

```yaml
# etcd-backup-cronjob.yaml
# Runs inside the cluster to back up etcd to a PersistentVolume or S3-compatible storage
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"          # Every 6 hours
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          hostNetwork: true
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          tolerations:
          - key: node-role.kubernetes.io/control-plane
            effect: NoSchedule
          containers:
          - name: etcd-backup
            image: registry.k8s.io/etcd:3.5.15-0
            command:
            - /bin/sh
            - -c
            - |
              ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db \
                --endpoints=https://127.0.0.1:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/server.crt \
                --key=/etc/kubernetes/pki/etcd/server.key && \
              echo "Backup successful" && \
              ls -la /backup/
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
            - name: backup-volume
              mountPath: /backup
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: backup-volume
            persistentVolumeClaim:
              claimName: etcd-backup-pvc
          restartPolicy: OnFailure
```

---

## Scenario 4 — PodDisruptionBudgets for Safe Upgrades

> **Use case:** Ensure workload availability during node drains and rolling upgrades.
> **Rule:** Every production workload MUST have a PDB before you upgrade.

### PDB with maxUnavailable (Recommended for Production)

```yaml
# pdb-maxunavailable.yaml
# Best practice: maxUnavailable scales automatically with replica count changes
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
  namespace: production
  labels:
    app: web-app
    managed-by: platform-team
spec:
  maxUnavailable: 1              # At most 1 pod can be disrupted at a time
  selector:
    matchLabels:
      app: web-app
```

### PDB with minAvailable

```yaml
# pdb-minavailable.yaml
# Use when you need to guarantee a specific minimum replica count
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-gateway-pdb
  namespace: production
spec:
  minAvailable: 2                # At least 2 pods must always be running
  selector:
    matchLabels:
      app: api-gateway
```

### PDB with Percentage

```yaml
# pdb-percentage.yaml
# Scales proportionally — great for autoscaled deployments
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: production
spec:
  maxUnavailable: "25%"          # At most 25% of pods disrupted at once
  selector:
    matchLabels:
      app: order-service
```

### Complete Production Deployment + PDB + TopologySpread

```yaml
# production-deployment-with-pdb.yaml
# Full production setup ensuring high availability during upgrades
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
  labels:
    app: payment-service
    version: v2.1.0
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0          # Zero-downtime during app deployments
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      # Spread pods across nodes — ensures draining one node
      # never takes down all replicas
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: payment-service
      # Anti-affinity for zone-level spread
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: payment-service
              topologyKey: topology.kubernetes.io/zone
      containers:
      - name: payment-app
        image: myregistry.io/payment-service:v2.1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]  # Graceful shutdown for in-flight requests
      terminationGracePeriodSeconds: 30
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-service-pdb
  namespace: production
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: payment-service
```

> **Why TopologySpread + PDB together?** A PDB says "only disrupt 1 pod at a time." But if all 4 pods are on the same node, draining that node evicts all 4 at once regardless of the PDB (the PDB only controls voluntary evictions *across* drain operations, but a single node drain evicts all local pods). TopologySpread ensures pods are physically distributed.

---

## Scenario 5 — Amazon EKS Cluster Upgrade

> **Use case:** Upgrading a managed EKS cluster in production. Control plane is managed by AWS; you manage add-ons and node groups.

### Upgrade Sequence

```
┌─────────────────────────────────────────────────────────────────┐
│              EKS UPGRADE ORDER OF OPERATIONS                    │
│                                                                 │
│  1. Pre-flight checks (deprecated APIs, add-on compatibility)   │
│  2. Upgrade Control Plane (AWS manages this)                    │
│  3. Upgrade EKS Add-ons (VPC CNI, kube-proxy, CoreDNS)         │
│  4. Upgrade Data Plane (Managed Node Groups / Karpenter / Self) │
│  5. Post-upgrade validation                                     │
└─────────────────────────────────────────────────────────────────┘
```

### CLI Commands

```bash
# ══════════════════════════════════════════════
# EKS CLUSTER UPGRADE — v1.32 to v1.33
# ══════════════════════════════════════════════

# --- Pre-flight: Check current version ---
aws eks describe-cluster \
  --name my-production-cluster \
  --query "cluster.version" \
  --output text

# --- Pre-flight: Check EKS Cluster Insights for blockers ---
aws eks list-insights \
  --cluster-name my-production-cluster

# --- Pre-flight: Check add-on versions ---
aws eks describe-addon \
  --cluster-name my-production-cluster \
  --addon-name vpc-cni \
  --query "addon.addonVersion"

aws eks describe-addon \
  --cluster-name my-production-cluster \
  --addon-name kube-proxy \
  --query "addon.addonVersion"

aws eks describe-addon \
  --cluster-name my-production-cluster \
  --addon-name coredns \
  --query "addon.addonVersion"

# ═══ STEP 1: UPGRADE CONTROL PLANE ═══
aws eks update-cluster-version \
  --name my-production-cluster \
  --kubernetes-version 1.33

# Monitor upgrade progress (takes 20-45 minutes)
aws eks describe-update \
  --name my-production-cluster \
  --update-id <update-id-from-above>

# Or watch:
watch -n 30 "aws eks describe-cluster \
  --name my-production-cluster \
  --query 'cluster.{status:status,version:version}'"

# ═══ STEP 2: UPGRADE ADD-ONS ═══
# Get compatible versions for the new Kubernetes version
aws eks describe-addon-versions \
  --kubernetes-version 1.33 \
  --addon-name vpc-cni \
  --query "addons[0].addonVersions[0].addonVersion"

# Update VPC CNI
aws eks update-addon \
  --cluster-name my-production-cluster \
  --addon-name vpc-cni \
  --addon-version v1.19.0-eksbuild.1 \
  --resolve-conflicts OVERWRITE

# Update kube-proxy
aws eks update-addon \
  --cluster-name my-production-cluster \
  --addon-name kube-proxy \
  --addon-version v1.33.0-eksbuild.1 \
  --resolve-conflicts OVERWRITE

# Update CoreDNS
aws eks update-addon \
  --cluster-name my-production-cluster \
  --addon-name coredns \
  --addon-version v1.11.4-eksbuild.1 \
  --resolve-conflicts OVERWRITE

# ═══ STEP 3: UPGRADE DATA PLANE (Managed Node Groups) ═══
# List node groups
aws eks list-nodegroups \
  --cluster-name my-production-cluster

# Upgrade node group (triggers rolling replacement)
aws eks update-nodegroup-version \
  --cluster-name my-production-cluster \
  --nodegroup-name production-workers \
  --kubernetes-version 1.33

# Monitor node group update
aws eks describe-nodegroup \
  --cluster-name my-production-cluster \
  --nodegroup-name production-workers \
  --query "nodegroup.{status:status,version:version}"

# ═══ STEP 4: VERIFY ═══
kubectl get nodes -o wide
kubectl get pods -A | grep -v Running | grep -v Completed
```

### EKS Upgrade with Terraform

```hcl
# eks-upgrade.tf
# Production EKS cluster managed via Terraform

resource "aws_eks_cluster" "production" {
  name     = "my-production-cluster"
  role_arn = aws_iam_role.eks_cluster.arn

  # Change this version to trigger upgrade
  version = "1.33"  # Previously "1.32"

  vpc_config {
    subnet_ids              = var.subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = false
  }

  # Ensure add-ons are upgraded before node groups
  depends_on = [
    aws_eks_addon.vpc_cni,
    aws_eks_addon.kube_proxy,
    aws_eks_addon.coredns,
  ]
}

resource "aws_eks_addon" "vpc_cni" {
  cluster_name                = aws_eks_cluster.production.name
  addon_name                  = "vpc-cni"
  addon_version               = "v1.19.0-eksbuild.1"  # Update for new k8s version
  resolve_conflicts_on_update = "OVERWRITE"
}

resource "aws_eks_addon" "kube_proxy" {
  cluster_name                = aws_eks_cluster.production.name
  addon_name                  = "kube-proxy"
  addon_version               = "v1.33.0-eksbuild.1"
  resolve_conflicts_on_update = "OVERWRITE"
}

resource "aws_eks_addon" "coredns" {
  cluster_name                = aws_eks_cluster.production.name
  addon_name                  = "coredns"
  addon_version               = "v1.11.4-eksbuild.1"
  resolve_conflicts_on_update = "OVERWRITE"
}

resource "aws_eks_node_group" "production_workers" {
  cluster_name    = aws_eks_cluster.production.name
  node_group_name = "production-workers"
  node_role_arn   = aws_iam_role.eks_node.arn
  subnet_ids      = var.private_subnet_ids

  # This follows the cluster version
  version = aws_eks_cluster.production.version

  scaling_config {
    desired_size = 6
    max_size     = 10
    min_size     = 3
  }

  update_config {
    max_unavailable_percentage = 33  # Surge upgrade: 33% recommended
  }

  instance_types = ["m6i.xlarge"]

  # Force new AMI on version change
  ami_type = "AL2_x86_64"

  lifecycle {
    # Prevent destruction during updates
    create_before_destroy = true
  }
}
```

### EKS Karpenter Node Upgrade

```yaml
# karpenter-nodepool.yaml
# When upgrading EKS with Karpenter, update the AMI family/version
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: production
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: production
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64"]
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["on-demand"]
      - key: node.kubernetes.io/instance-type
        operator: In
        values: ["m6i.xlarge", "m6i.2xlarge", "m7i.xlarge"]
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
  limits:
    cpu: "100"
    memory: 400Gi

---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: production
spec:
  amiSelectorTerms:
  - alias: al2023@v20241121        # ← Update this to trigger node rotation
  role: KarpenterNodeRole-my-production-cluster
  subnetSelectorTerms:
  - tags:
      karpenter.sh/discovery: my-production-cluster
  securityGroupSelectorTerms:
  - tags:
      karpenter.sh/discovery: my-production-cluster
```

---

## Scenario 6 — Azure AKS Cluster Upgrade

> **Use case:** Upgrading an AKS cluster with different strategies — in-place with max surge, blue-green node pools, and full cluster blue-green.

### Strategy 1: In-Place Upgrade with Max Surge

```bash
# ══════════════════════════════════════════════
# AKS IN-PLACE UPGRADE — v1.32 to v1.33
# ══════════════════════════════════════════════

# --- Pre-flight: Check current version ---
az aks show \
  --resource-group my-rg \
  --name my-aks-cluster \
  --query "kubernetesVersion" -o tsv

# --- Pre-flight: Check available upgrades ---
az aks get-upgrades \
  --resource-group my-rg \
  --name my-aks-cluster \
  -o table

# --- Configure max surge for the node pool (33% recommended) ---
az aks nodepool update \
  --resource-group my-rg \
  --cluster-name my-aks-cluster \
  --name nodepool1 \
  --max-surge 33%

# --- Upgrade control plane only first ---
az aks upgrade \
  --resource-group my-rg \
  --name my-aks-cluster \
  --kubernetes-version 1.33.0 \
  --control-plane-only \
  --yes

# --- Then upgrade node pools ---
az aks nodepool upgrade \
  --resource-group my-rg \
  --cluster-name my-aks-cluster \
  --name nodepool1 \
  --kubernetes-version 1.33.0

# --- Or upgrade everything at once ---
az aks upgrade \
  --resource-group my-rg \
  --name my-aks-cluster \
  --kubernetes-version 1.33.0 \
  --yes

# --- Verify ---
az aks show \
  --resource-group my-rg \
  --name my-aks-cluster \
  --query "kubernetesVersion" -o tsv

kubectl get nodes -o wide
```

### Strategy 2: Blue-Green Node Pool Upgrade

```bash
# ══════════════════════════════════════════════
# AKS BLUE-GREEN NODE POOL UPGRADE
# ══════════════════════════════════════════════

# Step 1: Upgrade control plane first
az aks upgrade \
  --resource-group my-rg \
  --name my-aks-cluster \
  --kubernetes-version 1.33.0 \
  --control-plane-only \
  --yes

# Step 2: Create a NEW "green" node pool with the new version
az aks nodepool add \
  --resource-group my-rg \
  --cluster-name my-aks-cluster \
  --name green \
  --kubernetes-version 1.33.0 \
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --mode System \
  --labels environment=green

# Step 3: Cordon the old "blue" node pool
# Mark blue nodes as unschedulable
kubectl cordon -l agentpool=nodepool1

# Step 4: Drain the old "blue" node pool
# Pods migrate to "green" nodes
kubectl drain -l agentpool=nodepool1 \
  --ignore-daemonsets \
  --delete-emptydir-data

# Step 5: Verify all workloads are running on green nodes
kubectl get pods -A -o wide | grep -v green

# Step 6: Delete the old "blue" node pool
az aks nodepool delete \
  --resource-group my-rg \
  --cluster-name my-aks-cluster \
  --name nodepool1 \
  --yes
```

### Strategy 3: Full Cluster Blue-Green (Zero Downtime)

```bash
# ══════════════════════════════════════════════
# AKS FULL CLUSTER BLUE-GREEN
# For mission-critical production environments
# ══════════════════════════════════════════════

# Step 1: Deploy a GREEN cluster with the new version
az aks create \
  --resource-group my-rg \
  --name my-aks-cluster-green \
  --kubernetes-version 1.33.0 \
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --network-plugin azure \
  --vnet-subnet-id /subscriptions/.../subnets/green-subnet \
  --generate-ssh-keys

# Step 2: Deploy all applications to the green cluster
az aks get-credentials --resource-group my-rg --name my-aks-cluster-green
kubectl apply -f manifests/

# Step 3: Run smoke tests and integration tests on green
# ... your testing pipeline ...

# Step 4: Switch traffic via Azure Traffic Manager / Front Door
az network traffic-manager endpoint update \
  --resource-group my-rg \
  --profile-name my-tm-profile \
  --name blue-endpoint \
  --type azureEndpoints \
  --endpoint-status Disabled

az network traffic-manager endpoint update \
  --resource-group my-rg \
  --profile-name my-tm-profile \
  --name green-endpoint \
  --type azureEndpoints \
  --endpoint-status Enabled

# Step 5: Monitor green cluster for stability (soak period)
# ... monitor for 24-72 hours ...

# Step 6: Decommission the BLUE cluster after soak
az aks delete \
  --resource-group my-rg \
  --name my-aks-cluster-blue \
  --yes --no-wait
```

### AKS Upgrade with Terraform

```hcl
# aks-upgrade.tf
resource "azurerm_kubernetes_cluster" "production" {
  name                = "my-aks-cluster"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = "myaks"

  # Change this to trigger upgrade
  kubernetes_version = "1.33.0"  # Previously "1.32.0"

  default_node_pool {
    name                = "system"
    node_count          = 3
    vm_size             = "Standard_D4s_v5"
    orchestrator_version = "1.33.0"

    # Surge upgrade configuration
    upgrade_settings {
      max_surge = "33%"           # 33% recommended for production
      drain_timeout_in_minutes = 30
    }
  }

  auto_upgrade_profile {
    # Options: patch, stable, rapid, node-image, none
    upgrade_channel = "stable"    # Automatic minor version upgrades
  }

  maintenance_window {
    allowed {
      day   = "Sunday"
      hours = [2, 3, 4, 5]       # 2am–6am Sunday only
    }
    not_allowed {
      start = "2026-11-20T00:00:00Z"  # Black Friday exclusion
      end   = "2026-12-01T00:00:00Z"
    }
  }

  identity {
    type = "SystemAssigned"
  }
}

# Additional node pool for workloads
resource "azurerm_kubernetes_cluster_node_pool" "user" {
  name                  = "userpool"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.production.id
  vm_size               = "Standard_D8s_v5"
  node_count            = 5
  orchestrator_version  = "1.33.0"

  upgrade_settings {
    max_surge = "33%"
  }
}
```

---

## Scenario 7 — Google GKE Cluster Upgrade

> **Use case:** Upgrading GKE clusters using release channels, maintenance windows, and fleet-based rollout strategies.

### GKE Release Channels Explained

```
┌─────────────────────────────────────────────────────────────────┐
│                 GKE RELEASE CHANNELS                            │
├─────────┬───────────────────────────────────────────────────────┤
│ Channel │ Description                                           │
├─────────┼───────────────────────────────────────────────────────┤
│ Rapid   │ Latest Kubernetes GA within days. For dev/staging.    │
│         │ Not recommended for production.                       │
├─────────┼───────────────────────────────────────────────────────┤
│ Regular │ Default channel. Balanced stability and features.     │
│         │ Good for general production workloads.                │
├─────────┼───────────────────────────────────────────────────────┤
│ Stable  │ Most conservative. Versions proven in Regular first.  │
│         │ Best for mission-critical production.                 │
├─────────┼───────────────────────────────────────────────────────┤
│ Extended│ Long-term support for a specific minor version.       │
│         │ For large enterprise stateful workloads.              │
└─────────┴───────────────────────────────────────────────────────┘

Promotion flow:  Rapid → Regular → Stable
                   ↓        ↓         ↓
                 Dev     Staging   Production
```

### GKE Upgrade Commands

```bash
# ══════════════════════════════════════════════
# GKE CLUSTER UPGRADE
# ══════════════════════════════════════════════

# --- Check current version ---
gcloud container clusters describe my-gke-cluster \
  --zone us-central1-a \
  --format="value(currentMasterVersion)"

# --- List available upgrades ---
gcloud container get-server-config \
  --zone us-central1-a \
  --format="yaml(validMasterVersions)"

# --- Set release channel (if not already on one) ---
gcloud container clusters update my-gke-cluster \
  --zone us-central1-a \
  --release-channel stable

# --- Manual control plane upgrade ---
gcloud container clusters upgrade my-gke-cluster \
  --zone us-central1-a \
  --master \
  --cluster-version 1.33.0-gke.100

# --- Upgrade node pools ---
# List node pools
gcloud container node-pools list \
  --cluster my-gke-cluster \
  --zone us-central1-a

# Upgrade specific node pool
gcloud container clusters upgrade my-gke-cluster \
  --zone us-central1-a \
  --node-pool default-pool \
  --cluster-version 1.33.0-gke.100

# --- Verify ---
gcloud container clusters describe my-gke-cluster \
  --zone us-central1-a \
  --format="table(currentMasterVersion, nodePools[].version)"

kubectl get nodes -o wide
```

### GKE Node Pool Surge Upgrade

```bash
# Configure surge upgrade for a node pool
gcloud container node-pools update default-pool \
  --cluster my-gke-cluster \
  --zone us-central1-a \
  --max-surge-upgrade 3 \
  --max-unavailable-upgrade 1
```

### GKE Blue-Green Node Pool Upgrade

```bash
# Configure blue-green upgrade strategy
gcloud container node-pools update default-pool \
  --cluster my-gke-cluster \
  --zone us-central1-a \
  --enable-blue-green-upgrade \
  --node-pool-soak-duration "3600s"      # 1-hour soak period
  --standard-rollout-policy \
    batch-node-count=3 \
    batch-soak-duration="600s"           # 10-min soak per batch
```

### GKE Maintenance Window & Exclusion

```bash
# Set maintenance window (Sunday 2am–6am UTC)
gcloud container clusters update my-gke-cluster \
  --zone us-central1-a \
  --maintenance-window-start 2026-01-01T02:00:00Z \
  --maintenance-window-end 2026-01-01T06:00:00Z \
  --maintenance-window-recurrence "FREQ=WEEKLY;BYDAY=SU"

# Add maintenance exclusion (no upgrades during peak)
gcloud container clusters update my-gke-cluster \
  --zone us-central1-a \
  --add-maintenance-exclusion-name "black-friday-2026" \
  --add-maintenance-exclusion-start "2026-11-20T00:00:00Z" \
  --add-maintenance-exclusion-end "2026-12-02T00:00:00Z" \
  --add-maintenance-exclusion-scope "no_minor_upgrades"
```

### GKE Fleet Rollout Sequencing

```yaml
# fleet-rollout-sequence.yaml
# Defines environment-based promotion: canary → staging → production
apiVersion: fleet.gke.io/v1alpha1
kind: RolloutSequence
metadata:
  name: production-rollout
spec:
  stages:
  - name: canary
    targetSelector:
      matchLabels:
        env: canary
    soakDuration: "7200s"           # 2-hour canary soak
    maxConcurrent: 1

  - name: staging
    targetSelector:
      matchLabels:
        env: staging
    soakDuration: "86400s"          # 24-hour staging soak
    maxConcurrent: 2

  - name: production-us
    targetSelector:
      matchLabels:
        env: production
        region: us
    soakDuration: "172800s"         # 48-hour soak before next region
    maxConcurrent: 3

  - name: production-eu
    targetSelector:
      matchLabels:
        env: production
        region: eu
    soakDuration: "172800s"
    maxConcurrent: 3
```

### GKE Upgrade with Terraform

```hcl
# gke-upgrade.tf
resource "google_container_cluster" "production" {
  name     = "my-gke-cluster"
  location = "us-central1"

  # Change this to trigger upgrade
  min_master_version = "1.33.0-gke.100"

  release_channel {
    channel = "STABLE"
  }

  maintenance_policy {
    recurring_window {
      start_time = "2026-01-01T02:00:00Z"
      end_time   = "2026-01-01T06:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SU"
    }

    maintenance_exclusion {
      exclusion_name = "black-friday"
      start_time     = "2026-11-20T00:00:00Z"
      end_time       = "2026-12-02T00:00:00Z"
      exclusion_options {
        scope = "NO_MINOR_UPGRADES"
      }
    }
  }

  # Enable Workload Identity
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  # Network configuration
  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.subnet.name
}

resource "google_container_node_pool" "production_nodes" {
  name     = "production-pool"
  location = "us-central1"
  cluster  = google_container_cluster.production.name

  version    = google_container_cluster.production.min_master_version
  node_count = 3

  node_config {
    machine_type = "e2-standard-8"
    disk_size_gb = 100
    disk_type    = "pd-ssd"

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform",
    ]
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }

  upgrade_settings {
    # Surge upgrade settings
    max_surge       = 3
    max_unavailable = 1

    # Or use blue-green strategy:
    # strategy = "BLUE_GREEN"
    # blue_green_settings {
    #   node_pool_soak_duration = "3600s"
    #   standard_rollout_policy {
    #     batch_node_count    = 3
    #     batch_soak_duration = "600s"
    #   }
    # }
  }
}
```

---

## Scenario 8 — Rolling vs Blue-Green vs Canary Upgrade Strategies

### Strategy Comparison

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    UPGRADE STRATEGY COMPARISON                             │
├────────────┬──────────────┬──────────────────┬─────────────────────────────┤
│ Aspect     │   Rolling    │   Blue-Green     │      Canary                 │
├────────────┼──────────────┼──────────────────┼─────────────────────────────┤
│ Downtime   │ Near-zero    │ Zero             │ Zero                        │
│ Cost       │ Low (reuse)  │ High (2x infra)  │ Low-Medium                  │
│ Rollback   │ Slow (undo)  │ Instant (switch) │ Fast (stop canary)          │
│ Risk       │ Medium       │ Low              │ Low                         │
│ Complexity │ Low          │ Medium           │ High                        │
│ Best For   │ Routine      │ Major upgrades   │ Validation-first upgrades   │
│            │ patch/minor  │ strict SLAs      │ large fleets                │
│ Data Sync  │ Not needed   │ Required for     │ Not needed                  │
│            │              │ stateful apps    │                             │
└────────────┴──────────────┴──────────────────┴─────────────────────────────┘
```

### Rolling Upgrade Example (Node-Level)

```bash
# Rolling upgrade script for self-managed clusters
#!/bin/bash
set -euo pipefail

NODES=$(kubectl get nodes -l node-role.kubernetes.io/worker -o name)
TARGET_VERSION="1.33.0-1.1"

for NODE in $NODES; do
  NODE_NAME=$(echo "$NODE" | cut -d'/' -f2)
  echo "===== Upgrading ${NODE_NAME} ====="

  # Drain
  kubectl drain "${NODE_NAME}" --ignore-daemonsets --delete-emptydir-data --timeout=300s

  # Upgrade (via SSH)
  ssh "${NODE_NAME}" << EOF
    sudo apt-mark unhold kubeadm kubelet kubectl
    sudo apt-get update
    sudo apt-get install -y kubeadm=${TARGET_VERSION} kubelet=${TARGET_VERSION} kubectl=${TARGET_VERSION}
    sudo apt-mark hold kubeadm kubelet kubectl
    sudo kubeadm upgrade node
    sudo systemctl daemon-reload
    sudo systemctl restart kubelet
EOF

  # Uncordon
  kubectl uncordon "${NODE_NAME}"

  # Wait for node to be Ready
  kubectl wait --for=condition=Ready "node/${NODE_NAME}" --timeout=120s

  # Health check: verify pods are rescheduled
  sleep 30
  kubectl get pods -A --field-selector "spec.nodeName=${NODE_NAME}" | grep -v Running | grep -v Completed

  echo "===== ${NODE_NAME} upgraded successfully ====="
done

echo "All worker nodes upgraded to ${TARGET_VERSION}!"
```

### Blue-Green Cluster Upgrade (Application-Level)

```yaml
# blue-green-deployment.yaml
# Kubernetes-native blue-green using Services and label selectors
---
# BLUE Deployment (current version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-blue
  namespace: production
  labels:
    app: web-app
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
      version: blue
  template:
    metadata:
      labels:
        app: web-app
        version: blue
    spec:
      containers:
      - name: web-app
        image: myregistry.io/web-app:v1.0.0
        ports:
        - containerPort: 8080

---
# GREEN Deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-green
  namespace: production
  labels:
    app: web-app
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
      version: green
  template:
    metadata:
      labels:
        app: web-app
        version: green
    spec:
      containers:
      - name: web-app
        image: myregistry.io/web-app:v2.0.0
        ports:
        - containerPort: 8080

---
# Service — switch between blue and green by changing selector
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: production
spec:
  selector:
    app: web-app
    version: blue           # ← Change to "green" to switch traffic
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

```bash
# Switch traffic from blue to green
kubectl patch service web-app -n production \
  -p '{"spec":{"selector":{"version":"green"}}}'

# Verify traffic is hitting green
kubectl get endpoints web-app -n production

# Rollback (if something goes wrong)
kubectl patch service web-app -n production \
  -p '{"spec":{"selector":{"version":"blue"}}}'

# Cleanup blue after validation
kubectl delete deployment web-app-blue -n production
```

### Canary Upgrade with Istio

```yaml
# canary-virtual-service.yaml
# Using Istio to do weighted traffic splitting for canary upgrades
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: web-app
  namespace: production
spec:
  hosts:
  - web-app
  http:
  - route:
    - destination:
        host: web-app
        subset: stable
      weight: 95                   # 95% to current stable version
    - destination:
        host: web-app
        subset: canary
      weight: 5                    # 5% to new canary version

---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: web-app
  namespace: production
spec:
  host: web-app
  subsets:
  - name: stable
    labels:
      version: v1.0.0
  - name: canary
    labels:
      version: v2.0.0
```

---

## Scenario 9 — Deprecated API Migration

> **Use case:** Before upgrading, you must migrate manifests from deprecated/removed API versions.

### Common API Deprecations (Reference Table)

| Old API | New API | Removed In |
|---------|---------|-----------|
| `extensions/v1beta1` Ingress | `networking.k8s.io/v1` Ingress | v1.22 |
| `rbac.authorization.k8s.io/v1beta1` | `rbac.authorization.k8s.io/v1` | v1.22 |
| `apiregistration.k8s.io/v1beta1` | `apiregistration.k8s.io/v1` | v1.22 |
| `policy/v1beta1` PodDisruptionBudget | `policy/v1` PodDisruptionBudget | v1.25 |
| `policy/v1beta1` PodSecurityPolicy | Removed (use Pod Security Standards) | v1.25 |
| `batch/v1beta1` CronJob | `batch/v1` CronJob | v1.25 |
| `autoscaling/v2beta1` HPA | `autoscaling/v2` HPA | v1.26 |
| `flowcontrol.apiserver.k8s.io/v1beta2` | `flowcontrol.apiserver.k8s.io/v1` | v1.32 |

### Scanning for Deprecated APIs

```bash
# Method 1: Using pluto (Fairwinds)
# Install
curl -L -o pluto.tar.gz \
  https://github.com/FairwindsOps/pluto/releases/latest/download/pluto_linux_amd64.tar.gz
tar -xzf pluto.tar.gz
sudo mv pluto /usr/local/bin/

# Scan live cluster
pluto detect-all-in-cluster

# Scan Helm releases
pluto detect-helm

# Scan local manifests
pluto detect -d ./manifests/

# Method 2: Using kubent (Kube No Trouble)
# Install
sh -c "$(curl -sSL https://git.io/install-kubent)"

# Scan cluster
kubent

# Scan files
kubent -f ./manifests/

# Method 3: kubectl with jq — find resources using old APIs
kubectl get apiservices -o json | jq '.items[] | select(.spec.version | test("beta"))'
```

### Migration Examples

```yaml
# ─── BEFORE: Deprecated Ingress (pre-v1.22) ───
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web-app
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: web-app
          servicePort: 80

# ─── AFTER: Current Ingress (v1.22+) ───
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app
spec:
  ingressClassName: nginx          # Replaces annotation
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix           # Required field
        backend:
          service:
            name: web-app
            port:
              number: 80           # Structured format
```

```yaml
# ─── BEFORE: Deprecated HPA (pre-v1.26) ───
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: web-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 70

# ─── AFTER: Current HPA (v1.26+) ───
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70     # Restructured target
```

```yaml
# ─── BEFORE: Deprecated PDB (pre-v1.25) ───
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web-app

# ─── AFTER: Current PDB (v1.25+) ───
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web-app
  # New optional field available in v1:
  unhealthyPodEvictionPolicy: IfHealthy  # Only evict unhealthy pods if healthy ones exist
```

---

## Scenario 10 — CNI & Add-on Upgrades

> **Use case:** Upgrading cluster networking (CNI), DNS (CoreDNS), storage (CSI), and ingress controllers alongside the Kubernetes version.

### CoreDNS Upgrade

```bash
# Check current CoreDNS version
kubectl get deployment coredns -n kube-system -o jsonpath='{.spec.template.spec.containers[0].image}'

# If kubeadm manages CoreDNS, it upgrades automatically with kubeadm upgrade apply
# For manual upgrade:
kubectl set image deployment/coredns \
  -n kube-system \
  coredns=registry.k8s.io/coredns/coredns:v1.11.4

# Verify rollout
kubectl rollout status deployment/coredns -n kube-system
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

### CoreDNS ConfigMap (Production)

```yaml
# coredns-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

### Calico CNI Upgrade

```bash
# Check current Calico version
kubectl calico version 2>/dev/null || \
  kubectl get daemonset calico-node -n kube-system \
    -o jsonpath='{.spec.template.spec.containers[0].image}'

# Upgrade Calico operator
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

# Upgrade Calico custom resources
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml

# Verify
kubectl get pods -n calico-system
kubectl get pods -n tigera-operator
```

### NGINX Ingress Controller Upgrade

```bash
# Using Helm
helm repo update

# Check available versions
helm search repo ingress-nginx/ingress-nginx --versions | head -5

# Upgrade
helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --version 4.12.0 \
  --reuse-values

# Verify
kubectl get pods -n ingress-nginx
kubectl get ingressclass
```

### CSI Driver Upgrade (EBS Example)

```yaml
# aws-ebs-csi-driver upgrade via Helm
# helm upgrade aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
#   --namespace kube-system \
#   --version 2.35.0

# Or if using EKS managed add-on:
# aws eks update-addon \
#   --cluster-name my-cluster \
#   --addon-name aws-ebs-csi-driver \
#   --addon-version v1.37.0-eksbuild.1

# StorageClass remains the same — no changes needed
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

---

## Scenario 11 — Upgrading Across Multiple Minor Versions

> **Use case:** Your cluster is at v1.29 and you need to get to v1.33. You cannot skip — you must upgrade 1.29 → 1.30 → 1.31 → 1.32 → 1.33.

### Sequential Upgrade Plan

```
┌─────────────────────────────────────────────────────────────────┐
│          MULTI-VERSION UPGRADE PATH: v1.29 → v1.33             │
│                                                                 │
│  v1.29.x ──→ v1.30.x ──→ v1.31.x ──→ v1.32.x ──→ v1.33.x    │
│     │           │           │           │           │           │
│   Check       Check       Check       Check       Check        │
│   release     release     release     release     release      │
│   notes       notes       notes       notes       notes        │
│     │           │           │           │           │           │
│   Scan        Scan        Scan        Scan        Scan         │
│   APIs        APIs        APIs        APIs        APIs         │
│     │           │           │           │           │           │
│   Backup      Backup      Backup      Backup      Backup      │
│   etcd        etcd        etcd        etcd        etcd         │
│     │           │           │           │           │           │
│   Upgrade     Upgrade     Upgrade     Upgrade     Upgrade      │
│   & Test      & Test      & Test      & Test      & Test       │
└─────────────────────────────────────────────────────────────────┘
```

### Automated Multi-Version Upgrade Script

```bash
#!/bin/bash
# multi-version-upgrade.sh
# Automates sequential minor version upgrades
# WARNING: Use in staging first! Review each hop carefully.
set -euo pipefail

CURRENT_VERSION=$(kubectl version -o json | jq -r '.serverVersion.minor' | tr -d '+')
TARGET_MINOR=33
PATCH="0-1.1"  # Adjust per your package repo

echo "Current minor version: 1.${CURRENT_VERSION}"
echo "Target minor version: 1.${TARGET_MINOR}"

if [ "$CURRENT_VERSION" -ge "$TARGET_MINOR" ]; then
  echo "Already at or beyond target version. Nothing to do."
  exit 0
fi

for ((MINOR = CURRENT_VERSION + 1; MINOR <= TARGET_MINOR; MINOR++)); do
  VERSION="1.${MINOR}.${PATCH}"
  echo ""
  echo "════════════════════════════════════════════"
  echo " UPGRADING TO v1.${MINOR}.0"
  echo "════════════════════════════════════════════"

  # 1. Backup etcd
  echo "[Step 1/6] Backing up etcd..."
  sudo ETCDCTL_API=3 etcdctl snapshot save "/backup/etcd-pre-1.${MINOR}.db" \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key

  # 2. Upgrade kubeadm
  echo "[Step 2/6] Upgrading kubeadm to ${VERSION}..."
  sudo apt-mark unhold kubeadm
  sudo apt-get update -qq
  sudo apt-get install -y -qq kubeadm="${VERSION}"
  sudo apt-mark hold kubeadm

  # 3. Plan
  echo "[Step 3/6] Running upgrade plan..."
  sudo kubeadm upgrade plan

  # 4. Apply
  echo "[Step 4/6] Applying upgrade..."
  sudo kubeadm upgrade apply "v1.${MINOR}.0" --yes

  # 5. Upgrade kubelet/kubectl
  echo "[Step 5/6] Upgrading kubelet and kubectl..."
  sudo apt-mark unhold kubelet kubectl
  sudo apt-get install -y -qq kubelet="${VERSION}" kubectl="${VERSION}"
  sudo apt-mark hold kubelet kubectl
  sudo systemctl daemon-reload
  sudo systemctl restart kubelet

  # 6. Verify
  echo "[Step 6/6] Verifying..."
  sleep 15
  kubectl get nodes
  kubectl get pods -n kube-system | grep -v Running | grep -v Completed || true

  echo "✓ Control plane upgraded to v1.${MINOR}.0"
  echo "  ⚠️  Don't forget to upgrade worker nodes separately!"
  echo ""
done

echo "════════════════════════════════════════════"
echo " CONTROL PLANE UPGRADE COMPLETE!"
echo " Final version: v1.${TARGET_MINOR}.0"
echo " Next: Upgrade each worker node"
echo "════════════════════════════════════════════"
```

---

## Scenario 12 — Terraform/IaC-Driven Cluster Upgrades

> **Use case:** Managing cluster lifecycle entirely through Infrastructure as Code. The upgrade is a code change, reviewed via PR, and applied through CI/CD.

### GitOps Upgrade Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│              GITOPS CLUSTER UPGRADE WORKFLOW                    │
│                                                                 │
│  Developer                                                      │
│     │                                                           │
│     ├─→ Open PR: "Bump k8s version 1.32 → 1.33"               │
│     │   └── terraform/cluster.tf: version = "1.33"              │
│     │                                                           │
│  CI Pipeline                                                    │
│     │                                                           │
│     ├─→ terraform plan (shows upgrade changes)                  │
│     ├─→ Automated tests (API deprecation scan, PDB check)       │
│     ├─→ Apply to staging cluster                                │
│     ├─→ Run integration tests on staging                        │
│     ├─→ Soak period (24-48 hours)                               │
│     │                                                           │
│  Approval                                                       │
│     │                                                           │
│     ├─→ PR approved by platform team lead                       │
│     ├─→ Merge to main                                           │
│     │                                                           │
│  CD Pipeline                                                    │
│     │                                                           │
│     ├─→ terraform apply (production)                            │
│     ├─→ Monitor rollout (alerts configured)                     │
│     └─→ Post-upgrade validation                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Pre-Upgrade CI Checks (GitHub Actions Example)

```yaml
# .github/workflows/k8s-upgrade-checks.yaml
name: Kubernetes Upgrade Pre-flight Checks

on:
  pull_request:
    paths:
    - 'terraform/**'
    - 'manifests/**'
    - 'helm/**'

jobs:
  deprecated-api-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Install pluto
      run: |
        curl -L -o pluto.tar.gz \
          https://github.com/FairwindsOps/pluto/releases/latest/download/pluto_linux_amd64.tar.gz
        tar -xzf pluto.tar.gz
        sudo mv pluto /usr/local/bin/

    - name: Scan manifests for deprecated APIs
      run: |
        pluto detect -d ./manifests/ \
          --target-versions k8s=v1.33.0 \
          --output markdown

    - name: Scan Helm templates
      run: |
        for chart in helm/*/; do
          echo "Scanning ${chart}..."
          helm template "${chart}" | pluto detect - \
            --target-versions k8s=v1.33.0
        done

  pdb-audit:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Check all Deployments have PDBs
      run: |
        echo "Checking for Deployments without PodDisruptionBudgets..."
        # Custom script to cross-reference deployments and PDBs
        ./scripts/audit-pdbs.sh

  terraform-plan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: 1.9.0

    - name: Terraform Plan
      run: |
        cd terraform/
        terraform init
        terraform plan -out=upgrade.plan

    - name: Show plan summary
      run: |
        cd terraform/
        terraform show -no-color upgrade.plan
```

---

## Production Upgrade Runbook Template

> Copy and customize this runbook for your team. Fill in the blanks for each upgrade cycle.

```markdown
# Kubernetes Upgrade Runbook
## Upgrade: v____ → v____
## Date: ____-____-____
## Owner: ________________

### Pre-Upgrade (Day -7 to Day -1)
- [ ] Review release notes for target version
- [ ] Run deprecated API scan (pluto/kubent)
- [ ] Verify add-on compatibility matrix
- [ ] Test upgrade in staging/dev environment
- [ ] Verify PDBs exist for all production workloads
- [ ] Confirm etcd backup is working and recent
- [ ] Schedule maintenance window
- [ ] Notify stakeholders (email/Slack)
- [ ] Prepare rollback plan

### Upgrade Day — Control Plane
- [ ] Take fresh etcd snapshot
- [ ] Verify etcd snapshot integrity
- [ ] Upgrade kubeadm on first control plane
- [ ] Run kubeadm upgrade plan
- [ ] Run kubeadm upgrade apply
- [ ] Upgrade kubelet and kubectl on first CP
- [ ] systemctl daemon-reload && restart kubelet
- [ ] Verify first CP node version
- [ ] Repeat for additional CP nodes (kubeadm upgrade node)

### Upgrade Day — Worker Nodes
- [ ] Drain worker-1
- [ ] Upgrade kubeadm, kubelet, kubectl on worker-1
- [ ] kubeadm upgrade node on worker-1
- [ ] daemon-reload && restart kubelet on worker-1
- [ ] Uncordon worker-1
- [ ] Verify pods rescheduled successfully
- [ ] (Repeat for each worker node)

### Post-Upgrade Validation
- [ ] All nodes show target version (kubectl get nodes)
- [ ] All system pods running (kubectl get pods -n kube-system)
- [ ] Application health checks passing
- [ ] Monitoring dashboards normal
- [ ] No increase in error rates
- [ ] DNS resolution working
- [ ] Ingress/Service connectivity verified
- [ ] PVs/PVCs bound and accessible

### Rollback Plan
- [ ] If control plane fails: restore etcd from snapshot
- [ ] If worker fails: uncordon and downgrade packages
- [ ] If application fails: roll back deployment to previous image
- [ ] Emergency contact: ________________
```

---

## Troubleshooting Common Upgrade Failures

### Problem 1: kubelet Fails to Start After Upgrade

```bash
# Symptoms: Node shows NotReady, kubelet service failing
# Root cause: systemd didn't reload the new unit file

# Fix:
sudo systemctl daemon-reload    # ← ALWAYS do this first
sudo systemctl restart kubelet

# Check logs:
sudo journalctl -u kubelet -f --no-pager | tail -50

# If kubelet config is wrong:
sudo cat /var/lib/kubelet/config.yaml
```

### Problem 2: kubeadm upgrade apply Fails Midway

```bash
# Symptoms: Upgrade started but didn't complete
# The cluster is in a mixed state

# Check what actually got upgraded:
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide

# Retry the upgrade (safe to re-run):
sudo kubeadm upgrade apply v1.33.0 --force

# If etcd is corrupted, restore from backup (see Scenario 3)
```

### Problem 3: Pods Stuck in Pending After Worker Drain

```bash
# Symptoms: After draining a worker, pods are Pending on other nodes
# Root cause: Not enough capacity on remaining nodes

# Check:
kubectl describe pod <pending-pod> | grep -A5 Events
kubectl top nodes

# Fixes:
# 1. Add temporary capacity
kubectl uncordon <another-node>

# 2. Check resource requests — may need to scale down non-critical workloads
kubectl get pods -A -o json | jq '[.items[] | select(.status.phase=="Pending") | {name: .metadata.name, namespace: .metadata.namespace}]'

# 3. Reduce resource requests temporarily
kubectl set resources deployment/<name> -c <container> --requests=cpu=100m,memory=128Mi
```

### Problem 4: CoreDNS CrashLooping After Upgrade

```bash
# Symptoms: CoreDNS pods restarting, DNS resolution fails
# Root cause: Incompatible Corefile or plugin version

# Check logs:
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Common fix: Reset Corefile to default
kubectl get configmap coredns -n kube-system -o yaml

# If the issue is a deprecated plugin in Corefile:
kubectl edit configmap coredns -n kube-system
# Remove or update the offending plugin line
# Then restart CoreDNS:
kubectl rollout restart deployment coredns -n kube-system
```

### Problem 5: Version Skew Between Nodes

```bash
# Symptoms: Some nodes upgraded, some not. API server rejecting requests.
# Root cause: Exceeded allowed version skew

# Check all versions:
kubectl get nodes -o custom-columns=\
  NAME:.metadata.name,\
  VERSION:.status.nodeInfo.kubeletVersion,\
  OS:.status.nodeInfo.osImage

# Fix: Continue upgrading remaining nodes to bring them within skew policy
# kubelet can be at most N-3 of apiserver
```

### Problem 6: PDB Blocking Drain

```bash
# Symptoms: kubectl drain hangs, cannot evict pods
# Root cause: PDB prevents eviction because it would violate minAvailable/maxUnavailable

# Check which PDB is blocking:
kubectl get pdb -A
kubectl describe pdb <name> -n <namespace>

# Options:
# 1. Scale up the deployment so more replicas are available
kubectl scale deployment <name> -n <namespace> --replicas=4

# 2. Temporarily relax the PDB (risky in production!)
kubectl delete pdb <name> -n <namespace>
# ... drain and upgrade ...
kubectl apply -f pdb.yaml  # Re-apply after

# 3. Use --disable-eviction flag (Kubernetes 1.26+)
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --disable-eviction
# ⚠️ This bypasses PDBs entirely — use as last resort
```

---

## Quick Reference Cheat Sheet

```bash
# ═══════════════════════════════════════════════════════════
# KUBERNETES UPGRADE CHEAT SHEET
# ═══════════════════════════════════════════════════════════

# ─── CHECK VERSIONS ───
kubectl version --short                       # Client + server
kubeadm version                               # kubeadm
kubelet --version                             # kubelet
kubectl get nodes -o wide                     # All node versions

# ─── KUBEADM UPGRADE (Control Plane) ───
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm=X.Y.Z-R && \
sudo apt-mark hold kubeadm
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply vX.Y.Z
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get install -y kubelet=X.Y.Z-R kubectl=X.Y.Z-R && \
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# ─── KUBEADM UPGRADE (Worker) ───
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# (SSH to worker)
sudo kubeadm upgrade node                     # NOT "apply"!
sudo systemctl daemon-reload && sudo systemctl restart kubelet
# (Exit SSH)
kubectl uncordon <node>

# ─── ETCD BACKUP ───
ETCDCTL_API=3 etcdctl snapshot save /backup/snap.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# ─── ETCD RESTORE ───
ETCDCTL_API=3 etcdctl snapshot restore /backup/snap.db \
  --data-dir=/var/lib/etcd-restored

# ─── EKS UPGRADE ───
aws eks update-cluster-version --name CLUSTER --kubernetes-version X.Y
aws eks update-addon --cluster-name CLUSTER --addon-name ADDON --addon-version VER
aws eks update-nodegroup-version --cluster-name CLUSTER --nodegroup-name NG

# ─── AKS UPGRADE ───
az aks get-upgrades --resource-group RG --name CLUSTER -o table
az aks upgrade --resource-group RG --name CLUSTER --kubernetes-version X.Y.Z
az aks nodepool update --cluster-name CLUSTER --name POOL --max-surge 33%

# ─── GKE UPGRADE ───
gcloud container clusters upgrade CLUSTER --master --cluster-version VER
gcloud container clusters upgrade CLUSTER --node-pool POOL --cluster-version VER
gcloud container clusters update CLUSTER --release-channel stable

# ─── SCAN DEPRECATED APIs ───
pluto detect-all-in-cluster
kubent

# ─── PDB CHECK ───
kubectl get pdb -A

# ─── DRAIN TROUBLESHOOTING ───
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --timeout=300s
kubectl drain <node> --disable-eviction        # Bypass PDBs (last resort)
```

---

> **Remember:** The golden rule of Kubernetes upgrades is **always upgrade one minor version at a time**, **always control plane first**, and **always backup etcd before you start**. Test in staging before touching production. Every. Single. Time.
