# SPIFFE Federation with ACME (Let's Encrypt) - Complete Guide

## Hybrid Federation: https_spiffe + https_web (cert-manager / Let's Encrypt)

**Document Version**: 3.0  
**Last Updated**: February 2, 2026  
**OpenShift Version**: 4.18+  
**Tested On**: AWS ↔ GCP Multi-Cloud Federation  
**Target Audience**: DevOps Engineers, Security Teams, Platform Architects

---

## Table of Contents

1. [Introduction](#introduction)
2. [What is ACME?](#what-is-acme)
3. [What is SPIFFE/SPIRE?](#what-is-spiffespire)
4. [Why Combine SPIFFE + cert-manager/ACME?](#why-combine-spiffe--cert-manageracme)
5. [Real-World Example: Netflix-Style Architecture](#real-world-example-netflix-style-architecture)
6. [Federation Profiles Explained](#federation-profiles-explained)
7. [Prerequisites](#prerequisites)
8. [Phase 1: Install ZTWIM Operator](#phase-1-install-ztwim-operator)
9. [Phase 2: Deploy SPIRE Stack on Cluster 1 (https_spiffe)](#phase-2-deploy-spire-stack-on-cluster-1-https_spiffe)
10. [Phase 3: Deploy SPIRE Stack on Cluster 2 (https_web + cert-manager)](#phase-3-deploy-spire-stack-on-cluster-2-https_web--cert-manager)
11. [Phase 4: Install cert-manager and Issue Let's Encrypt Certificate](#phase-4-install-cert-manager-and-issue-lets-encrypt-certificate)
12. [Phase 5: Patch SpireServer to Use External Certificate](#phase-5-patch-spireserver-to-use-external-certificate)
13. [Phase 6: Fetch Trust Bundles](#phase-6-fetch-trust-bundles)
14. [Phase 7: Create ClusterFederatedTrustDomain](#phase-7-create-clusterfederatedtrustdomain)
15. [Phase 8: Verify Federation](#phase-8-verify-federation)
16. [Phase 9: Cross-Cluster mTLS Testing](#phase-9-cross-cluster-mtls-testing)
17. [Comprehensive Test Suite](#comprehensive-test-suite)
18. [Troubleshooting](#troubleshooting)
19. [Best Practices](#best-practices)
20. [Glossary](#glossary)
21. [Quick Reference](#quick-reference)
22. [Document History](#document-history)

---

## Introduction

This guide provides a **complete, step-by-step walkthrough** for setting up SPIFFE federation between two OpenShift clusters using a **hybrid approach**:

- **Cluster 1**: Uses `https_spiffe` profile (self-signed SPIRE certificate)
- **Cluster 2**: Uses `https_web` profile with **cert-manager** and **Let's Encrypt** for publicly trusted certificates

### What You'll Accomplish

By following this guide, you will:

1. ✅ Install the Zero Trust Workload Identity Manager (ZTWIM) operator on both clusters
2. ✅ Deploy SPIRE stacks with different federation profiles
3. ✅ Configure cert-manager with Let's Encrypt on Cluster 2
4. ✅ Establish **bidirectional federation** between clusters
5. ✅ Test **cross-cluster mTLS** communication using SPIFFE identities
6. ✅ Understand the complete federation flow with practical examples

---

## What is ACME?

### Definition

**ACME** = **A**utomated **C**ertificate **M**anagement **E**nvironment (RFC 8555)

ACME is an internet protocol that automates:

1. **Domain validation** – Proving you own/control a domain
2. **Certificate issuance** – Obtaining TLS certificates
3. **Certificate renewal** – Renewing before expiration

### How ACME Works

```
┌─────────────────┐                    ┌──────────────────┐
│ Your Server      │  1. "I want cert   │ ACME Server       │
│ (SPIRE +         │     for            │ (Let's Encrypt)   │
│  cert-manager)   │     federation.    │                   │
│                  │     example.com"   │                   │
│                  │ ──────────────────▶│                   │
│                  │                    │                   │
│                  │  2. "Prove it:     │                   │
│                  │     serve file at   │                   │
│                  │     /.well-known/   │                   │
│                  │     acme-challenge/ │                   │
│                  │     <token>"       │                   │
│                  │ ◀──────────────────│                   │
│                  │                    │                   │
│                  │  3. (cert-manager   │                   │
│                  │     serves file,    │                   │
│                  │     Let's Encrypt   │                   │
│                  │     verifies)       │                   │
│                  │                    │                   │
│                  │  4. "Here's your    │                   │
│                  │     certificate!"   │                   │
│                  │ ◀──────────────────│                   │
└─────────────────┘                    └──────────────────┘
```

### ACME Challenge Types

| Challenge Type   | How It Works                    | When to Use                    |
|------------------|----------------------------------|--------------------------------|
| **HTTP-01**      | Serve a file via HTTP on port 80| Most common; **we use this**   |
| **DNS-01**       | Create a DNS TXT record         | Wildcards, no HTTP access     |
| **TLS-ALPN-01**  | Special TLS handshake on 443    | When only 443 is open         |

---

## What is SPIFFE/SPIRE?

### SPIFFE (Specification)

**SPIFFE** = **S**ecure **P**roduction **I**dentity **F**ramework **F**or **E**veryone

SPIFFE defines:

- **SPIFFE ID**: A URI-based identity (e.g., `spiffe://example.com/ns/default/sa/myapp`)
- **SVID**: SPIFFE Verifiable Identity Document (X.509 certificate or JWT)
- **Trust Bundle**: Collection of CA certificates for a trust domain

### SPIRE (Implementation)

**SPIRE** = **SP**IFFE **R**untime **E**nvironment

SPIRE is the reference implementation that:

- Issues SVIDs to workloads
- Manages trust bundles
- Handles **federation** between trust domains

### How Workload Identity Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│ SPIFFE WORKLOAD IDENTITY                                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              │
│  │ Workload     │◀───▶│ SPIRE Agent  │◀───▶│ SPIRE Server │              │
│  │ (Your App)   │ SVID│ (per node)   │     │ (central)    │              │
│  └──────────────┘    └──────────────┘    └──────────────┘              │
│         │                    │                    │                     │
│         ▼                    ▼                    ▼                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              │
│  │ SPIFFE ID:   │    │ Workload API  │    │ Trust Bundle: │              │
│  │ spiffe://    │    │ (Unix socket  │    │ CA certs,     │              │
│  │ domain/ns/   │    │ via CSI       │    │ JWT keys      │              │
│  │ app/sa/x     │    │ driver)       │    │               │              │
│  └──────────────┘    └──────────────┘    └──────────────┘              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Why Combine SPIFFE + cert-manager/ACME?

### The Federation Challenge

When two SPIRE servers need to trust each other:

```
┌─────────────────┐         Federation          ┌─────────────────┐
│ CLUSTER 1        │  ◀─────────────────────────▶ │ CLUSTER 2        │
│ SPIRE Server     │                               │ SPIRE Server     │
│                  │  "How do I trust you?"        │                  │
└─────────────────┘                               └─────────────────┘
```

### Three Solutions (Federation Profiles)

| Profile                     | Certificate Type        | Trust Model           | Use Case           |
|-----------------------------|-------------------------|------------------------|--------------------|
| **https_spiffe**            | Self-signed SPIRE cert  | Manual bundle exchange | Internal clusters  |
| **https_web (ACME)**        | Let's Encrypt cert      | Public CA trust        | Internet-facing    |
| **https_web (servingCert)** | cert-manager → Secret   | ExternalSecretRef      | Hybrid (this guide)|

### The Hybrid Approach

```
┌─────────────────────────────────────────────────────────────────────────┐
│ HYBRID FEDERATION (https_spiffe + https_web with cert-manager)           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────────────┐    ┌──────────────────────────┐           │
│  │ CLUSTER 1 (INTERNAL)      │    │ CLUSTER 2 (EXTERNAL)     │           │
│  │                           │    │                           │           │
│  │ Profile: https_spiffe     │    │ Profile: https_web        │           │
│  │ Cert: Self-signed SPIRE   │    │ Cert: cert-manager →      │           │
│  │ Internet: Not required    │    │       Let's Encrypt       │           │
│  │                           │    │ Internet: Required        │           │
│  │ ✓ Fast setup              │    │ ✓ Publicly trusted        │           │
│  │ ✓ No external deps        │    │ ✓ No -k flag needed       │           │
│  │ ✓ Internal only           │    │ ✓ Partner-friendly        │           │
│  └──────────────────────────┘    └──────────────────────────┘           │
│                                                                         │
│  ◀────────────────────── BIDIRECTIONAL TRUST ──────────────────────▶    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example: Netflix-Style Architecture

### The Business Scenario

Imagine **Netflix** has two OpenShift clusters:

- **Cluster 1 (AWS)**: Internal content processing (video encoding, ML recommendations)
- **Cluster 2 (GCP)**: Public-facing streaming (user auth, video delivery)

| Cluster     | Platform | Purpose                          | Exposure        |
|------------|----------|-----------------------------------|-----------------|
| **Cluster 1** | AWS      | Encoding, ML, content management   | Internal only    |
| **Cluster 2** | GCP      | User auth, streaming, payments    | Internet-facing  |

### Why This Hybrid Setup?

**Cluster 1 (Internal)** – Uses `https_spiffe`:
- No internet exposure needed
- Like an **internal company ID badge** - works inside the building

**Cluster 2 (Public-Facing)** – Uses `https_web` with cert-manager:
- Exposed to internet; partners/auditors need standard TLS
- Like a **government-issued passport** - recognized everywhere

### Cross-Cluster mTLS Use Case

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CROSS-CLUSTER mTLS COMMUNICATION                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Cluster 2 (GCP)                              Cluster 1 (AWS)           │
│  ┌─────────────────┐                         ┌─────────────────┐        │
│  │  mtls-client    │                         │  mtls-server    │        │
│  │                 │  ──── TLSv1.3 ────────▶ │                 │        │
│  │  SPIFFE ID:     │      (Mutual Auth)      │  SPIFFE ID:     │        │
│  │  spiffe://gcp/  │                         │  spiffe://aws/  │        │
│  │  .../mtls-client│                         │  .../mtls-server│        │
│  │                 │                         │                 │        │
│  │  Certificate:   │                         │  Verifies with: │        │
│  │  - svid.pem     │                         │  - Cluster 2 CA │        │
│  │  - Combined CA  │                         │    (federated)  │        │
│  └─────────────────┘                         └─────────────────┘        │
│                                                                         │
│  Both sides verify certificates using FEDERATED TRUST BUNDLES           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Federation Profiles Explained

### Profile 1: https_spiffe (Self-Signed)

```yaml
federation:
  bundleEndpoint:
    profile: https_spiffe
  managedRoute: "true"
```

**Characteristics:**
- Certificate signed by SPIRE's own CA
- Requires manual trust bundle exchange
- `curl -k` needed (self-signed)

### Profile 2: https_web with cert-manager (serving cert)

```yaml
federation:
  bundleEndpoint:
    profile: https_web
    httpsWeb:
      servingCert:
        externalSecretRef: spire-server-federation-tls
        fileSyncInterval: 86400
```

**Characteristics:**
- Certificate from Let's Encrypt via cert-manager
- Publicly trusted; no `-k` needed
- Requires internet for ACME challenge

---

## Prerequisites

Before starting, ensure you have:

- [ ] **Two OpenShift 4.18+ clusters** with admin access
- [ ] **oc CLI** installed and configured
- [ ] **Network connectivity** to both clusters' ingress
- [ ] **Internet access** on Cluster 2 (for ACME certificate validation)
- [ ] **Kubeconfig files** for both clusters

---

## Phase 1: Install ZTWIM Operator

> **Run on BOTH clusters**

### Step 1.1: Set Up Environment Variables

```bash
# Set your kubeconfig paths
export KUBECONFIG1=~/Downloads/cluster1.kubeconfig
export KUBECONFIG2=~/Downloads/cluster2.kubeconfig

# Verify access
KUBECONFIG=$KUBECONFIG1 oc whoami
KUBECONFIG=$KUBECONFIG2 oc whoami
```

### Step 1.2: Install Operator on Cluster 1

```bash
export KUBECONFIG=$KUBECONFIG1

cat <<'EOF' | oc apply -f -
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
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  name: openshift-zero-trust-workload-identity-manager
  channel: stable-v1
EOF

# Wait for operator
oc wait --for=condition=Available deployment/zero-trust-workload-identity-manager-controller-manager \
  -n zero-trust-workload-identity-manager --timeout=5m

oc get pods -n zero-trust-workload-identity-manager
```

### Step 1.3: Install Operator on Cluster 2

```bash
export KUBECONFIG=$KUBECONFIG2

cat <<'EOF' | oc apply -f -
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
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  name: openshift-zero-trust-workload-identity-manager
  channel: stable-v1
EOF

# Wait for operator
oc wait --for=condition=Available deployment/zero-trust-workload-identity-manager-controller-manager \
  -n zero-trust-workload-identity-manager --timeout=5m

oc get pods -n zero-trust-workload-identity-manager
```

**Expected Output:**
```
NAME                                                              READY   STATUS    RESTARTS   AGE
zero-trust-workload-identity-manager-controller-manager-xxxxx      1/1     Running   0          30s
```

---

## Phase 2: Deploy SPIRE Stack on Cluster 1 (https_spiffe)

### Step 2.1: Get Cluster Domains

```bash
export KUBECONFIG=$KUBECONFIG1

# Get Cluster 1 domain
export CLUSTER1_APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
echo "CLUSTER1_APP_DOMAIN: ${CLUSTER1_APP_DOMAIN}"

# Get Cluster 2 domain (switch kubeconfig temporarily)
export KUBECONFIG=$KUBECONFIG2
export CLUSTER2_APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
echo "CLUSTER2_APP_DOMAIN: ${CLUSTER2_APP_DOMAIN}"

# Switch back to Cluster 1
export KUBECONFIG=$KUBECONFIG1
```

> **Note:** If you get permission errors, set the domains manually from your console URLs:
> ```bash
> export CLUSTER1_APP_DOMAIN=apps.your-cluster1-domain.com
> export CLUSTER2_APP_DOMAIN=apps.your-cluster2-domain.com
> ```

### Step 2.2: Deploy SPIRE Stack on Cluster 1

```bash
export KUBECONFIG=$KUBECONFIG1
export APP_DOMAIN=${CLUSTER1_APP_DOMAIN}
export JWT_ISSUER_ENDPOINT=oidc-discovery.${APP_DOMAIN}
export CLUSTER_NAME=cluster1

echo "Deploying SPIRE Stack on Cluster 1 with https_spiffe profile..."

# ZeroTrustWorkloadIdentityManager
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: ${APP_DOMAIN}
  clusterName: ${CLUSTER_NAME}
EOF

# SPIRE Components
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
  federation:
    bundleEndpoint:
      profile: https_spiffe
    managedRoute: "true"
    federatesWith:
    - trustDomain: ${CLUSTER2_APP_DOMAIN}
      bundleEndpointUrl: https://federation.${CLUSTER2_APP_DOMAIN}
      bundleEndpointProfile: https_web
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

# Verify deployment
echo "Waiting for SPIRE pods..."
sleep 30
oc get pods -n zero-trust-workload-identity-manager
oc get routes -n zero-trust-workload-identity-manager
```

**Expected Output:**
```
NAME                                                              READY   STATUS    RESTARTS   AGE
spire-agent-xxxxx                                                 1/1     Running   0          30s
spire-server-0                                                    2/2     Running   0          30s
spire-spiffe-csi-driver-xxxxx                                     2/2     Running   0          30s
spire-spiffe-oidc-discovery-provider-xxxxx                        1/1     Running   0          30s
zero-trust-workload-identity-manager-controller-manager-xxxxx     1/1     Running   0          5m

NAME                            HOST/PORT                           PATH   SERVICES      PORT         TERMINATION
spire-server-federation         federation.apps.cluster1.com               spire-server  federation   passthrough
```

---

## Phase 3: Deploy SPIRE Stack on Cluster 2 (https_web + cert-manager)

### Step 3.1: Deploy SPIRE Stack on Cluster 2

```bash
export KUBECONFIG=$KUBECONFIG2
export APP_DOMAIN=${CLUSTER2_APP_DOMAIN}
export JWT_ISSUER_ENDPOINT=oidc-discovery.${APP_DOMAIN}
export CLUSTER_NAME=cluster2

echo "Deploying SPIRE Stack on Cluster 2 with https_web profile..."

# ZeroTrustWorkloadIdentityManager
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: ${APP_DOMAIN}
  clusterName: ${CLUSTER_NAME}
EOF

# SPIRE Components (with servingCert placeholder)
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
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        servingCert: {}
    managedRoute: "true"
    federatesWith:
    - trustDomain: ${CLUSTER1_APP_DOMAIN}
      bundleEndpointUrl: https://federation.${CLUSTER1_APP_DOMAIN}
      bundleEndpointProfile: https_spiffe
      endpointSpiffeId: spiffe://${CLUSTER1_APP_DOMAIN}/spire/server
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

# Verify deployment
echo "Waiting for SPIRE pods..."
sleep 30
oc get pods -n zero-trust-workload-identity-manager
oc get routes -n zero-trust-workload-identity-manager
```

---

## Phase 4: Install cert-manager and Issue Let's Encrypt Certificate

> **Run on Cluster 2 only**

### Step 4.1: Install cert-manager Operator

```bash
export KUBECONFIG=$KUBECONFIG2

# Create namespace and install operator
oc create namespace cert-manager-operator --dry-run=client -o yaml | oc apply -f -

oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cert-manager-operator
  namespace: cert-manager-operator
spec:
  targetNamespaces:
  - cert-manager-operator
EOF

oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-cert-manager-operator
  namespace: cert-manager-operator
spec:
  channel: stable-v1
  installPlanApproval: Automatic
  name: openshift-cert-manager-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Wait for cert-manager pods
echo "Waiting for cert-manager to be ready..."
sleep 60
oc get pods -n cert-manager
```

**Expected Output:**
```
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-xxxxx                        1/1     Running   0          30s
cert-manager-cainjector-xxxxx             1/1     Running   0          30s
cert-manager-webhook-xxxxx                1/1     Running   0          30s
```

### Step 4.2: Create ACME Issuer (Let's Encrypt)

```bash
export KUBECONFIG=$KUBECONFIG2
export APP_DOMAIN=${CLUSTER2_APP_DOMAIN}

oc apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-http01
  namespace: zero-trust-workload-identity-manager
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: openshift-default
EOF

oc get issuer -n zero-trust-workload-identity-manager
```

**Expected Output:**
```
NAME                 READY   STATUS                                                 AGE
letsencrypt-http01   True    The ACME account was registered with the ACME server   10s
```

### Step 4.3: Create Certificate and RBAC

```bash
export KUBECONFIG=$KUBECONFIG2
export APP_DOMAIN=${CLUSTER2_APP_DOMAIN}

# Create Certificate
oc apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: spire-server-federation-tls
  namespace: zero-trust-workload-identity-manager
spec:
  secretName: spire-server-federation-tls
  commonName: federation.${APP_DOMAIN}
  dnsNames:
    - federation.${APP_DOMAIN}
  usages:
    - server auth
  issuerRef:
    kind: Issuer
    name: letsencrypt-http01
EOF

# Create RBAC for router
oc create role secret-reader \
  --verb=get,list,watch \
  --resource=secrets \
  --resource-name=spire-server-federation-tls \
  -n zero-trust-workload-identity-manager

oc create rolebinding secret-reader-binding \
  --role=secret-reader \
  --serviceaccount=openshift-ingress:router \
  -n zero-trust-workload-identity-manager

# Wait for certificate to be ready
echo "Waiting for Let's Encrypt certificate..."
while true; do
  READY=$(oc get certificate spire-server-federation-tls -n zero-trust-workload-identity-manager -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null)
  if [ "$READY" = "True" ]; then
    echo "Certificate is READY!"
    break
  fi
  echo "Certificate not ready yet, waiting..."
  sleep 10
done

oc get certificate -n zero-trust-workload-identity-manager
```

**Expected Output:**
```
NAME                          READY   SECRET                        AGE
spire-server-federation-tls   True    spire-server-federation-tls   60s
```

---

## Phase 5: Patch SpireServer to Use External Certificate

> **Run on Cluster 2 only**

```bash
export KUBECONFIG=$KUBECONFIG2

# Patch SpireServer to use cert-manager certificate
oc patch spireserver cluster --type=merge -p '{"spec":{"federation":{"bundleEndpoint":{"profile":"https_web","httpsWeb":{"servingCert":{"externalSecretRef":"spire-server-federation-tls","fileSyncInterval":86400}}}}}}'

# Verify
oc get spireserver cluster -o jsonpath='{.spec.federation.bundleEndpoint.httpsWeb.servingCert}' | jq .
```

**Expected Output:**
```json
{
  "externalSecretRef": "spire-server-federation-tls",
  "fileSyncInterval": 86400
}
```

---

## Phase 6: Fetch Trust Bundles

> **Run from a machine that can reach both clusters**

```bash
# Fetch bundles from both federation endpoints
echo "Fetching Cluster 1 bundle (https_spiffe - needs -k)..."
curl -sk "https://federation.${CLUSTER1_APP_DOMAIN}" -o /tmp/cluster1-bundle.json

echo "Fetching Cluster 2 bundle (https_web - Let's Encrypt)..."
curl -s "https://federation.${CLUSTER2_APP_DOMAIN}" -o /tmp/cluster2-bundle.json

# Verify bundles are valid JSON
echo "Cluster 1 bundle keys: $(jq '.keys | length' /tmp/cluster1-bundle.json)"
echo "Cluster 2 bundle keys: $(jq '.keys | length' /tmp/cluster2-bundle.json)"
```

**Expected Output:**
```
Fetching Cluster 1 bundle (https_spiffe - needs -k)...
Fetching Cluster 2 bundle (https_web - Let's Encrypt)...
Cluster 1 bundle keys: 2
Cluster 2 bundle keys: 2
```

> **Note:** Cluster 2 bundle fetched **without `-k`** because it has a Let's Encrypt certificate!

---

## Phase 7: Create ClusterFederatedTrustDomain

### Step 7.1: On Cluster 1 – Trust Cluster 2

```bash
export KUBECONFIG=$KUBECONFIG1

BUNDLE_C2=$(cat /tmp/cluster2-bundle.json)

cat <<EOF | oc apply -f -
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: cluster-12-federation
spec:
  trustDomain: ${CLUSTER2_APP_DOMAIN}
  bundleEndpointURL: https://federation.${CLUSTER2_APP_DOMAIN}
  bundleEndpointProfile:
    type: https_web
  className: zero-trust-workload-identity-manager-spire
  trustDomainBundle: |
$(echo "$BUNDLE_C2" | sed 's/^/    /')
EOF

oc get clusterfederatedtrustdomain
```

### Step 7.2: On Cluster 2 – Trust Cluster 1

```bash
export KUBECONFIG=$KUBECONFIG2

BUNDLE_C1=$(cat /tmp/cluster1-bundle.json)

cat <<EOF | oc apply -f -
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: cluster-21-federation
spec:
  trustDomain: ${CLUSTER1_APP_DOMAIN}
  bundleEndpointURL: https://federation.${CLUSTER1_APP_DOMAIN}
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://${CLUSTER1_APP_DOMAIN}/spire/server
  className: zero-trust-workload-identity-manager-spire
  trustDomainBundle: |
$(echo "$BUNDLE_C1" | sed 's/^/    /')
EOF

oc get clusterfederatedtrustdomain
```

---

## Phase 8: Verify Federation

### Step 8.1: Check Federation on Both Clusters

```bash
# Cluster 1
export KUBECONFIG=$KUBECONFIG1
echo "=== Cluster 1 - Bundle List ==="
oc -n zero-trust-workload-identity-manager exec spire-server-0 -c spire-server -- \
  /spire-server bundle list -socketPath /tmp/spire-server/private/api.sock

# Cluster 2
export KUBECONFIG=$KUBECONFIG2
echo "=== Cluster 2 - Bundle List ==="
oc -n zero-trust-workload-identity-manager exec spire-server-0 -c spire-server -- \
  /spire-server bundle list -socketPath /tmp/spire-server/private/api.sock
```

**Expected:** Each cluster shows the **other** cluster's trust domain in its bundle list.

### Step 8.2: Test Federation Endpoints

```bash
# Test Cluster 1 (https_spiffe - needs -k)
echo "Cluster 1 federation endpoint:"
curl -sk -o /dev/null -w "HTTP %{http_code}\n" "https://federation.${CLUSTER1_APP_DOMAIN}"

# Test Cluster 2 (https_web - Let's Encrypt, no -k needed!)
echo "Cluster 2 federation endpoint:"
curl -s -o /dev/null -w "HTTP %{http_code}\n" "https://federation.${CLUSTER2_APP_DOMAIN}"

# Verify Let's Encrypt certificate
echo "Cluster 2 certificate issuer:"
echo | openssl s_client -connect federation.${CLUSTER2_APP_DOMAIN}:443 -servername federation.${CLUSTER2_APP_DOMAIN} 2>/dev/null | openssl x509 -noout -issuer
```

**Expected Output:**
```
Cluster 1 federation endpoint:
HTTP 200
Cluster 2 federation endpoint:
HTTP 200
Cluster 2 certificate issuer:
issuer=C=US, O=Let's Encrypt, CN=R13
```

---

## Phase 9: Cross-Cluster mTLS Testing

This is the most important test - it proves that workloads in different clusters can authenticate each other using federated SPIFFE identities.

### Step 9.1: Deploy mTLS Server on Cluster 1

```bash
export KUBECONFIG=$KUBECONFIG1

# Create namespace and service account
oc create namespace federation-test --dry-run=client -o yaml | oc apply -f -
oc create serviceaccount mtls-server -n federation-test --dry-run=client -o yaml | oc apply -f -

# ClusterSPIFFEID for the server (federates with Cluster 2)
cat <<EOF | oc apply -f -
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: mtls-server-workload
spec:
  className: zero-trust-workload-identity-manager-spire
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels:
      spiffe.io/spiffe-id: "true"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: federation-test
  federatesWith:
    - "${CLUSTER2_APP_DOMAIN}"
EOF

# ConfigMap for spiffe-helper
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: spiffe-helper-config
  namespace: federation-test
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

# mTLS Server Pod with spiffe-helper sidecar
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: mtls-server
  namespace: federation-test
  labels:
    app: mtls-server
    spiffe.io/spiffe-id: "true"
spec:
  serviceAccountName: mtls-server
  containers:
  - name: server
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["/bin/bash", "-c"]
    args:
      - |
        echo "Waiting for certificates..."
        while [ ! -f /certs/svid.pem ]; do sleep 2; done
        echo "Certificates found! Starting openssl s_server on port 8443..."
        openssl s_server -accept 8443 -cert /certs/svid.pem -key /certs/svid_key.pem -CAfile /certs/bundle.pem -Verify 1 -www
    ports:
    - containerPort: 8443
    volumeMounts:
    - name: certs
      mountPath: /certs
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
    - name: certs
      mountPath: /certs
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
  - name: certs
    emptyDir: {}
  - name: spiffe-helper-config
    configMap:
      name: spiffe-helper-config
EOF

# Service and Route
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: mtls-server
  namespace: federation-test
spec:
  selector:
    app: mtls-server
  ports:
  - port: 8443
    targetPort: 8443
    name: https
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: mtls-server
  namespace: federation-test
spec:
  host: mtls-server.${CLUSTER1_APP_DOMAIN}
  to:
    kind: Service
    name: mtls-server
  port:
    targetPort: https
  tls:
    termination: passthrough
EOF

# Wait for pod
echo "Waiting for mtls-server pod..."
oc wait --for=condition=Ready pod/mtls-server -n federation-test --timeout=120s
oc get pods -n federation-test
```

### Step 9.2: Deploy mTLS Client on Cluster 2

```bash
export KUBECONFIG=$KUBECONFIG2

# Create namespace and service account
oc create namespace federation-test --dry-run=client -o yaml | oc apply -f -
oc create serviceaccount mtls-client -n federation-test --dry-run=client -o yaml | oc apply -f -

# ClusterSPIFFEID for the client (federates with Cluster 1)
cat <<EOF | oc apply -f -
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: mtls-client-workload
spec:
  className: zero-trust-workload-identity-manager-spire
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels:
      spiffe.io/spiffe-id: "true"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: federation-test
  federatesWith:
    - "${CLUSTER1_APP_DOMAIN}"
EOF

# ConfigMap for spiffe-helper
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: spiffe-helper-config
  namespace: federation-test
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

# mTLS Client Pod with spiffe-helper sidecar
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: mtls-client
  namespace: federation-test
  labels:
    app: mtls-client
    spiffe.io/spiffe-id: "true"
spec:
  serviceAccountName: mtls-client
  containers:
  - name: client
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["sleep", "infinity"]
    volumeMounts:
    - name: certs
      mountPath: /certs
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
    - name: certs
      mountPath: /certs
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
  - name: certs
    emptyDir: {}
  - name: spiffe-helper-config
    configMap:
      name: spiffe-helper-config
EOF

# Wait for pod
echo "Waiting for mtls-client pod..."
oc wait --for=condition=Ready pod/mtls-client -n federation-test --timeout=120s
oc get pods -n federation-test
```

### Step 9.3: Create Combined CA Bundle

For cross-cluster mTLS, we need a bundle containing CA certificates from **both** trust domains:

```bash
# Fetch both federation bundles and extract CA certs
echo "Creating combined CA bundle..."

# Extract Cluster 1 CA
curl -sk "https://federation.${CLUSTER1_APP_DOMAIN}" | \
  jq -r '.keys[] | select(.use=="x509-svid") | .x5c[0]' | \
  base64 -d | openssl x509 -inform DER -out /tmp/cluster1-ca.pem

# Extract Cluster 2 CA
curl -s "https://federation.${CLUSTER2_APP_DOMAIN}" | \
  jq -r '.keys[] | select(.use=="x509-svid") | .x5c[0]' | \
  base64 -d | openssl x509 -inform DER -out /tmp/cluster2-ca.pem

# Combine
cat /tmp/cluster1-ca.pem /tmp/cluster2-ca.pem > /tmp/combined-bundle.pem
echo "Combined bundle has $(grep -c 'BEGIN CERTIFICATE' /tmp/combined-bundle.pem) certificates"

# Copy to client pod
export KUBECONFIG=$KUBECONFIG2
oc cp /tmp/combined-bundle.pem federation-test/mtls-client:/certs/combined-bundle.pem -c client
```

### Step 9.4: Test mTLS Connection

```bash
export KUBECONFIG=$KUBECONFIG2

SERVER_HOST="mtls-server.${CLUSTER1_APP_DOMAIN}"

echo "============================================"
echo "CROSS-CLUSTER mTLS TEST"
echo "============================================"
echo "Client: Cluster 2 (${CLUSTER2_APP_DOMAIN})"
echo "Server: Cluster 1 (${CLUSTER1_APP_DOMAIN})"
echo "============================================"

oc exec -n federation-test mtls-client -c client -- bash -c "
echo 'Q' | timeout 15 openssl s_client \
  -connect ${SERVER_HOST}:443 \
  -servername ${SERVER_HOST} \
  -cert /certs/svid.pem \
  -key /certs/svid_key.pem \
  -CAfile /certs/combined-bundle.pem \
  2>&1 | grep -E '(Verification|verify return|CONNECTED|Verify return code)'
"
```

**Expected Output (SUCCESS):**
```
verify return:1
verify return:1
CONNECTED(00000003)
Verification: OK
Verify return code: 0 (ok)
```

### Step 9.5: Verify SPIFFE IDs

```bash
# Check server's SPIFFE ID (from Cluster 1)
export KUBECONFIG=$KUBECONFIG1
echo "Server SPIFFE ID:"
oc exec -n federation-test mtls-server -c server -- \
  openssl x509 -in /certs/svid.pem -noout -text 2>/dev/null | grep -A1 "Subject Alternative Name"

# Check client's SPIFFE ID (from Cluster 2)
export KUBECONFIG=$KUBECONFIG2
echo "Client SPIFFE ID:"
oc exec -n federation-test mtls-client -c client -- \
  openssl x509 -in /certs/svid.pem -noout -text 2>/dev/null | grep -A1 "Subject Alternative Name"
```

**Expected Output:**
```
Server SPIFFE ID:
            X509v3 Subject Alternative Name: 
                URI:spiffe://apps.cluster1.example.com/ns/federation-test/sa/mtls-server

Client SPIFFE ID:
            X509v3 Subject Alternative Name: 
                URI:spiffe://apps.cluster2.example.com/ns/federation-test/sa/mtls-client
```

---

## Comprehensive Test Suite

### Test Results Summary

| Test ID | Description | Expected Result | Status |
|---------|-------------|-----------------|--------|
| P1 | ZTWIM operator installs | Pods Running | ✅ |
| P2 | SpireServer on Cluster 1 | spire-server-0 2/2 Running | ✅ |
| P3 | SpireServer on Cluster 2 | spire-server-0 2/2 Running | ✅ |
| P4 | cert-manager issues certificate | Certificate Ready=True | ✅ |
| P5 | Bundle fetch Cluster 1 | Valid JSON, 2 keys | ✅ |
| P6 | Bundle fetch Cluster 2 (no -k) | Valid JSON, 2 keys | ✅ |
| P7 | CFTD on Cluster 1 | Created | ✅ |
| P8 | CFTD on Cluster 2 | Created | ✅ |
| P9 | Federation bundle list | Shows remote trust domain | ✅ |
| P10 | mTLS connection | Verify return code: 0 | ✅ |

### Key Verification Commands

```bash
# Check SPIRE pods
oc get pods -n zero-trust-workload-identity-manager

# Check federation routes
oc get routes -n zero-trust-workload-identity-manager

# Check certificate (Cluster 2)
oc get certificate -n zero-trust-workload-identity-manager

# Check ClusterFederatedTrustDomain
oc get clusterfederatedtrustdomain

# Check bundle list from SPIRE server
oc -n zero-trust-workload-identity-manager exec spire-server-0 -c spire-server -- \
  /spire-server bundle list -socketPath /tmp/spire-server/private/api.sock
```

---

## Troubleshooting

### Problem: ACME challenge fails

```
Error: acme: error: 403 :: urn:ietf:params:acme:error:unauthorized
```

**Solution:**
1. Verify the federation route is reachable from the internet
2. Check DNS for `federation.${APP_DOMAIN}` resolves correctly
3. Ensure HTTP-01 challenge can reach port 80

### Problem: Certificate not Ready

**Solution:**
```bash
oc describe certificate spire-server-federation-tls -n zero-trust-workload-identity-manager
oc get challenges -n zero-trust-workload-identity-manager
```

### Problem: mTLS verification fails

```
verify error:num=19:self-signed certificate in certificate chain
```

**Solution:** Use the combined bundle containing both CAs:
```bash
cat cluster1-ca.pem cluster2-ca.pem > combined-bundle.pem
```

### Problem: Permission denied on `oc get dns cluster`

**Solution:** Set domains manually:
```bash
export CLUSTER1_APP_DOMAIN=apps.your-cluster1-domain.com
export CLUSTER2_APP_DOMAIN=apps.your-cluster2-domain.com
```

---

## Best Practices

| Category | Recommendation |
|----------|----------------|
| **Security** | Use `https_spiffe` for internal, `https_web` for external |
| **Testing** | Use Let's Encrypt staging first to avoid rate limits |
| **Operations** | Document trust domains and backup bundles |
| **mTLS** | Always use combined bundles for cross-cluster communication |

---

## Glossary

| Term | Definition |
|------|------------|
| **ACME** | Automated Certificate Management Environment |
| **SPIFFE** | Secure Production Identity Framework For Everyone |
| **SPIRE** | SPIFFE Runtime Environment |
| **SVID** | SPIFFE Verifiable Identity Document |
| **Trust Domain** | SPIFFE identity namespace (e.g., apps.example.com) |
| **Trust Bundle** | CA certificates for a trust domain |
| **Federation** | Trust between SPIRE servers in different trust domains |
| **https_spiffe** | Federation using SPIRE's self-signed certificate |
| **https_web** | Federation using publicly trusted certificate |
| **externalSecretRef** | Reference to cert-manager-issued Secret |
| **spiffe-helper** | Sidecar that writes SVIDs to files |
| **ZTWIM** | Zero Trust Workload Identity Manager |

---

## Quick Reference

### Complete Deployment Checklist

- [ ] Phase 1: Install ZTWIM operator on both clusters
- [ ] Phase 2: Deploy SPIRE stack on Cluster 1 (https_spiffe)
- [ ] Phase 3: Deploy SPIRE stack on Cluster 2 (https_web)
- [ ] Phase 4: Install cert-manager and issue certificate
- [ ] Phase 5: Patch SpireServer with externalSecretRef
- [ ] Phase 6: Fetch trust bundles
- [ ] Phase 7: Create ClusterFederatedTrustDomain on both clusters
- [ ] Phase 8: Verify federation
- [ ] Phase 9: Test cross-cluster mTLS

### Key URLs

```bash
# Federation endpoints
https://federation.${CLUSTER1_APP_DOMAIN}  # https_spiffe (use -k)
https://federation.${CLUSTER2_APP_DOMAIN}  # https_web (Let's Encrypt)

# mTLS server
https://mtls-server.${CLUSTER1_APP_DOMAIN}
```

---

## Document History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | Feb 2, 2026 | Initial guide |
| 2.0 | Feb 2, 2026 | Added diagrams, test suite |
| 3.0 | Feb 2, 2026 | Added complete mTLS testing with step-by-step commands |

---

*Document created: February 2, 2026*  
*Tested on: AWS (Cluster 1) ↔ GCP (Cluster 2) Multi-Cloud Federation*  
*All tests passed: Federation established, Let's Encrypt certificate working, cross-cluster mTLS verified*
