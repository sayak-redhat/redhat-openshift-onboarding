# ZTWIM Stage Deployment Guide

Complete guide for deploying Zero Trust Workload Identity Manager (ZTWIM) stage builds on OpenShift clusters.

---

## Table of Contents
- [Prerequisites](#prerequisites)
- [Choose Your Deployment Path](#choose-your-deployment-path)
- [Path A: Cucushift Clusters (workflow-launch)](#path-a-cucushift-clusters-workflow-launch)
- [Path B: Regular Clusters (launch)](#path-b-regular-clusters-launch)
- [Deploy Custom Resources (Common for Both Paths)](#deploy-custom-resources-common-for-both-paths)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Quick Reference](#quick-reference)

---

## Prerequisites

- Access to ClusterBot
- `oc` CLI installed
- `kubectl` (optional)
- `envsubst` (for environment variable substitution)

---

## Choose Your Deployment Path

### Path A: Cucushift Clusters (workflow-launch cucushift-*)
**✅ Has brew registry pull secrets by default**
- Uses: `brew.registry.redhat.io/rh-osbs/iib:1091451`
- Best for: QE testing with brew builds

### Path B: Regular Clusters (launch)
**❌ Does NOT have brew registry pull secrets**
- Uses: `quay.io/redhat-user-workloads/zero-trust-workload-tenant/xxx`
- Best for: Testing with public Quay.io images
- No need to create pull secrets

---

## Path A: Cucushift Clusters (workflow-launch)

### Step 1: Launch Cucushift Cluster

```bash
workflow-launch cucushift-installer-rehearse-gcp-ipi 4.18
```

Wait for the cluster to be ready and download the kubeconfig.

### Step 2: Create Catalog Source (Brew Registry)

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

### Step 3: Verify Catalog Source

```bash
# Check CatalogSource status
oc get catalogsource rh-brew-stage-operators -n openshift-marketplace

# Check catalog pod is running
oc get pods -n openshift-marketplace | grep rh-brew-stage

# Verify connection state is READY
oc describe catalogsource rh-brew-stage-operators -n openshift-marketplace | grep -A5 "Connection State"

# Verify operators available
oc get packagemanifests -l catalog=rh-brew-stage-operators

# Check ZTWIM operator specifically
oc get packagemanifest openshift-zero-trust-workload-identity-manager -o yaml | grep -A5 catalogSource
```

**Expected:** Pod should be 1/1 Running and connection state should be READY

**If TRANSIENT_FAILURE:**
```bash
oc logs -n openshift-marketplace -l olm.catalogSource=rh-brew-stage-operators
oc delete pod -n openshift-marketplace -l olm.catalogSource=rh-brew-stage-operators
```

### Step 4: Install ZTWIM Operator

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

### Step 5: Verify Operator Installation

```bash
# Watch CSV installation progress
oc get csv -n zero-trust-workload-identity-manager -w

# Wait until PHASE shows: Succeeded (Ctrl+C to exit watch)

# Verify operator pod is running
oc get pods -n zero-trust-workload-identity-manager
```

### Step 6: Deploy Custom Resources

Continue to [Deploy Custom Resources](#deploy-custom-resources-common-for-both-paths) section.

---

## Path B: Regular Clusters (launch)

### Step 1: Launch Regular Cluster

```bash
launch openshift-install 4.18
```

Wait for the cluster to be ready and download the kubeconfig.

### Step 2: Create Catalog Source (Quay.io)

**Note:** Update `<VERSION_TAG>` with the actual version tag from Quay.io

```bash
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ztwim-stage-operators
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  # Replace <VERSION_TAG> with actual version (e.g., latest, v1.0.0-preview)
  image: quay.io/redhat-user-workloads/zero-trust-workload-tenant/openshift-zero-trust-workload-identity-manager-index:latest
  displayName: ZTWIM Stage Catalog (Quay)
  publisher: Red Hat
  updateStrategy:
    registryPoll:
      interval: 5m
EOF
```

**Find the correct Quay.io image:**
1. Visit: https://quay.io/repository/redhat-user-workloads/zero-trust-workload-tenant/openshift-zero-trust-workload-identity-manager-index
2. Look for the latest tag or specific version
3. Update the image field in the CatalogSource above

### Step 3: Verify Catalog Source

```bash
# Check CatalogSource status
oc get catalogsource ztwim-stage-operators -n openshift-marketplace

# Check catalog pod is running
oc get pods -n openshift-marketplace | grep ztwim-stage

# Verify connection state is READY
oc describe catalogsource ztwim-stage-operators -n openshift-marketplace | grep -A5 "Connection State"

# Verify operators available
oc get packagemanifests -l catalog=ztwim-stage-operators

# Check ZTWIM operator specifically
oc get packagemanifest openshift-zero-trust-workload-identity-manager -o yaml | grep -A5 catalogSource
```

**Expected:** Pod should be 1/1 Running and connection state should be READY

**If TRANSIENT_FAILURE or ImagePullBackOff:**
```bash
oc logs -n openshift-marketplace -l olm.catalogSource=ztwim-stage-operators
oc get catalogsource ztwim-stage-operators -n openshift-marketplace -o yaml
oc delete pod -n openshift-marketplace -l olm.catalogSource=ztwim-stage-operators
```

### Step 4: Install ZTWIM Operator

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

### Step 5: Verify Operator Installation

```bash
# Watch CSV installation progress
oc get csv -n zero-trust-workload-identity-manager -w

# Wait until PHASE shows: Succeeded (Ctrl+C to exit watch)

# Verify operator pod is running
oc get pods -n zero-trust-workload-identity-manager
```

### Step 6: Deploy Custom Resources

Continue to [Deploy Custom Resources](#deploy-custom-resources-common-for-both-paths) section.

---

## Deploy Custom Resources (Common for Both Paths)

After the operator is successfully installed, deploy the ZTWIM operands.

### Step 1: Set Environment Variables

```bash
export APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
export JWT_ISSUER_ENDPOINT=oidc-discovery.${APP_DOMAIN}
export CLUSTER_NAME=test01

# Verify variables are set correctly
echo "APP_DOMAIN: $APP_DOMAIN"
echo "JWT_ISSUER_ENDPOINT: $JWT_ISSUER_ENDPOINT"
echo "CLUSTER_NAME: $CLUSTER_NAME"
```

### Step 2: Create ZeroTrustWorkloadIdentityManager CR

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

### Step 3: Create SPIRE Components

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

### Step 4: Optional - Refresh UI

If the catalog source or resources don't appear in the OpenShift Console UI:

```bash
oc delete pods -n openshift-console -l app=console
```

---

## Verification

### Check All Pods

```bash
oc get pods -n zero-trust-workload-identity-manager
```

**Expected Pods (All Running with full readiness):**
- `spire-agent-*` (one per node)
- `spire-server-0`
- `spire-spiffe-csi-driver-*` (one per node)
- `spire-spiffe-oidc-discovery-provider-*`
- `zero-trust-workload-identity-manager-controller-manager-*`

### Check All Custom Resources

```bash
oc get zerotrustworkloadidentitymanager,spireserver,spireagent,spiffecsidriver,spireoidcdiscoveryprovider -n zero-trust-workload-identity-manager
```

### Check Individual Resources

```bash
# ZeroTrustWorkloadIdentityManager
oc get zerotrustworkloadidentitymanager cluster -o yaml

# SpireServer
oc get spireserver cluster -o yaml

# SpireAgent
oc get spireagent cluster -o yaml

# SpiffeCSIDriver
oc get spiffecsidriver cluster -o yaml

# SpireOIDCDiscoveryProvider
oc get spireoidcdiscoveryprovider cluster -o yaml
```

### Check PVC

```bash
oc get pvc -n zero-trust-workload-identity-manager
```

**Expected:** A 2Gi PVC for spire-server in "Bound" status

### Check ConfigMap

```bash
oc get configmap spire-bundle -n zero-trust-workload-identity-manager -o yaml
```

---

## Troubleshooting

### CatalogSource Issues

#### CatalogSource stuck in TRANSIENT_FAILURE

```bash
# Check pod logs
oc logs -n openshift-marketplace -l olm.catalogSource=rh-brew-stage-operators
# OR for regular clusters:
oc logs -n openshift-marketplace -l olm.catalogSource=ztwim-stage-operators

# Restart the catalog pod
oc delete pod -n openshift-marketplace -l olm.catalogSource=rh-brew-stage-operators
# OR for regular clusters:
oc delete pod -n openshift-marketplace -l olm.catalogSource=ztwim-stage-operators

# Check the CatalogSource spec
oc get catalogsource -n openshift-marketplace -o yaml
```

#### CatalogSource pod ImagePullBackOff (Regular clusters only)

```bash
# Verify the image exists and tag is correct
oc describe catalogsource ztwim-stage-operators -n openshift-marketplace

# Check if the image path is correct (should be public on Quay.io)
# Update the CatalogSource with correct image tag:
oc edit catalogsource ztwim-stage-operators -n openshift-marketplace
```

### Operator Installation Issues

#### CSV not appearing

```bash
# Check subscription status
oc get subscription -n zero-trust-workload-identity-manager -o yaml

# Check install plan
oc get installplan -n zero-trust-workload-identity-manager

# Check for events
oc get events -n zero-trust-workload-identity-manager --sort-by='.lastTimestamp'

# Check OLM logs
oc logs -n openshift-operator-lifecycle-manager deployment/catalog-operator
```

#### CSV stuck in "Installing" phase

```bash
# Describe the CSV
oc describe csv -n zero-trust-workload-identity-manager

# Check events
oc get events -n zero-trust-workload-identity-manager --sort-by='.lastTimestamp'

# Check if there are approval requirements
oc get installplan -n zero-trust-workload-identity-manager -o yaml
```

### Operator Pod Issues

#### Operator pod not starting

```bash
# Describe the pod
oc describe pod -n zero-trust-workload-identity-manager -l control-plane=controller-manager

# Check pod logs
oc logs -n zero-trust-workload-identity-manager -l control-plane=controller-manager

# Check events
oc get events -n zero-trust-workload-identity-manager --sort-by='.lastTimestamp'
```

### Custom Resource Issues

#### Resources not being created

```bash
# Check operator logs
oc logs -n zero-trust-workload-identity-manager deployment/zero-trust-workload-identity-manager-controller-manager

# Check if CRDs are installed
oc get crd | grep spire

# Describe the CR to see status/conditions
oc describe spireserver cluster
```

#### SPIRE Server pod not starting

```bash
# Check PVC is bound
oc get pvc -n zero-trust-workload-identity-manager

# Describe the pod
oc describe pod spire-server-0 -n zero-trust-workload-identity-manager

# Check logs
oc logs spire-server-0 -n zero-trust-workload-identity-manager

# Check events
oc get events -n zero-trust-workload-identity-manager --sort-by='.lastTimestamp'
```

#### SPIRE Agent pods not starting

```bash
# Check DaemonSet status
oc get daemonset -n zero-trust-workload-identity-manager

# Describe agent pod
oc describe pod -n zero-trust-workload-identity-manager -l app=spire-agent

# Check logs
oc logs -n zero-trust-workload-identity-manager -l app=spire-agent

# Verify node attestation configuration
oc get spireagent cluster -o yaml
```

---

## Quick Reference

### Common Commands

| Command | Description |
|---------|-------------|
| `oc get catalogsource -n openshift-marketplace` | Check all catalog sources |
| `oc get packagemanifests` | List all available operators |
| `oc get subscription -n zero-trust-workload-identity-manager` | Check subscription status |
| `oc get csv -n zero-trust-workload-identity-manager` | Check CSV status |
| `oc get pods -n zero-trust-workload-identity-manager` | Check all ZTWIM pods |
| `oc get zerotrustworkloadidentitymanager,spireserver,spireagent,spiffecsidriver,spireoidcdiscoveryprovider` | Check all ZTWIM CRs |
| `oc logs -n zero-trust-workload-identity-manager deployment/zero-trust-workload-identity-manager-controller-manager` | Check operator logs |

### Key Differences Between Deployment Paths

| Aspect | Cucushift Clusters | Regular Clusters |
|--------|-------------------|------------------|
| **Cluster Launch** | `workflow-launch cucushift-*` | `launch` |
| **Brew Registry Access** | ✅ Yes (built-in) | ❌ No |
| **Image Source** | `brew.registry.redhat.io` | `quay.io/redhat-user-workloads` |
| **CatalogSource Name** | `rh-brew-stage-operators` | `ztwim-stage-operators` |
| **IIB Version** | `1091451` | N/A (uses Quay.io tags) |
| **Pull Secret Setup** | Not needed | Not needed (public Quay.io) |

### Environment Variables

```bash
# Set these before deploying custom resources
export APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
export JWT_ISSUER_ENDPOINT=oidc-discovery.${APP_DOMAIN}
export CLUSTER_NAME=test01
```

### Quick Cleanup

```bash
# Delete all ZTWIM custom resources
oc delete spireoidcdiscoveryprovider cluster
oc delete spiffecsidriver cluster
oc delete spireagent cluster
oc delete spireserver cluster
oc delete zerotrustworkloadidentitymanager cluster

# Uninstall operator
oc delete subscription openshift-zero-trust-workload-identity-manager -n zero-trust-workload-identity-manager
oc delete csv -n zero-trust-workload-identity-manager -l operators.coreos.com/openshift-zero-trust-workload-identity-manager.zero-trust-workload-identity-manager

# Delete namespace
oc delete namespace zero-trust-workload-identity-manager

# Delete catalog source (cucushift)
oc delete catalogsource rh-brew-stage-operators -n openshift-marketplace
# OR for regular clusters:
oc delete catalogsource ztwim-stage-operators -n openshift-marketplace
```

---

## Version Information

- **OpenShift Version:** 4.18
- **IIB Version (Cucushift):** 1091451
- **Channel:** stable-v1
- **Last Updated:** January 2026

---

## Additional Resources

- **SPIRE Documentation:** https://spiffe.io/docs/latest/spire/
- **OpenShift Operators:** https://docs.openshift.com/container-platform/latest/operators/understanding/olm/olm-understanding-olm.html
- **Quay.io Repository:** https://quay.io/repository/redhat-user-workloads/zero-trust-workload-tenant

---

## Notes

1. **envsubst requirement:** Environment variables in the custom resources section use shell variable expansion. Ensure your shell supports this or manually replace the variables.

2. **Quay.io image tags:** For regular clusters, always verify the latest available image tag at the Quay.io repository URL provided above.

3. **Cluster monitoring:** The namespace is labeled with `openshift.io/cluster-monitoring: "true"` to enable monitoring integration.

4. **Persistence:** SpireServer uses a 2Gi PVC with ReadWriteOncePod access mode for data persistence.

5. **Node attestation:** The deployment uses Kubernetes PSAT (Projected Service Account Token) for secure node attestation.

6. **Trust domain:** The trust domain is set to match the cluster's application domain for proper SPIFFE identity management.
