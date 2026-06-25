# ZTWIM Stage Build — Deploy & Registry Reference

**Owner:** Sayak Das
**Last Updated:** 2026-06-25
**Purpose:** Single reference for deploying ZTWIM stage builds on any OpenShift cluster.

---

## Table of Contents

1. [Quick Deploy (cucushift cluster)](#1-quick-deploy-cucushift-cluster)
2. [Daily Commands](#2-daily-commands)
3. [Deploy on Bare OCP Cluster (Full E2E)](#3-deploy-on-bare-ocp-cluster-full-e2e)
4. [Deploy Paths — How to Choose](#4-deploy-paths--how-to-choose)
5. [Registry Credentials](#5-registry-credentials)
6. [IIB Reference Table](#6-iib-reference-table)
7. [Stage Build Manifest (v1.1.0)](#7-stage-build-manifest-v110)
8. [Mirror Images to Quay.io](#8-mirror-images-to-quayio)
9. [OLM Upgrade Testing](#9-olm-upgrade-testing)
10. [Clean Uninstall](#10-clean-uninstall)
11. [OpenShift Cluster from Scratch (AWS)](#11-openshift-cluster-from-scratch-aws)
12. [Troubleshooting & Lessons Learned](#12-troubleshooting--lessons-learned)

---

## 1. Quick Deploy (cucushift cluster)

For `workflow-launch cucushift-*` clusters that already have brew credentials:

```bash
# CatalogSource
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: rh-brew-stage-operators
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: brew.registry.redhat.io/rh-osbs/iib:1091450
  displayName: Red Hat Brew Stage Catalog
  publisher: grpc
  updateStrategy:
    registryPoll:
      interval: 5m
EOF

# Operator
oc apply -f - <<EOF
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
  name: ztwim-og
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

# Wait for Succeeded
oc get csv -n zero-trust-workload-identity-manager -w
```

---

## 2. Daily Commands

### Status at a Glance

```bash
echo "=== CSV ===" && oc get csv -n zero-trust-workload-identity-manager && \
echo "=== Pods ===" && oc get pods -n zero-trust-workload-identity-manager && \
echo "=== CRs ===" && oc get spireserver,spireagent,spiffecsidriver,spireoidcdiscoveryprovider
```

### SPIRE Server

```bash
# Health check
oc exec spire-server-0 -n zero-trust-workload-identity-manager -c spire-server -- \
  /spire-server healthcheck -socketPath /tmp/spire-server/private/api.sock

# List registered workloads
oc exec spire-server-0 -n zero-trust-workload-identity-manager -c spire-server -- \
  /spire-server entry show -socketPath /tmp/spire-server/private/api.sock

# List attested agents
oc exec spire-server-0 -n zero-trust-workload-identity-manager -c spire-server -- \
  /spire-server agent list -socketPath /tmp/spire-server/private/api.sock

# Logs
oc logs spire-server-0 -n zero-trust-workload-identity-manager -c spire-server --tail=30
```

### Trust Bundle

```bash
oc get cm spire-bundle -n zero-trust-workload-identity-manager -o jsonpath='{.data.bundle\.crt}' | \
  openssl x509 -noout -issuer -subject -dates
```

### Restart Pods

```bash
# Restart SPIRE server
oc delete pod spire-server-0 -n zero-trust-workload-identity-manager

# Restart all SPIRE agents
oc delete pods -n zero-trust-workload-identity-manager -l app=spire-agent
```

### Quick Workload Test

```bash
oc create namespace spire-test --dry-run=client -o yaml | oc apply -f -

oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: test-workload
spec:
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  workloadSelectorTemplates:
  - "k8s:ns:spire-test"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-sleep
  namespace: spire-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-sleep
  template:
    metadata:
      labels:
        app: test-sleep
    spec:
      containers:
      - name: sleep
        image: registry.access.redhat.com/ubi9/ubi-minimal
        command: ["sleep", "3600"]
        volumeMounts:
        - name: spiffe-workload-api
          mountPath: /spiffe-workload-api
          readOnly: true
      volumes:
      - name: spiffe-workload-api
        csi:
          driver: csi.spiffe.io
          readOnly: true
EOF

# Verify
oc get pods -n spire-test -w
oc exec -n spire-test deploy/test-sleep -- ls /spiffe-workload-api/

# Cleanup
oc delete namespace spire-test
oc delete clusterspiffeid test-workload
```

---

## 3. Deploy on Bare OCP Cluster (Full E2E)

Follow top-to-bottom on any fresh OCP cluster (ClusterBot, openshift-install, ROSA).

### Step 1: Mirror IIB to Quay (one-time, from laptop)

```bash
podman login brew.registry.redhat.io
podman login quay.io -u rh-ee-sayadas

podman pull brew.registry.redhat.io/rh-osbs/iib:1164247
podman tag brew.registry.redhat.io/rh-osbs/iib:1164247 quay.io/rh-ee-sayadas/ocp4.22/ztwim:1.1.0
podman push quay.io/rh-ee-sayadas/ocp4.22/ztwim:1.1.0
```

Make the quay.io repo **PUBLIC**: Settings → Make Public.

### Step 2: Add registry credentials to cluster pull secret

```bash
stage_auth=$(jq -c '.auths["registry.stage.redhat.io"]' ${XDG_RUNTIME_DIR}/containers/auth.json)

oc get secret pull-secret -n openshift-config \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d > pull_secret.json

jq --argjson auth "$stage_auth" \
  '.auths["registry.stage.redhat.io"] = $auth' \
  pull_secret.json > pull_secret_updated.json

jq '.auths["brew.registry.redhat.io"] = .auths["registry.redhat.io"]' \
  pull_secret_updated.json > pull_secret_final.json

cat pull_secret_final.json | jq '.auths | keys'

oc set data secret/pull-secret -n openshift-config \
  --from-file=.dockerconfigjson=pull_secret_final.json

oc get mcp -w
# Wait: UPDATED=True UPDATING=False on both master and worker
```

### Step 3: Apply IDMS (stage → brew fallback)

```bash
oc apply -f - <<EOF
apiVersion: config.openshift.io/v1
kind: ImageDigestMirrorSet
metadata:
  name: ztwim-stage-to-brew-mirror
spec:
  imageDigestMirrors:
  - mirrors:
    - brew.registry.redhat.io
    source: registry.stage.redhat.io
    mirrorSourcePolicy: AllowContactingSource
EOF

oc get mcp -w   # Wait for UPDATED=True again
```

### Step 4: CatalogSource

```bash
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: stage-catalog-422-1.1.0
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/rh-ee-sayadas/ztwim:1.1.0
  displayName: ZTWIM 1.1.0 Stage Index
  publisher: RH Stage
  priority: 1000
  updateStrategy:
    registryPoll:
      interval: 5m
EOF

# WAIT until READY
sleep 60
oc get pods -n openshift-marketplace | grep stage
oc get catalogsource stage-catalog-422-1.1.0 -n openshift-marketplace \
  -o jsonpath='{.status.connectionState.lastObservedState}'
```

### Step 5: Install Operator

```bash
oc create namespace zero-trust-workload-identity-manager

oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: ztwim-og
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
  source: stage-catalog-422-1.1.0
  sourceNamespace: openshift-marketplace
  name: openshift-zero-trust-workload-identity-manager
  channel: stable-v1
  installPlanApproval: Manual
EOF
```

### Step 6: Approve InstallPlan

```bash
sleep 120
oc get installplan -n zero-trust-workload-identity-manager
oc patch installplan <PLAN_NAME> -n zero-trust-workload-identity-manager \
  --type merge -p '{"spec":{"approved":true}}'
oc get csv -n zero-trust-workload-identity-manager -w
```

### Step 7: Deploy Operands

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
  trustDomain: ${APP_DOMAIN}
  clusterName: ${CLUSTER_NAME}
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

### Step 8: Verify

```bash
oc get pods -n zero-trust-workload-identity-manager
oc exec spire-server-0 -n zero-trust-workload-identity-manager -c spire-server -- \
  /spire-server healthcheck -socketPath /tmp/spire-server/private/api.sock
```

---

## 4. Deploy Paths — How to Choose

| Cluster Source | Command | Brew Creds? | CatalogSource Image |
|---|---|:-:|---|
| **cucushift** | `workflow-launch cucushift-installer-rehearse-gcp-ipi 4.21` | Yes | `brew.registry.redhat.io/rh-osbs/iib:<tag>` |
| **plain launch** | `launch 4.21 gcp` | No | `quay.io/rh-ee-sayadas/ztwim:1.1.0` or add brew creds manually |

**Decision:** cucushift = use brew IIB directly. Plain launch = use quay.io or add creds.

---

## 5. Registry Credentials

| Registry | Auth Source | Login |
|---|---|---|
| `registry.stage.redhat.io` | [Terms-Based Token](https://access.stage.redhat.com/terms-based-registry/token/sayadas_stage_build_cred) | `podman login registry.stage.redhat.io` |
| `brew.registry.redhat.io` | Same as `registry.redhat.io` | `podman login brew.registry.redhat.io` |
| `registry-proxy.engineering.redhat.com` | Kerberos / Red Hat SSO | `podman login registry-proxy.engineering.redhat.com` |
| `quay.io` | User: `rh-ee-sayadas` | `podman login quay.io -u rh-ee-sayadas` |

---

## 6. IIB Reference Table

### ZTWIM 1.1.0

| OCP | IIB Tag | brew (needs pull secret) | quay.io (mirrored) |
|:-:|:-:|---|---|
| 4.18 | 1157868 | `brew.registry.redhat.io/rh-osbs/iib:1157868` | — |
| 4.19 | 1157869 | `brew.registry.redhat.io/rh-osbs/iib:1157869` | — |
| 4.20 | 1157867 | `brew.registry.redhat.io/rh-osbs/iib:1157867` | — |
| 4.21 | 1157870 | `brew.registry.redhat.io/rh-osbs/iib:1157870` | `quay.io/rh-ee-sayadas/iib:1157870` |

> `registry-proxy` is only reachable from your laptop (VPN). Cluster nodes cannot resolve it. Use quay.io for CatalogSource — always works without auth.

---

## 7. Stage Build Manifest (v1.1.0)

| Component | Image |
|---|---|
| Operator | `registry.stage.redhat.io/zero-trust-workload-identity-manager/zero-trust-workload-identity-manager-rhel9@sha256:305bb1...` |
| SPIRE Server | `registry.stage.redhat.io/.../spiffe-spire-server-rhel9@sha256:7d71c6...` |
| SPIRE Agent | `registry.stage.redhat.io/.../spiffe-spire-agent-rhel9@sha256:356f75...` |
| SPIFFE CSI Driver | `registry.stage.redhat.io/.../spiffe-csi-driver-rhel9@sha256:d69ea9...` |
| OIDC Discovery Provider | `registry.stage.redhat.io/.../spiffe-spire-oidc-discovery-provider-rhel9@sha256:3655f4...` |
| Controller Manager | `registry.stage.redhat.io/.../spiffe-spire-controller-manager-rhel9@sha256:b74141...` |
| spiffe-helper | `registry.stage.redhat.io/.../spiffe-helper-rhel9@sha256:977148...` |
| CSI Node Driver Registrar | `registry.redhat.io/openshift4/ose-csi-node-driver-registrar-rhel9@sha256:cdef55...` |
| CSI Init Container | `registry.access.redhat.com/ubi9@sha256:a7241c...` |

Verify running images match:

```bash
oc get pods -n zero-trust-workload-identity-manager \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.containers[*]}  {.name}: {.image}{"\n"}{end}{end}'
```

---

## 8. Mirror Images to Quay.io

```bash
podman login brew.registry.redhat.io
podman login quay.io -u rh-ee-sayadas

podman pull brew.registry.redhat.io/rh-osbs/iib:1157870
podman tag brew.registry.redhat.io/rh-osbs/iib:1157870 quay.io/rh-ee-sayadas/iib:1157870
podman push quay.io/rh-ee-sayadas/iib:1157870
```

Then use `quay.io/rh-ee-sayadas/iib:1157870` in your CatalogSource.

---

## 9. OLM Upgrade Testing

### Verify upgrade graph

```bash
CATALOG_POD=$(oc get pods -n openshift-marketplace \
  -l olm.catalogSource=stage-catalog-421-1.1.0 -o jsonpath='{.items[0].metadata.name}')

oc exec -n openshift-marketplace $CATALOG_POD -- \
  cat /configs/openshift-zero-trust-workload-identity-manager/channel.yaml
```

Expected: `v1.0.0 → v1.0.1 → v1.1.0`

### Approve upgrades sequentially

```bash
oc get installplan -n zero-trust-workload-identity-manager

# Approve each step (replace <PLAN_NAME>)
oc patch installplan <PLAN_NAME> -n zero-trust-workload-identity-manager \
  --type merge -p '{"spec":{"approved":true}}'
```

### If InstallPlan doesn't appear

```bash
oc delete pod -n openshift-operator-lifecycle-manager -l app=catalog-operator
sleep 60
oc get installplan -n zero-trust-workload-identity-manager
```

### Verify final state

```bash
oc get csv -n zero-trust-workload-identity-manager
# VERSION: 1.1.0, Phase: Succeeded
```

---

## 10. Clean Uninstall

```bash
# Operands
oc delete spireoidcdiscoveryprovider cluster 2>/dev/null
oc delete spiffecsidriver cluster 2>/dev/null
oc delete spireagent cluster 2>/dev/null
oc delete spireserver cluster 2>/dev/null
oc delete zerotrustworkloadidentitymanager cluster 2>/dev/null

# Operator
oc delete subscription --all -n zero-trust-workload-identity-manager
oc delete csv --all -n zero-trust-workload-identity-manager
oc delete installplan --all -n zero-trust-workload-identity-manager

# Namespace
oc delete namespace zero-trust-workload-identity-manager

# Catalog
oc delete catalogsource stage-catalog-421-1.1.0 -n openshift-marketplace 2>/dev/null
oc delete catalogsource rh-brew-stage-operators -n openshift-marketplace 2>/dev/null
```

---

## 11. OpenShift Cluster from Scratch (AWS)

### Download installer

| Source | URL | Use for |
|---|---|---|
| CI releases | https://amd64.ocp.releases.ci.openshift.org | Nightlies |
| Customer Portal | https://access.redhat.com/downloads/content/290 | GA releases |

### Create cluster

```bash
cd ~/Catlog-image-Installer
tar -xvf ~/Downloads/openshift-install-linux-4.19.11.tar.gz
./openshift-install create cluster --dir aws
```

| Prompt | Value |
|---|---|
| Platform | `aws` |
| Region | `us-east-1` |
| Base Domain | `qe.devcluster.openshift.com` |
| Cluster Name | `sayadas-aws-osc` (must start with Kerberos ID) |
| Pull Secret | Paste from `keys/pull-secret.json` |

### Access

```bash
export KUBECONFIG=~/Catlog-image-Installer/aws/auth/kubeconfig
oc whoami
```

### Destroy (when done)

```bash
./openshift-install destroy cluster --dir aws
```

---

## 12. Troubleshooting & Lessons Learned

### Golden Rule

> **CatalogSource FIRST → wait for READY → Subscription SECOND.** Never the other way around.

### Pre-Flight Checklist

1. **Stage registry creds are fresh** — tokens expire. Re-login if you see `BundleUnpackFailed`
2. **IDMS is applied** — redirects stage pulls to brew as fallback
3. **No orphaned CSVs** — if reinstalling, clean everything first

### Common Problems

| Problem | Cause | Fix |
|---|---|---|
| CatalogSource `ImagePullBackOff` | brew creds missing or `registry-proxy` used | Use quay.io mirrored IIB |
| `BundleUnpackFailed: DeadlineExceeded` | Expired stage registry token | Re-login, update pull secret, wait MCP |
| `ResolutionFailed` | Orphaned CSVs from previous install | Delete all CSVs, Subs, InstallPlans |
| `AtLatestKnown` but new version exists | OLM catalog-operator cache is stale | `oc delete pod -l app=catalog-operator -n openshift-operator-lifecycle-manager` |
| OLM picks wrong catalog | Multiple catalogs have same package | Use `priority: 1000` on CatalogSource |
| Subscription created, no InstallPlan | Sub created before catalog was READY | Delete sub + catalog, recreate catalog FIRST |
| Upgrade stuck at intermediate version | Manual approval, InstallPlan pending | Approve the pending InstallPlan |

### Key Namespaces

| Namespace | What lives there |
|---|---|
| `openshift-marketplace` | CatalogSource pod |
| `zero-trust-workload-identity-manager` | Operator + all SPIRE pods |
| `cert-manager` | cert-manager pods (if using upstream CA) |

### Key Paths Inside SPIRE Server Pod

| Path | What it is |
|---|---|
| `/spire-server` | SPIRE server binary |
| `/tmp/spire-server/private/api.sock` | API socket (use with `-socketPath`) |
| `/run/spire/data/datastore.sqlite3` | SQLite datastore |
