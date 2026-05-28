# Zero-Trust Workload Identity Manager — Codebase Map

A visual guide to navigating the [zero-trust-workload-identity-manager](https://github.com/openshift/zero-trust-workload-identity-manager) repository. Intended for new contributors, QE engineers, and anyone needing to understand how the operator is structured.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     OLM (Operator Lifecycle Manager)            │
│  Subscription → CSV → Deploys operator Pod in                  │
│                       zero-trust-workload-identity-manager ns   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              cmd/zero-trust-workload-identity-manager/main.go   │
│                                                                 │
│  1. Register schemes (K8s, OpenShift, SPIFFE, OLM)             │
│  2. Build label-filtered cache (managed-by label)              │
│  3. Create controller-runtime Manager                          │
│  4. Register 5 controllers ──────────────────────────┐         │
│  5. Start healthz/readyz + mgr.Start()               │         │
└──────────────────────────────────────────────────────┼─────────┘
                                                       │
            ┌──────────────┬──────────────┬────────────┼──────────────┐
            ▼              ▼              ▼            ▼              ▼
   ┌──────────────┐ ┌────────────┐ ┌───────────┐ ┌──────────┐ ┌──────────────┐
   │    ZTWIM     │ │SpireServer │ │SpireAgent │ │SpiffeCSI │ │  SpireOIDC   │
   │  Controller  │ │ Controller │ │Controller │ │Controller│ │  Controller  │
   └──────┬───────┘ └─────┬──────┘ └─────┬─────┘ └────┬─────┘ └──────┬───────┘
          │               │              │             │              │
          ▼               ▼              ▼             ▼              ▼
    Aggregates       Creates:       Creates:      Creates:       Creates:
    operand        • StatefulSet  • DaemonSet   • DaemonSet    • Deployment
    statuses       • ConfigMaps   • ConfigMap   • CSIDriver    • ConfigMap
    into ZTWIM     • Services     • Service     • ServiceAcct  • Services
    Ready cond.    • RBAC (8)     • RBAC        • SCC          • RBAC
                   • Webhook      • SCC                        • Route
                   • Route        • ServiceAcct                • ClusterSPIFFEIDs
                   • ServiceAcct                               • ServiceAcct
```

---

## Directory Structure

```
zero-trust-workload-identity-manager/
│
├── api/v1alpha1/                          ◄── CRD TYPE DEFINITIONS
│   ├── zero_trust_workload_identity_manager_types.go   (ZeroTrustWorkloadIdentityManager)
│   ├── spire_server_config_types.go                    (SpireServer)
│   ├── spire_agent_config_types.go                     (SpireAgent)
│   ├── spiffe_csi_config_types.go                      (SpiffeCSIDriver)
│   ├── spire_oidc_discovery_provider_types.go           (SpireOIDCDiscoveryProvider)
│   ├── conditions.go                                    (Ready, Failed, Progressing, ResourceConflict)
│   └── zz_generated.deepcopy.go                         (auto-generated)
│
├── cmd/zero-trust-workload-identity-manager/
│   └── main.go                            ◄── ENTRYPOINT: registers all 5 controllers
│
├── pkg/
│   ├── controller/                        ◄── ALL RECONCILER LOGIC
│   │   ├── zero-trust-workload-identity-manager/   → ZTWIM (parent, status aggregator)
│   │   ├── spire-server/                           → SpireServer reconciler
│   │   │   ├── controller.go                         (Reconcile entrypoint)
│   │   │   ├── statefulset.go                        (StatefulSet create/update)
│   │   │   ├── configmap.go                          (server + controller-manager ConfigMaps)
│   │   │   ├── service.go                            (Services)
│   │   │   ├── rbac.go                               (8 RBAC resources)
│   │   │   ├── webhook.go                            (ValidatingWebhookConfig)
│   │   │   ├── routes.go                             (Federation Route)
│   │   │   ├── service_account.go
│   │   │   └── *_test.go                             (unit tests per resource)
│   │   ├── spire-agent/                            → SpireAgent reconciler
│   │   │   ├── controller.go / daemonset.go / configmap.go / rbac.go
│   │   │   ├── scc.go / service.go / service_account.go
│   │   │   └── *_test.go
│   │   ├── spiffe-csi-driver/                      → SpiffeCSIDriver reconciler
│   │   │   ├── controller.go / daemonset.go / csi_driver.go
│   │   │   ├── scc.go / service_account.go
│   │   │   └── *_test.go
│   │   ├── spire-oidc-discovery-provider/          → SpireOIDCDiscoveryProvider reconciler
│   │   │   ├── controller.go / deployments.go / configmaps.go
│   │   │   ├── clusterspiffeid.go / routes.go / rbac.go
│   │   │   ├── service.go / service_account.go
│   │   │   └── *_test.go
│   │   ├── status/                        ◄── STATUS MANAGEMENT
│   │   │   └── manager.go                   (AddCondition → SetReady → ApplyStatus)
│   │   └── utils/                         ◄── SHARED UTILITIES
│   │       ├── constants.go                 (namespaces, image envs, config keys)
│   │       ├── labels.go                    (managed-by labels, component labels)
│   │       ├── resource_comparison.go       (needsUpdate for each resource type)
│   │       ├── resource_ownership.go        (conflict detection: CheckResourceConflict)
│   │       ├── validation.go                (CR spec validation)
│   │       ├── utils.go                     (create-only mode, predicates, hashing)
│   │       ├── proxy.go                     (OLM proxy env injection)
│   │       └── relatedImages.go             (RELATED_IMAGE_* env → container images)
│   │
│   ├── client/                            ◄── CLIENT ABSTRACTIONS
│   │   ├── client.go                        (CustomCtrlClient: CRUD + retry)
│   │   ├── cache.go                         (label-filtered informer cache)
│   │   └── fakes/                           (counterfeiter fakes for unit tests)
│   │
│   ├── operator/assets/                   ◄── EMBEDDED BINDATA (compiled YAML)
│   └── version/                           ◄── Build version/commit ldflags
│
├── config/                                ◄── KUSTOMIZE & OLM CONFIG
│   ├── crd/bases/                           (5 operator CRDs + 3 SPIFFE CRDs)
│   ├── rbac/                                (manager ClusterRole, bindings)
│   ├── samples/                             (sample CRs for all kinds)
│   ├── manager/                             (Deployment manifest)
│   └── manifests/                           (OLM CSV base)
│
├── bundle/                                ◄── OLM BUNDLE (generated)
├── bindata/                               ◄── YAML TEMPLATES for operand resources
├── test/e2e/                              ◄── GINKGO V2 E2E TESTS
│   └── e2e_test.go                          (Ordered suite: install, attest, config, create-only)
├── Makefile                               ◄── build, test, manifests, bundle, docker
└── Dockerfile / Dockerfile.coverage
```

---

## CRD Kinds (api/v1alpha1)

All CRs are **cluster-scoped singletons** — they always use `metadata.name: cluster`.

| Kind | File | Purpose |
|------|------|---------|
| **ZeroTrustWorkloadIdentityManager** | `zero_trust_workload_identity_manager_types.go` | Global config: `trustDomain`, `clusterName`, `bundleConfigMap`; aggregates operand status |
| **SpireServer** | `spire_server_config_types.go` | SPIRE server: JWT issuer, CA/TTL, persistence, datastore, federation, upstream authority |
| **SpireAgent** | `spire_agent_config_types.go` | Node agent: socket path, node/workload attestors, kubelet cert verification |
| **SpiffeCSIDriver** | `spiffe_csi_config_types.go` | CSI driver: agent socket path, plugin name |
| **SpireOIDCDiscoveryProvider** | `spire_oidc_discovery_provider_types.go` | OIDC provider: JWT issuer, replicas, managed Route, external TLS secret |

**Shared types:** `CommonConfig` (labels, resources, affinity, tolerations, nodeSelector) is embedded in all operand specs.

**External CRDs** (from spire-controller-manager, bundled in `config/crd/bases/`):
- `ClusterSPIFFEID`, `ClusterStaticEntry`, `ClusterFederatedTrustDomain` (`spire.spiffe.io/v1alpha1`)

---

## Controllers — What Each Reconciles

### ZTWIM Controller (`pkg/controller/zero-trust-workload-identity-manager/`)
Does **not** deploy workloads. It:
- Aggregates status from all 4 operand CRs into `status.operands[]`
- Sets `OperandsAvailable`, `Ready`, `CreateOnlyMode` conditions
- Syncs OLM `Upgradeable` to `OperatorCondition`
- Watches operand CR status changes and re-enqueues ZTWIM

### SpireServer Controller (`pkg/controller/spire-server/`)

| Resource | Name |
|----------|------|
| StatefulSet | `spire-server` (server + controller-manager sidecar) |
| ConfigMaps | Server config, controller-manager config, trust bundle |
| Services | `spire-server`, `spire-controller-manager-webhook` |
| RBAC | ClusterRoles/Bindings (server, bundle, controller-manager), Roles (leader election, external cert) |
| ValidatingWebhook | `spire-controller-manager-webhook` |
| Route | `spire-server-federation` (when federation enabled) |
| ServiceAccount | `spire-server` |

### SpireAgent Controller (`pkg/controller/spire-agent/`)

| Resource | Name |
|----------|------|
| DaemonSet | `spire-agent` |
| ConfigMap | Agent config |
| ServiceAccount | `spire-agent` |
| Service | `spire-agent` |
| RBAC | ClusterRole/Binding |
| SCC | `spire-agent` (OpenShift SecurityContextConstraints) |

### SpiffeCSIDriver Controller (`pkg/controller/spiffe-csi-driver/`)

| Resource | Name |
|----------|------|
| DaemonSet | `spire-spiffe-csi-driver` |
| CSIDriver | `csi.spiffe.io` |
| ServiceAccount | `spire-spiffe-csi-driver` |
| SCC | `spire-spiffe-csi-driver` |

### SpireOIDCDiscoveryProvider Controller (`pkg/controller/spire-oidc-discovery-provider/`)

| Resource | Name |
|----------|------|
| Deployment | `spire-spiffe-oidc-discovery-provider` |
| ConfigMap | OIDC config |
| Services | `spire-spiffe-oidc-discovery-provider` |
| ServiceAccount | `spire-spiffe-oidc-discovery-provider` |
| RBAC | Role/RoleBinding for external cert |
| Route | `spire-oidc-discovery-provider` |
| ClusterSPIFFEIDs | OIDC + default fallback |

---

## Reconciler Pattern (All Controllers Follow This)

```
Reconcile(req)
│
├─ 1. Get operand CR (e.g., SpireServer "cluster")
│     └─ Not found? → return (deleted or not yet created)
│
├─ 2. SetInitialReconciliationStatus(Ready=False, Progressing)
│     └─ defer statusMgr.ApplyStatus()  ◄── always writes status at end
│
├─ 3. Get ZTWIM parent CR ("cluster")
│     └─ Not found? → set Ready=False "ZTWIM CR not found"
│
├─ 4. SetControllerReference(ZTWIM → operand CR)
│
├─ 5. Check CREATE_ONLY_MODE env
│
├─ 6. ValidateConfiguration(spec)
│     └─ Invalid? → set ConfigurationValid=False, return
│
├─ 7. Reconcile sub-resources (in order):
│     ├─ ServiceAccount     ─┐
│     ├─ RBAC (Roles, CRBs) │  For each:
│     ├─ ConfigMaps          │    Get existing → not found? → Create
│     ├─ Services            │    exists? → needsUpdate? → Update
│     ├─ SCC (OpenShift)     │    AlreadyExists on Create? → ResourceConflict!
│     ├─ Workload (STS/DS)   │    set sub-condition per resource type
│     ├─ Route               │
│     └─ Webhook            ─┘
│
├─ 8. Check workload health (replicas ready, generation match)
│
└─ 9. SetReadyCondition() ◄── auto-aggregates all sub-conditions
       └─ All True?  → Ready=True
       └─ Any False? → Ready=False (Failed or Progressing)
```

---

## Ownership & Watch Graph

```
          ┌──────────────────────────────────┐
          │  ZeroTrustWorkloadIdentityManager │  ◄── User creates this first
          │         (name: "cluster")         │
          └─────────┬──────┬──────┬──────┬───┘
         owns       │      │      │      │        (ownerReference)
            ┌───────┘      │      │      └────────┐
            ▼              ▼      ▼               ▼
     ┌────────────┐  ┌──────────┐ ┌──────────┐  ┌──────────────┐
     │SpireServer │  │SpireAgent│ │SpiffeCSI │  │SpireOIDC     │
     │ "cluster"  │  │"cluster" │ │"cluster" │  │"cluster"     │
     └─────┬──────┘  └────┬─────┘ └────┬─────┘  └──────┬───────┘
           │ owns         │ owns       │ owns          │ owns
           ▼              ▼            ▼               ▼
     StatefulSet     DaemonSet    DaemonSet       Deployment
     ConfigMaps      ConfigMap    CSIDriver       ConfigMap
     Services        Service     ServiceAcct      Services
     RBAC (8)        RBAC        SCC              Route
     Webhook         SCC                          RBAC
     Route           ServiceAcct                  ClusterSPIFFEIDs
     ServiceAcct                                  ServiceAcct
```

**Watch behavior:**
- Each controller watches its operand CR + its owned K8s resources (filtered by `app.kubernetes.io/managed-by` label)
- All secondary watches map back to the `cluster` operand CR name
- ZTWIM watches all 4 operand CR status changes and re-aggregates its own `Ready`

---

## Status Management (`pkg/controller/status/`)

The `status.Manager` is the central pattern used by all reconcilers:

| Step | Method | What it does |
|------|--------|-------------|
| 1 | `SetInitialReconciliationStatus()` | Sets `Ready=False, Reason=Progressing` at reconcile start |
| 2 | `AddCondition(type, reason, msg, status)` | Accumulates sub-conditions during reconcile |
| 3 | `SetReadyCondition()` | Auto-aggregates: `Ready=True` if all sub-conditions pass; `Failed` if any is `False`; `Progressing` for rollout states |
| 4 | `ApplyStatus()` | Writes conditions via `StatusUpdateWithRetry`; skips write if unchanged |

**Health helpers:** `CheckStatefulSetHealth`, `CheckDaemonSetHealth`, `CheckDeploymentHealth` — verify replicas, generation, and conditions.

**ZTWIM is special:** It manually sets `Ready` (doesn't auto-aggregate) to distinguish between progressing vs failed operands.

---

## Shared Utilities (`pkg/controller/utils/`)

| File | What it provides |
|------|-----------------|
| `constants.go` | Operator namespace, `RELATED_IMAGE_*` env var names, ConfigMap data keys (`agent.conf`, `server.conf`, `controller-manager-config.yaml`) |
| `labels.go` | Standard K8s labels (`app.kubernetes.io/managed-by`, component labels), watch predicates |
| `resource_comparison.go` | `ResourceNeedsUpdate()` — type-aware diff for Services, RBAC, SCC, StatefulSet, DaemonSet, Deployment, CSIDriver, webhooks, ClusterSPIFFEID |
| `resource_ownership.go` | Conflict detection: `CheckResourceConflict()`, `HandleCreateConflict()` — prevents overwriting resources not labeled as operator-managed |
| `validation.go` | CR spec validation (affinity, tolerations, nodeSelector, resources, labels) |
| `utils.go` | Create-only mode (`CREATE_ONLY_MODE`), watch predicates, config hashing, owner-ref helpers |
| `proxy.go` | OLM proxy env var injection, trusted CA bundle ConfigMap mounting |
| `relatedImages.go` | Reads operand container images from `RELATED_IMAGE_*` env vars set by OLM |

---

## Client & Cache (`pkg/client/`)

**`CustomCtrlClient`** — wraps controller-runtime client with:
- Standard CRUD + Patch
- `UpdateWithRetry` / `StatusUpdateWithRetry` (automatic conflict retry)
- `Exists`, `CreateOrUpdateObject`

**`NewCacheBuilder()`** — configures manager cache with:
- **Label selector** `app.kubernetes.io/managed-by=zero-trust-workload-identity-manager` on all watched operand resources
- Pre-registers informers for CRs, Deployments, DaemonSets, StatefulSets, Routes, ClusterSPIFFEIDs
- Resources without the managed-by label are invisible to the cache (this is how conflict detection works)

**Fakes:** `pkg/client/fakes/` — counterfeiter-generated for unit tests

---

## Test Structure

| Tier | Location | Framework | Run with |
|------|----------|-----------|----------|
| **Unit** | `pkg/controller/**/*_test.go` (~38 files) | Go `testing` + counterfeiter fakes | `make test` |
| **Integration** | (envtest, no dedicated dir) | controller-runtime envtest | `make test` |
| **E2E** | `test/e2e/e2e_test.go` | Ginkgo v2 + Gomega | `make test-e2e` |

### E2E test contexts:
- **Installation** — CRDs, operator deployment, operand CRs, ZTWIM status aggregation
- **OperatorCondition** — Upgradeable True/False under failure/recovery
- **SpireAgent attestation** — SVID rotation
- **Common configurations** — Subscription log level, CR-driven resources/affinity/labels for all operands
- **CreateOnlyMode** — `CREATE_ONLY_MODE` env via Subscription patch

---

## Build & Makefile

| Target | Purpose |
|--------|---------|
| `make build` | manifests + generate + fmt + vet + compile |
| `make test` | Unit tests with envtest (excludes e2e) |
| `make test-e2e` | E2E suite (needs live OpenShift cluster) |
| `make manifests` | controller-gen → CRDs + RBAC |
| `make generate` | DeepCopy generation |
| `make lint` | golangci-lint |
| `make docker-build` | Operator container image |
| `make bundle` | OLM bundle generation + validation |
| `make install` / `uninstall` | CRD install/remove on cluster |
| `make deploy` / `undeploy` | Manager deployment |

---

## Key Conventions

| Convention | Value |
|-----------|-------|
| All CRs | name `cluster` (singletons) |
| Operator namespace | `zero-trust-workload-identity-manager` |
| Managed-by label | `app.kubernetes.io/managed-by=zero-trust-workload-identity-manager` |
| Cache filtering | Only sees resources with managed-by label |
| Config keys | `agent.conf`, `server.conf`, `controller-manager-config.yaml` |
| Images | `RELATED_IMAGE_*` env vars on operator Deployment (set by OLM/CSV) |
| Unit test framework | Go `testing` + counterfeiter fakes |
| E2E test framework | Ginkgo v2 + Gomega (live cluster required) |

---

## Quick Navigation Guide

| "I want to..." | Look at... |
|----------------|------------|
| Understand CRD fields | `api/v1alpha1/*_types.go` |
| See how SpireServer reconciles | `pkg/controller/spire-server/controller.go` |
| Understand what triggers a resource update | `pkg/controller/utils/resource_comparison.go` |
| See how status conditions work | `pkg/controller/status/manager.go` |
| Debug resource conflict errors | `pkg/controller/utils/resource_ownership.go` |
| Find what labels go on resources | `pkg/controller/utils/labels.go` |
| Run unit tests | `make test` |
| Run E2E tests | `make test-e2e` (needs live cluster) |
| Find sample CRs to apply | `config/samples/` |
| Understand OLM installation | `bundle/` + `config/manifests/` |
| Check RBAC permissions | `config/rbac/role.yaml` |
| See container image sources | `pkg/controller/utils/relatedImages.go` |
| Understand proxy/CA handling | `pkg/controller/utils/proxy.go` |
