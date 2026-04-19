# SPIFFE Federation with ACME (Let's Encrypt) + Amazon RDS PostgreSQL

## Hybrid Federation: https_spiffe + https_web (cert-manager / Let's Encrypt)

**Document Version**: 3.0
**Last Updated**: April 19, 2026
**OpenShift Version**: 4.18+ (Tested on 4.19)
**Datastore**: Amazon RDS PostgreSQL (production-grade, replaces default SQLite)
**Tested On**: AWS Multi-Cluster Federation

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Phase 0: Discover Cluster Information](#phase-0-discover-cluster-information)
5. [Phase 1: Create Amazon RDS PostgreSQL Instances (AWS Console)](#phase-1-create-amazon-rds-postgresql-instances-aws-console)
6. [Phase 2: Configure Network Access (Security Groups)](#phase-2-configure-network-access-security-groups)
7. [Phase 3: Cluster 1 Setup](#phase-3-cluster-1-setup)
8. [Phase 4: Cluster 2 Setup](#phase-4-cluster-2-setup)
9. [Phase 5: Federation Trust Setup](#phase-5-federation-trust-setup)
10. [Phase 6: Cross-Cluster mTLS Testing](#phase-6-cross-cluster-mtls-testing)
11. [Phase 7: Verify Data in RDS (Optional)](#phase-7-verify-data-in-rds-optional)
12. [Execution Order Summary](#execution-order-summary)
13. [Troubleshooting](#troubleshooting)
14. [Cleanup](#cleanup)
15. [Glossary](#glossary)

---

## Overview

This guide provides a complete, tested walkthrough for setting up **SPIFFE federation** between two OpenShift clusters using **Amazon RDS PostgreSQL** as the SPIRE datastore. The setup uses a **hybrid approach**:

- **Cluster 1**: `https_spiffe` profile (self-signed SPIRE certificate)
- **Cluster 2**: `https_web` profile with **cert-manager** and **Let's Encrypt** for publicly trusted certificates
- **Datastore**: One Amazon RDS PostgreSQL instance per cluster (production configuration)

### Why RDS Instead of SQLite?

| Aspect | SQLite (Default) | RDS PostgreSQL (This Guide) |
|--------|-----------------|----------------------------|
| Production readiness | Not recommended | Production-grade |
| High availability | No (single file on PVC) | Multi-AZ available |
| Backups | Manual PVC backup | Automated snapshots |
| Scalability | Limited by PVC I/O | Scales with instance class |
| Disaster recovery | Manual | Point-in-time restore |

### Why PostgreSQL?

SPIRE Server supports only `sqlite3`, `postgres`, and `mysql`. Among AWS RDS engines (Aurora, PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, Db2), **only PostgreSQL and MySQL are supported by SPIRE**. PostgreSQL is recommended.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         FEDERATION ARCHITECTURE                                   │
│                                                                                   │
│  ┌─────────────────────────────────┐    ┌─────────────────────────────────┐       │
│  │ CLUSTER 1 (VPC-A)               │    │ CLUSTER 2 (VPC-B)               │       │
│  │                                  │    │                                  │       │
│  │  Profile: https_spiffe           │    │  Profile: https_web              │       │
│  │  Cert: Self-signed SPIRE         │    │  Cert: Let's Encrypt             │       │
│  │                                  │    │  (via cert-manager)              │       │
│  │  ┌────────────┐                  │    │  ┌────────────┐                  │       │
│  │  │ SPIRE      │◄────── Federation ────►  │ SPIRE      │                  │       │
│  │  │ Server     │                  │    │  │ Server     │                  │       │
│  │  └─────┬──────┘                  │    │  └─────┬──────┘                  │       │
│  │        │                         │    │        │                         │       │
│  │        ▼                         │    │        ▼                         │       │
│  │  ┌────────────┐                  │    │  ┌────────────┐                  │       │
│  │  │ RDS        │                  │    │  │ RDS        │                  │       │
│  │  │ PostgreSQL │ (in VPC-A)       │    │  │ PostgreSQL │ (in VPC-B)       │       │
│  │  └────────────┘                  │    │  └────────────┘                  │       │
│  └─────────────────────────────────┘    └─────────────────────────────────┘       │
│                                                                                   │
│  ◄────────────────────── BIDIRECTIONAL TRUST ──────────────────────►              │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Critical Networking Requirement

> **Each RDS instance MUST be in the same VPC as its corresponding OpenShift cluster.**
>
> OpenShift clusters on AWS each create their own VPC during installation. RDS instances in a different VPC cannot be reached by cluster pods — connections will time out silently. There is no cross-VPC routing unless VPC Peering or Transit Gateway is explicitly configured.

---

## Prerequisites

- [ ] **Two OpenShift 4.18+ clusters** on AWS with admin access
- [ ] **oc CLI** installed and configured
- [ ] **AWS Console access** with permissions to create RDS instances, security groups, and subnet groups
- [ ] **jq** installed on your local machine
- [ ] **Kubeconfig files** for both clusters
- [ ] **Internet access** on Cluster 2 (required for Let's Encrypt ACME certificate validation)

---

## Phase 0: Discover Cluster Information

Throughout this guide, placeholders like `<CLUSTER1_APP_DOMAIN>` appear in commands. Before you begin, gather these values for your environment using the methods below. The guide also uses shell variables (`${CLUSTER1_APP_DOMAIN}`) — these are set in Phase 3 Step 1 and Phase 4 Step 1.

### Placeholders Used in This Guide

| Placeholder | Description | Where You Get It |
|-------------|-------------|-----------------|
| `<KUBECONFIG1>` | Kubeconfig path for Cluster 1 | OpenShift install output: `export KUBECONFIG=...` |
| `<KUBECONFIG2>` | Kubeconfig path for Cluster 2 | OpenShift install output: `export KUBECONFIG=...` |
| `<CLUSTER1_APP_DOMAIN>` | Apps domain for Cluster 1 (e.g., `apps.mycluster1.example.com`) | Step 0.1 below |
| `<CLUSTER2_APP_DOMAIN>` | Apps domain for Cluster 2 (e.g., `apps.mycluster2.example.com`) | Step 0.1 below |
| `<CLUSTER1_VPC_ID>` | VPC ID where Cluster 1 runs (e.g., `vpc-0abc123...`) | Step 0.2 below |
| `<CLUSTER2_VPC_ID>` | VPC ID where Cluster 2 runs (e.g., `vpc-0def456...`) | Step 0.2 below |
| `<VPC_CIDR>` | CIDR block of the VPC (e.g., `10.0.0.0/16`) | Step 0.3 below |
| `<RDS_ENDPOINT_C1>` | RDS endpoint for Cluster 1 (e.g., `spire-ds1.xxx.region.rds.amazonaws.com`) | After Phase 1 — from RDS Console → Connectivity & security |
| `<RDS_ENDPOINT_C2>` | RDS endpoint for Cluster 2 | After Phase 1 — from RDS Console → Connectivity & security |
| `<RDS_MASTER_PASSWORD>` | Master password you set when creating RDS | You choose this during RDS creation |
| `<SPIRE_USER_PASSWORD>` | Password for the `spire_server` database user | You choose this in Phase 3 Step 3 |

### Step 0.1: Find Apps Domains

The **apps domain** is the base domain for all OpenShift routes. You need it for trust domains, federation endpoints, and route hostnames.

**From the install output:**

```
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.mycluster.example.com
                                                                           └──────────────────────────┘
                                                                           This is the apps domain
```

**Or using the oc CLI:**

```bash
export KUBECONFIG=<KUBECONFIG1>
oc get ingresses.config cluster -o jsonpath='{.spec.domain}'
```

**Expected output:**

```
apps.mycluster1.example.com
```

Repeat for Cluster 2 with `<KUBECONFIG2>`.

### Step 0.2: Find VPC IDs

Each OpenShift cluster on AWS creates its own VPC. You need the VPC IDs to create RDS in the correct network.

**Option A — AWS Console:**
1. Go to **EC2 → Instances** → search for your cluster name
2. Click any worker instance → **Details** tab → note the **VPC ID**

**Option B — AWS Console (Security Groups):**
1. Go to **EC2 → Security Groups** → search for your cluster name
2. The **VPC ID** column shows the VPC

### Step 0.3: Find VPC CIDR

```bash
export KUBECONFIG=<KUBECONFIG1>
oc get nodes -o wide
```

**Expected output:**

```
NAME                          STATUS   ROLES                  AGE   VERSION    INTERNAL-IP   ...
ip-10-0-21-169.ec2.internal   Ready    control-plane,master   79m   v1.32.x    10.0.21.169   ...
ip-10-0-23-130.ec2.internal   Ready    worker                 64m   v1.32.x    10.0.23.130   ...
ip-10-0-34-242.ec2.internal   Ready    worker                 72m   v1.32.x    10.0.34.242   ...
```

All nodes in `10.0.x.x` → VPC CIDR is `10.0.0.0/16`.

---

## Phase 1: Create Amazon RDS PostgreSQL Instances (AWS Console)

### Step 1.1: Create RDS Instance for Cluster 1

Navigate to **AWS Console → RDS → Create database** and select every field as shown:

| Section | Field | Value |
|---------|-------|-------|
| **Method** | Creation method | **Standard create** (NOT Easy create) |
| **Engine** | Engine type | **PostgreSQL** |
| | Engine version | **PostgreSQL 15.x** (e.g., 15.13-R2) |
| | Enable RDS Extended Support | **Uncheck** |
| **Template** | Template | **Free tier** |
| **Settings** | DB instance identifier | **`spire-ds1`** (or any unique name) |
| | Master username | **`postgres`** |
| | Credentials management | **Self managed** |
| | Master password | **Your strong password** (note it down!) |
| **Auth** | Authentication | **Password authentication** |
| **Instance** | DB instance class | **Burstable classes → db.t4g.micro** |
| **Storage** | Storage type | **General Purpose SSD (gp2)** |
| | Allocated storage | **20 GiB** |
| **Connectivity** | Compute resource | **Don't connect to an EC2 compute resource** |
| | **VPC** | **Cluster 1's VPC** (match the VPC ID from Phase 0) |
| | DB subnet group | **Create new DB Subnet Group** (if none exists for this VPC) |
| | Public access | **No** |
| | VPC security group | **Create new → name it `spire-rds-sg`** |
| | Availability Zone | **No preference** |
| **Proxy** | Create an RDS Proxy | **Do NOT check** |
| **CA** | Certificate authority | **rds-ca-rsa2048-g1 (default)** |
| **Monitoring** | Database Insights | **Database Insights - Standard** |
| | Performance Insights | **Uncheck** |
| | Enhanced Monitoring | **Uncheck** |
| | Log exports | **Uncheck ALL** |
| | DevOps Guru | **Do NOT turn on** |
| **Additional Config** | **Initial database name** | **`spire`** (EXPAND this section! DO NOT leave blank!) |
| (expand it!) | Enable automated backups | **Uncheck** |
| | Deletion protection | **Uncheck** |

Click **Create database**. Wait 5-10 minutes for status to show **"Available"**.

> **If the VPC doesn't appear in the dropdown**: Create a DB subnet group first at **RDS → Subnet groups → Create DB subnet group**. Select the VPC, choose at least 2 subnets from different AZs, then retry.

### Step 1.2: Create RDS Instance for Cluster 2

Repeat with **identical settings** except:

| Field | Cluster 1 | Cluster 2 |
|-------|-----------|-----------|
| DB instance identifier | `spire-ds1` | **`spire-ds2`** |
| VPC | Cluster 1's VPC | **Cluster 2's VPC** |
| VPC security group | Create new → `spire-rds-sg` | **Create new → `spire-rds-sg-c2`** |

### Step 1.3: Note the RDS Endpoints

After both show "Available", click each instance → **Connectivity & security** tab → copy the **Endpoint**:

| Instance | Endpoint | Port |
|----------|----------|------|
| spire-ds1 | `spire-ds1.<identifier>.<region>.rds.amazonaws.com` | 5432 |
| spire-ds2 | `spire-ds2.<identifier>.<region>.rds.amazonaws.com` | 5432 |

Record these in your environment table from Phase 0.

---

## Phase 2: Configure Network Access (Security Groups)

Each RDS security group needs an inbound rule allowing OpenShift pods (port 5432).

### Step 2.1: Update spire-rds-sg (Cluster 1)

1. Go to **EC2 → Security Groups** → find **`spire-rds-sg`**
2. Click **Inbound rules** → **Edit inbound rules** → **Add rule**:

| Type | Protocol | Port range | Source | Description |
|------|----------|------------|--------|-------------|
| **PostgreSQL** | TCP | 5432 | **Your VPC CIDR** (e.g., `10.0.0.0/16`) | OpenShift workers |

3. Click **Save rules**

### Step 2.2: Update spire-rds-sg-c2 (Cluster 2)

Repeat the same for **`spire-rds-sg-c2`** with the same CIDR.

> **Why is this needed?** Security groups act as firewalls for RDS. Without this rule, OpenShift pods cannot reach the database and connections silently time out.

---

## Phase 3: Cluster 1 Setup

> **Run all commands in this section against Cluster 1.**

### C1-Step 1: Set Environment

```bash
export KUBECONFIG1=<PATH_TO_CLUSTER1_KUBECONFIG>
export KUBECONFIG2=<PATH_TO_CLUSTER2_KUBECONFIG>
export CLUSTER1_APP_DOMAIN=<CLUSTER1_APPS_DOMAIN>
export CLUSTER2_APP_DOMAIN=<CLUSTER2_APPS_DOMAIN>

export KUBECONFIG=$KUBECONFIG1
oc whoami
```

**Expected output:**

```
system:admin
```

---

### C1-Step 2: Test RDS Connectivity

```bash
oc run psql-test --rm -it --restart=Never \
  --image=registry.access.redhat.com/ubi9/ubi:latest \
  -n default -- bash -c "
    dnf install -y postgresql > /dev/null 2>&1
    echo '=== Testing connection to RDS 1 ==='
    pg_isready -h <RDS_ENDPOINT_C1> -p 5432 -U postgres
    echo '=== Connecting to RDS 1 ==='
    PGPASSWORD=<RDS_MASTER_PASSWORD> psql 'host=<RDS_ENDPOINT_C1> port=5432 user=postgres dbname=spire sslmode=require' -c 'SELECT version();'
  "
```

**Expected output:**

```
=== Testing connection to RDS 1 ===
<RDS_ENDPOINT_C1>:5432 - accepting connections
=== Connecting to RDS 1 ===
                                      version
--------------------------------------------------------------------------------
 PostgreSQL 15.x on aarch64-unknown-linux-gnu, compiled by gcc ...
(1 row)
```

> **If "Connection timed out"**: RDS is in the wrong VPC or security group rule is missing. See [Troubleshooting](#troubleshooting).

---

### C1-Step 3: Create SPIRE Database User

```bash
oc run psql-setup --rm -it --restart=Never \
  --image=registry.access.redhat.com/ubi9/ubi:latest \
  -n default -- bash -c "
    dnf install -y postgresql > /dev/null 2>&1
    PGPASSWORD=<RDS_MASTER_PASSWORD> psql 'host=<RDS_ENDPOINT_C1> port=5432 user=postgres dbname=spire sslmode=require' <<'SQL'
CREATE USER spire_server WITH PASSWORD '<SPIRE_USER_PASSWORD>';
GRANT CONNECT ON DATABASE spire TO spire_server;
GRANT USAGE ON SCHEMA public TO spire_server;
GRANT CREATE ON SCHEMA public TO spire_server;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO spire_server;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO spire_server;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO spire_server;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO spire_server;
\du spire_server
SQL
  "
```

**Expected output:**

```
CREATE ROLE
GRANT
GRANT
GRANT
ALTER DEFAULT PRIVILEGES
ALTER DEFAULT PRIVILEGES
GRANT
GRANT
             List of roles
  Role name   | Attributes | Member of
--------------+------------+-----------
 spire_server |            | {}
```

**Minimum privileges granted:**

| Privilege | Purpose |
|-----------|---------|
| `CONNECT` | Connect to `spire` database |
| `USAGE ON SCHEMA` | Access the public schema |
| `CREATE ON SCHEMA` | Create tables during first-boot migration |
| `SELECT/INSERT/UPDATE/DELETE` | Normal SPIRE operations |
| `USAGE ON SEQUENCES` | Auto-increment primary keys |

**Not granted** (security boundary): SUPERUSER, CREATEDB, CREATEROLE, DROP DATABASE.

---

### C1-Step 4: Install ZTWIM Operator

```bash
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
```

```bash
echo "Waiting for ZTWIM operator deployment to be created..."
until oc get deployment/zero-trust-workload-identity-manager-controller-manager -n zero-trust-workload-identity-manager &>/dev/null; do
  echo "Deployment not found yet, waiting..."
  sleep 10
done
echo "Deployment found. Waiting for it to become available..."
oc wait --for=condition=Available deployment/zero-trust-workload-identity-manager-controller-manager \
  -n zero-trust-workload-identity-manager --timeout=5m
oc get pods -n zero-trust-workload-identity-manager
```

**Expected output:**

```
Waiting for ZTWIM operator deployment to be created...
Deployment not found yet, waiting...
Deployment found. Waiting for it to become available...
deployment.apps/zero-trust-workload-identity-manager-controller-manager condition met
NAME                                                              READY   STATUS    RESTARTS   AGE
zero-trust-workload-identity-manager-controller-manager-xxxxx      1/1     Running   0          20s
```

---

### C1-Step 5: Deploy SPIRE Stack (https_spiffe + RDS PostgreSQL)

```bash
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: ${CLUSTER1_APP_DOMAIN}
  clusterName: cluster1
EOF
```

> **Note the `datastore` section below** — this is where RDS replaces SQLite. The `connectionString` uses standard PostgreSQL format with `sslmode=require` for encrypted connections.

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
  jwtIssuer: https://oidc-discovery.${CLUSTER1_APP_DOMAIN}
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: postgres
    connectionString: "host=<RDS_ENDPOINT_C1> port=5432 user=spire_server password=<SPIRE_USER_PASSWORD> dbname=spire sslmode=require"
    maxOpenConns: 100
    maxIdleConns: 10
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
  jwtIssuer: https://oidc-discovery.${CLUSTER1_APP_DOMAIN}
EOF
```

```bash
echo "Waiting for SPIRE pods..."
sleep 30
oc get pods -n zero-trust-workload-identity-manager
oc get routes -n zero-trust-workload-identity-manager
```

**Expected output:**

```
NAME                                                              READY   STATUS    RESTARTS   AGE
spire-agent-xxxxx                                                 1/1     Running   0          30s
spire-server-0                                                    2/2     Running   0          30s
spire-spiffe-csi-driver-xxxxx                                     2/2     Running   0          30s
spire-spiffe-oidc-discovery-provider-xxxxx                        1/1     Running   0          30s
zero-trust-workload-identity-manager-controller-manager-xxxxx     1/1     Running   0          2m

NAME                            HOST/PORT                                     SERVICES       PORT
spire-server-federation         federation.<CLUSTER1_APP_DOMAIN>              spire-server   federation
spire-oidc-discovery-provider   oidc-discovery.<CLUSTER1_APP_DOMAIN>          spire-spiffe-oidc-discovery-provider   https
```

---

### C1-Step 6: Verify SPIRE is Using RDS

```bash
oc logs spire-server-0 -c spire-server -n zero-trust-workload-identity-manager 2>/dev/null | grep -i "sql\|datastore\|postgres\|database" | head -10
```

**Expected output:**

```
time="..." level=info msg="Opening SQL database" db_type=postgres subsystem_name=sql
time="..." level=info msg="Initializing new database" subsystem_name=sql
time="..." level=info msg="Connected to SQL database" read_only=false subsystem_name=sql type=postgres version=17.6
time="..." level=info msg="Configured DataStore" reconfigurable=false subsystem_name=catalog
```

> If you see `sqlite3` instead of `postgres`, the connectionString is not configured correctly.

---

### C1-Step 7: Verify Federation (run AFTER Phase 4 is complete)

```bash
echo "=== Cluster 1 - Bundle List ==="
oc -n zero-trust-workload-identity-manager exec spire-server-0 -c spire-server -- \
  /spire-server bundle list -socketPath /tmp/spire-server/private/api.sock
```

**Expected output:**

```
=== Cluster 1 - Bundle List ===
****************************************
* <CLUSTER2_APP_DOMAIN>
****************************************
-----BEGIN CERTIFICATE-----
...certificate data...
-----END CERTIFICATE-----
```

```bash
echo "Cluster 1 federation endpoint:"
curl -sk -o /dev/null -w "HTTP %{http_code}\n" "https://federation.${CLUSTER1_APP_DOMAIN}"
```

**Expected output:** `HTTP 200`

---

### C1-Step 8: Deploy mTLS Server (run AFTER Phase 5)

```bash
oc create namespace federation-test --dry-run=client -o yaml | oc apply -f -
oc create serviceaccount mtls-server -n federation-test --dry-run=client -o yaml | oc apply -f -
```

```bash
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
```

```bash
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
```

```bash
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
```

```bash
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
```

```bash
echo "Waiting for mtls-server pod..."
oc wait --for=condition=Ready pod/mtls-server -n federation-test --timeout=120s
oc get pods -n federation-test
```

**Expected output:**

```
pod/mtls-server condition met
NAME          READY   STATUS    RESTARTS   AGE
mtls-server   2/2     Running   0          30s
```

---

### C1-Step 9: Verify Server SPIFFE ID

```bash
echo "Server SPIFFE ID:"
oc exec -n federation-test mtls-server -c server -- \
  openssl x509 -in /certs/svid.pem -noout -text 2>/dev/null | grep -A1 "Subject Alternative Name"
```

**Expected output:**

```
            X509v3 Subject Alternative Name:
                URI:spiffe://<CLUSTER1_APP_DOMAIN>/ns/federation-test/sa/mtls-server
```

---
---
---

## Phase 4: Cluster 2 Setup

> **Run all commands in this section against Cluster 2.**

### C2-Step 1: Set Environment

```bash
export KUBECONFIG1=<PATH_TO_CLUSTER1_KUBECONFIG>
export KUBECONFIG2=<PATH_TO_CLUSTER2_KUBECONFIG>
export CLUSTER1_APP_DOMAIN=<CLUSTER1_APPS_DOMAIN>
export CLUSTER2_APP_DOMAIN=<CLUSTER2_APPS_DOMAIN>

export KUBECONFIG=$KUBECONFIG2
oc whoami
```

**Expected output:** `system:admin`

---

### C2-Step 2: Test RDS Connectivity

```bash
oc run psql-test --rm -it --restart=Never \
  --image=registry.access.redhat.com/ubi9/ubi:latest \
  -n default -- bash -c "
    dnf install -y postgresql > /dev/null 2>&1
    echo '=== Testing connection to RDS 2 ==='
    pg_isready -h <RDS_ENDPOINT_C2> -p 5432 -U postgres
    echo '=== Connecting to RDS 2 ==='
    PGPASSWORD=<RDS_MASTER_PASSWORD> psql 'host=<RDS_ENDPOINT_C2> port=5432 user=postgres dbname=spire sslmode=require' -c 'SELECT version();'
  "
```

**Expected output:**

```
=== Testing connection to RDS 2 ===
<RDS_ENDPOINT_C2>:5432 - accepting connections
=== Connecting to RDS 2 ===
 PostgreSQL 15.x ...
(1 row)
```

> **Important**: The `sslmode=require` is required. Without it, RDS may reject connections with "no encryption" error.

---

### C2-Step 3: Create SPIRE Database User

```bash
oc run psql-setup --rm -it --restart=Never \
  --image=registry.access.redhat.com/ubi9/ubi:latest \
  -n default -- bash -c "
    dnf install -y postgresql > /dev/null 2>&1
    PGPASSWORD=<RDS_MASTER_PASSWORD> psql 'host=<RDS_ENDPOINT_C2> port=5432 user=postgres dbname=spire sslmode=require' <<'SQL'
CREATE USER spire_server WITH PASSWORD '<SPIRE_USER_PASSWORD>';
GRANT CONNECT ON DATABASE spire TO spire_server;
GRANT USAGE ON SCHEMA public TO spire_server;
GRANT CREATE ON SCHEMA public TO spire_server;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO spire_server;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO spire_server;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO spire_server;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO spire_server;
\du spire_server
SQL
  "
```

**Expected output:** Same as C1-Step 3 (`CREATE ROLE`, `GRANT` x6, user listed).

---

### C2-Step 4: Install ZTWIM Operator

```bash
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
```

```bash
echo "Waiting for ZTWIM operator deployment to be created..."
until oc get deployment/zero-trust-workload-identity-manager-controller-manager -n zero-trust-workload-identity-manager &>/dev/null; do
  echo "Deployment not found yet, waiting..."
  sleep 10
done
echo "Deployment found. Waiting for it to become available..."
oc wait --for=condition=Available deployment/zero-trust-workload-identity-manager-controller-manager \
  -n zero-trust-workload-identity-manager --timeout=5m
oc get pods -n zero-trust-workload-identity-manager
```

**Expected output:** Controller manager pod `1/1 Running`.

---

### C2-Step 5: Deploy SPIRE Stack (https_web + RDS PostgreSQL)

```bash
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: ${CLUSTER2_APP_DOMAIN}
  clusterName: cluster2
EOF
```

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
  jwtIssuer: https://oidc-discovery.${CLUSTER2_APP_DOMAIN}
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: postgres
    connectionString: "host=<RDS_ENDPOINT_C2> port=5432 user=spire_server password=<SPIRE_USER_PASSWORD> dbname=spire sslmode=require"
    maxOpenConns: 100
    maxIdleConns: 10
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
  jwtIssuer: https://oidc-discovery.${CLUSTER2_APP_DOMAIN}
EOF
```

```bash
echo "Waiting for SPIRE pods..."
sleep 30
oc get pods -n zero-trust-workload-identity-manager
oc get routes -n zero-trust-workload-identity-manager
```

**Expected output:** `spire-server-0` at `2/2 Running`, federation route created.

---

### C2-Step 6: Verify SPIRE is Using RDS

```bash
oc logs spire-server-0 -c spire-server -n zero-trust-workload-identity-manager 2>/dev/null | grep -i "sql\|datastore\|postgres\|database" | head -10
```

**Expected output:** `db_type=postgres`, `Connected to SQL database`, `type=postgres`.

---

### C2-Step 7: Install cert-manager

```bash
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
```

```bash
echo "Waiting for cert-manager pods to be created..."
until oc get pods -n cert-manager 2>/dev/null | grep -q cert-manager; do
  echo "cert-manager pods not found yet, waiting..."
  sleep 10
done
echo "cert-manager pods found. Waiting for them to be ready..."
oc wait --for=condition=Ready pods --all -n cert-manager --timeout=5m
oc get pods -n cert-manager
```

**Expected output:**

```
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-xxxxx                        1/1     Running   0          30s
cert-manager-cainjector-xxxxx             1/1     Running   0          45s
cert-manager-webhook-xxxxx                1/1     Running   0          45s
```

---

### C2-Step 8: Create ACME Issuer (Let's Encrypt)

This registers an account with Let's Encrypt and configures HTTP-01 challenge validation.

```bash
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
```

```bash
oc get issuer -n zero-trust-workload-identity-manager
```

**Expected output:**

```
NAME                 READY   AGE
letsencrypt-http01   True    10s
```

---

### C2-Step 9: Create Certificate and RBAC

This requests a publicly trusted certificate for the federation endpoint from Let's Encrypt.

```bash
oc apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: spire-server-federation-tls
  namespace: zero-trust-workload-identity-manager
spec:
  secretName: spire-server-federation-tls
  commonName: federation.${CLUSTER2_APP_DOMAIN}
  dnsNames:
    - federation.${CLUSTER2_APP_DOMAIN}
  usages:
    - server auth
  issuerRef:
    kind: Issuer
    name: letsencrypt-http01
EOF
```

```bash
oc create role secret-reader \
  --verb=get,list,watch \
  --resource=secrets \
  --resource-name=spire-server-federation-tls \
  -n zero-trust-workload-identity-manager

oc create rolebinding secret-reader-binding \
  --role=secret-reader \
  --serviceaccount=openshift-ingress:router \
  -n zero-trust-workload-identity-manager
```

```bash
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

**Expected output:**

```
Certificate is READY!
NAME                          READY   SECRET                        AGE
spire-server-federation-tls   True    spire-server-federation-tls   60s
```

---

### C2-Step 10: Patch SpireServer to Use Certificate

This tells SPIRE to use the Let's Encrypt certificate for its federation endpoint.

```bash
oc patch spireserver cluster --type=merge -p '{"spec":{"federation":{"bundleEndpoint":{"profile":"https_web","httpsWeb":{"servingCert":{"externalSecretRef":"spire-server-federation-tls","fileSyncInterval":86400}}}}}}'
```

```bash
oc get spireserver cluster -o jsonpath='{.spec.federation.bundleEndpoint.httpsWeb.servingCert}' | python3 -m json.tool
```

**Expected output:**

```json
{
    "externalSecretRef": "spire-server-federation-tls",
    "fileSyncInterval": 86400
}
```

---

### C2-Step 11: Verify Federation (run AFTER Phase 5)

```bash
echo "=== Cluster 2 - Bundle List ==="
oc -n zero-trust-workload-identity-manager exec spire-server-0 -c spire-server -- \
  /spire-server bundle list -socketPath /tmp/spire-server/private/api.sock
```

**Expected:** Shows Cluster 1's trust domain and certificate.

```bash
echo "Cluster 2 federation endpoint:"
curl -s -o /dev/null -w "HTTP %{http_code}\n" "https://federation.${CLUSTER2_APP_DOMAIN}"
```

**Expected output:** `HTTP 200` (no `-k` flag needed — Let's Encrypt certificate is publicly trusted!)

---

### C2-Step 12: Deploy mTLS Client (run AFTER Phase 5)

```bash
oc create namespace federation-test --dry-run=client -o yaml | oc apply -f -
oc create serviceaccount mtls-client -n federation-test --dry-run=client -o yaml | oc apply -f -
```

```bash
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
```

```bash
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
```

```bash
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
```

```bash
echo "Waiting for mtls-client pod..."
oc wait --for=condition=Ready pod/mtls-client -n federation-test --timeout=120s
oc get pods -n federation-test
```

**Expected output:**

```
NAME          READY   STATUS    RESTARTS   AGE
mtls-client   2/2     Running   0          15s
```

---

### C2-Step 13: Verify Client SPIFFE ID

```bash
echo "Client SPIFFE ID:"
oc exec -n federation-test mtls-client -c client -- \
  openssl x509 -in /certs/svid.pem -noout -text 2>/dev/null | grep -A1 "Subject Alternative Name"
```

**Expected output:**

```
            X509v3 Subject Alternative Name:
                URI:spiffe://<CLUSTER2_APP_DOMAIN>/ns/federation-test/sa/mtls-client
```

---
---
---

## Phase 5: Federation Trust Setup

> **Run LOCAL-Step 1 from either terminal. Run LOCAL-Step 2 from Cluster 1 terminal. Run LOCAL-Step 3 from Cluster 2 terminal.**

### LOCAL-Step 1: Fetch Trust Bundles

```bash
echo "Fetching Cluster 1 bundle (https_spiffe - needs -k for self-signed)..."
curl -sk "https://federation.${CLUSTER1_APP_DOMAIN}" -o /tmp/cluster1-bundle.json

echo "Fetching Cluster 2 bundle (https_web - Let's Encrypt, no -k needed)..."
curl -s "https://federation.${CLUSTER2_APP_DOMAIN}" -o /tmp/cluster2-bundle.json

echo "Cluster 1 bundle keys: $(jq '.keys | length' /tmp/cluster1-bundle.json)"
echo "Cluster 2 bundle keys: $(jq '.keys | length' /tmp/cluster2-bundle.json)"
```

**Expected output:**

```
Cluster 1 bundle keys: 2
Cluster 2 bundle keys: 2
```

---

### LOCAL-Step 2: Create ClusterFederatedTrustDomain on Cluster 1

> Run from **Cluster 1 terminal**.

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

**Expected output:**

```
clusterfederatedtrustdomain.spire.spiffe.io/cluster-12-federation created
NAME                    TRUST DOMAIN              ENDPOINT URL
cluster-12-federation   <CLUSTER2_APP_DOMAIN>     https://federation.<CLUSTER2_APP_DOMAIN>
```

---

### LOCAL-Step 3: Create ClusterFederatedTrustDomain on Cluster 2

> Run from **Cluster 2 terminal**.

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

**Expected output:**

```
clusterfederatedtrustdomain.spire.spiffe.io/cluster-21-federation created
NAME                    TRUST DOMAIN              ENDPOINT URL
cluster-21-federation   <CLUSTER1_APP_DOMAIN>     https://federation.<CLUSTER1_APP_DOMAIN>
```

---
---
---

## Phase 6: Cross-Cluster mTLS Testing

### LOCAL-Step 4: Create Combined CA Bundle

```bash
curl -sk "https://federation.${CLUSTER1_APP_DOMAIN}" | \
  jq -r '.keys[] | select(.use=="x509-svid") | .x5c[0]' | \
  base64 -d | openssl x509 -inform DER -out /tmp/cluster1-ca.pem

curl -s "https://federation.${CLUSTER2_APP_DOMAIN}" | \
  jq -r '.keys[] | select(.use=="x509-svid") | .x5c[0]' | \
  base64 -d | openssl x509 -inform DER -out /tmp/cluster2-ca.pem

cat /tmp/cluster1-ca.pem /tmp/cluster2-ca.pem > /tmp/combined-bundle.pem
echo "Combined bundle has $(grep -c 'BEGIN CERTIFICATE' /tmp/combined-bundle.pem) certificates"
```

**Expected output:** `Combined bundle has 2 certificates`

---

### LOCAL-Step 5: Copy Bundle to Client and Test mTLS

> Run from **Cluster 2 terminal**.

```bash
export KUBECONFIG=$KUBECONFIG2

oc cp /tmp/combined-bundle.pem federation-test/mtls-client:/certs/combined-bundle.pem -c client

echo "============================================"
echo "CROSS-CLUSTER mTLS TEST"
echo "Client: Cluster 2 (${CLUSTER2_APP_DOMAIN})"
echo "Server: Cluster 1 (${CLUSTER1_APP_DOMAIN})"
echo "============================================"

oc exec -n federation-test mtls-client -c client -- bash -c "
echo 'Q' | timeout 15 openssl s_client \
  -connect mtls-server.${CLUSTER1_APP_DOMAIN}:443 \
  -servername mtls-server.${CLUSTER1_APP_DOMAIN} \
  -cert /certs/svid.pem \
  -key /certs/svid_key.pem \
  -CAfile /certs/combined-bundle.pem \
  2>&1 | grep -E '(Verification|verify return|CONNECTED|Verify return code)'
"
```

**Expected output (SUCCESS):**

```
verify return:1
verify return:1
CONNECTED(00000003)
Verification: OK
Verify return code: 0 (ok)
```

---

### LOCAL-Step 6: Verify Let's Encrypt Certificate

```bash
echo "Cluster 2 certificate issuer:"
echo | openssl s_client -connect federation.${CLUSTER2_APP_DOMAIN}:443 \
  -servername federation.${CLUSTER2_APP_DOMAIN} 2>/dev/null | \
  openssl x509 -noout -issuer
```

**Expected output:**

```
issuer=C=US, O=Let's Encrypt, CN=R13
```

---
---

## Phase 7: Verify Data in RDS (Optional)

You can inspect what SPIRE stored in the database.

### Check RDS 1 (from Cluster 1 terminal)

```bash
oc run psql-check --rm -it --restart=Never \
  --image=registry.access.redhat.com/ubi9/ubi:latest \
  -n default -- bash -c "
    dnf install -y postgresql > /dev/null 2>&1
    PGPASSWORD=<SPIRE_USER_PASSWORD> psql 'host=<RDS_ENDPOINT_C1> port=5432 user=spire_server dbname=spire sslmode=require' <<'SQL'
\dt
SELECT COUNT(*) AS registration_entries FROM registered_entries;
SELECT id, spiffe_id, parent_id FROM registered_entries;
SELECT COUNT(*) AS attested_nodes FROM attested_node_entries;
SELECT spiffe_id, data_type FROM attested_node_entries;
SELECT COUNT(*) AS bundles FROM bundles;
SELECT trust_domain FROM bundles;
SQL
  "
```

**Expected output summary:**

| Table | Expected Data |
|-------|--------------|
| Tables created | 13 tables (registered_entries, bundles, attested_node_entries, etc.) |
| Registration entries | 3+ (OIDC provider, mtls-server, default SA) |
| Attested nodes | 3 (one per worker node, type `k8s_psat`) |
| Bundles | 2 (own trust domain + federated trust domain) |

### Check RDS 2 (from Cluster 2 terminal)

Same command with `<RDS_ENDPOINT_C2>`. Cluster 2 will show more entries (7+) because cert-manager components are also registered.

---
---

## Execution Order Summary

| Order | Where to Run | Steps | What It Does |
|-------|-------------|-------|-------------|
| 1 | Local machine | Phase 0 | Discover domains, VPCs, CIDR |
| 2 | AWS Console | Phase 1 | Create two RDS PostgreSQL instances |
| 3 | AWS Console | Phase 2 | Add security group inbound rules |
| 4 | Cluster 1 terminal | C1-Step 1 → C1-Step 6 | RDS test, DB user, ZTWIM, SPIRE stack |
| 5 | Cluster 2 terminal | C2-Step 1 → C2-Step 10 | RDS test, DB user, ZTWIM, SPIRE, cert-manager, certificate |
| 6 | Either terminal | LOCAL-Step 1 | Fetch trust bundles |
| 7 | Cluster 1 terminal | LOCAL-Step 2 | Create federation trust on Cluster 1 |
| 8 | Cluster 2 terminal | LOCAL-Step 3 | Create federation trust on Cluster 2 |
| 9 | Cluster 1 terminal | C1-Step 7 | Verify federation on Cluster 1 |
| 10 | Cluster 2 terminal | C2-Step 11 | Verify federation on Cluster 2 |
| 11 | Cluster 1 terminal | C1-Step 8 → C1-Step 9 | Deploy mTLS server |
| 12 | Cluster 2 terminal | C2-Step 12 → C2-Step 13 | Deploy mTLS client |
| 13 | Cluster 2 terminal | LOCAL-Step 4 → LOCAL-Step 6 | Combined bundle + mTLS test |
| 14 | Either terminal | Phase 7 (optional) | Verify data in RDS |

---

## Troubleshooting

### RDS Connection Times Out

```
psql: error: could not connect to server: Connection timed out
```

| Cause | How to Verify | Solution |
|-------|--------------|----------|
| RDS in wrong VPC | RDS Console → instance → Connectivity → VPC vs EC2 → worker → VPC | Delete and recreate RDS in the correct VPC |
| Security group missing rule | EC2 → Security Groups → check inbound rules | Add PostgreSQL (5432) with source = your VPC CIDR |
| VPC not in RDS dropdown | RDS creation page → VPC dropdown | Create a DB subnet group first (RDS → Subnet groups) |

### RDS Password Authentication Failed

```
FATAL: password authentication failed for user "postgres"
FATAL: no pg_hba.conf entry for host "x.x.x.x", user "postgres", database "spire", no encryption
```

**Solution:** Add `sslmode=require` to your psql connection string. RDS may reject unencrypted connections.

### DB Instance Identifier Already Exists

**Solution:** The old instance is still deleting (takes 5-10 min). Wait or use a different name.

### SPIRE Server CrashLoopBackOff

```bash
oc logs spire-server-0 -c spire-server -n zero-trust-workload-identity-manager
```

| Log Message | Solution |
|-------------|----------|
| `connection refused` | Check RDS is "Available" and security group rules |
| `password authentication failed` | Verify credentials in connectionString match the DB user |
| `database "spire" does not exist` | Recreate RDS with initial database name = `spire` |
| `permission denied for schema public` | Re-run the GRANT commands from Step 3 |

### Certificate Not Ready

```bash
oc describe certificate spire-server-federation-tls -n zero-trust-workload-identity-manager
oc get challenges -n zero-trust-workload-identity-manager
```

| Cause | Solution |
|-------|----------|
| DNS not resolving | `dig federation.<APP_DOMAIN>` must return an IP |
| Port 80 blocked | HTTP-01 challenge needs inbound port 80 on the cluster ingress |
| Rate limit | Wait 1 hour or use Let's Encrypt staging server |

### mTLS Verification Fails

```
verify error:num=19:self-signed certificate in certificate chain
```

**Solution:** Use the **combined** bundle (both cluster CAs), not a single cluster's bundle.

---

## Cleanup

### Delete OpenShift Resources

```bash
# Cluster 1
export KUBECONFIG=$KUBECONFIG1
oc delete namespace federation-test
oc delete clusterspiffeid mtls-server-workload
oc delete clusterfederatedtrustdomain cluster-12-federation

# Cluster 2
export KUBECONFIG=$KUBECONFIG2
oc delete namespace federation-test
oc delete clusterspiffeid mtls-client-workload
oc delete clusterfederatedtrustdomain cluster-21-federation
```

### Delete AWS Resources

1. **RDS → Databases** → Delete `spire-ds1` and `spire-ds2` (uncheck "Create final snapshot")
2. **EC2 → Security Groups** → Delete `spire-rds-sg` and `spire-rds-sg-c2`
3. **RDS → Subnet groups** → Delete any subnet groups created for this test

---

## Glossary

| Term | Definition |
|------|------------|
| **ACME** | Automated Certificate Management Environment — protocol for TLS certificate issuance (RFC 8555) |
| **SPIFFE** | Secure Production Identity Framework For Everyone — workload identity standard |
| **SPIRE** | SPIFFE Runtime Environment — reference implementation |
| **SVID** | SPIFFE Verifiable Identity Document (X.509 cert or JWT) |
| **Trust Domain** | SPIFFE identity namespace (e.g., `apps.example.com`) |
| **Trust Bundle** | Collection of CA certificates for a trust domain |
| **Federation** | Establishing trust between SPIRE servers in different trust domains |
| **https_spiffe** | Federation profile using SPIRE's self-signed certificate |
| **https_web** | Federation profile using a publicly trusted certificate |
| **ZTWIM** | Zero Trust Workload Identity Manager — Red Hat operator for SPIRE on OpenShift |
| **RDS** | Amazon Relational Database Service |
| **VPC** | Virtual Private Cloud — isolated virtual network on AWS |
| **Security Group** | Virtual firewall controlling inbound/outbound traffic to AWS resources |
| **DB Subnet Group** | Subnets within a VPC that RDS can use |
| **sslmode=require** | PostgreSQL parameter enforcing encrypted TLS connections |
| **spiffe-helper** | Sidecar that writes SVID certificates to files for application use |

---

## Differences from the SQLite-Based Guide

| Aspect | SQLite Guide | This RDS Guide |
|--------|-------------|----------------|
| `databaseType` | `sqlite3` | `postgres` |
| `connectionString` | `/run/spire/data/datastore.sqlite3` | `host=<RDS> port=5432 user=spire_server password=<PASS> dbname=spire sslmode=require` |
| `maxIdleConns` | `2` | `10` (increased for remote database) |
| Infrastructure | None | RDS instances + VPC networking + security groups |
| Database user | None | `CREATE USER` with minimum SPIRE privileges |
| Connectivity test | None | `pg_isready` and `psql` from OpenShift pods |
| All other phases | Same | Same |

---

*Document created: April 19, 2026*
*Tested on: AWS OpenShift 4.19 with Amazon RDS PostgreSQL 15.x (db.t4g.micro)*
*Datastore: Two separate RDS instances, one per cluster, in separate VPCs*
*All tests passed: RDS connectivity, SPIRE datastore, federation, Let's Encrypt certificate, cross-cluster mTLS*
