# CKA Exam Domain Coverage Map

This matrix aligns exercises with official CKA exam domains and weighted percentages. Use this to track your preparation coverage.

## Official CKA Domains

| Domain | Official weight | Exercises | Notes |
|---|---:|---|---|
| **Storage** | 10% | 12, 25 | PV/PVC, StorageClass, and binding |
| **Troubleshooting** | 30% | 11, 17, 29 | Cluster, node, and workload troubleshooting |
| **Workloads & Scheduling** | 15% | 01, 02, 03, 06, 07, 10, 16, 20, 21, 22, 23, 24 | Pods, admission, workload controllers, autoscaling, and scheduling |
| **Cluster Architecture, Installation & Configuration** | 25% | 04, 08, 09, 13, 14, 27 | RBAC, kubeadm, cluster lifecycle, and extensions |
| **Services & Networking** | 20% | 05, 15, 19, 28, 32 | Services, NetworkPolicy, Ingress, Gateway API, and EndpointSlices |

The weights are official domain weights, not a promise about the number of questions or minutes assigned to each domain. See the [Linux Foundation CKA FAQ](https://docs.linuxfoundation.org/tc-docs/certification/faq-cka-ckad-cks) and the [current CNCF curriculum](https://github.com/cncf/curriculum/blob/master/CKA_Curriculum_v1.37.pdf).


**Total Estimated Time: 15-21 hours of hands-on practice**

---

## Exam Readiness Checklist

### ✓ Storage (10%)
- [ ] Exercise 12: PV, PVC, StorageClass basics
- [ ] Exercise 25: WaitForFirstConsumer binding
- **Gotchas:** Namespace mismatch, storageClassName exactness, WFFC timing

### ✓ Troubleshooting (30%)
- [ ] Exercise 11: Cluster troubleshooting scenarios
- [ ] Exercise 17: kubectl debug usage
- [ ] Exercise 29: etcd endpoint troubleshooting
- **Gotchas:** Log locations, API server audit logs, certificate paths

### ✓ Workloads & Scheduling (15%)
- [ ] Exercise 02: Multi-container pods
- [ ] Exercise 06: Deployment rollout/rollback
- [ ] Exercise 07: StatefulSet fundamentals
- [ ] Exercise 16: HPA with metrics-server
- [ ] Exercise 20: Pod Security Standards and namespace admission
- [ ] Exercise 21: Jobs and CronJobs
- [ ] Exercise 22: PriorityClass usage
- [ ] Exercise 23: Resource requests/limits, tuning
- **Gotchas:** HPA target.type vs averageUtilization, initContainers vs sidecar, Pod disruption budgets

### ✓ Services & Networking (20%)
- [ ] Exercise 05: NetworkPolicy basics
- [ ] Exercise 19: Ingress configuration
- [ ] Exercise 28: Multi-namespace NetworkPolicy, CNI awareness
- [ ] Exercise 32: Service selector and EndpointSlice troubleshooting
- **Gotchas:** Egress + ingress both required, DNS rules critical, label case-sensitivity, CNI enforcement variation

### ✓ Cluster Architecture, Installation & Configuration (25%)
- [ ] Exercise 04: RBAC (Roles, ClusterRoles, bindings)
- [ ] Exercise 08: Node drain/cordon
- [ ] Exercise 09: kubeadm cluster upgrade
- [ ] Exercise 10: Static pods
- [ ] Exercise 13: Helm chart install/upgrade
- [ ] Exercise 24: PriorityClass patch
- [ ] Exercise 27: CNI Tigera (Calico) install
- **Gotchas:** kubelet config is persistent, kubeadm token expiry (24h), RBAC namespace scoping, static pod node name suffix

### Supplemental Implementation Practice
- [ ] Exercises 18 and 26: Docker and CRI-dockerd installation
- [ ] Exercise 30: Application TLS configuration
- [ ] Exercise 31: Argo CD installation and GitOps setup

---

## Study Path Recommendations

### **Path A: Exam Simulation (Realistic Order)**
Follow official CKA domain weights → mock exams → focus weak areas

1. Start with **Cluster Architecture** (25%) — sets up cluster fundamentals
2. Then **Workloads & Scheduling** (15%) — workloads and scheduling tasks
3. Then **Services & Networking** (20%) — connectivity scenarios
4. Then **Troubleshooting** (30%) — debugging existing clusters
5. Finally **Storage** (10%) — often time-permitting on exams
6. Use mock exams after each domain

### **Path B: Hands-On Learning (Sequential)**
Follow exercises in order with increasing complexity

1. **Basic Concepts:** 01-03 (Pods, multi-container, ConfigMap)
2. **Cluster Setup:** 04, 08-10, 18, 26-27 (RBAC, nodes, kubeadm, CRI, CNI)
3. **Workloads:** 02, 06-07, 21-23 (Deployments, StatefulSets, Jobs, Priority, HPA)
4. **Networking:** 05, 19, 28, 32 (Services, Ingress, NetworkPolicy, EndpointSlices)
5. **Storage:** 12, 25 (PV/PVC, WFFC)
6. **Advanced:** 13-15, 24, 29-31 (Helm, Kustomize, Gateway API, upgrades, troubleshooting)

### **Path C: Targeted Weak Areas**
Use the matrix to focus on your weakest domain(s)

- Weak at **Storage**? → 12 → 25 → mock exams targeting storage questions
- Weak at **RBAC**? → 04 → 09 (kubeadm) → mock exams
- Weak at **Networking**? → 05 → 28 → 19 → 32 → mock exams

---

## Illustrative Time Equivalents

These figures apply the official domain percentages to a 120-minute exam for planning only. They are not per-domain time limits from the exam provider.

| Domain | Equivalent time out of 120 minutes | Exercise Prep | Total |
|--------|-----------|---------------|-------|
| Storage | 12 min | 2-3h | 2.5-3.5h |
| Troubleshooting | 36 min | 2-3h | 2.5-3.5h |
| Workloads & Scheduling | 18 min | 3-4h | 3.5-4.5h |
| Services & Networking | 24 min | 2-3h | 2.5-3.5h |
| Cluster Architecture, Installation & Configuration | 30 min | 4-5h | 4.5-5.5h |
| **TOTAL** | **120 min** | **13-18h** | **15-21h** |

Use this only to compare domain weights. In the exam, manage time by task and move on when stuck.

---

## Assessment: Am I Ready?

### 🟢 Green (Exam Ready)
- Completed all exercises ✓
- Passed all mock exams (70%+ score)
- Can complete 2-3 exercises under timed conditions (15 min per exercise)
- Understand all gotchas in "What tripped me up" sections

### 🟡 Yellow (Almost Ready)
- Completed most exercises
- Struggled with 1-2 domains
- Understand concepts but slow on execution
- **Action:** Focus on timed drills, mock exams

### 🔴 Red (Need More Work)
- Incomplete exercises in 2+ domains
- Struggling with RBAC or networking concepts
- Haven't run mock exams yet
- **Action:** Go back to foundational exercises, don't rush to mock exams
