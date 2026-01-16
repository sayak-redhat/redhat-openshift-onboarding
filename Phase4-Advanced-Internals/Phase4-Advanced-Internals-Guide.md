# 📚 Phase 4: Advanced Internals Study Guide
## Deep Dive into Kubernetes & OpenShift Architecture

---

## Table of Contents

1. [Module 1: API Server Deep Dive](#module-1-api-server-deep-dive)
2. [Module 2: etcd - The Brain of Kubernetes](#module-2-etcd---the-brain-of-kubernetes)
3. [Module 3: Scheduler Internals](#module-3-scheduler-internals)
4. [Module 4: Controller Manager](#module-4-controller-manager)
5. [Module 5: Networking Internals (CNI)](#module-5-networking-internals-cni)
6. [Module 6: Storage Internals (CSI)](#module-6-storage-internals-csi)
7. [Module 7: Advanced Debugging](#module-7-advanced-debugging)
8. [Module 8: Performance & Troubleshooting](#module-8-performance--troubleshooting)

---

# Module 1: API Server Deep Dive

## 1.1 What is the API Server?

**Simple Explanation**: The API Server is the front door to Kubernetes. Every request (from kubectl, operators, kubelet) goes through it. It validates, processes, and stores requests in etcd.

**Real-life Analogy**: The API Server is like a bank teller. Everyone who wants to do something with their money (resources) has to go through the teller (API Server), who validates their identity, checks their permissions, and processes their request.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          API Server Request Flow                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   kubectl apply -f pod.yaml                                                  │
│         │                                                                    │
│         │ HTTPS request                                                      │
│         ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                       1. AUTHENTICATION                              │   │
│   │                       "Who are you?"                                 │   │
│   │                                                                      │   │
│   │   Methods:                                                           │   │
│   │   • X.509 client certificates                                       │   │
│   │   • Bearer tokens (ServiceAccount tokens)                           │   │
│   │   • OpenID Connect (OAuth)                                          │   │
│   │   • Webhook authentication                                          │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         │ User: "sayadas", Groups: ["developers"]                           │
│         ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                       2. AUTHORIZATION                               │   │
│   │                       "Can you do this?"                             │   │
│   │                                                                      │   │
│   │   Checks RBAC:                                                       │   │
│   │   • Does user have Role/ClusterRole?                                │   │
│   │   • Does role allow "create" on "pods"?                             │   │
│   │   • Is action allowed in this namespace?                            │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         │ Authorized!                                                        │
│         ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                     3. ADMISSION CONTROL                             │   │
│   │                     "Should we allow this?"                          │   │
│   │                                                                      │   │
│   │   ┌─────────────────┐    ┌─────────────────┐                        │   │
│   │   │   Mutating      │ →  │   Validating    │                        │   │
│   │   │   Webhooks      │    │   Webhooks      │                        │   │
│   │   │                 │    │                 │                        │   │
│   │   │ • Add defaults  │    │ • Check policy  │                        │   │
│   │   │ • Inject sidecar│    │ • Validate spec │                        │   │
│   │   │ • Add labels    │    │ • Deny bad pods │                        │   │
│   │   └─────────────────┘    └─────────────────┘                        │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         │ Admitted!                                                          │
│         ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                          4. STORAGE                                  │   │
│   │                          "Store in etcd"                             │   │
│   │                                                                      │   │
│   │   Pod object serialized and stored in etcd                          │   │
│   │   Key: /registry/pods/namespace/pod-name                            │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         │ Stored!                                                            │
│         ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                       5. NOTIFICATION                                │   │
│   │                       "Tell watchers"                                │   │
│   │                                                                      │   │
│   │   Controllers/kubelet watching "pods" get notified:                 │   │
│   │   "New pod created! Go do something!"                               │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 1.2 Admission Controllers

Admission controllers are plugins that intercept requests AFTER authentication/authorization but BEFORE the object is stored.

### Built-in Admission Controllers

| Controller | What it does |
|------------|--------------|
| **NamespaceLifecycle** | Prevents operations in non-existent namespaces |
| **LimitRanger** | Enforces default limits on resources |
| **ServiceAccount** | Auto-creates ServiceAccount and token |
| **PodSecurity** | Enforces Pod Security Standards |
| **ResourceQuota** | Enforces namespace quotas |
| **MutatingAdmissionWebhook** | Calls external webhooks to modify |
| **ValidatingAdmissionWebhook** | Calls external webhooks to validate |

### How Webhooks Work

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Admission Webhook Flow                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. Request comes in: "Create Pod X"                                        │
│         │                                                                    │
│         ▼                                                                    │
│   2. API Server calls Mutating Webhooks                                      │
│      ┌─────────────────────────────────────────────────────────────────┐    │
│      │  Webhook 1: Istio Sidecar Injector                              │    │
│      │  "Add istio-proxy container to this pod"                        │    │
│      │                                                                  │    │
│      │  Webhook 2: SPIFFE CSI Driver Webhook                           │    │
│      │  "Add CSI volume for SPIFFE workload API"                       │    │
│      └─────────────────────────────────────────────────────────────────┘    │
│         │                                                                    │
│         ▼ (Pod now modified with sidecars/volumes)                          │
│                                                                              │
│   3. API Server calls Validating Webhooks                                    │
│      ┌─────────────────────────────────────────────────────────────────┐    │
│      │  Webhook 1: OPA/Gatekeeper                                      │    │
│      │  "Check: Is image from allowed registry?"                       │    │
│      │  "Check: Does pod have required labels?"                        │    │
│      │                                                                  │    │
│      │  Webhook 2: Security Policy Webhook                             │    │
│      │  "Check: Is pod not running as root?"                           │    │
│      └─────────────────────────────────────────────────────────────────┘    │
│         │                                                                    │
│         ▼ (All validations passed)                                          │
│                                                                              │
│   4. Store in etcd                                                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 1.3 Watching API Server

```bash
# Enable verbose output to see API calls
kubectl get pods -v=8

# Sample output shows:
# GET https://api.cluster.example.com:6443/api/v1/namespaces/default/pods
# Request Headers:
#   Accept: application/json
#   User-Agent: kubectl/v1.28.0
# Response Status: 200 OK

# Watch API server metrics
oc get --raw /metrics | grep apiserver_request

# Check API server logs (on master node)
oc debug node/<master-node>
chroot /host
crictl logs $(crictl ps --name=kube-apiserver -q)

# Check enabled admission plugins
oc get cm -n openshift-kube-apiserver config -o yaml | grep -A 50 "admission"
```

---

# Module 2: etcd - The Brain of Kubernetes

## 2.1 What is etcd?

**Simple Explanation**: etcd is a distributed key-value database that stores ALL Kubernetes state. Every pod, service, secret, configmap - everything is stored in etcd.

**Real-life Analogy**: etcd is like the hotel's master registry book. It knows every guest (pod), every room assignment (scheduling), every special request (configs). If you lose this book, the hotel can't function.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           etcd Data Model                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   etcd stores data as key-value pairs organized hierarchically:             │
│                                                                              │
│   /registry/                                                                 │
│   ├── pods/                                                                  │
│   │   ├── default/                                                          │
│   │   │   ├── nginx-abc123    → {pod spec JSON}                            │
│   │   │   └── redis-xyz789    → {pod spec JSON}                            │
│   │   └── kube-system/                                                       │
│   │       └── coredns-xxx     → {pod spec JSON}                            │
│   │                                                                          │
│   ├── services/                                                              │
│   │   └── default/                                                          │
│   │       └── kubernetes      → {service spec JSON}                         │
│   │                                                                          │
│   ├── secrets/                                                               │
│   │   └── default/                                                          │
│   │       └── my-secret       → {secret data JSON}                          │
│   │                                                                          │
│   ├── configmaps/                                                            │
│   │   └── kube-system/                                                       │
│   │       └── kube-proxy      → {configmap data JSON}                       │
│   │                                                                          │
│   └── customresources/                                                       │
│       └── spireservers/                                                      │
│           └── cluster         → {spireserver spec JSON}                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 2.2 etcd Cluster Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      etcd Cluster (3 nodes for HA)                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐         │
│   │   etcd-0        │    │   etcd-1        │    │   etcd-2        │         │
│   │   (Leader)      │◄──►│   (Follower)    │◄──►│   (Follower)    │         │
│   │                 │    │                 │    │                 │         │
│   │   Handles all   │    │   Replicates    │    │   Replicates    │         │
│   │   writes        │    │   from leader   │    │   from leader   │         │
│   └─────────────────┘    └─────────────────┘    └─────────────────┘         │
│            │                     │                     │                     │
│            └─────────────────────┼─────────────────────┘                     │
│                                  │                                           │
│                          Raft Consensus                                      │
│                                                                              │
│   • Writes go to leader, replicated to followers                            │
│   • Reads can go to any node                                                 │
│   • Quorum (2 of 3) required for writes                                     │
│   • If leader fails, new election happens                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 2.3 etcd Operations

```bash
# Access etcd on OpenShift
oc debug node/<master-node>
chroot /host

# Find etcd pod
crictl ps | grep etcd

# Access etcd shell
oc exec -it etcd-<master-node> -n openshift-etcd -- /bin/sh

# Inside etcd pod, use etcdctl
etcdctl --endpoints=https://localhost:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status

# List keys (careful - don't do in production!)
etcdctl ... get / --prefix --keys-only | head -50

# Check cluster health
etcdctl ... endpoint health

# Check cluster member list
etcdctl ... member list

# Defragment (maintenance)
etcdctl ... defrag

# Check database size
etcdctl ... endpoint status --write-out=table
```

## 2.4 etcd Backup & Restore

```bash
# Backup etcd (CRITICAL for disaster recovery!)
etcdctl snapshot save /backup/etcd-snapshot.db

# Verify backup
etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table

# Restore (emergency only!)
etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore

# OpenShift specific backup
oc get etcdbackups -n openshift-etcd

# Create backup via OpenShift
cat <<EOF | oc apply -f -
apiVersion: operator.openshift.io/v1alpha1
kind: EtcdBackup
metadata:
  name: backup-$(date +%Y%m%d)
  namespace: openshift-etcd
spec:
  pvcName: etcd-backup-pvc
EOF
```

---

# Module 3: Scheduler Internals

## 3.1 How the Scheduler Works

**Simple Explanation**: The Scheduler decides which node should run a pod. It finds nodes that CAN run the pod, scores them, and picks the best one.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Scheduler Decision Process                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   New Pod (Pending, no node assigned)                                        │
│         │                                                                    │
│         ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    1. FILTERING                                      │   │
│   │                    "Which nodes CAN run this pod?"                   │   │
│   │                                                                      │   │
│   │   Checks:                                                            │   │
│   │   ├── NodeSelector: Does node have required labels?                 │   │
│   │   ├── Resources: Does node have enough CPU/memory?                  │   │
│   │   ├── Taints: Does pod tolerate node's taints?                      │   │
│   │   ├── Affinity: Does pod's affinity match node?                     │   │
│   │   ├── Ports: Is required hostPort available?                        │   │
│   │   └── Volume: Can required volumes be mounted?                      │   │
│   │                                                                      │   │
│   │   Available Nodes: [node-1, node-2, node-3]                         │   │
│   │   After filtering: [node-1, node-3]  (node-2 failed resource check) │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    2. SCORING                                        │   │
│   │                    "Which node is BEST?"                             │   │
│   │                                                                      │   │
│   │   Scoring plugins (each gives 0-100):                               │   │
│   │   ├── LeastRequestedPriority: More free resources = higher score   │   │
│   │   ├── BalancedResourceAllocation: Even CPU/memory = higher score   │   │
│   │   ├── NodeAffinity: Preferred affinity match = higher score        │   │
│   │   ├── InterPodAffinity: Pod affinity match = higher score          │   │
│   │   └── ImageLocality: Image already on node = higher score          │   │
│   │                                                                      │   │
│   │   node-1: 85 points                                                  │   │
│   │   node-3: 72 points                                                  │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    3. BINDING                                        │   │
│   │                    "Assign pod to winning node"                      │   │
│   │                                                                      │   │
│   │   Winner: node-1 (85 points)                                        │   │
│   │   Pod updated: spec.nodeName = "node-1"                             │   │
│   │   kubelet on node-1 sees pod and starts it                          │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 3.2 Scheduling Constraints

### Node Selector

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  nodeSelector:
    gpu: "true"           # Only schedule on nodes with label gpu=true
  containers:
  - name: app
    image: my-gpu-app
```

### Affinity & Anti-Affinity

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  affinity:
    # Node affinity - prefer certain nodes
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: zone
            operator: In
            values: ["us-west-1a", "us-west-1b"]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: type
            operator: In
            values: ["high-memory"]
    
    # Pod affinity - schedule near other pods
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: cache
        topologyKey: "kubernetes.io/hostname"
    
    # Pod anti-affinity - spread pods apart
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web
          topologyKey: "kubernetes.io/hostname"
```

### Taints and Tolerations

```bash
# Add taint to node (repels pods)
oc adm taint nodes node-1 dedicated=database:NoSchedule

# Pod must have toleration to run on tainted node
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-pod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
```

## 3.3 Debugging Scheduling

```bash
# See why pod is pending
oc describe pod pending-pod | grep -A 20 "Events:"

# Common messages:
# - "Insufficient cpu" - no node has enough CPU
# - "Insufficient memory" - no node has enough memory
# - "node(s) didn't match node selector" - no node has required label
# - "node(s) had taint that pod didn't tolerate"

# Check node capacity
oc describe node node-1 | grep -A 10 "Allocatable:"

# Check what's using resources on node
oc describe node node-1 | grep -A 30 "Non-terminated Pods:"

# Check scheduler logs
oc logs -n openshift-kube-scheduler kube-scheduler-<master>
```

---

# Module 4: Controller Manager

## 4.1 What is the Controller Manager?

**Simple Explanation**: The Controller Manager runs all the built-in controllers that maintain cluster state. Each controller watches specific resources and makes changes to achieve desired state.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Built-in Controllers                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                 Deployment Controller                                │   │
│   │                                                                      │   │
│   │   Watches: Deployments                                              │   │
│   │   Creates: ReplicaSets                                              │   │
│   │                                                                      │   │
│   │   "User wants 3 replicas → Create/update ReplicaSet"               │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                 ReplicaSet Controller                                │   │
│   │                                                                      │   │
│   │   Watches: ReplicaSets                                              │   │
│   │   Creates: Pods                                                      │   │
│   │                                                                      │   │
│   │   "ReplicaSet wants 3 pods, only 2 exist → Create 1 more pod"      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                 Node Controller                                      │   │
│   │                                                                      │   │
│   │   Watches: Nodes                                                     │   │
│   │   Actions: Mark unhealthy, evict pods                               │   │
│   │                                                                      │   │
│   │   "Node hasn't reported in 5 min → Mark NotReady, evict pods"      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                 Service Controller                                   │   │
│   │                                                                      │   │
│   │   Watches: Services (type LoadBalancer)                             │   │
│   │   Creates: Cloud load balancers                                     │   │
│   │                                                                      │   │
│   │   "Service type=LoadBalancer → Create AWS ELB"                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                 Endpoint Controller                                  │   │
│   │                                                                      │   │
│   │   Watches: Services, Pods                                           │   │
│   │   Creates: Endpoints                                                 │   │
│   │                                                                      │   │
│   │   "Service selects pods with app=nginx → Update endpoints list"    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   And many more: Job, CronJob, StatefulSet, DaemonSet, Garbage Collection  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# Module 5: Networking Internals (CNI)

## 5.1 Container Network Interface (CNI)

**Simple Explanation**: CNI is a standard for how container runtimes configure networking for containers. OpenShift uses OVN-Kubernetes as its CNI plugin.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CNI Plugin Flow                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Pod created → kubelet calls CNI plugin → Network configured                │
│                                                                              │
│   1. kubelet: "I need to start pod X"                                        │
│         │                                                                    │
│         ▼                                                                    │
│   2. CNI plugin: "I'll create network namespace"                            │
│         │                                                                    │
│         ▼                                                                    │
│   3. CNI plugin: "I'll create veth pair (virtual ethernet)"                 │
│      ┌─────────────────────────────────────────────────────────────────┐    │
│      │  Pod namespace          │    Host namespace                     │    │
│      │                         │                                        │    │
│      │   eth0 ◄───────────────►│  vethXXX                              │    │
│      │   10.128.1.5            │    │                                   │    │
│      │                         │    ▼                                   │    │
│      │                         │  OVS Bridge                           │    │
│      │                         │    │                                   │    │
│      │                         │    ▼                                   │    │
│      │                         │  Physical NIC                         │    │
│      └─────────────────────────────────────────────────────────────────┘    │
│         │                                                                    │
│         ▼                                                                    │
│   4. CNI plugin: "I'll assign IP from IPAM (10.128.1.5)"                    │
│         │                                                                    │
│         ▼                                                                    │
│   5. CNI plugin: "I'll configure routes and iptables"                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 5.2 OVN-Kubernetes Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      OVN-Kubernetes Components                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Control Plane:                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  OVN-Kubernetes Master                                              │   │
│   │  • Watches Kubernetes resources (pods, services, network policies)  │   │
│   │  • Programs OVN northbound database                                 │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  OVN Northbound Database                                            │   │
│   │  • Logical network topology                                         │   │
│   │  • Logical switches, routers, ACLs                                  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼ (ovn-northd translates to southbound)                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  OVN Southbound Database                                            │   │
│   │  • Physical network bindings                                        │   │
│   │  • Flow rules                                                        │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│   Node Level:                                                                │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  ovn-controller (per node)                                          │   │
│   │  • Reads southbound database                                        │   │
│   │  • Programs OVS flows                                               │   │
│   │                                                                      │   │
│   │  Open vSwitch (OVS)                                                 │   │
│   │  • Actual packet forwarding                                         │   │
│   │  • Implements tunnels (GENEVE)                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 5.3 Network Debugging

```bash
# Check pod network
oc exec my-pod -- ip addr
oc exec my-pod -- ip route
oc exec my-pod -- cat /etc/resolv.conf

# Check DNS resolution
oc exec my-pod -- nslookup kubernetes.default.svc.cluster.local

# Check connectivity
oc exec my-pod -- curl -v http://other-service:8080

# Debug network from node
oc debug node/node-1
chroot /host
ovs-vsctl show
ovs-ofctl dump-flows br-int | head -50

# Check OVN pods
oc get pods -n openshift-ovn-kubernetes

# Check network policy
oc get networkpolicy -n my-namespace

# TCP dump (on node)
tcpdump -i any port 8080
```

---

# Module 6: Storage Internals (CSI)

## 6.1 Container Storage Interface (CSI)

**Simple Explanation**: CSI is a standard for how Kubernetes talks to storage systems. Each storage provider has a CSI driver (AWS EBS, GCP PD, NFS, etc.).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CSI Architecture                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   PVC created → CSI Provisioner → Storage Backend → PV created → Mounted    │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    CSI Controller (Deployment)                       │   │
│   │                                                                      │   │
│   │   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │   │
│   │   │  Provisioner    │  │   Attacher      │  │   Resizer       │    │   │
│   │   │                 │  │                 │  │                 │    │   │
│   │   │  Creates/deletes│  │  Attaches volume│  │  Resizes volume │    │   │
│   │   │  volumes        │  │  to nodes       │  │                 │    │   │
│   │   └─────────────────┘  └─────────────────┘  └─────────────────┘    │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              │ gRPC calls                                    │
│                              ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    CSI Driver Container                              │   │
│   │                                                                      │   │
│   │   Implements CSI spec:                                               │   │
│   │   • CreateVolume / DeleteVolume                                     │   │
│   │   • ControllerPublishVolume / ControllerUnpublishVolume            │   │
│   │   • NodeStageVolume / NodeUnstageVolume                             │   │
│   │   • NodePublishVolume / NodeUnpublishVolume                         │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              │ API calls                                     │
│                              ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    Storage Backend                                   │   │
│   │                                                                      │   │
│   │   AWS EBS, GCP PD, Azure Disk, NFS, Ceph, etc.                     │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 6.2 SPIFFE CSI Driver (What you used!)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SPIFFE CSI Driver                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   When pod mounts:                                                           │
│   volumes:                                                                   │
│   - name: spiffe-workload-api                                               │
│     csi:                                                                     │
│       driver: csi.spiffe.io                                                 │
│       readOnly: true                                                         │
│                                                                              │
│   SPIFFE CSI Driver:                                                         │
│   1. Creates a directory for this pod                                        │
│   2. Creates a Unix socket in that directory                                │
│   3. Connects socket to SPIRE Agent's Workload API                          │
│   4. Mounts directory into pod at specified path                            │
│                                                                              │
│   Result in pod:                                                             │
│   /spiffe-workload-api/                                                      │
│   └── spire-agent.sock   ← Unix socket to SPIRE Agent                       │
│                                                                              │
│   Pod can then:                                                              │
│   • Request SVIDs via Workload API                                          │
│   • Get trust bundles                                                        │
│   • Validate other workloads' identities                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# Module 7: Advanced Debugging

## 7.1 Systematic Debugging Approach

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Debugging Checklist                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. DESCRIBE - What does Kubernetes think is happening?                    │
│      oc describe pod/deployment/service <name>                              │
│      Look at: Events, Conditions, Status                                    │
│                                                                              │
│   2. LOGS - What is the application saying?                                 │
│      oc logs <pod> -c <container>                                           │
│      oc logs <pod> --previous   (crashed container)                         │
│                                                                              │
│   3. EVENTS - What happened recently?                                       │
│      oc get events --sort-by='.lastTimestamp'                               │
│                                                                              │
│   4. EXEC - What's happening inside?                                        │
│      oc exec -it <pod> -- /bin/bash                                         │
│      Check: processes, files, network, env                                  │
│                                                                              │
│   5. DEBUG - Access the node                                                │
│      oc debug node/<node>                                                   │
│      Check: kubelet, container runtime, system resources                    │
│                                                                              │
│   6. API - Is the object correct?                                           │
│      oc get <resource> -o yaml                                              │
│      Check: spec, status, annotations                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 7.2 Common Issues and Solutions

### Pod Stuck in Pending

```bash
# Check why
oc describe pod <pod-name> | grep -A 20 "Events:"

# Possible causes:
# 1. Insufficient resources
oc describe node | grep -A 10 "Allocated resources:"

# 2. No nodes match selector
oc get nodes --show-labels | grep <required-label>

# 3. PVC pending
oc get pvc
oc describe pvc <pvc-name>

# 4. Image pull issues
oc describe pod <pod-name> | grep "Failed to pull"
```

### Pod Stuck in CrashLoopBackOff

```bash
# Check logs
oc logs <pod-name> --previous

# Check exit code
oc get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'

# Common causes:
# Exit 1: Application error
# Exit 137: OOMKilled (memory limit hit)
# Exit 139: Segfault

# Check if OOMKilled
oc describe pod <pod-name> | grep OOMKilled
```

### Service Not Reaching Pods

```bash
# Check endpoints
oc get endpoints <service-name>
# If empty, selector doesn't match any pods

# Check labels match
oc get svc <service-name> -o jsonpath='{.spec.selector}'
oc get pods --show-labels

# Test from inside cluster
oc run debug --rm -it --image=busybox -- wget -qO- http://<service>:port

# Check NetworkPolicy
oc get networkpolicy -n <namespace>
```

## 7.3 Debug Commands Reference

```bash
# ═══════════════════════════════════════════════════════════════════════════
# POD DEBUGGING
# ═══════════════════════════════════════════════════════════════════════════

# Describe pod
oc describe pod <pod>

# Get logs
oc logs <pod>
oc logs <pod> -c <container>
oc logs <pod> --previous
oc logs <pod> -f               # Follow
oc logs <pod> --tail=100       # Last 100 lines
oc logs <pod> --since=1h       # Last hour

# Execute in pod
oc exec -it <pod> -- /bin/bash
oc exec <pod> -- cat /app/config

# Copy files
oc cp <pod>:/path/to/file ./local-file
oc cp ./local-file <pod>:/path/to/file

# Port forward
oc port-forward <pod> 8080:80

# ═══════════════════════════════════════════════════════════════════════════
# NODE DEBUGGING
# ═══════════════════════════════════════════════════════════════════════════

# Debug node (starts debug pod)
oc debug node/<node>
# Then: chroot /host

# Inside debug pod:
crictl ps                      # List containers
crictl logs <container-id>     # Container logs
journalctl -u kubelet         # Kubelet logs
systemctl status kubelet
ss -tulpn                      # Listening ports
df -h                          # Disk space
free -h                        # Memory

# ═══════════════════════════════════════════════════════════════════════════
# CLUSTER-WIDE
# ═══════════════════════════════════════════════════════════════════════════

# Events (cluster-wide)
oc get events -A --sort-by='.lastTimestamp' | tail -50

# All resources in namespace
oc get all -n <namespace>

# API resources
oc api-resources

# Check cluster operators (OpenShift)
oc get clusteroperators
oc describe clusteroperator <name>

# Cluster health
oc adm top nodes
oc adm top pods -A
```

---

# Module 8: Performance & Troubleshooting

## 8.1 Resource Management

```yaml
# Best practice: Always set requests AND limits
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: my-app
    resources:
      requests:
        memory: "256Mi"    # Guaranteed minimum
        cpu: "250m"        # 0.25 CPU cores
      limits:
        memory: "512Mi"    # Maximum (OOMKilled if exceeded)
        cpu: "500m"        # Throttled if exceeded
```

## 8.2 Monitoring Commands

```bash
# Node resource usage
oc adm top nodes

# Pod resource usage
oc adm top pods -n <namespace>
oc adm top pods --containers  # Per container

# Resource quotas
oc describe resourcequota -n <namespace>

# Limit ranges
oc describe limitrange -n <namespace>

# Prometheus queries (via console or API)
# CPU usage: sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)
# Memory: sum(container_memory_working_set_bytes) by (pod)
```

## 8.3 Performance Tuning Tips

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Performance Best Practices                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. Set appropriate resource requests/limits                               │
│      - Requests too low: Pod gets scheduled but starves                     │
│      - Requests too high: Wasted cluster capacity                           │
│      - Limits too low: OOMKilled/throttled                                  │
│      - Limits too high: One pod can starve others                          │
│                                                                              │
│   2. Use horizontal pod autoscaling                                         │
│      oc autoscale deployment my-app --min=2 --max=10 --cpu-percent=80      │
│                                                                              │
│   3. Use pod disruption budgets for availability                           │
│      Ensures minimum pods during updates/maintenance                        │
│                                                                              │
│   4. Use appropriate storage class                                          │
│      - SSD for databases                                                    │
│      - HDD for logs/backups                                                 │
│                                                                              │
│   5. Optimize container images                                              │
│      - Use minimal base images (UBI-minimal, Alpine)                        │
│      - Multi-stage builds                                                    │
│      - Don't include unnecessary tools                                      │
│                                                                              │
│   6. Use readiness/liveness probes properly                                 │
│      - Readiness: When to send traffic                                      │
│      - Liveness: When to restart                                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# Quiz and Review

## Quiz Questions

1. **What are the stages of API request processing?**

2. **What happens if etcd is lost?**

3. **How does the scheduler decide where to place a pod?**

4. **What is CNI?**

5. **What does CSI stand for and what does it do?**

6. **How do you debug a pod that won't start?**

## Answers

1. Authentication → Authorization (RBAC) → Admission Control → Storage (etcd)

2. All cluster state is lost. You need to restore from backup. This is why etcd backup is critical!

3. Filtering (which nodes CAN run it) → Scoring (which is BEST) → Binding

4. Container Network Interface - standard for container networking plugins

5. Container Storage Interface - standard for storage plugins (AWS EBS, GCP PD, etc.)

6. `oc describe pod`, check Events, `oc logs`, `oc get events`, check node resources

---

## Continuous Learning

Phase 4 is about ongoing deep learning. Keep exploring:

1. **Read source code** - Kubernetes is open source!
2. **Follow KEPs** - Kubernetes Enhancement Proposals
3. **Experiment** - Break things in a test cluster
4. **Contribute** - File bugs, improve docs
5. **Stay updated** - Follow release notes

---

**Congratulations! You've completed all four phases! 🎉**

You now have a solid foundation in:
- ✅ Linux and Container fundamentals
- ✅ Kubernetes core concepts
- ✅ OpenShift enterprise features
- ✅ Operator Framework
- ✅ Cluster internals

Keep learning, keep building, keep breaking things! 🚀

---

*Phase 4 Complete! The journey continues...*
