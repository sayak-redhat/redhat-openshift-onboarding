# Deploy Stage Builds - Cucushift Cluster (Workflow-Launch)

**Use this guide for clusters created with:** `workflow-launch cucushift-*`

These clusters have brew registry pull secrets by default.

---

## Step 1: Launch OpenShift Cluster (ClusterBot)

Launch a cluster using ClusterBot with the following workflow:

```bash
workflow-launch cucushift-installer-rehearse-gcp-ipi 4.18
```

Wait for the cluster to be ready and download the kubeconfig.

---

## Step 2: Create the Catalog Source

Create a CatalogSource that points to the Brew stage IIB (Index Image Build):

```bash
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: rh-brew-stage-operators
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: brew.registry.redhat.io/rh-osbs/iib:1091451
  displayName: Red Hat Brew Stage Catalog
  publisher: Red Hat
  updateStrategy:
    registryPoll:
      interval: 5m
EOF
```

---

## Step 3: Verify Catalog Source

### 3.1 Check CatalogSource status:

```bash
oc get catalogsource rh-brew-stage-operators -n openshift-marketplace
```

### 3.2 Check catalog pod is running:

```bash
oc get pods -n openshift-marketplace | grep rh-brew-stage
```

**Expected:** Pod should be 1/1 Running

### 3.3 Verify connection state is READY:

```bash
oc describe catalogsource rh-brew-stage-operators -n openshift-marketplace | grep -A5 "Connection State"
```

**Expected:** lastObservedState: READY

**Note:** If TRANSIENT_FAILURE, check pod logs:

```bash
oc logs -n openshift-marketplace -l olm.catalogSource=rh-brew-stage-operators
```

### 3.4 Verify operators are available in catalog:

```bash
oc get packagemanifests -l catalog=rh-brew-stage-operators
```

### 3.5 Check ZTWIM operator specifically:

```bash
oc get packagemanifest openshift-zero-trust-workload-identity-manager -o yaml | grep -A5 catalogSource
```

---

## Step 4: Refresh UI (Optional)

If the catalog source doesn't appear in the OpenShift Console UI:

```bash
oc delete pods -n openshift-console -l app=console
```

---

## Step 5: Install the ZTWIM Operator

Create the Namespace, OperatorGroup, and Subscription:

```bash
oc apply -f - <<EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: zero-trust-workload-identity-manager
  labels:
    openshift.io/cluster-monitoring: "true"

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
  source: rh-brew-stage-operators
  sourceNamespace: openshift-marketplace
  name: openshift-zero-trust-workload-identity-manager
  channel: stable-v1
EOF
```

---

## Step 6: Verify Operator Installation

### 6.1 Watch CSV installation progress:

```bash
oc get csv -n zero-trust-workload-identity-manager -w
```

Wait until PHASE shows: **Succeeded**

### 6.2 Verify operator pod is running:

```bash
oc get pods -n zero-trust-workload-identity-manager
```

---

## Step 7: Deploy ZTWIM Operands (Custom Resources)

### 7.1 Set environment variables:

```bash
export APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
export JWT_ISSUER_ENDPOINT=oidc-discovery.${APP_DOMAIN}
export CLUSTER_NAME=test01
```

### 7.2 Create ZeroTrustWorkloadIdentityManager CR:

```bash
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: ${APP_DOMAIN}
  clusterName: ${CLUSTER_NAME}
  bundleConfigMap: spire-bundle
  labels:
    environment: development
    managed-by: ztwim-operator
EOF
```

### 7.3 Create SPIRE Components:

```bash
oc apply -f - <<EOF
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: "SPIRE Server CA"
    country: "US"
    organization: "RH"
  jwtIssuer: https://${JWT_ISSUER_ENDPOINT}
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

---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireAgent
metadata:
  name: cluster
spec:
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
  jwtIssuer: https://${JWT_ISSUER_ENDPOINT}
EOF
```

---

## Step 8: Verify Complete Deployment

```bash
oc get pods -n zero-trust-workload-identity-manager
```

**Expected:** All pods Running with full readiness:
- spire-agent (one per node)
- spire-server-0
- spire-spiffe-csi-driver (one per node)
- spire-spiffe-oidc-discovery-provider
- zero-trust-workload-identity-manager-controller-manager

Verify all CRs:

```bash
oc get zerotrustworkloadidentitymanager,spireserver,spireagent,spiffecsidriver,spireoidcdiscoveryprovider -n zero-trust-workload-identity-manager
```

---

## Troubleshooting

### CatalogSource stuck in TRANSIENT_FAILURE:

```bash
oc logs -n openshift-marketplace -l olm.catalogSource=rh-brew-stage-operators
oc delete pod -n openshift-marketplace -l olm.catalogSource=rh-brew-stage-operators
```

### CSV not appearing:

```bash
oc get subscription -n zero-trust-workload-identity-manager -o yaml
oc get installplan -n zero-trust-workload-identity-manager
```

### Operator pod not starting:

```bash
oc describe csv -n zero-trust-workload-identity-manager
oc get events -n zero-trust-workload-identity-manager --sort-by='.lastTimestamp'
```

---

## Quick Reference

| Command | Description |
|---------|-------------|
| `oc get catalogsource -n openshift-marketplace` | Check catalog status |
| `oc get packagemanifests -l catalog=rh-brew-stage-operators` | List catalog operators |
| `oc get subscription -n zero-trust-workload-identity-manager` | Check subscription |
| `oc get csv -n zero-trust-workload-identity-manager` | Check CSV status |
| `oc get pods -n zero-trust-workload-identity-manager` | Check all pods |

---

**Document Version:** 1.0  
**IIB Version:** 1091451  
**OpenShift Version:** 4.18  
**Cluster Type:** Cucushift (workflow-launch)  
**Last Updated:** January 2026
