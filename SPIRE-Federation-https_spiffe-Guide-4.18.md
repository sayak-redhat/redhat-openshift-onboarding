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

## Phase 5: Test Workloads with mTLS

Now let's deploy real mTLS workloads to verify cross-cluster authentication using SPIFFE identities.

### What is mTLS?

```
┌──────────────────────────────────────────────────────────────────┐
│                    mTLS vs Normal TLS                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   Normal TLS:   Client ─────────────────► Server shows ID        │
│                 (Only server proves identity)                    │
│                                                                   │
│   Mutual TLS:   Client shows ID ◄────────► Server shows ID       │
│                 (BOTH sides prove identity)                      │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Test Architecture

```
┌─────────────────────────────────┐         ┌─────────────────────────────────┐
│     Cluster 1 (Server)          │         │     Cluster 2 (Client)          │
│                                 │         │                                 │
│   ┌─────────────────────────┐   │  mTLS   │   ┌─────────────────────────┐   │
│   │   mTLS Server           │   │ ◄═════► │   │   mTLS Client           │   │
│   │   - Uses SPIFFE SVID    │   │         │   │   - Uses SPIFFE SVID    │   │
│   │   - Requires client cert│   │         │   │   - Presents client cert│   │
│   └─────────────────────────┘   │         │   └─────────────────────────┘   │
│                                 │         │                                 │
│   FederatesWith: Cluster 2      │         │   FederatesWith: Cluster 1      │
└─────────────────────────────────┘         └─────────────────────────────────┘
```

---

### Step 5.1: Deploy mTLS Server (Cluster 1)

#### 5.1.1 Create Namespace and ClusterSPIFFEID

```bash
# Set the remote cluster's domain (Cluster 2's apps domain)
export REMOTE_DOMAIN="<CLUSTER2_APPS_DOMAIN>"

# Create namespace
oc create namespace mtls-server

# Create ClusterSPIFFEID with federation
oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: mtls-server
spec:
  className: zero-trust-workload-identity-manager-spire
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels:
      app: mtls-server
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: mtls-server
  federatesWith:
    - "${REMOTE_DOMAIN}"
EOF
```

#### 5.1.2 Create spiffe-helper ConfigMap

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: spiffe-helper-config
  namespace: mtls-server
data:
  helper.conf: |
    agent_address = "/spiffe-workload-api/spire-agent.sock"
    cmd = ""
    cert_dir = "/certs"
    svid_file_name = "svid.pem"
    svid_key_file_name = "svid_key.pem"
    svid_bundle_file_name = "bundle.pem"
    renew_signal = ""
EOF
```

#### 5.1.3 Deploy mTLS Server Deployment

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mtls-server-sa
  namespace: mtls-server
---
apiVersion: v1
kind: Service
metadata:
  name: mtls-server
  namespace: mtls-server
spec:
  selector:
    app: mtls-server
  ports:
  - port: 8443
    targetPort: 8443
    name: https
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mtls-server
  namespace: mtls-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mtls-server
  template:
    metadata:
      labels:
        app: mtls-server
    spec:
      serviceAccountName: mtls-server-sa
      containers:
      - name: server
        image: registry.access.redhat.com/ubi9/ubi:latest
        command:
        - /bin/bash
        - -c
        - |
          dnf install -y openssl &>/dev/null
          echo "Waiting for SVID files..."
          while [ ! -f /certs/svid.pem ]; do
            sleep 5
          done
          echo "✓ SVID ready! Starting mTLS server on port 8443..."
          openssl x509 -in /certs/svid.pem -noout -subject
          while true; do
            openssl s_server \
              -cert /certs/svid.pem \
              -key /certs/svid_key.pem \
              -CAfile /certs/bundle.pem \
              -Verify 1 \
              -verify_return_error \
              -accept 8443 \
              -www 2>&1
            sleep 1
          done
        ports:
        - containerPort: 8443
        volumeMounts:
        - name: certs
          mountPath: /certs
          readOnly: true
      - name: spiffe-helper
        image: ghcr.io/spiffe/spiffe-helper:0.8.0
        args: ["-config", "/config/helper.conf"]
        volumeMounts:
        - name: spiffe-workload-api
          mountPath: /spiffe-workload-api
          readOnly: true
        - name: certs
          mountPath: /certs
        - name: helper-config
          mountPath: /config
      volumes:
      - name: spiffe-workload-api
        csi:
          driver: csi.spiffe.io
          readOnly: true
      - name: certs
        emptyDir: {}
      - name: helper-config
        configMap:
          name: spiffe-helper-config
EOF

# Wait for server to be ready
echo "Waiting for mTLS server..."
oc wait -n mtls-server --for=condition=Ready pod -l app=mtls-server --timeout=180s
```

#### 5.1.4 Create TLS Passthrough Route (Cluster 1)

```bash
# Get cluster's apps domain
CLUSTER1_DOMAIN=$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
CLUSTER1_DOMAIN="apps.${CLUSTER1_DOMAIN}"

oc apply -f - <<EOF
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: mtls-secure
  namespace: mtls-server
spec:
  host: mtls-secure.${CLUSTER1_DOMAIN}
  port:
    targetPort: https
  tls:
    termination: passthrough
  to:
    kind: Service
    name: mtls-server
    weight: 100
EOF

# Get the route URL
echo "Server Route: https://$(oc get route mtls-secure -n mtls-server -o jsonpath='{.spec.host}')"
```

---

### Step 5.2: Deploy mTLS Client (Cluster 2)

#### 5.2.1 Create Namespace and ClusterSPIFFEID

```bash
# Set the remote cluster's domain (Cluster 1's apps domain)
export REMOTE_DOMAIN="<CLUSTER1_APPS_DOMAIN>"

# Create namespace
oc create namespace mtls-client

# Create ClusterSPIFFEID with federation
oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: mtls-client
spec:
  className: zero-trust-workload-identity-manager-spire
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels:
      app: mtls-client
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: mtls-client
  federatesWith:
    - "${REMOTE_DOMAIN}"
EOF
```

#### 5.2.2 Create spiffe-helper ConfigMap

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: spiffe-helper-config
  namespace: mtls-client
data:
  helper.conf: |
    agent_address = "/spiffe-workload-api/spire-agent.sock"
    cmd = ""
    cert_dir = "/certs"
    svid_file_name = "svid.pem"
    svid_key_file_name = "svid_key.pem"
    svid_bundle_file_name = "bundle.pem"
    renew_signal = ""
EOF
```

#### 5.2.3 Deploy mTLS Client Pod

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mtls-client-sa
  namespace: mtls-client
---
apiVersion: v1
kind: Pod
metadata:
  name: mtls-client
  namespace: mtls-client
  labels:
    app: mtls-client
spec:
  serviceAccountName: mtls-client-sa
  containers:
  - name: client
    image: registry.access.redhat.com/ubi9/ubi:latest
    command:
    - /bin/bash
    - -c
    - |
      dnf install -y openssl &>/dev/null
      echo "Waiting for SVID files..."
      while [ ! -f /certs/svid.pem ]; do
        sleep 5
      done
      echo "✓ SVID ready! Client is ready for mTLS testing."
      openssl x509 -in /certs/svid.pem -noout -subject
      sleep infinity
    volumeMounts:
    - name: certs
      mountPath: /certs
      readOnly: true
  - name: spiffe-helper
    image: ghcr.io/spiffe/spiffe-helper:0.8.0
    args: ["-config", "/config/helper.conf"]
    volumeMounts:
    - name: spiffe-workload-api
      mountPath: /spiffe-workload-api
      readOnly: true
    - name: certs
      mountPath: /certs
    - name: helper-config
      mountPath: /config
  volumes:
  - name: spiffe-workload-api
    csi:
      driver: csi.spiffe.io
      readOnly: true
  - name: certs
    emptyDir: {}
  - name: helper-config
    configMap:
      name: spiffe-helper-config
EOF

# Wait for client to be ready
echo "Waiting for mTLS client..."
oc wait -n mtls-client --for=condition=Ready pod mtls-client --timeout=180s
```

---

### Step 5.3: Verify SVID Files

#### On Cluster 1 (Server):

```bash
# Check SVID files exist
oc exec -n mtls-server $(oc get pod -n mtls-server -l app=mtls-server -o name | head -1) -c server -- ls -la /certs/

# View the SPIFFE ID
oc exec -n mtls-server $(oc get pod -n mtls-server -l app=mtls-server -o name | head -1) -c server -- \
  openssl x509 -in /certs/svid.pem -noout -subject -ext subjectAltName
```

**Expected Output:**
```
-rw-r--r--. 1 ... bundle.pem
-rw-r--r--. 1 ... svid.pem
-rw-------. 1 ... svid_key.pem

subject=C=US, O=SPIRE
X509v3 Subject Alternative Name:
    URI:spiffe://<CLUSTER1_APPS_DOMAIN>/ns/mtls-server/sa/mtls-server-sa
```

#### On Cluster 2 (Client):

```bash
# Check SVID files exist
oc exec -n mtls-client mtls-client -c client -- ls -la /certs/

# View the SPIFFE ID
oc exec -n mtls-client mtls-client -c client -- \
  openssl x509 -in /certs/svid.pem -noout -subject -ext subjectAltName
```

**Expected Output:**
```
-rw-r--r--. 1 ... bundle.pem
-rw-r--r--. 1 ... svid.pem
-rw-------. 1 ... svid_key.pem

subject=C=US, O=SPIRE
X509v3 Subject Alternative Name:
    URI:spiffe://<CLUSTER2_APPS_DOMAIN>/ns/mtls-client/sa/mtls-client-sa
```

---

### Step 5.4: Verify FederatesWith in SPIRE Entries

#### On Cluster 1:

```bash
oc -n zero-trust-workload-identity-manager exec spire-server-0 -c spire-server -- \
  /spire-server entry show -socketPath /tmp/spire-server/private/api.sock | grep -A10 "mtls-server"
```

**Expected:** Should show `FederatesWith: <CLUSTER2_APPS_DOMAIN>`

#### On Cluster 2:

```bash
oc -n zero-trust-workload-identity-manager exec spire-server-0 -c spire-server -- \
  /spire-server entry show -socketPath /tmp/spire-server/private/api.sock | grep -A10 "mtls-client"
```

**Expected:** Should show `FederatesWith: <CLUSTER1_APPS_DOMAIN>`

---

### Step 5.5: Execute mTLS Test 🔐

This is the key test! The client on Cluster 2 will connect to the server on Cluster 1 using mutual TLS with SPIFFE certificates.

**Run on Cluster 2:**

```bash
# Set the server hostname (from Cluster 1's route)
SERVER_HOST="mtls-secure.<CLUSTER1_APPS_DOMAIN>"

echo "=========================================="
echo "🔐 mTLS TEST: Cluster 2 Client → Cluster 1 Server"
echo "=========================================="
echo ""
echo "Client presents: Cluster 2 SPIFFE SVID"
echo "Server presents: Cluster 1 SPIFFE SVID"
echo "Both verify certificates using federated trust bundle"
echo ""

# Execute mTLS connection
oc exec -n mtls-client mtls-client -c client -- \
  openssl s_client \
    -connect ${SERVER_HOST}:443 \
    -cert /certs/svid.pem \
    -key /certs/svid_key.pem \
    -CAfile /certs/bundle.pem \
    -verify 1 \
    -brief \
    </dev/null 2>&1
```

---

### Step 5.6: Expected mTLS Test Output ✅

```
==========================================
🔐 mTLS TEST: Cluster 2 Client → Cluster 1 Server
==========================================

Client presents: Cluster 2 SPIFFE SVID
Server presents: Cluster 1 SPIFFE SVID
Both verify certificates using federated trust bundle

verify depth is 1
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Ciphersuite: TLS_AES_256_GCM_SHA384
Peer certificate: C=US, O=SPIRE
Hash used: SHA256
Signature type: ecdsa_secp256r1_sha256
Verification error: self-signed certificate in certificate chain
DONE

==========================================
```

### Understanding the Output

| Field | Meaning | Expected Value |
|-------|---------|----------------|
| `CONNECTION ESTABLISHED` | mTLS handshake succeeded | ✅ Must see this |
| `Protocol version` | TLS version used | TLSv1.2 or TLSv1.3 |
| `Ciphersuite` | Encryption algorithm | Strong cipher (AES_256) |
| `Peer certificate: C=US, O=SPIRE` | Server's SPIFFE cert | Confirms SPIFFE identity |
| `self-signed certificate` | SPIRE's CA is self-signed | ⚠️ Expected (not an error!) |

> **Note:** The "self-signed certificate in certificate chain" message is **normal** for SPIFFE/SPIRE. Trust is established through federation bundle exchange, not public PKI. The key indicator is `CONNECTION ESTABLISHED`.

---

### Step 5.7: Test Results Summary

| Test | What it Proves | Expected |
|------|----------------|----------|
| Server SVID files exist | SPIRE issued identity to server | ✅ 3 files in /certs |
| Client SVID files exist | SPIRE issued identity to client | ✅ 3 files in /certs |
| Server has FederatesWith | Server trusts Cluster 2 | ✅ Remote domain listed |
| Client has FederatesWith | Client trusts Cluster 1 | ✅ Remote domain listed |
| mTLS Connection | Cross-cluster authentication | ✅ CONNECTION ESTABLISHED |

### 🎉 If all tests pass, federation is fully working!

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

### Remove mTLS Test Workloads

**On Cluster 1 (Server):**
```bash
oc delete namespace mtls-server
oc delete clusterspiffeid mtls-server
```

**On Cluster 2 (Client):**
```bash
oc delete namespace mtls-client
oc delete clusterspiffeid mtls-client
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


