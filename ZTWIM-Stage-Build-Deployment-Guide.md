# ZTWIM v1.0.1 — Stage Build Deployment Guide

**Operator:** Zero Trust Workload Identity Manager (ZTWIM)  
**Version:** v1.0.1  
**Build System:** Konflux  
**Reference Doc:** [Stage Build Testing Doc](https://docs.google.com/document/d/1dTwo6_qVbNUpnOQjRiqvl0lJHYij40RZPqtW8XsdkYA/edit?tab=t.0)

---

## 1. Index Images (IIB) per OCP Version

| OCP Version | IIB Image | Registry Proxy |
|:-----------:|-----------|----------------|
| **4.18** | `brew.registry.redhat.io/rh-osbs/iib:1142067` | `registry-proxy.engineering.redhat.com/rh-osbs/iib:1142067` |
| **4.19** | `brew.registry.redhat.io/rh-osbs/iib:1142066` | `registry-proxy.engineering.redhat.com/rh-osbs/iib:1142066` |
| **4.20** | `brew.registry.redhat.io/rh-osbs/iib:1142069` | `registry-proxy.engineering.redhat.com/rh-osbs/iib:1142069` |
| **4.21** | `brew.registry.redhat.io/rh-osbs/iib:1142065` | `registry-proxy.engineering.redhat.com/rh-osbs/iib:1142065` |

## 2. Component Images

| Component | Image |
|-----------|-------|
| Operator | `registry.stage.redhat.io/zero-trust-workload-identity-manager/zero-trust-workload-identity-manager-rhel9@sha256:c90ed9e428a4e5f8cc85e88de0ef1e80834700a0354f796d13154e50cd95a2fe` |
| SPIRE Server | `registry.stage.redhat.io/zero-trust-workload-identity-manager/spiffe-spire-server-rhel9@sha256:1adc34cd1e87024bab6850b2cda63743f42c57ae7c3ae483574760294b1a91a9` |
| SPIRE Agent | `registry.stage.redhat.io/zero-trust-workload-identity-manager/spiffe-spire-agent-rhel9@sha256:402eabe171ba8129489ddd12eccea03e475da226fcd230eba5bfdeff3d73dc8e` |
| SPIFFE CSI Driver | `registry.stage.redhat.io/zero-trust-workload-identity-manager/spiffe-csi-driver-rhel9@sha256:e15e0b3442602833be8e3297cfac1333ce9e45ebc0d9a893f2a758702aff75e1` |
| SPIRE OIDC Discovery Provider | `registry.stage.redhat.io/zero-trust-workload-identity-manager/spiffe-spire-oidc-discovery-provider-rhel9@sha256:47142dd4fefad52dcd1c749532f7de761bb9d463a9d90349c8987c769f3e3438` |
| SPIRE Controller Manager | `registry.stage.redhat.io/zero-trust-workload-identity-manager/spiffe-spire-controller-manager-rhel9@sha256:fcd19d7b5c774ada1c7f9abf3367796ef7a113c4d2e2566ffc3cc0f0be222c4d` |
| Node Driver Registrar | `registry.redhat.io/openshift4/ose-csi-node-driver-registrar-rhel9@sha256:715fed4adf4280d9a8e92a09cdcc35ed9bb95e623b1d5fc2f1b753981ecf41f4` |
| CSI Init Container | `registry.access.redhat.com/ubi9@sha256:f3b9841ea7615b38fd271cf18ae235d8228d3369036eb175ded171c936808255` |

---

## 3. Prerequisites

### 3.1 Provision a Cluster

Use clusterbot to launch a cluster on the desired OCP version:

```
workflow-launch cucushift-installer-rehearse-gcp-ipi 4.18
```

Replace `4.18` with `4.19`, `4.20`, or `4.21` as needed.

### 3.2 Verify Brew Registry Access

The cluster must have a pull secret configured for `brew.registry.redhat.io`. Verify:

```bash
oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | python3 -m json.tool | grep brew
```

If missing, update the global pull secret with Brew credentials before proceeding.

---

## 4. Deploy the Operator (Stage Build)

### Step 1 — Set Variables for Your OCP Version

Pick the IIB matching your cluster version:

```bash
# --- For OCP 4.18 ---
export CATALOG_IMG=brew.registry.redhat.io/rh-osbs/iib:1142067
export CATALOG_NAME=stage-catalog-418-1.0.1

# --- For OCP 4.19 ---
# export CATALOG_IMG=brew.registry.redhat.io/rh-osbs/iib:1142066
# export CATALOG_NAME=stage-catalog-419-1.0.1

# --- For OCP 4.20 ---
# export CATALOG_IMG=brew.registry.redhat.io/rh-osbs/iib:1142069
# export CATALOG_NAME=stage-catalog-420-1.0.1

# --- For OCP 4.21 ---
# export CATALOG_IMG=brew.registry.redhat.io/rh-osbs/iib:1142065
# export CATALOG_NAME=stage-catalog-421-1.0.1
```

### Step 2 — Create CatalogSource

```bash
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: $CATALOG_NAME
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: $CATALOG_IMG
  displayName: Red Hat Brew Stage Catalog
  publisher: grpc
  updateStrategy:
    registryPoll:
      interval: 5m
EOF
```

### Step 3 — Wait for CatalogSource to Become READY

```bash
oc wait --timeout=5m -n openshift-marketplace catalogsource/$CATALOG_NAME \
  --for=jsonpath='{.status.connectionState.lastObservedState}=READY'
```

> **Note:** IIB images are ~2 GB. The catalog pod may take several minutes to initialize and could enter `CrashLoopBackOff` during startup. This is normal — the pod will self-heal after a few restarts as the gRPC server needs time to load the large index database. If the `oc wait` times out, monitor the pod and retry:
>
> ```bash
> # Watch the catalog pod until it shows 1/1 Running
> watch -n 10 'oc get pods -n openshift-marketplace | grep $CATALOG_NAME'
>
> # Then verify READY state
> oc get catalogsource $CATALOG_NAME -n openshift-marketplace \
>   -o jsonpath='{.status.connectionState.lastObservedState}'
> ```

### Step 4 — Create Namespace, OperatorGroup, and Subscription

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: zero-trust-workload-identity-manager
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: zero-trust-workload-identity-manager-og
  namespace: zero-trust-workload-identity-manager
spec:
  upgradeStrategy: Default
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-zero-trust-workload-identity-manager
  namespace: zero-trust-workload-identity-manager
spec:
  source: $CATALOG_NAME
  sourceNamespace: openshift-marketplace
  name: openshift-zero-trust-workload-identity-manager
  channel: stable-v1
EOF
```

### Step 5 — Wait for Operator to Install

```bash
# Wait for CSV to reach Succeeded
oc get csv -n zero-trust-workload-identity-manager -w

# Wait for controller-manager deployment
oc wait --for=condition=Available \
  deployment/zero-trust-workload-identity-manager-controller-manager \
  -n zero-trust-workload-identity-manager --timeout=2m
```

---

## 5. Deploy the Operands

### Step 1 — Set Cluster Variables

```bash
export NS=zero-trust-workload-identity-manager
export APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
export JWT_ISSUER_ENDPOINT=oidc-discovery.${APP_DOMAIN}
export CLUSTER_NAME=test01
```

### Step 2 — Create the Parent CR (ZeroTrustWorkloadIdentityManager)

This **must** be created first. All sub-controllers (SpireServer, SpireAgent, CSI, OIDC) depend on this parent object — without it, they will fail with `"ZeroTrustWorkloadIdentityManager cluster not found"`.

```bash
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: $APP_DOMAIN
  clusterName: $CLUSTER_NAME
EOF
```

### Step 3 — Apply Operand Custom Resources

```bash
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  trustDomain: $APP_DOMAIN
  clusterName: $CLUSTER_NAME
  caSubject:
    commonName: $APP_DOMAIN
    country: "US"
    organization: "RH"
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
    maxOpenConns: 100
    maxIdleConns: 2
    connMaxLifetime: 3600
  jwtIssuer: https://$JWT_ISSUER_ENDPOINT
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireAgent
metadata:
  name: cluster
spec:
  trustDomain: $APP_DOMAIN
  clusterName: $CLUSTER_NAME
  nodeAttestor:
    k8sPSATEnabled: "true"
  workloadAttestors:
    k8sEnabled: "true"
    workloadAttestorsVerification:
      type: "auto"
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpiffeCSIDriver
metadata:
  name: cluster
spec: {}
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireOIDCDiscoveryProvider
metadata:
  name: cluster
spec:
  trustDomain: $APP_DOMAIN
  jwtIssuer: https://$JWT_ISSUER_ENDPOINT
EOF
```

### Step 4 — Wait for Operands

```bash
oc rollout status statefulset/spire-server -n $NS --timeout=2m
oc rollout status daemonset/spire-agent -n $NS --timeout=2m
oc rollout status daemonset/spire-spiffe-csi-driver -n $NS --timeout=2m
oc wait --for=condition=Available deployment/spire-spiffe-oidc-discovery-provider -n $NS --timeout=2m
```

### Step 5 — Verify Everything Running

```bash
oc get pods -n $NS -o wide
```

Expected: `spire-server`, `spire-agent` (one per worker), `spire-spiffe-csi-driver` (one per worker), `spire-spiffe-oidc-discovery-provider`, and `controller-manager` — all Running.

---

## 6. Quick Validation

```bash
# CSV status
oc get csv -n $NS

# All pods
oc get pods -n $NS

# Agent logs (check attestation)
POD_AGENT=$(oc get pods -n $NS -l app.kubernetes.io/name=spire-agent -o jsonpath='{.items[0].metadata.name}')
oc logs -n $NS $POD_AGENT --tail=20

# Server logs
POD_SERVER=$(oc get pods -n $NS -l app.kubernetes.io/name=spire-server -o jsonpath='{.items[0].metadata.name}')
oc logs -n $NS $POD_SERVER --tail=20
```

---

## 7. Cleanup

```bash
# Delete operand CRs first
oc delete spireoidcdiscoveryprovider cluster 2>/dev/null
oc delete spiffecsidrivers cluster 2>/dev/null
oc delete spireagent cluster 2>/dev/null
oc delete spireserver cluster 2>/dev/null

# Wait for operand pods to terminate
sleep 30

# Delete the operator namespace
oc delete namespace zero-trust-workload-identity-manager

# Delete the CatalogSource
oc delete catalogsource $CATALOG_NAME -n openshift-marketplace
```

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| CatalogSource stuck in `TRANSIENT_FAILURE` | IIB image (~2 GB) takes too long to start; startup probe kills it | Wait — pod self-heals after a few restarts. Monitor with `watch oc get pods -n openshift-marketplace` |
| `oc wait` for CatalogSource times out | Same as above | Re-run the wait command after the pod shows `1/1 Running` |
| No CSV appears after Subscription created | CatalogSource not READY yet | Fix catalog first, then Subscription auto-resolves |
| `ConstraintsNotSatisfiable` on Subscription | Channel mismatch or package name wrong | Verify `channel: stable-v1` and `name: openshift-zero-trust-workload-identity-manager` |
| Image pull failures | Missing Brew pull secret | Add `brew.registry.redhat.io` credentials to cluster pull secret |
| `ZeroTrustWorkloadIdentityManager "cluster" not found` in logs | Parent CR not created | Create the `ZeroTrustWorkloadIdentityManager` CR before operand CRs (Section 5, Step 2) |
| Operand pods not starting | CRDs not installed or CR spec incorrect | Check `oc get crd | grep spire` and operator logs |

---

## 9. Per-Version Quick Copy-Paste Blocks

### OCP 4.18

```bash
export CATALOG_IMG=brew.registry.redhat.io/rh-osbs/iib:1142067
export CATALOG_NAME=stage-catalog-418-1.0.1
```

### OCP 4.19

```bash
export CATALOG_IMG=brew.registry.redhat.io/rh-osbs/iib:1142066
export CATALOG_NAME=stage-catalog-419-1.0.1
```

### OCP 4.20

```bash
export CATALOG_IMG=brew.registry.redhat.io/rh-osbs/iib:1142069
export CATALOG_NAME=stage-catalog-420-1.0.1
```

### OCP 4.21

```bash
export CATALOG_IMG=brew.registry.redhat.io/rh-osbs/iib:1142065
export CATALOG_NAME=stage-catalog-421-1.0.1
```

After setting the variables, follow Steps 2–5 from Section 4 above.

---

*Document created: 2026-05-12*  
*ZTWIM version: v1.0.1 (Konflux stage builds)*
