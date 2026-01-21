# Deploy Stage Builds - Regular Cluster (Launch)

**Use this guide for clusters created with:** `launch` command (without brew registry access)

These clusters do NOT have brew registry pull secrets by default. This guide uses Quay.io images.

---

## Step 1: Launch OpenShift Cluster (ClusterBot)

Launch a cluster using ClusterBot:

```bash
launch openshift-install 4.18
```

Wait for the cluster to be ready and download the kubeconfig.

---

## Step 2: Create the Catalog Source

Create a CatalogSource that points to the Quay.io stage IIB:

**Note:** Replace `<VERSION_TAG>` with the specific version tag from the Red Hat user workloads repository.

```bash
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ztwim-stage-operators
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/redhat-user-workloads/zero-trust-workload-tenant/openshift-zero-trust-workload-identity-manager-bundle:<VERSION_TAG>
  displayName: ZTWIM Stage Catalog (Quay)
  publisher: Red Hat
  updateStrategy:
    registryPoll:
      interval: 5m
EOF
```

**Alternative:** If you have the specific Quay.io index image:

```bash
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ztwim-stage-operators
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/redhat-user-workloads/zero-trust-workload-tenant/openshift-zero-trust-workload-identity-manager-index:latest
  displayName: ZTWIM Stage Catalog (Quay)
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
oc get catalogsource ztwim-stage-operators -n openshift-marketplace
```

### 3.2 Check catalog pod is running:

```bash
oc get pods -n openshift-marketplace | grep ztwim-stage
```

**Expected:** Pod should be 1/1 Running

### 3.3 Verify connection state is READY:

```bash
oc describe catalogsource ztwim-stage-operators -n openshift-marketplace | grep -A5 "Connection State"
```

**Expected:** lastObservedState: READY

**Note:** If TRANSIENT_FAILURE, check pod logs:

```bash
oc logs -n openshift-marketplace -l olm.catalogSource=ztwim-stage-operators
```

### 3.4 Verify operators are available in catalog:

```bash
oc get packagemanifests -l catalog=ztwim-stage-operators
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
  source: ztwim-stage-operators
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
oc logs -n openshift-marketplace -l olm.catalogSource=ztwim-stage-operators
oc delete pod -n openshift-marketplace -l olm.catalogSource=ztwim-stage-operators
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

### Image pull errors:

If you see ImagePullBackOff errors, verify the Quay.io image path and tag:

```bash
oc get catalogsource ztwim-stage-operators -n openshift-marketplace -o yaml
```

---

## Quick Reference

| Command | Description |
|---------|-------------|
| `oc get catalogsource -n openshift-marketplace` | Check catalog status |
| `oc get packagemanifests -l catalog=ztwim-stage-operators` | List catalog operators |
| `oc get subscription -n zero-trust-workload-identity-manager` | Check subscription |
| `oc get csv -n zero-trust-workload-identity-manager` | Check CSV status |
| `oc get pods -n zero-trust-workload-identity-manager` | Check all pods |

---

## Finding the Correct Quay.io Image

To find the correct image tag from Quay.io:

1. Visit: https://quay.io/repository/redhat-user-workloads/zero-trust-workload-tenant/openshift-zero-trust-workload-identity-manager-bundle
2. Look for the latest tag or specific version you need
3. Update the CatalogSource image field accordingly

---

**Document Version:** 1.0  
**OpenShift Version:** 4.18  
**Cluster Type:** Regular launch (no brew registry)  
**Image Source:** Quay.io  
**Last Updated:** January 2026
