# SPIRE Federation Guide - `https_spiffe` Profile

## Complete Step-by-Step Guide for Two OpenShift 4.18 Clusters

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Architecture](#architecture)
4. [Phase 1: Install ZTWIM Operator](#phase-1-install-ztwim-operator)
5. [Phase 2: Deploy SPIRE Stack](#phase-2-deploy-spire-stack)
6. [Phase 3: Configure Federation](#phase-3-configure-federation)
7. [Phase 4: Verify Federation](#phase-4-verify-federation)
8. [Phase 5: Test Workloads](#phase-5-test-workloads)
9. [Troubleshooting](#troubleshooting)
10. [Cleanup](#cleanup)

---

## Overview

### What is SPIRE Federation?

SPIRE Federation allows workloads running on different clusters to **trust and authenticate each other** using SPIFFE identities. Think of it like two countries agreeing to recognize each other's passports.

### What is `https_spiffe` Profile?

The `https_spiffe` profile uses **SPIRE's own self-signed certificates** for the federation endpoint. This requires a manual "trust bootstrap" where you exchange trust bundles between clusters before they can communicate securely.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     FEDERATION OVERVIEW                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Cluster 1                                       Cluster 2             │
│   ┌───────────────────┐                           ┌───────────────────┐ │
│   │   SPIRE Server    │     Trust Exchange        │   SPIRE Server    │ │
│   │                   │◄─────────────────────────►│                   │ │
│   │   Trust Domain:   │                           │   Trust Domain:   │ │
│   │   apps.cluster1   │                           │   apps.cluster2   │ │
│   └───────────────────┘                           └───────────────────┘ │
│           │                                               │             │
│           ▼                                               ▼             │
│   ┌───────────────────┐                           ┌───────────────────┐ │
│   │   Workload A      │     mTLS Authentication   │   Workload B      │ │
│   │   (test-client)   │◄─────────────────────────►│   (test-server)   │ │
│   └───────────────────┘                           └───────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

### Required Access

- [ ] Two OpenShift 4.18+ clusters
- [ ] `cluster-admin` access on both clusters
- [ ] `oc` CLI installed
- [ ] `envsubst` available (for variable substitution)

### Get Cluster Information

**On each cluster, run:**

```bash
# Get your cluster's apps domain
oc get dns cluster -o jsonpath='{.spec.baseDomain}'
```

**Note down both domains - you'll need them throughout this guide.**

Example:
- Cluster 1: `ci-ln-abc123.gcp-2.ci.openshift.org` → Apps domain: `apps.ci-ln-abc123.gcp-2.ci.openshift.org`
- Cluster 2: `ci-ln-xyz789.aws-4.ci.openshift.org` → Apps domain: `apps.ci-ln-xyz789.aws-4.ci.openshift.org`

---

## Architecture

### Components We'll Deploy

| Component | Purpose |
|-----------|---------|
| **ZTWIM Operator** | Manages all SPIRE components |
| **SpireServer** | Issues SPIFFE identities, manages federation |
| **SpireAgent** | Runs on each node, delivers identities to workloads |
| **SpiffeCSIDriver** | Mounts workload API socket into pods |
| **SpireOIDCDiscoveryProvider** | Enables OIDC authentication |
| **ClusterFederatedTrustDomain** | Declares trust in a remote cluster |
| **ClusterSPIFFEID** | Defines which pods get SPIFFE identities |

---

## Phase 1: Install ZTWIM Operator

### Why?

The ZTWIM (Zero Trust Workload Identity Manager) operator is available from the **Red Hat Operators catalog** in OpenShift. No additional CatalogSource is required!

### Step 1.1: Install Operator via OLM

**Run on BOTH clusters:**

```bash
oc apply -f - <<EOF
---
# 1. Namespace for the operator
apiVersion: v1
kind: Namespace
metadata:
  name: zero-trust-workload-identity-manager
  labels:
    openshift.io/cluster-monitoring: "true"

---
# 2. OperatorGroup
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: zero-trust-workload-identity-manager-og
  namespace: zero-trust-workload-identity-manager
spec:
  upgradeStrategy: Default

---
# 3. Subscription
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-zero-trust-workload-identity-manager
  namespace: zero-trust-workload-identity-manager
spec:
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  name: openshift-zero-trust-workload-identity-manager
  channel: stable-v1
EOF
```

### Step 1.2: Wait for Operator to Install

```bash
# Watch CSV status (wait for "Succeeded")
oc get csv -n zero-trust-workload-identity-manager -w
```

**Expected Output:**
```
NAME                                          DISPLAY                                VERSION   PHASE
zero-trust-workload-identity-manager.v1.0.0   Zero Trust Workload Identity Manager   1.0.0     Succeeded
```

Press `Ctrl+C` once you see `Succeeded`.

### Step 1.3: Verify Operator Pod is Running

```bash
oc get pods -n zero-trust-workload-identity-manager
```

**Expected Output:**
```
NAME                                                              READY   STATUS    RESTARTS   AGE
zero-trust-workload-identity-manager-controller-manager-xxxxx     1/1     Running   0          2m
```

---

## Phase 2: Deploy SPIRE Stack

### Important: Set Environment Variables First

Before deploying the SPIRE stack, you need to set up environment variables on each cluster.

### Step 2.1: Set Environment Variables

**On Cluster 1:**
```bash
# Set cluster-specific variables
export APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
export JWT_ISSUER_ENDPOINT=oidc-discovery.${APP_DOMAIN}
export CLUSTER_NAME=cluster1

# Verify variables
echo "APP_DOMAIN: ${APP_DOMAIN}"
echo "JWT_ISSUER_ENDPOINT: ${JWT_ISSUER_ENDPOINT}"
echo "CLUSTER_NAME: ${CLUSTER_NAME}"
```

**On Cluster 2:**
```bash
# Set cluster-specific variables
export APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
export JWT_ISSUER_ENDPOINT=oidc-discovery.${APP_DOMAIN}
export CLUSTER_NAME=cluster2

# Verify variables
echo "APP_DOMAIN: ${APP_DOMAIN}"
echo "JWT_ISSUER_ENDPOINT: ${JWT_ISSUER_ENDPOINT}"
echo "CLUSTER_NAME: ${CLUSTER_NAME}"
```

**Write down both APP_DOMAIN values - you'll need them for federation!**

---

### Step 2.2: Deploy SPIRE Stack (Both Clusters)

**Run on BOTH clusters (variables will be substituted):**

```bash
cat <<'EOF' | envsubst | oc apply -f -
---
# 1. ZeroTrustWorkloadIdentityManager (Main CR)
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: ${APP_DOMAIN}
  clusterName: ${CLUSTER_NAME}

---
# 2. SpireServer
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: ${APP_DOMAIN}
    country: "US"
    organization: "RH"
  persistence:
    type: pvc
    size: "1Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
    maxOpenConns: 100
    maxIdleConns: 2
    connMaxLifetime: 3600
  jwtIssuer: https://${JWT_ISSUER_ENDPOINT}

---
# 3. SpireAgent
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
# 4. SpiffeCSIDriver
apiVersion: operator.openshift.io/v1alpha1
kind: SpiffeCSIDriver
metadata:
  name: cluster
spec: {}

---
# 5. SpireOIDCDiscoveryProvider
apiVersion: operator.openshift.io/v1alpha1
kind: SpireOIDCDiscoveryProvider
metadata:
  name: cluster
spec:
  jwtIssuer: https://${JWT_ISSUER_ENDPOINT}
EOF
```

---

### Step 2.3: Enable Federation on SpireServer

After the base stack is deployed, update the SpireServer to enable federation.

**On Cluster 1 (replace `<CLUSTER2_APPS_DOMAIN>` with Cluster 2's actual apps domain):**

```bash
# First, get Cluster 2's domain (run on Cluster 2 first if you don't have it)
# export CLUSTER2_DOMAIN=apps.ci-ln-xyz789.aws-4.ci.openshift.org

oc patch spireserver cluster --type=merge -p '
{
  "spec": {
    "federation": {
      "bundleEndpoint": {
        "profile": "https_spiffe"
      },
      "managedRoute": "true",
      "federatesWith": [
        {
          "trustDomain": "<CLUSTER2_APPS_DOMAIN>",
          "bundleEndpointUrl": "https://federation.<CLUSTER2_APPS_DOMAIN>",
          "bundleEndpointProfile": "https_spiffe",
          "endpointSpiffeId": "spiffe://<CLUSTER2_APPS_DOMAIN>/spire/server"
        }
      ]
    }
  }
}'
```

**On Cluster 2 (replace `<CLUSTER1_APPS_DOMAIN>` with Cluster 1's actual apps domain):**

```bash
oc patch spireserver cluster --type=merge -p '
{
  "spec": {
    "federation": {
      "bundleEndpoint": {
        "profile": "https_spiffe"
      },
      "managedRoute": "true",
      "federatesWith": [
        {
          "trustDomain": "<CLUSTER1_APPS_DOMAIN>",
          "bundleEndpointUrl": "https://federation.<CLUSTER1_APPS_DOMAIN>",
          "bundleEndpointProfile": "https_spiffe",
          "endpointSpiffeId": "spiffe://<CLUSTER1_APPS_DOMAIN>/spire/server"
        }
      ]
    }
  }
}'
```

---

### Step 2.4: Verify SPIRE Stack is Running

**Run on BOTH clusters:**

```bash
# Check all pods are running
oc get pods -n zero-trust-workload-identity-manager
```

**Expected Output:**
```
NAME                                                              READY   STATUS    RESTARTS   AGE
spire-agent-xxxxx                                                 1/1     Running   0          3m
spire-agent-yyyyy                                                 1/1     Running   0          3m
spire-agent-zzzzz                                                 1/1     Running   0          3m
spire-server-0                                                    2/2     Running   0          3m
spire-spiffe-csi-driver-xxxxx                                     2/2     Running   0          3m
spire-spiffe-csi-driver-yyyyy                                     2/2     Running   0          3m
spire-spiffe-csi-driver-zzzzz                                     2/2     Running   0          3m
spire-spiffe-oidc-discovery-provider-xxxxx                        1/1     Running   0          3m
zero-trust-workload-identity-manager-controller-manager-xxxxx     1/1     Running   0          10m
```

### Step 2.5: Verify Federation Routes Exist

```bash
oc get routes -n zero-trust-workload-identity-manager
```

**Expected Output:**
```
NAME                            HOST/PORT                                    TERMINATION
spire-oidc-discovery-provider   oidc-discovery.apps.<domain>                 reencrypt/Redirect
spire-server-federation         federation.apps.<domain>                     passthrough/Redirect
```

---

## Phase 3: Configure Federation

### Understanding the Bootstrap Problem

With `https_spiffe`, each cluster uses a self-signed certificate. Before they can trust each other, we need to manually exchange their "trust bundles" (CA certificates).

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     TRUST BOOTSTRAP PROCESS                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Step 1: Fetch Cluster 1's bundle                                     │
│   Step 2: Fetch Cluster 2's bundle                                     │
│   Step 3: Give Cluster 1 → Cluster 2's bundle (so it can trust C2)     │
│   Step 4: Give Cluster 2 → Cluster 1's bundle (so it can trust C1)     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Step 3.1: Fetch Trust Bundles

**Run from any machine that can reach both clusters:**

```bash
# Fetch Cluster 1's bundle
curl -sk https://federation.<CLUSTER1_APPS_DOMAIN> > /tmp/cluster1-bundle.json

# Fetch Cluster 2's bundle
curl -sk https://federation.<CLUSTER2_APPS_DOMAIN> > /tmp/cluster2-bundle.json

# Verify bundles are valid JSON with keys
cat /tmp/cluster1-bundle.json | jq '.keys | length'
cat /tmp/cluster2-bundle.json | jq '.keys | length'
```

**Expected:** Both should return `2` (one x509-svid key, one jwt-svid key)

---

### Step 3.2: Create ClusterFederatedTrustDomain on Cluster 1

This tells Cluster 1 to trust Cluster 2.

```bash
# Create the CFDT YAML file
cat <<'EOF' > /tmp/cfdt-cluster1.yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: federation-to-cluster2
spec:
  trustDomain: <CLUSTER2_APPS_DOMAIN>
  bundleEndpointURL: https://federation.<CLUSTER2_APPS_DOMAIN>
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://<CLUSTER2_APPS_DOMAIN>/spire/server
  className: zero-trust-workload-identity-manager-spire
  trustDomainBundle: |
EOF

# Append Cluster 2's bundle
cat /tmp/cluster2-bundle.json | sed 's/^/    /' >> /tmp/cfdt-cluster1.yaml

# Apply on Cluster 1
oc apply -f /tmp/cfdt-cluster1.yaml

# Verify
oc get clusterfederatedtrustdomain
```

---

### Step 3.3: Create ClusterFederatedTrustDomain on Cluster 2

This tells Cluster 2 to trust Cluster 1.

```bash
# Create the CFDT YAML file
cat <<'EOF' > /tmp/cfdt-cluster2.yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: federation-to-cluster1
spec:
  trustDomain: <CLUSTER1_APPS_DOMAIN>
  bundleEndpointURL: https://federation.<CLUSTER1_APPS_DOMAIN>
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://<CLUSTER1_APPS_DOMAIN>/spire/server
  className: zero-trust-workload-identity-manager-spire
  trustDomainBundle: |
EOF

# Append Cluster 1's bundle
cat /tmp/cluster1-bundle.json | sed 's/^/    /' >> /tmp/cfdt-cluster2.yaml

# Apply on Cluster 2
oc apply -f /tmp/cfdt-cluster2.yaml

# Verify
oc get clusterfederatedtrustdomain
```

---

## Phase 4: Verify Federation

### Step 4.1: Check Bundle Sync on Cluster 1

```bash
oc -n zero-trust-workload-identity-manager exec spire-server-0 -c spire-server -- \
  /spire-server bundle list -socketPath /tmp/spire-server/private/api.sock
```

**Expected:** Should show Cluster 2's trust domain with its certificate:
```
****************************************
* <CLUSTER2_APPS_DOMAIN>
****************************************
-----BEGIN CERTIFICATE-----
...certificate data...
-----END CERTIFICATE-----
```

### Step 4.2: Check Bundle Sync on Cluster 2

```bash
oc -n zero-trust-workload-identity-manager exec spire-server-0 -c spire-server -- \
  /spire-server bundle list -socketPath /tmp/spire-server/private/api.sock
```

**Expected:** Should show Cluster 1's trust domain with its certificate.

### ✅ Federation is Working!

If both clusters show the other's trust domain in the bundle list, **federation is successfully established!**

---

## Phase 5: Test Workloads

Now let's deploy test workloads to verify they can get SPIFFE identities that include federation information.

### Step 5.1: Create Test Namespace (Both Clusters)

```bash
oc create namespace federation-test
oc create serviceaccount test-workload -n federation-test
```

### Step 5.2: Create ClusterSPIFFEID (Cluster 1)

```bash
oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: federation-test-workload
spec:
  className: zero-trust-workload-identity-manager-spire
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels:
      spiffe.io/spiffe-id: "true"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: federation-test
  workloadSelectorTemplates:
    - "k8s:pod-label:spiffe.io/spiffe-id:true"
  federatesWith:
    - "<CLUSTER2_APPS_DOMAIN>"
EOF
```

### Step 5.3: Create ClusterSPIFFEID (Cluster 2)

```bash
oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: federation-test-workload
spec:
  className: zero-trust-workload-identity-manager-spire
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels:
      spiffe.io/spiffe-id: "true"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: federation-test
  workloadSelectorTemplates:
    - "k8s:pod-label:spiffe.io/spiffe-id:true"
  federatesWith:
    - "<CLUSTER1_APPS_DOMAIN>"
EOF
```

### Step 5.4: Create spiffe-helper ConfigMap (Both Clusters)

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: spiffe-helper-config
  namespace: federation-test
data:
  helper.conf: |
    agent_address = "/spiffe-workload-api/spire-agent.sock"
    cmd = ""
    cert_dir = "/svids"
    svid_file_name = "svid.pem"
    svid_key_file_name = "svid.key"
    svid_bundle_file_name = "bundle.pem"
    renew_signal = ""
EOF
```

### Step 5.5: Deploy Test Pod (Cluster 1)

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-client
  namespace: federation-test
  labels:
    app: test-client
    spiffe.io/spiffe-id: "true"
spec:
  serviceAccountName: test-workload
  containers:
  - name: client
    image: registry.access.redhat.com/ubi9/ubi-minimal:latest
    command: ["sleep", "infinity"]
    volumeMounts:
    - name: svids
      mountPath: /svids
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      runAsNonRoot: true
      seccompProfile:
        type: RuntimeDefault
  - name: spiffe-helper
    image: ghcr.io/spiffe/spiffe-helper:0.8.0
    args: ["-config", "/etc/spiffe-helper/helper.conf"]
    volumeMounts:
    - name: spiffe-workload-api
      mountPath: /spiffe-workload-api
      readOnly: true
    - name: svids
      mountPath: /svids
    - name: spiffe-helper-config
      mountPath: /etc/spiffe-helper
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      runAsNonRoot: true
      seccompProfile:
        type: RuntimeDefault
  volumes:
  - name: spiffe-workload-api
    csi:
      driver: csi.spiffe.io
      readOnly: true
  - name: svids
    emptyDir: {}
  - name: spiffe-helper-config
    configMap:
      name: spiffe-helper-config
EOF

# Wait for pod to be ready
oc get pod test-client -n federation-test -w
```

### Step 5.6: Deploy Test Pod (Cluster 2)

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-server
  namespace: federation-test
  labels:
    app: test-server
    spiffe.io/spiffe-id: "true"
spec:
  serviceAccountName: test-workload
  containers:
  - name: server
    image: registry.access.redhat.com/ubi9/ubi-minimal:latest
    command: ["sleep", "infinity"]
    volumeMounts:
    - name: svids
      mountPath: /svids
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      runAsNonRoot: true
      seccompProfile:
        type: RuntimeDefault
  - name: spiffe-helper
    image: ghcr.io/spiffe/spiffe-helper:0.8.0
    args: ["-config", "/etc/spiffe-helper/helper.conf"]
    volumeMounts:
    - name: spiffe-workload-api
      mountPath: /spiffe-workload-api
      readOnly: true
    - name: svids
      mountPath: /svids
    - name: spiffe-helper-config
      mountPath: /etc/spiffe-helper
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      runAsNonRoot: true
      seccompProfile:
        type: RuntimeDefault
  volumes:
  - name: spiffe-workload-api
    csi:
      driver: csi.spiffe.io
      readOnly: true
  - name: svids
    emptyDir: {}
  - name: spiffe-helper-config
    configMap:
      name: spiffe-helper-config
EOF

# Wait for pod to be ready
oc get pod test-server -n federation-test -w
```

### Step 5.7: Verify Workload Identities

**On Cluster 1:**

```bash
# Check SVID files exist
oc exec -n federation-test test-client -c client -- ls -la /svids/

# View the SPIFFE ID
oc exec -n federation-test test-client -c client -- \
  cat /svids/svid.pem | openssl x509 -text -noout | grep -A1 "Subject Alternative Name"
```

**Expected:**
```
URI:spiffe://<CLUSTER1_APPS_DOMAIN>/ns/federation-test/sa/test-workload
```

**On Cluster 2:**

```bash
# Check SVID files exist
oc exec -n federation-test test-server -c server -- ls -la /svids/

# View the SPIFFE ID
oc exec -n federation-test test-server -c server -- \
  cat /svids/svid.pem | openssl x509 -text -noout | grep -A1 "Subject Alternative Name"
```

**Expected:**
```
URI:spiffe://<CLUSTER2_APPS_DOMAIN>/ns/federation-test/sa/test-workload
```

### Step 5.8: Verify FederatesWith in Entry Registration

**On Cluster 1:**

```bash
oc -n zero-trust-workload-identity-manager exec spire-server-0 -c spire-server -- \
  /spire-server entry show -socketPath /tmp/spire-server/private/api.sock | grep -A15 "federation-test"
```

**Expected:** Should show `FederatesWith: <CLUSTER2_APPS_DOMAIN>`

---

## Troubleshooting

### Operator Not Installing

```bash
# Check subscription status
oc get subscription -n zero-trust-workload-identity-manager

# Check install plan
oc get installplan -n zero-trust-workload-identity-manager

# Check operator logs
oc logs -n zero-trust-workload-identity-manager -l name=zero-trust-workload-identity-manager --tail=50
```

### Bundle Fetch Fails

```bash
# Check if route exists
oc get route -n zero-trust-workload-identity-manager

# Test connectivity
curl -vk https://federation.<APPS_DOMAIN>

# Check spire-server logs
oc logs -n zero-trust-workload-identity-manager spire-server-0 -c spire-server --tail=50
```

### CFDT Not Syncing

```bash
# Check CFDT status
oc describe clusterfederatedtrustdomain

# Check spire-controller-manager logs
oc logs -n zero-trust-workload-identity-manager -l app=spire-controller-manager --tail=50
```

### Workload Not Getting SVID

```bash
# Check if ClusterSPIFFEID matches
oc get clusterspiffeid

# Check spire-agent logs
oc logs -n zero-trust-workload-identity-manager -l app=spire-agent --tail=50

# Verify CSI driver is working
oc get pods -n zero-trust-workload-identity-manager | grep csi
```

---

## Cleanup

### Remove Test Workloads

```bash
oc delete namespace federation-test
oc delete clusterspiffeid federation-test-workload
```

### Remove Federation

```bash
oc delete clusterfederatedtrustdomain --all
```

### Remove SPIRE Stack

```bash
oc delete spireoidcdiscoveryprovider cluster
oc delete spiffecsidriver cluster
oc delete spireagent cluster
oc delete spireserver cluster
oc delete zerotrustworkloadidentitymanager cluster
```

### Remove Operator

```bash
oc delete subscription openshift-zero-trust-workload-identity-manager -n zero-trust-workload-identity-manager
oc delete csv -n zero-trust-workload-identity-manager --all
oc delete namespace zero-trust-workload-identity-manager
```

---

## Quick Reference: All-in-One Commands

### Set Variables for Both Clusters

```bash
# Export these variables on each cluster before running commands
export APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
export JWT_ISSUER_ENDPOINT=oidc-discovery.${APP_DOMAIN}
export CLUSTER_NAME=cluster1  # or cluster2 for the second cluster
```

### Verify All Components

```bash
# Check all resources
oc get zerotrustworkloadidentitymanager,spireserver,spireagent,spiffecsidriver,spireoidcdiscoveryprovider
oc get pods -n zero-trust-workload-identity-manager
oc get routes -n zero-trust-workload-identity-manager
oc get clusterfederatedtrustdomain
oc get clusterspiffeid
```

---

## Summary

### What We Achieved

1. ✅ Installed ZTWIM operator from **Red Hat Operators catalog** (no CatalogSource needed!)
2. ✅ Deployed SPIRE stack using environment variables for configuration
3. ✅ Configured federation using `https_spiffe` profile
4. ✅ Exchanged trust bundles via ClusterFederatedTrustDomain
5. ✅ Deployed test workloads with federated SPIFFE identities
6. ✅ Verified cross-cluster trust establishment

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Trust Domain** | A SPIFFE identity realm (usually = apps domain) |
| **Bundle** | CA certificate + JWT signing keys |
| **CFDT** | ClusterFederatedTrustDomain - declares trust in remote cluster |
| **ClusterSPIFFEID** | Defines which pods get SPIFFE identities |
| **federatesWith** | Which remote trust domains the workload can communicate with |

### Key Differences from Stage Deployment

| Aspect | Stage Deployment | GA Deployment |
|--------|------------------|---------------|
| Catalog Source | `rh-brew-stage-operators` (custom) | `redhat-operators` (built-in) |
| Registry | `brew.registry.redhat.io` | Red Hat's production registry |
| Extra Steps | Create CatalogSource | None needed |

### Next Steps

- Deploy real applications that use SPIFFE/SPIRE for mTLS
- Integrate with Envoy proxy for automatic mTLS
- Use OIDC federation for cloud provider authentication

---

## References

- [SPIFFE Documentation](https://spiffe.io/docs/)
- [SPIRE Documentation](https://spiffe.io/docs/latest/spire/)
- [OpenShift Operators](https://docs.openshift.com/container-platform/latest/operators/)
- [Red Hat Operators Catalog](https://catalog.redhat.com/software/operators/search)

---

**Document Version:** 2.0  
**Last Updated:** January 2026  
**Tested On:** OpenShift 4.18, ZTWIM Operator v1.0.0 (GA)
