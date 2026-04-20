# SPIFFE Federation Flow — Operators, Components & Data Flow

Visual reference for the SPIFFE cross-cluster federation setup using Amazon RDS PostgreSQL, ZTWIM Operator, cert-manager, and Let's Encrypt.

---

## 1. End-to-End Setup Flow

This diagram shows the complete execution order — what runs where, and in what sequence.

```mermaid
flowchart TD
    subgraph phase0 [Phase 0 — Discovery]
        A1["Find Apps Domains\n(oc get ingresses.config)"] --> A2["Find VPC IDs\n(AWS Console → EC2)"]
        A2 --> A3["Find VPC CIDR\n(oc get nodes -o wide)"]
    end

    subgraph phase1 [Phase 1 — AWS Console: Create RDS]
        B1["Create RDS spire-ds1\nin Cluster 1 VPC"] --> B2["Create RDS spire-ds2\nin Cluster 2 VPC"]
        B2 --> B3["Note endpoints from\nConnectivity & security tab"]
    end

    subgraph phase2 [Phase 2 — AWS Console: Security Groups]
        C1["Add inbound rule to spire-rds-sg\nPostgreSQL 5432 ← VPC CIDR"] --> C2["Add inbound rule to spire-rds-sg-c2\nPostgreSQL 5432 ← VPC CIDR"]
    end

    subgraph phase3 [Phase 3 — Cluster 1]
        D1["Test RDS connectivity\n(pg_isready + psql)"] --> D2["Create spire_server DB user\n(GRANT minimum privileges)"]
        D2 --> D3["Install ZTWIM Operator\n(Subscription CRD)"]
        D3 --> D4["Deploy SPIRE Stack\n(SpireServer + SpireAgent +\nCSIDriver + OIDC)"]
        D4 --> D5["Verify SPIRE uses postgres\n(check logs)"]
    end

    subgraph phase4 [Phase 4 — Cluster 2]
        E1["Test RDS connectivity"] --> E2["Create spire_server DB user"]
        E2 --> E3["Install ZTWIM Operator"]
        E3 --> E4["Deploy SPIRE Stack"]
        E4 --> E5["Install cert-manager Operator"]
        E5 --> E6["Create ACME Issuer\n(Let's Encrypt)"]
        E6 --> E7["Request TLS Certificate\n(Certificate CRD)"]
        E7 --> E8["Patch SpireServer to use\nLet's Encrypt cert"]
    end

    subgraph phase5 [Phase 5 — Federation Trust]
        F1["Fetch trust bundles\nfrom both clusters"] --> F2["Create ClusterFederatedTrustDomain\non Cluster 1 → trusts Cluster 2"]
        F2 --> F3["Create ClusterFederatedTrustDomain\non Cluster 2 → trusts Cluster 1"]
    end

    subgraph phase6 [Phase 6 — mTLS Testing]
        G1["Deploy mtls-server on Cluster 1\n(ClusterSPIFFEID + Pod + Route)"] --> G2["Deploy mtls-client on Cluster 2\n(ClusterSPIFFEID + Pod)"]
        G2 --> G3["Build combined CA bundle\n(both cluster CAs)"]
        G3 --> G4["openssl s_client → s_server\nCross-cluster mTLS handshake"]
        G4 --> G5{"Verify return code: 0 (ok)?"}
        G5 -->|Yes| G6["Federation SUCCESS"]
        G5 -->|No| G7["Check bundles,\ncerts, routes"]
    end

    phase0 --> phase1 --> phase2 --> phase3 --> phase4 --> phase5 --> phase6
```

---

## 2. Operator and Component Interaction

This diagram shows every operator, the CRDs it watches, the components it creates, and their significance.

```mermaid
flowchart TB
    subgraph ztwimOp [ZTWIM Operator — Zero Trust Workload Identity Manager]
        direction TB
        ZO["Controller Manager\n(runs in zero-trust-workload-identity-manager namespace)"]

        subgraph ztwimCRDs [CRDs Watched by ZTWIM]
            CR1["ZeroTrustWorkloadIdentityManager\n● Sets trustDomain\n● Sets clusterName"]
            CR2["SpireServer\n● CA subject, JWT issuer\n● Datastore config (RDS)\n● Federation profile\n● Bundle endpoint"]
            CR3["SpireAgent\n● Node attestor (k8s_psat)\n● Workload attestors"]
            CR4["SpiffeCSIDriver\n● Exposes SPIRE socket\n  to pods via CSI volume"]
            CR5["SpireOIDCDiscoveryProvider\n● OIDC endpoint for\n  JWT-SVID validation"]
        end

        subgraph ztwimCreates [Components Created]
            CC1["spire-server\n(StatefulSet, 1 replica)\nManages identities,\nbundles, registration"]
            CC2["spire-agent\n(DaemonSet, every node)\nProvides workload API\nvia Unix socket"]
            CC3["spire-spiffe-csi-driver\n(DaemonSet)\nMounts SPIRE socket\ninto pod volumes"]
            CC4["spire-spiffe-oidc-discovery-provider\n(Deployment)\nServes OIDC discovery\nfor JWT validation"]
            CC5["federation Route\nExposes bundle endpoint\nat federation.<apps-domain>"]
        end

        ZO --> CR1 & CR2 & CR3 & CR4 & CR5
        CR2 --> CC1
        CR3 --> CC2
        CR4 --> CC3
        CR5 --> CC4
        CR2 --> CC5
    end

    subgraph certMgrOp [cert-manager Operator — Cluster 2 Only]
        direction TB
        CMO["cert-manager Controller\n(runs in cert-manager namespace)"]

        subgraph certCRDs [CRDs Watched by cert-manager]
            CM1["Issuer\n● ACME server URL\n● Account key secret\n● HTTP-01 solver config"]
            CM2["Certificate\n● DNS name for federation endpoint\n● References Issuer\n● server auth usage"]
        end

        subgraph certCreates [Resources Created]
            CMC1["letsencrypt-account-key\n(Secret)\nACME account private key"]
            CMC2["HTTP-01 Challenge Pod + Route\nTemporary pod to prove\ndomain ownership"]
            CMC3["spire-server-federation-tls\n(Secret: tls.crt + tls.key)\nPublicly trusted certificate\nfrom Let's Encrypt"]
        end

        CMO --> CM1 & CM2
        CM1 --> CMC1
        CM2 --> CMC2
        CMC2 -->|"ACME validation\npasses"| CMC3
    end

    subgraph spireCRDs [SPIRE-Level CRDs — Identity & Federation]
        direction TB
        SP1["ClusterSPIFFEID\n● spiffeIDTemplate\n● podSelector + namespaceSelector\n● federatesWith list"]
        SP2["ClusterFederatedTrustDomain\n● Remote trustDomain\n● Bundle endpoint URL\n● Profile: https_web or https_spiffe\n● Initial trust bundle"]
    end

    CC1 -->|"reads"| SP1
    CC1 -->|"reads"| SP2
    CMC3 -->|"mounted by\nSpireServer patch"| CC1

    style ztwimOp fill:transparent
    style certMgrOp fill:transparent
    style spireCRDs fill:transparent
```

### Significance of Each Operator

| Operator | Installed On | Role | What Happens Without It |
|----------|-------------|------|------------------------|
| **ZTWIM Operator** | Both clusters | Manages the entire SPIRE stack lifecycle — installs, configures, and reconciles SPIRE Server, Agent, CSI Driver, and OIDC Provider based on CRDs | No SPIRE components exist; no workload identities, no federation |
| **cert-manager Operator** | Cluster 2 only | Automates TLS certificate lifecycle — requests, validates (HTTP-01), issues, and renews certificates from Let's Encrypt | Cluster 2 cannot use `https_web` profile; federation endpoint would need manual certificate management or fall back to `https_spiffe` |

### Significance of Each CRD

| CRD | Operator | What It Does |
|-----|----------|-------------|
| `ZeroTrustWorkloadIdentityManager` | ZTWIM | Sets the trust domain and cluster name — the identity root for all workloads |
| `SpireServer` | ZTWIM | Configures the SPIRE Server: CA, datastore (RDS connection), federation profile, bundle endpoints |
| `SpireAgent` | ZTWIM | Configures node attestation (k8s_psat) and workload attestation methods |
| `SpiffeCSIDriver` | ZTWIM | Enables pods to mount the SPIRE Agent socket as a CSI volume (no hostPath needed) |
| `SpireOIDCDiscoveryProvider` | ZTWIM | Exposes OIDC discovery endpoint for services that validate JWT-SVIDs |
| `Issuer` | cert-manager | Registers an ACME account with Let's Encrypt and defines the challenge solver |
| `Certificate` | cert-manager | Requests a specific TLS certificate; cert-manager handles the full ACME flow |
| `ClusterSPIFFEID` | SPIRE (via ZTWIM) | Registers workload identity rules — maps pod labels to SPIFFE IDs and federation peers |
| `ClusterFederatedTrustDomain` | SPIRE (via ZTWIM) | Establishes trust with a remote SPIRE server — configures bundle endpoint and initial trust bundle |

---

## 3. Data Flow — Identity Issuance and Federation

### 3a. How a Pod Gets Its SPIFFE Identity

```mermaid
sequenceDiagram
    participant Pod as Application Pod
    participant Helper as spiffe-helper<br/>(sidecar)
    participant Agent as SPIRE Agent<br/>(DaemonSet on same node)
    participant Server as SPIRE Server
    participant RDS as Amazon RDS<br/>PostgreSQL

    Note over Pod,RDS: Pod starts with CSI volume mounting SPIRE Agent socket

    Helper->>Agent: Connect to Unix socket<br/>/spiffe-workload-api/spire-agent.sock
    Agent->>Agent: Identify pod via<br/>Kubernetes API (k8s attestor)
    Agent->>Server: Request SVID for<br/>spiffe://trust-domain/ns/.../sa/...
    Server->>RDS: Lookup registered_entries<br/>and selectors tables
    RDS-->>Server: Entry found,<br/>federatesWith confirmed
    Server->>Server: Sign X.509 SVID<br/>with SPIRE CA
    Server-->>Agent: X.509 SVID + bundle<br/>(includes federated CAs)
    Agent-->>Helper: SVID cert, key, bundle
    Helper->>Helper: Write to /certs/ directory:<br/>svid.pem, svid_key.pem, bundle.pem
    Pod->>Pod: Application reads certs<br/>from /certs/ and starts TLS
```

### 3b. Federation Bundle Exchange

```mermaid
sequenceDiagram
    participant S1 as SPIRE Server<br/>Cluster 1 (https_spiffe)
    participant FE1 as Federation Endpoint<br/>federation.cluster1.example.com
    participant FE2 as Federation Endpoint<br/>federation.cluster2.example.com
    participant S2 as SPIRE Server<br/>Cluster 2 (https_web)
    participant LE as Let's Encrypt

    Note over S1,S2: Initial Setup — Bidirectional Trust

    LE->>S2: Issue TLS certificate<br/>(via cert-manager ACME flow)

    Note over S1,FE1: Cluster 1 serves bundle with self-signed SPIRE cert
    Note over S2,FE2: Cluster 2 serves bundle with Let's Encrypt cert

    S1->>FE2: Fetch Cluster 2 bundle<br/>(HTTPS — trusted via Let's Encrypt CA)
    FE2-->>S1: Trust bundle (JSON)<br/>Contains Cluster 2 CA public keys

    S2->>FE1: Fetch Cluster 1 bundle<br/>(HTTPS — verified via SPIFFE ID)
    FE1-->>S2: Trust bundle (JSON)<br/>Contains Cluster 1 CA public keys

    Note over S1,S2: Periodic refresh — bundles auto-update as CAs rotate
```

### 3c. Cross-Cluster mTLS Handshake

```mermaid
sequenceDiagram
    participant Client as mtls-client<br/>(Cluster 2)
    participant Router1 as OpenShift Router<br/>(Cluster 1, passthrough)
    participant Server as mtls-server<br/>(Cluster 1)

    Note over Client,Server: Both pods have SPIFFE SVIDs from their respective SPIRE Servers

    Client->>Router1: TLS ClientHello<br/>SNI: mtls-server.cluster1.example.com
    Router1->>Server: Forward (TLS passthrough)

    Server-->>Client: ServerHello + Server Certificate<br/>spiffe://cluster1/ns/federation-test/sa/mtls-server
    Server->>Client: CertificateRequest<br/>(demands client cert)
    Client-->>Server: Client Certificate<br/>spiffe://cluster2/ns/federation-test/sa/mtls-client

    Note over Server: Validates client cert against<br/>combined bundle (includes Cluster 2 CA)
    Note over Client: Validates server cert against<br/>combined bundle (includes Cluster 1 CA)

    Server-->>Client: Handshake Complete
    Client->>Server: Encrypted application data
    Note over Client,Server: Verify return code: 0 (ok)<br/>Cross-cluster mTLS SUCCESS
```

---

## 4. What SPIRE Stores in RDS PostgreSQL

```mermaid
erDiagram
    bundles {
        string trust_domain PK "e.g. spiffe://apps.cluster1.example.com"
        blob data "CA certificates (X.509)"
    }
    registered_entries {
        int id PK "Auto-increment"
        string spiffe_id "e.g. spiffe://trust-domain/ns/federation-test/sa/mtls-server"
        string parent_id "SPIRE Agent SPIFFE ID"
    }
    attested_node_entries {
        string spiffe_id PK "e.g. spiffe://trust-domain/spire/agent/k8s_psat/cluster1/node-uuid"
        string data_type "k8s_psat"
    }
    selectors {
        int id PK
        int registered_entry_id FK
        string type "e.g. k8s:pod-label"
        string value "e.g. spiffe.io/spiffe-id:true"
    }
    federated_registration_entries {
        int registered_entry_id FK
        string bundle_id FK "Trust domain of federation peer"
    }
    federated_trust_domains {
        string trust_domain PK "Remote trust domain"
        string bundle_endpoint_url "URL to fetch remote bundle"
        string bundle_endpoint_profile "https_web or https_spiffe"
    }
    dns_names {
        int id PK
        int registered_entry_id FK
        string value "DNS SAN for the SVID"
    }
    ca_journals {
        int id PK
        blob data "CA signing key rotation history"
    }
    migrations {
        int version PK "Schema version"
    }

    registered_entries ||--o{ selectors : "has"
    registered_entries ||--o{ federated_registration_entries : "federates via"
    registered_entries ||--o{ dns_names : "has"
    federated_registration_entries }o--|| bundles : "references"
    federated_trust_domains }o--|| bundles : "syncs into"
```

### What Each Table Stores

| Table | Records | Example |
|-------|---------|---------|
| `bundles` | Trust domain CA certificates (own + federated) | 2 entries: own cluster + peer cluster |
| `registered_entries` | Workload identities registered with SPIRE | `spiffe://cluster1/ns/federation-test/sa/mtls-server` |
| `attested_node_entries` | SPIRE Agents that proved their identity | 3 per cluster (one per worker node, type `k8s_psat`) |
| `selectors` | Pod matching rules (labels, service accounts) | `k8s:pod-label:spiffe.io/spiffe-id:true` |
| `federated_registration_entries` | Links workloads to their federation peers | mtls-server → federates with Cluster 2 domain |
| `federated_trust_domains` | Remote bundle endpoints and their profiles | `https_web` endpoint for Cluster 2 |
| `ca_journals` | CA key rotation history | Tracks SPIRE CA signing key changes |
| `migrations` | Database schema version | Ensures SPIRE can upgrade schema across versions |

---

## 5. Component Map — What Runs Where

```mermaid
flowchart LR
    subgraph cluster1 [Cluster 1 — https_spiffe Profile]
        direction TB
        subgraph ns1_ztwim [Namespace: zero-trust-workload-identity-manager]
            Z1_CM["ZTWIM Controller Manager\n(Deployment)"]
            Z1_SS["spire-server\n(StatefulSet, 1 pod)"]
            Z1_OIDC["OIDC Discovery Provider\n(Deployment)"]
            Z1_FedRoute["Route: federation.cluster1...\n(self-signed SPIRE cert)"]
        end
        subgraph ns1_agents [Every Worker Node]
            Z1_Agent["spire-agent\n(DaemonSet pod)"]
            Z1_CSI["spiffe-csi-driver\n(DaemonSet pod)"]
        end
        subgraph ns1_test [Namespace: federation-test]
            Z1_Server["mtls-server Pod\n(server + spiffe-helper)"]
            Z1_Svc["Service + Route\nmtls-server.cluster1..."]
        end
        Z1_SS --> RDS1
        Z1_Agent -->|"workload API\n(Unix socket)"| Z1_CSI
        Z1_CSI -->|"CSI volume\nmount"| Z1_Server
    end

    subgraph cluster2 [Cluster 2 — https_web Profile]
        direction TB
        subgraph ns2_ztwim [Namespace: zero-trust-workload-identity-manager]
            Z2_CM["ZTWIM Controller Manager\n(Deployment)"]
            Z2_SS["spire-server\n(StatefulSet, 1 pod)"]
            Z2_OIDC["OIDC Discovery Provider\n(Deployment)"]
            Z2_FedRoute["Route: federation.cluster2...\n(Let's Encrypt cert)"]
        end
        subgraph ns2_certmgr [Namespace: cert-manager]
            CM_ctrl["cert-manager\n(Deployment)"]
            CM_inj["cert-manager-cainjector\n(Deployment)"]
            CM_wh["cert-manager-webhook\n(Deployment)"]
        end
        subgraph ns2_agents [Every Worker Node]
            Z2_Agent["spire-agent\n(DaemonSet pod)"]
            Z2_CSI["spiffe-csi-driver\n(DaemonSet pod)"]
        end
        subgraph ns2_test [Namespace: federation-test]
            Z2_Client["mtls-client Pod\n(client + spiffe-helper)"]
        end
        Z2_SS --> RDS2
        CM_ctrl -->|"issues cert"| Z2_FedRoute
        Z2_Agent -->|"workload API"| Z2_CSI
        Z2_CSI -->|"CSI volume"| Z2_Client
    end

    subgraph aws [AWS]
        RDS1[("RDS PostgreSQL\nspire-ds1\n(Cluster 1 VPC)")]
        RDS2[("RDS PostgreSQL\nspire-ds2\n(Cluster 2 VPC)")]
    end

    Z1_FedRoute <-->|"Bundle\nexchange"| Z2_FedRoute
    Z2_Client -->|"mTLS via\nOpenShift Router"| Z1_Svc
```

---

*Generated from: [SPIFFE-Federation-RDS-PostgreSQL-Guide-4.19.md](https://github.com/sayak-redhat/redhat-openshift-onboarding/blob/main/SPIFFE-Federation-RDS-PostgreSQL-Guide-4.19.md)*
