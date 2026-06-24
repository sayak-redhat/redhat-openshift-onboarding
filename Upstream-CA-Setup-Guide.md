# ZTWIM UpstreamAuthority with cert-manager — Complete Guide

**Feature:** SPIRE Server uses cert-manager as its CA (upstream authority)  
**Result:** cert-manager controls the root of trust → SPIRE's intermediate CA is signed by cert-manager  
**Tested on:** OpenShift 4.19, ZTWIM Operator v1.0.1, cert-manager Operator v1.19.0  
**Date:** 2026-05-20

---

## Table of Contents

1. [Foundational Concepts](#foundational-concepts)
2. [What Is UpstreamAuthority?](#what-is-upstreamauthority)
3. [Architecture — How It Works](#architecture--how-it-works)
4. [Operator & Operand Install Order](#operator--operand-install-order)
5. [Prerequisites](#prerequisites)
6. [Production-Safe Setup (No Agent Restart)](#production-safe-setup-no-agent-restart)
7. [Step-by-Step Commands with YAML Explanations](#step-by-step-commands-with-yaml-explanations)
8. [Verification Commands](#verification-commands)
9. [What NOT To Do (Common Mistakes)](#what-not-to-do-common-mistakes)
10. [Troubleshooting](#troubleshooting)
11. [Evidence from Testing](#evidence-from-testing)
12. [Glossary](#glossary)

---

## Foundational Concepts

### What is an SSL/TLS Certificate?

A certificate is like an **ID card for computers**:

```
WITHOUT certificate:
  Pod A ──── sends data ────► Pod B (could be a fake!)

WITH certificate:
  Pod A ──── "Who are you?" ────► Pod B
  Pod B ──── "Here's my cert" ──► Pod A
  Pod A checks cert ✅ → encrypted communication starts
```

Three things a certificate provides:
- **Identity** — proves WHO you are (like a passport)
- **Encryption** — scrambles data so only sender/receiver can read
- **Integrity** — proves data wasn't changed in transit

---

### What is SPIRE/SPIFFE?

**SPIFFE** = Secure Production Identity Framework For Everyone
- A standard that defines HOW workloads identify themselves
- Identity format: `spiffe://trust-domain/path` (e.g., `spiffe://cluster.local/ns/default/sa/myapp`)

**SPIRE** = the software that IMPLEMENTS SPIFFE
- Automatically gives every pod a certificate (called **SVID**)
- Certificates rotate automatically before expiry
- No human involvement needed

```
SPIRE Server (central CA — issues identity certificates)
    │
    │ issues SVIDs
    ▼
SPIRE Agent (runs on each node, DaemonSet)
    │
    │ delivers via workload API
    ▼
Pod gets SVID (its identity certificate)
    │
    │ uses SVID for
    ▼
mTLS communication with other pods ✅
```

---

### What is ZTWIM Operator?

**ZTWIM** = Zero Trust Workload Identity Manager

It's a Kubernetes **operator** that manages the lifecycle of SPIRE components:

| CR (Custom Resource) | What it creates |
|---|---|
| `SpireServer` | StatefulSet (spire-server-0), ConfigMaps, RBAC |
| `SpireAgent` | DaemonSet (spire-agent on every node) |
| `SpireCSIDriver` | DaemonSet (CSI driver for mounting SVIDs) |
| `SpireOIDCDiscoveryProvider` | Deployment (OIDC endpoint for JWT verification) |

---

### What is a CA (Certificate Authority)?

A CA is a trusted entity that signs certificates. Think of it as a government office that stamps passports.

```
Root CA (the government — ultimate trust)
    │ signs
Intermediate CA (a state office — delegated authority)
    │ signs
Leaf Certificate (your passport — individual identity)
```

Anyone who trusts the Root CA automatically trusts everything it has signed.

---

---

## What Is UpstreamAuthority?

### Without UpstreamAuthority (default — self-signed)

```
SPIRE Server
  ├── Creates its own CA certificate
  ├── Signs it with its OWN key (self-signed)
  ├── No external CA involved
  └── Trust chain: SPIRE CA → Workload SVIDs

Problem: Nobody outside the cluster can verify SPIRE's CA.
         No enterprise CA governance.
```

### With UpstreamAuthority (cert-manager)

```
cert-manager Root CA ("SPIRE Upstream CA")
  │
  │ signs (using its own CA key from spire-ca-secret)
  ▼
SPIRE Intermediate CA (self_signed=false)
  │
  │ signs
  ▼
Workload SVIDs (each pod's identity certificate)

Benefit: Enterprise CA controls the root of trust.
         Auditable, rotatable, externally verifiable.
```

### With UpstreamAuthority (Vault)

```
Vault PKI Engine Root CA
  │
  │ signs (using Vault's PKI secrets engine)
  ▼
SPIRE Intermediate CA (self_signed=false)
  │
  │ signs
  ▼
Workload SVIDs (each pod's identity certificate)

Benefit: Enterprise Vault controls the root of trust.
         HSM-backed keys, fine-grained audit, centralized PKI.
```

**How SPIRE authenticates to Vault (step by step):**

```
1. SPIRE server pod starts
2. Pod has a projected ServiceAccount token (mounted by the operator)
3. SPIRE sends this token to Vault: "I'm spire-server, authenticate me"
4. Vault verifies token with Kubernetes API → grants access
5. SPIRE asks Vault to sign its intermediate CA certificate
6. Vault's PKI engine signs it with Vault's root CA
7. SPIRE uses the signed intermediate to issue workload SVIDs
```

---

### Key Concepts Explained

#### What is a CA (Certificate Authority)?

A CA is a trusted entity that signs certificates. Think of it as a government office that stamps passports — anyone who trusts the government also trusts the passport.

```
Root CA (the government)
    │ signs
Intermediate CA (a state office)
    │ signs
Leaf Certificate (your passport)
```

#### What is cert-manager?

cert-manager is a Kubernetes-native tool that automates certificate lifecycle. It has:

| Object | Purpose | Scope |
|---|---|---|
| **Issuer** | Signs certificates | One namespace only |
| **ClusterIssuer** | Signs certificates | All namespaces |
| **Certificate** | Requests a cert and stores it in a Secret | Namespace-scoped |
| **CertificateRequest** | Short-lived object — a signing request | Auto-created, auto-deleted |

#### What is the Trust Bundle?

The trust bundle (`bundle.crt`) is a ConfigMap containing the root CA certificate. SPIRE distributes it to all agents so they can verify the server's identity.

```
SPIRE Server writes bundle.crt → spire-bundle ConfigMap
    │
    │ mounted into every agent pod
    ▼
Agent reads bundle → "I trust certificates signed by this CA"
    │
    ▼
Agent verifies server's certificate chain → connects securely ✅
```

#### Why CA-type Issuer? (NOT selfSigned)

| Issuer Type | How It Signs | Works for SPIRE? | Why |
|---|---|---|---|
| **selfSigned** | Uses the requester's own private key | **NO** | SPIRE doesn't expose its key to cert-manager |
| **CA** | Uses its own CA key (from a Secret) | **YES** | cert-manager has its own key, doesn't need SPIRE's |
| **ACME/Let's Encrypt** | External validation + issuance | **NO** | Won't issue CA certificates (isCA: true) |
| **Vault** | HashiCorp Vault PKI engine | **YES** | Can issue intermediate CA certs |

---

## Architecture — How It Works

### The Three-Object CA Chain

```
┌─────────────────────────────────────────────────────────────┐
│ Object 1: selfsigned-issuer (ClusterIssuer)                 │
│   Type: selfSigned                                          │
│   Purpose: Bootstrap ONLY — creates the root CA certificate │
│   Used once, then never again                               │
└──────────────────────────┬──────────────────────────────────┘
                           │ signs (one-time bootstrap)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ Object 2: spire-ca (Certificate)                            │
│   isCA: true                                                │
│   commonName: "SPIRE Upstream CA"                           │
│   secretName: spire-ca-secret (stores key + cert)           │
│   Purpose: This IS the root CA                              │
└──────────────────────────┬──────────────────────────────────┘
                           │ provides key to
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ Object 3: spire-ca-issuer (Issuer)                          │
│   Type: CA (backed by spire-ca-secret)                      │
│   Purpose: Signs SPIRE's intermediate CA certificate        │
│   SPIRE points to THIS in SpireServer CR                    │
└──────────────────────────┬──────────────────────────────────┘
                           │ signs (every CA rotation)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ SPIRE Server Intermediate CA                                │
│   self_signed=false                                         │
│   Used to sign workload SVIDs                               │
└──────────────────────────┬──────────────────────────────────┘
                           │ signs
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ Workload SVID (each pod's identity)                         │
│   spiffe://trust-domain/ns/default/sa/myapp                 │
└─────────────────────────────────────────────────────────────┘
```

### What Happens at SPIRE Startup

```
1. SPIRE Server starts
2. Loads cert-manager UpstreamAuthority plugin
3. Generates a key pair for its intermediate CA
4. Creates a CertificateRequest in cert-manager namespace
5. cert-manager's spire-ca-issuer signs it with root CA key
6. SPIRE receives signed certificate
7. SPIRE publishes root CA cert in trust bundle (spire-bundle ConfigMap)
8. SPIRE uses intermediate CA to sign all workload SVIDs
```

### What the Operator Auto-Creates (DO NOT create manually)

When you set `upstreamAuthority.certManager` in SpireServer CR, the ZTWIM operator automatically:

1. Updates `spire-server` ConfigMap with cert-manager plugin block
2. Creates a `Role` in `cert-manager` namespace (permission to create CertificateRequests)
3. Creates a `RoleBinding` in `cert-manager` namespace (binds to spire-server ServiceAccount)

---

## Operator & Operand Install Order

### What Gets Installed and When

```
┌─────────────────────────────────────────────────────────────────────┐
│ Step 1: Install cert-manager operator (via OperatorHub)             │
│   Creates: cert-manager namespace, cert-manager pods               │
│   Wait for: cert-manager, cainjector, webhook all Running           │
└─────────────────────────────────┬───────────────────────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Step 2: Create cert-manager CA chain (3 YAMLs — you do this)       │
│   Creates: selfsigned-issuer, spire-ca Certificate, spire-ca-issuer │
│   Wait for: Issuer shows READY=True                                 │
└─────────────────────────────────┬───────────────────────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Step 3: Install ZTWIM operator (via CatalogSource/OperatorHub)      │
│   Creates: controller-manager pod                                   │
│   Wait for: controller-manager Running                              │
└─────────────────────────────────┬───────────────────────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Step 4: Create ZeroTrustWorkloadIdentityManager CR (master switch)  │
│   Required: trustDomain, clusterName                                │
│   Without this CR → operator won't reconcile any operands           │
└─────────────────────────────────┬───────────────────────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Step 5: Create ALL operands WITH upstreamAuthority (you do this)    │
│   Apply single YAML containing:                                     │
│     - SpireServer (includes upstreamAuthority.certManager)          │
│     - SpireAgent                                                    │
│     - SpiffeCSIDriver                                               │
│     - SpireOIDCDiscoveryProvider                                    │
│   Operator reconciles → StatefulSet, ConfigMaps, RBAC, DaemonSets  │
│   spire-server-0 boots → uses cert-manager CA from first startup   │
│   Agents boot → get correct bundle → no restart needed ✅           │
└─────────────────────────────────────────────────────────────────────┘
```

### Operand Creation Timeline

```
Second 0:   You create ZeroTrustWorkloadIdentityManager CR (master switch)
Second 2:   You apply operands YAML (SpireServer + Agent + CSI + OIDC)
Second 5:   Operator reconciles SpireServer → creates StatefulSet, ConfigMaps, RBAC
Second 10:  spire-server-0 pod starts → sees upstreamAuthority config
Second 12:  SPIRE creates CertificateRequest → cert-manager signs it
Second 14:  SPIRE activates cert-manager-signed CA → fills spire-bundle
Second 18:  Operator sees server Ready → reconciles Agent DaemonSet
Second 22:  Agents start → mount spire-bundle (with cert-manager root) → connect
Second 25:  All pods Running ✅
```

> **KEY:** Create `ZeroTrustWorkloadIdentityManager` CR first, then include `upstreamAuthority` in the SpireServer CR — no patching needed, no restart required.

---

## Prerequisites

| Requirement | Why |
|---|---|
| OpenShift cluster | Platform |
| ZTWIM operator installed | Manages SpireServer/SpireAgent |
| cert-manager operator installed | Provides Issuer/Certificate resources |
| `cert-manager` namespace exists | Where cert-manager components run |
| cert-manager pods Running | Must be healthy before SPIRE can use it |

### Verify Prerequisites

```bash
# cert-manager pods healthy
oc get pods -n cert-manager

# ZTWIM operator and operands
oc get pods -n zero-trust-workload-identity-manager

# SpireServer exists
oc get spireserver cluster -n zero-trust-workload-identity-manager
```

---

## Production-Safe Setup (No Agent Restart)

### The Golden Rule

> **Set up `upstreamAuthority` BEFORE deploying SPIRE operands** (fresh install)  
> OR  
> **Wait for natural CA rotation** after adding upstreamAuthority to an existing cluster

### Why Agents Crash (and how to avoid it)

```
DANGEROUS (agent restart needed):
  1. SPIRE running with self-signed CA → agents trust CA-A
  2. Delete PVC → force new CA from cert-manager → agents don't trust CA-B → CRASH

SAFE (no agent restart):
  Option A: Fresh install with upstreamAuthority from day 1
    → Agents boot and immediately trust the cert-manager root CA

  Option B: Add upstreamAuthority to existing cluster, DON'T delete PVC
    → SPIRE continues with old CA until natural rotation
    → At rotation time, SPIRE includes BOTH old and new CA in bundle
    → Agents update trust bundle → seamless transition
```

### Recommended Flow for Fresh Install (SMOOTHEST)

```
Step 1: Install cert-manager operator
Step 2: Create cert-manager CA chain (3 objects)
Step 3: Install ZTWIM operator
Step 4: Create ZeroTrustWorkloadIdentityManager CR (master switch — operator needs this!)
Step 5: Apply ALL operands in one YAML (SpireServer WITH upstreamAuthority + Agent + CSI + OIDC)
Step 6: SPIRE boots → asks cert-manager to sign CA → agents get correct bundle → DONE
```

---

## Step-by-Step Commands with YAML Explanations

### Step 1: Create the cert-manager CA Chain

```bash
cat <<EOF | oc apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: spire-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: "SPIRE Upstream CA"
  secretName: spire-ca-secret
  duration: 87600h
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: spire-ca-issuer
  namespace: cert-manager
spec:
  ca:
    secretName: spire-ca-secret
EOF
```

#### YAML Breakdown — Object 1: `selfsigned-issuer`

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer           # Cluster-wide (not namespace-scoped)
metadata:
  name: selfsigned-issuer     # Name referenced by the Certificate below
spec:
  selfSigned: {}              # Type: selfSigned
                              # Used ONLY to bootstrap the root CA cert
                              # After bootstrap, this issuer is not used by SPIRE
```

**Why selfSigned here?** We need to create the FIRST certificate (root CA). There's nobody above it to sign it, so it signs itself. This is standard PKI bootstrapping — every CA chain starts with a self-signed root.

**Why ClusterIssuer (not Issuer)?** So it can be referenced from any namespace. The Certificate below is in `cert-manager` namespace.

---

#### YAML Breakdown — Object 2: `spire-ca` Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: spire-ca              # Just a name for this resource
  namespace: cert-manager     # MUST be in cert-manager namespace
                              # (ClusterIssuer stores secrets here by default)
spec:
  isCA: true                  # CRITICAL — marks this as a CA certificate
                              # Without this, it can't sign other certificates
  
  commonName: "SPIRE Upstream CA"  # The CN that will appear in the cert
                                   # This shows up in SPIRE's trust bundle
                                   # Choose a descriptive name
  
  secretName: spire-ca-secret      # cert-manager stores the generated key + cert HERE
                                   # Contains: tls.crt (cert), tls.key (private key), ca.crt
  
  duration: 87600h                 # 10 years validity
                                   # Root CAs should be long-lived
  
  issuerRef:
    name: selfsigned-issuer        # Signed by our bootstrap issuer
    kind: ClusterIssuer            # It's a ClusterIssuer (not namespace Issuer)
    group: cert-manager.io         # API group (always cert-manager.io)
```

**Why `isCA: true`?** Without this flag, the certificate's Basic Constraints won't include `CA:TRUE`, and it cannot be used to sign other certificates. SPIRE needs a CA certificate as its root.

**Why `namespace: cert-manager`?** ClusterIssuers store their secret in the `cert-manager` namespace by default. The Issuer in the next step (namespace-scoped) needs to access this secret from the same namespace.

**Why 87600h?** Root CAs should be long-lived (10 years). Intermediate CAs (SPIRE's) rotate frequently, but the root should be stable.

---

#### YAML Breakdown — Object 3: `spire-ca-issuer`

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer                  # Namespace-scoped (exists in cert-manager ns)
metadata:
  name: spire-ca-issuer       # Name SPIRE references in SpireServer CR
  namespace: cert-manager     # MUST match SpireServer's certManager.namespace
spec:
  ca:
    secretName: spire-ca-secret  # Uses the root CA key from Object 2
                                 # cert-manager reads the private key from this secret
                                 # and uses it to sign any CertificateRequest
```

**Why `ca:` type (not `selfSigned:`)?** This is the critical difference:
- `selfSigned:` → cert-manager needs the REQUESTER's private key → SPIRE doesn't expose it → FAILS
- `ca:` → cert-manager uses ITS OWN key (from `spire-ca-secret`) → works perfectly

**Why `Issuer` (not `ClusterIssuer`)?** The developer's design uses namespace-scoped Issuer. This is slightly more secure because the Issuer's scope is limited to the `cert-manager` namespace. Either works — just match `issuerKind` in SpireServer CR.

**Why `namespace: cert-manager`?** SPIRE creates its CertificateRequests in this namespace (as configured in `certManager.namespace`). The Issuer must be in the same namespace to process them.

---

### Step 2: Verify cert-manager Objects Are Ready

```bash
oc get clusterissuer selfsigned-issuer
oc get certificate spire-ca -n cert-manager
oc get issuer spire-ca-issuer -n cert-manager
oc get secret spire-ca-secret -n cert-manager
```

Expected output:
```
NAME                READY   AGE
selfsigned-issuer   True    30s

NAME       READY   AGE
spire-ca   True    30s

NAME              READY   STATUS                AGE
spire-ca-issuer   True    Signing CA verified   30s

NAME              TYPE                DATA   AGE
spire-ca-secret   kubernetes.io/tls   3      30s
```

**All must show `READY=True`** before proceeding.

---

### Step 3: Create ZeroTrustWorkloadIdentityManager CR

This is the **master switch** — the operator will not reconcile any operands without it.

```bash
export APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
export JWT_ISSUER_ENDPOINT=oidc-discovery.${APP_DOMAIN}
export CLUSTER_NAME=test01

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

Verify it was accepted:
```bash
oc get zerotrustworkloadidentitymanager cluster
```

---

### Step 4: Create All Operands with UpstreamAuthority

#### Option A: Fresh Install (recommended — no disruption, no restart)

Apply all operands in a single YAML with `upstreamAuthority` included in the SpireServer CR from the start.

**Apply the complete operands YAML (env vars already set in Step 3):**

```bash
cat <<EOF | envsubst | oc apply -f -
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
  namespace: zero-trust-workload-identity-manager
spec:
  trustDomain: ${APP_DOMAIN}
  clusterName: ${CLUSTER_NAME}
  caSubject:
    commonName: ${APP_DOMAIN}
    country: US
    organization: RH
  persistence:
    type: pvc
    size: 1Gi
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
    maxOpenConns: 100
    maxIdleConns: 2
    connMaxLifetime: 3600
  jwtIssuer: https://${JWT_ISSUER_ENDPOINT}
  upstreamAuthority:
    certManager:
      namespace: cert-manager
      issuerName: spire-ca-issuer
      issuerKind: Issuer
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireAgent
metadata:
  name: cluster
  namespace: zero-trust-workload-identity-manager
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
  namespace: zero-trust-workload-identity-manager
spec: {}
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireOIDCDiscoveryProvider
metadata:
  name: cluster
  namespace: zero-trust-workload-identity-manager
spec:
  jwtIssuer: https://${JWT_ISSUER_ENDPOINT}
EOF
```

**Why this works without restart:** The `upstreamAuthority.certManager` is part of SpireServer from second 0. When `spire-server-0` starts for the first time, it finds no cached CA in the PVC, immediately creates a CertificateRequest in cert-manager namespace, receives a signed intermediate CA, and publishes the cert-manager root in the trust bundle. Agents boot and trust the correct CA from the start.

#### Option B: Add to Existing Cluster (patch — requires waiting for CA rotation)

```bash
oc patch spireserver cluster -n zero-trust-workload-identity-manager \
  --type=merge -p '{
    "spec": {
      "upstreamAuthority": {
        "certManager": {
          "namespace": "cert-manager",
          "issuerName": "spire-ca-issuer",
          "issuerKind": "Issuer"
        }
      }
    }
  }'
```

> **Warning:** If you patch an existing cluster, SPIRE will continue using its cached self-signed CA until the next natural rotation. The `upstream_authority_id` in logs will be empty until rotation occurs. Do NOT delete the PVC to force rotation in production — agents will crash.

#### SpireServer YAML Field Explanation

```yaml
spec:
  upstreamAuthority:          # Top-level field added by PR-113
    certManager:              # Provider: cert-manager (alternative: vault)
      namespace: cert-manager
        # Where SPIRE creates CertificateRequest objects
        # Must match where the Issuer lives (for namespace-scoped Issuers)
        # RBAC (Role/RoleBinding) is auto-created here by the operator
      
      issuerName: spire-ca-issuer
        # The name of the Issuer or ClusterIssuer that will sign
        # SPIRE's intermediate CA certificate
      
      issuerKind: Issuer
        # "Issuer" = namespace-scoped (must be in certManager.namespace)
        # "ClusterIssuer" = cluster-wide (can be anywhere)
        # Default: "Issuer" if not specified
      
      # issuerGroup: cert-manager.io  (optional, default: cert-manager.io)
```

---

### Step 4: Wait for SPIRE to Use cert-manager

**For fresh install:** SPIRE will use cert-manager immediately on first startup.

**For existing cluster (Option B):** SPIRE will use cert-manager at the NEXT CA rotation. Do NOT delete the PVC!

```bash
# Watch for the operator to reconcile
oc get spireserver cluster -n zero-trust-workload-identity-manager \
  -o jsonpath='{range .status.conditions[*]}{.type}{" => "}{.status}{"\n"}{end}'
```

All conditions should be `True`.

---

### Step 5: Verify Everything

```bash
# Check SPIRE logs for cert-manager plugin
oc logs spire-server-0 -n zero-trust-workload-identity-manager -c spire-server | \
  grep -iE "cert-manager|upstream|self_signed|CA activated"

# Check trust bundle (should show CN=SPIRE Upstream CA)
oc get cm spire-bundle -n zero-trust-workload-identity-manager \
  -o jsonpath='{.data.bundle\.crt}' | openssl x509 -noout -issuer -subject

# Check from inside the pod
oc exec spire-server-0 -n zero-trust-workload-identity-manager -c spire-server -- \
  /opt/spire/bin/spire-server localauthority x509 show \
  -socketPath /tmp/spire-server/private/api.sock

# Match Subject Key IDs (definitive proof)
echo "=== cert-manager root CA ==="
oc get secret spire-ca-secret -n cert-manager -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -text | grep -A 1 "Subject Key Identifier"

echo "=== SPIRE's upstream authority ==="
oc exec spire-server-0 -n zero-trust-workload-identity-manager -c spire-server -- \
  /opt/spire/bin/spire-server localauthority x509 show \
  -socketPath /tmp/spire-server/private/api.sock | grep "Upstream"
```

**Success indicators:**
- Logs show `self_signed=false`
- Logs show `upstream_authority_id=` with a VALUE (not empty)
- Bundle shows `issuer=CN=SPIRE Upstream CA`
- Subject Key IDs match between cert-manager secret and SPIRE's upstream authority

---

## Verification Commands

### Quick Health Check

```bash
# All pods running
oc get pods -n zero-trust-workload-identity-manager

# SpireServer all conditions True
oc get spireserver cluster -n zero-trust-workload-identity-manager \
  -o jsonpath='{range .status.conditions[*]}{.type}{" => "}{.status}{"\n"}{end}'

# cert-manager Issuer ready
oc get issuer spire-ca-issuer -n cert-manager
```

### Verify cert-manager is the CA (not self-signed)

```bash
# Method 1: Check SPIRE logs
oc logs spire-server-0 -n zero-trust-workload-identity-manager -c spire-server | \
  grep "self_signed"
# Expected: self_signed=false

# Method 2: Check bundle issuer
oc get cm spire-bundle -n zero-trust-workload-identity-manager \
  -o jsonpath='{.data.bundle\.crt}' | openssl x509 -noout -issuer -subject
# Expected: issuer=CN=SPIRE Upstream CA, subject=CN=SPIRE Upstream CA

# Method 3: Inside pod — local authority
oc exec spire-server-0 -n zero-trust-workload-identity-manager -c spire-server -- \
  /opt/spire/bin/spire-server localauthority x509 show \
  -socketPath /tmp/spire-server/private/api.sock
# Expected: "Upstream authority Subject Key ID: <non-empty>"

# Method 4: Fingerprint match (strongest proof)
oc get secret spire-ca-secret -n cert-manager -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -fingerprint -subject
oc get cm spire-bundle -n zero-trust-workload-identity-manager \
  -o jsonpath='{.data.bundle\.crt}' | openssl x509 -noout -fingerprint -subject
# Expected: SAME fingerprint = same certificate = cert-manager is the root
```

### Check RBAC (auto-created by operator)

```bash
# Role in cert-manager namespace
oc get role -n cert-manager | grep spire

# RoleBinding in cert-manager namespace
oc get rolebinding -n cert-manager | grep spire

# Can SPIRE create CertificateRequests?
oc auth can-i create certificaterequests.cert-manager.io \
  --as=system:serviceaccount:zero-trust-workload-identity-manager:spire-server \
  -n cert-manager
# Expected: yes
```

### Check ConfigMap (auto-generated by operator)

```bash
oc get cm spire-server -n zero-trust-workload-identity-manager \
  -o yaml | grep -A 20 "UpstreamAuthority"
```

---

## What NOT To Do (Common Mistakes)

### Mistake 1: Using selfSigned Issuer for SPIRE

```yaml
# WRONG — causes CrashLoopBackOff
kind: ClusterIssuer
metadata:
  name: spire-issuer
spec:
  selfSigned: {}    # ← cert-manager needs SPIRE's private key → can't find it → CRASH

# Error: "Annotation cert-manager.io/private-key-secret-name missing"
```

**Fix:** Use `ca:` type issuer backed by a real secret.

---

### Mistake 2: Deleting PVC on Existing Cluster

```bash
# WRONG — causes agent crash (trust bundle mismatch)
oc delete pvc spire-data-spire-server-0 -n zero-trust-workload-identity-manager
```

**Fix:** Don't delete PVC. Let SPIRE rotate naturally, or do a fresh install.

---

### Mistake 3: Using ACME/Let's Encrypt

```yaml
# WRONG — Let's Encrypt won't issue CA certificates
kind: ClusterIssuer
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
```

**Fix:** ACME only issues leaf certs for public domains. SPIRE needs a CA cert. Use `ca:` type.

---

### Mistake 4: Checking Wrong Namespace for CertificateRequests

```bash
# WRONG
oc get certificaterequests -n zero-trust-workload-identity-manager

# CORRECT — CRs go to the namespace specified in certManager.namespace
oc get certificaterequests -n cert-manager
```

---

### Mistake 5: Setting caValidity too low

```yaml
# WRONG — caValidity must be > defaultX509Validity (1h)
spec:
  caValidity: 10m0s    # ← Less than default 1h → TTLConfigurationValid=False
```

**Fix:** Set `caValidity` to at least greater than `defaultX509Validity` (default 1h). Recommended: `24h0m0s` or higher.

---

## Troubleshooting

### SPIRE in CrashLoopBackOff

```bash
# Check crash logs
oc logs spire-server-0 -n zero-trust-workload-identity-manager -c spire-server --previous

# Common errors:
# 1. "private-key-secret-name missing" → Using selfSigned issuer, switch to CA type
# 2. "ca_validity must be greater than default_ca_ttl" → Increase caValidity
# 3. "connection refused" → cert-manager not running
```

### Agents in CrashLoopBackOff

```bash
# Check agent logs
oc logs <agent-pod-name> -n zero-trust-workload-identity-manager

# Common cause: Trust bundle changed (new CA), agents have old bundle
# Fix: Delete agent pods (DaemonSet recreates them with new bundle)
oc delete $(oc get pods -n zero-trust-workload-identity-manager -o name | grep spire-agent) \
  -n zero-trust-workload-identity-manager
```

### upstream_authority_id is Empty in Logs

This means SPIRE loaded an old cached CA from its PVC journal. It will use cert-manager at the next natural rotation.

**To force immediate use (testing only — NOT for production):**
```bash
oc scale statefulset spire-server -n zero-trust-workload-identity-manager --replicas=0
sleep 15
oc delete pvc spire-data-spire-server-0 -n zero-trust-workload-identity-manager --wait=true
oc scale statefulset spire-server -n zero-trust-workload-identity-manager --replicas=1
# Then restart agents after server is Running
```

### CertificateRequest Not Appearing

CertificateRequests are short-lived (seconds). To catch them:
```bash
# Watch in real-time DURING spire-server startup
oc get certificaterequests -n cert-manager -w
```

---

## Evidence from Testing

### Cluster 1 — Successful Output

**SPIRE Logs:**
```
level=info msg="Configured plugin" plugin_name=cert-manager plugin_type=UpstreamAuthority
level=info msg="Plugin loaded" plugin_name=cert-manager plugin_type=UpstreamAuthority
level=info msg="Journal loaded" jwt_keys=0 x509_cas=0
level=info msg="Waiting for certificaterequest to be signed" name=spiffe-ca-8zlp5 namespace=cert-manager
level=info msg="X509 CA prepared" self_signed=false upstream_authority_id=46ed8a46e1513fb629d69532a26df2e422ce4e81
level=info msg="X509 CA activated" upstream_authority_id=46ed8a46e1513fb629d69532a26df2e422ce4e81
```

**Trust Bundle:**
```
issuer=CN=SPIRE Upstream CA
subject=CN=SPIRE Upstream CA
```

**Local Authority (inside pod):**
```
Active X.509 authority:
  Authority ID: 4607fea44156c4babfdb14ac0b97e97ad83cfa88
  Upstream authority Subject Key ID: 46ed8a46e1513fb629d69532a26df2e422ce4e81
```

### Cluster 2 — Successful Output

**SPIRE Logs:**
```
level=info msg="Journal loaded" jwt_keys=0 x509_cas=0
level=info msg="X509 CA prepared" self_signed=false upstream_authority_id=f88003831a893f8b8160fbc0c6fd2cd042f41ea3
level=info msg="X509 CA activated" upstream_authority_id=f88003831a893f8b8160fbc0c6fd2cd042f41ea3
```

**Subject Key ID Match:**
```
cert-manager root (spire-ca-secret): F8:80:03:83:1A:89:3F:8B:81:60:FB:C0:C6:FD:2C:D0:42:F4:1E:A3
SPIRE upstream authority:            f88003831a893f8b8160fbc0c6fd2cd042f41ea3
→ MATCH ✅ (same bytes, different formatting)
```

### Cluster 3 — Fresh Install with upstreamAuthority from Day 1 (2026-05-21)

**Install order used:**
1. cert-manager operator + CA chain (selfsigned-issuer → spire-ca → spire-ca-issuer)
2. ZTWIM operator v1.0.1 (via CatalogSource)
3. `ZeroTrustWorkloadIdentityManager` CR (required — operator won't reconcile without it)
4. All operands in one YAML (SpireServer with `upstreamAuthority` + SpireAgent + CSI + OIDC)

**SPIRE Logs (no restart needed):**
```
level=info msg="Configured plugin" plugin_name=cert-manager plugin_type=UpstreamAuthority
level=info msg="Plugin loaded" plugin_name=cert-manager plugin_type=UpstreamAuthority
level=info msg="Waiting for certificaterequest to be signed" name=spiffe-ca-qjs2x namespace=cert-manager
level=info msg="X509 CA prepared" self_signed=false upstream_authority_id=9fe5993e62ad68c70481183f445c6e61d4e2b294
level=info msg="X509 CA activated" upstream_authority_id=9fe5993e62ad68c70481183f445c6e61d4e2b294
level=warning msg="UpstreamAuthority plugin does not support JWT-SVIDs."
```

**Subject Key ID Match:**
```
cert-manager root (spire-ca-secret): 9F:E5:99:3E:62:AD:68:C7:04:81:18:3F:44:5C:6E:61:D4:E2:B2:94
SPIRE upstream authority:            9fe5993e62ad68c70481183f445c6e61d4e2b294
→ MATCH ✅ (same bytes, different formatting)
```

**Pods (all Running, no restarts):**
```
spire-server-0                                                    2/2     Running   0          24s
spire-agent-2xt97                                                 0/1     Running   0          25s
spire-agent-8bkpd                                                 0/1     Running   0          25s
spire-agent-p9d5p                                                 0/1     Running   0          25s
spire-spiffe-csi-driver-2dqpp                                     2/2     Running   0          26s
spire-spiffe-csi-driver-xxcwm                                     2/2     Running   0          26s
spire-spiffe-csi-driver-zbp62                                     2/2     Running   0          26s
spire-spiffe-oidc-discovery-provider-6d99b4bf74-lg82r             0/1     Running   0          26s
zero-trust-workload-identity-manager-controller-manager-969pv7z   1/1     Running   0          65s
```

**Key lesson:** The `ZeroTrustWorkloadIdentityManager` CR must be created BEFORE operands — without it the operator reports `Ready => False : Failed to retrieve ZeroTrustWorkloadIdentityManager from cluster` and won't create StatefulSets/DaemonSets.

---

## Glossary

| Term | Plain English |
|---|---|
| **CA** | Certificate Authority — trusted stamp maker that signs certificates |
| **CSR** | Certificate Signing Request — application form asking CA to sign a cert |
| **SVID** | SPIFFE Verifiable Identity Document — a pod's identity certificate |
| **SPIFFE** | Standard for workload identity URIs (`spiffe://domain/path`) |
| **SPIRE** | Software that implements SPIFFE — issues SVIDs to pods |
| **ZTWIM** | Zero Trust Workload Identity Manager — the OpenShift operator |
| **UpstreamAuthority** | External CA (cert-manager/Vault) that signs SPIRE's own CA cert |
| **Trust Domain** | The SPIFFE namespace for a cluster (e.g., `apps.example.com`) |
| **Trust Bundle** | Collection of CA certs used to verify SVIDs (`spire-bundle` ConfigMap) |
| **bundle.crt** | SPIRE server's root cert — agents need it to trust the server |
| **CertificateRequest** | Short-lived cert-manager object — SPIRE asks cert-manager to sign |
| **ClusterIssuer** | Cluster-wide cert-manager signer (any namespace) |
| **Issuer** | Namespace-scoped cert-manager signer (one namespace only) |
| **selfSigned** | Issuer type that signs with the requester's own key (NOT for SPIRE) |
| **CA Issuer** | Issuer backed by a real root CA key (correct for SPIRE) |
| **mTLS** | Mutual TLS — both sides verify each other's certificate |
| **distroless** | Container image with no shell — use binary path directly |
| **PVC** | PersistentVolumeClaim — SPIRE stores its CA journal here |
| **RBAC** | Role-Based Access Control — permissions in Kubernetes |
| **OLM** | Operator Lifecycle Manager — installs/upgrades operators in OpenShift |
| **Operand** | The thing an operator manages (SpireServer, SpireAgent, etc.) |
| **StatefulSet** | Kubernetes resource for pods needing stable identity + storage |
| **DaemonSet** | Kubernetes resource that runs one pod per node |
| **Subject Key ID** | Fingerprint of a certificate's public key (used to match CA chains) |

---

## Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│ UPSTREAM AUTHORITY SETUP — QUICK REFERENCE                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ INSTALL ORDER:                                                   │
│ 1. Install cert-manager operator (wait for pods Ready)           │
│ 2. Create CA chain (selfsigned-issuer → Certificate → CA Issuer) │
│ 3. Install ZTWIM operator (wait for controller-manager Ready)    │
│ 4. Create ZeroTrustWorkloadIdentityManager CR (master switch)    │
│ 5. Apply ALL operands YAML with upstreamAuthority in SpireServer │
│    (SpireServer + SpireAgent + SpiffeCSIDriver + OIDC Provider)  │
│                                                                  │
│ VERIFICATION:                                                    │
│ 6. Logs show: self_signed=false                                  │
│ 7. Logs show: upstream_authority_id=<non-empty>                  │
│ 8. Bundle shows: CN=SPIRE Upstream CA                            │
│ 9. Subject Key IDs match between secret and SPIRE               │
│                                                                  │
│ KEY RULES:                                                       │
│ • NEVER use selfSigned issuer for SPIRE → use CA type            │
│ • NEVER delete PVC in production → let SPIRE rotate naturally    │
│ • ALWAYS include upstreamAuthority in SpireServer from the start │
│ • ALWAYS check cert-manager namespace for CertificateRequests    │
│ • ALWAYS verify issuer is Ready before creating operands         │
│ • If patching later: upstream_authority_id will be empty until    │
│   next CA rotation (SPIRE uses cached CA from PVC)               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```
