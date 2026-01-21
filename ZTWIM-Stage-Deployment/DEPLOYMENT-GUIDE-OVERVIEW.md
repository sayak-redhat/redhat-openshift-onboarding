# ZTWIM Stage Deployment - Overview

This directory contains deployment guides and YAML files for deploying ZTWIM (Zero Trust Workload Identity Manager) stage builds on OpenShift clusters.

## Choose Your Deployment Path

### Path A: Cucushift Clusters (workflow-launch cucushift-*)
**Has brew registry pull secrets by default**

- **Guide:** `deploy-stage-cucushift-cluster.md`
- **YAML Files:**
  - `catalog-source-cucushift.yaml` - CatalogSource using brew registry
  - `ztwim-operator-install-cucushift.yaml` - Operator installation

**Quick Start:**
```bash
# Create catalog source
oc apply -f catalog-source-cucushift.yaml

# Install operator
oc apply -f ztwim-operator-install-cucushift.yaml

# Deploy operands (after setting env vars)
export APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
export JWT_ISSUER_ENDPOINT=oidc-discovery.${APP_DOMAIN}
export CLUSTER_NAME=test01
envsubst < ztwim-custom-resources.yaml | oc apply -f -
```

---

### Path B: Regular Clusters (launch)
**Does NOT have brew registry pull secrets**

- **Guide:** `deploy-stage-regular-cluster.md`
- **YAML Files:**
  - `catalog-source-regular.yaml` - CatalogSource using Quay.io
  - `ztwim-operator-install-regular.yaml` - Operator installation

**Quick Start:**
```bash
# Update image tag in catalog-source-regular.yaml first
# Then create catalog source
oc apply -f catalog-source-regular.yaml

# Install operator
oc apply -f ztwim-operator-install-regular.yaml

# Deploy operands (after setting env vars)
export APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
export JWT_ISSUER_ENDPOINT=oidc-discovery.${APP_DOMAIN}
export CLUSTER_NAME=test01
envsubst < ztwim-custom-resources.yaml | oc apply -f -
```

---

## Common Resources

**Custom Resources (Operands):**
- `ztwim-custom-resources.yaml` - All ZTWIM custom resources (same for both paths)
  - ZeroTrustWorkloadIdentityManager
  - SpireServer
  - SpireAgent
  - SpiffeCSIDriver
  - SpireOIDCDiscoveryProvider

---

## File Structure

```
.
├── DEPLOYMENT-GUIDE-OVERVIEW.md           # This file
│
├── deploy-stage-cucushift-cluster.md      # Complete guide for cucushift clusters
├── deploy-stage-regular-cluster.md        # Complete guide for regular clusters
│
├── catalog-source-cucushift.yaml          # CatalogSource (brew registry)
├── catalog-source-regular.yaml            # CatalogSource (Quay.io)
│
├── ztwim-operator-install-cucushift.yaml  # Operator install (cucushift)
├── ztwim-operator-install-regular.yaml    # Operator install (regular)
│
└── ztwim-custom-resources.yaml            # Operands (common for both)
```

---

## Key Differences Between Paths

| Aspect | Cucushift Clusters | Regular Clusters |
|--------|-------------------|------------------|
| **Cluster Launch** | `workflow-launch cucushift-*` | `launch` |
| **Brew Registry Access** | ✅ Yes (built-in) | ❌ No |
| **Image Source** | brew.registry.redhat.io | quay.io/redhat-user-workloads |
| **CatalogSource Name** | rh-brew-stage-operators | ztwim-stage-operators |
| **IIB Version** | 1091451 | N/A (uses Quay.io tags) |
| **Pull Secret Setup** | Not needed | Not needed (public Quay.io) |

---

## Deployment Workflow

### Phase 1: Cluster Setup
1. Launch cluster with appropriate command
2. Download kubeconfig
3. Verify cluster access

### Phase 2: Catalog Source
1. Choose correct CatalogSource YAML
2. Apply catalog source
3. Verify catalog pod is running
4. Confirm connection state is READY

### Phase 3: Operator Installation
1. Apply operator installation YAML
2. Wait for CSV to reach "Succeeded" phase
3. Verify operator pod is running

### Phase 4: Deploy Operands
1. Set environment variables
2. Apply custom resources
3. Verify all pods are running
4. Confirm all CRs are ready

---

## Verification Commands

```bash
# Check catalog
oc get catalogsource -n openshift-marketplace

# Check operator subscription
oc get subscription -n zero-trust-workload-identity-manager

# Check CSV status
oc get csv -n zero-trust-workload-identity-manager

# Check all pods
oc get pods -n zero-trust-workload-identity-manager

# Check all custom resources
oc get zerotrustworkloadidentitymanager,spireserver,spireagent,spiffecsidriver,spireoidcdiscoveryprovider -n zero-trust-workload-identity-manager
```

---

## Troubleshooting Resources

Both deployment guides include comprehensive troubleshooting sections for:
- CatalogSource connectivity issues
- CSV installation problems
- Operator pod failures
- Image pull errors (regular clusters)
- Custom resource deployment issues

Refer to the specific deployment guide for detailed troubleshooting steps.

---

## Version Information

- **OpenShift Version:** 4.18
- **IIB Version (Cucushift):** 1091451
- **Channel:** stable-v1
- **Last Updated:** January 2026

---

## Notes

1. **envsubst requirement:** If using `ztwim-custom-resources.yaml` with environment variables, ensure `envsubst` is installed or manually replace the variables.

2. **Quay.io image tags:** For regular clusters, verify the latest available image tag at:
   https://quay.io/repository/redhat-user-workloads/zero-trust-workload-tenant/openshift-zero-trust-workload-identity-manager-index

3. **Cluster monitoring:** The namespace is labeled with `openshift.io/cluster-monitoring: "true"` to enable monitoring.

4. **Persistence:** SpireServer uses a 2Gi PVC with ReadWriteOncePod access mode.

5. **Node attestation:** The deployment uses Kubernetes PSAT (Projected Service Account Token) for node attestation.
