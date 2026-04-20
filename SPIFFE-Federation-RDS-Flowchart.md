# SPIFFE Federation — Visual Guide for Beginners

A beginner-friendly visual reference for the SPIFFE cross-cluster federation setup using Amazon RDS PostgreSQL, the ZTWIM Operator, cert-manager, and Let's Encrypt.

> **Reading tip**: Every section starts with a plain-English explanation before the diagram. If a technical term appears, it is explained inline in parentheses.

---

## 1. The Big Picture

**Goal**: Two separate OpenShift clusters need to **trust each other** so that an application on Cluster 1 can securely communicate with an application on Cluster 2 — and both sides can verify "yes, I'm really talking to who I think I'm talking to."

Think of it like **two companies that want their employees to visit each other's offices**. Each company:
1. Sets up an **ID card system** (SPIRE) that issues digital passports to every employee (application pod)
2. Stores the employee registry in a **secure database** (Amazon RDS PostgreSQL)
3. **Exchanges their list of trusted ID issuers** with the other company (federation)
4. One company gets their trust list **notarized by a public authority** (Let's Encrypt via cert-manager) so anyone can verify it without special setup

The end result: an app on Cluster 2 shows its ID card to an app on Cluster 1, Cluster 1 checks the card against the trusted issuer list, and the secure connection is established.

```mermaid
flowchart LR
    A["Set up Database\n(where identities\nare stored)"] --> B["Install Identity System\n(issues digital passports\nto every app)"]
    B --> C["Exchange Trust Lists\n(each cluster learns\nwho the other trusts)"]
    C --> D["Get Public Certificate\n(so outsiders can\nverify Cluster 2)"]
    D --> E["Secure Connection\n(apps prove identity\nto each other)"]
```

---

## 2. End-to-End Setup Flow

> **What this shows**: The step-by-step order of everything you do, from creating databases in AWS all the way to testing the secure connection. Each box is one action, grouped by where you perform it.

```mermaid
flowchart TD
    subgraph phase0 [Phase 0 — Gather Information]
        A1["Find your cluster domains\n(the base URL for each cluster)"] --> A2["Find your VPC IDs\n(the private network each\ncluster runs inside on AWS)"]
        A2 --> A3["Find your network range\n(which IP addresses your\ncluster machines use)"]
    end

    subgraph phase1 [Phase 1 — Create Databases in AWS]
        B1["Create Database 1\nin Cluster 1's network"] --> B2["Create Database 2\nin Cluster 2's network"]
        B2 --> B3["Copy the database addresses\nfrom the AWS dashboard"]
    end

    subgraph phase2 [Phase 2 — Open Firewall for Databases]
        C1["Allow Cluster 1 machines\nto reach Database 1\n(port 5432)"] --> C2["Allow Cluster 2 machines\nto reach Database 2\n(port 5432)"]
    end

    subgraph phase3 [Phase 3 — Set Up Cluster 1]
        D1["Test: can pods\nreach the database?"] --> D2["Create a database user\nwith limited permissions"]
        D2 --> D3["Install the Identity Operator\n(manages the whole\nidentity system)"]
        D3 --> D4["Deploy the Identity System\n(SPIRE Server, Agent,\nand supporting components)"]
        D4 --> D5["Verify the identity system\nis using the database"]
    end

    subgraph phase4 [Phase 4 — Set Up Cluster 2]
        E1["Test: can pods\nreach the database?"] --> E2["Create a database user"]
        E2 --> E3["Install the Identity Operator"]
        E3 --> E4["Deploy the Identity System"]
        E4 --> E5["Install the Certificate Manager\n(automates getting certificates\nfrom Let's Encrypt)"]
        E5 --> E6["Request a public certificate\nfor the federation endpoint"]
        E6 --> E7["Tell the identity system\nto use the new certificate"]
    end

    subgraph phase5 [Phase 5 — Connect the Two Clusters]
        F1["Download each cluster's\ntrust list"] --> F2["Give Cluster 1 the\ntrust list from Cluster 2"]
        F2 --> F3["Give Cluster 2 the\ntrust list from Cluster 1"]
    end

    subgraph phase6 [Phase 6 — Test the Secure Connection]
        G1["Deploy a test server\non Cluster 1"] --> G2["Deploy a test client\non Cluster 2"]
        G2 --> G3["Client connects to server\nusing mutual TLS"]
        G3 --> G4{"Did both sides\nverify each other?"}
        G4 -->|Yes| G5["SUCCESS"]
        G4 -->|No| G6["Check trust lists\nand certificates"]
    end

    phase0 --> phase1 --> phase2 --> phase3 --> phase4 --> phase5 --> phase6
```

---

## 3. What Each Operator Does and Why

> **What is an Operator?** An Operator is a piece of software that runs inside OpenShift and automates a complex task. You tell it *what* you want (by creating a configuration file called a CRD), and the Operator figures out *how* to make it happen — installing software, configuring it, and keeping it running.

### The Two Operators in This Setup

```mermaid
flowchart TB
    subgraph ztwim [ZTWIM Operator — The Identity Manager]
        direction TB
        ZO["You install this operator\non BOTH clusters"]

        subgraph ztwimYouGive [Configuration files you create]
            CR1["Identity Manager Config\n— Sets your cluster's\n   identity domain name\n— Sets the cluster name"]
            CR2["Identity Server Config\n— How to sign ID cards\n— Database connection\n— Federation settings"]
            CR3["Identity Agent Config\n— How to verify pods\n   are who they claim to be"]
            CR4["Secure Socket Config\n— How to deliver the\n   identity socket to pods"]
            CR5["OIDC Provider Config\n— Publishes identity info\n   for external verification"]
        end

        subgraph ztwimCreates [What the operator creates for you]
            CC1["Identity Server\n(one per cluster)\nSigns ID cards, stores\nthem in the database"]
            CC2["Identity Agent\n(one per machine)\nDelivers ID cards\nto pods on that machine"]
            CC3["Secure Socket Driver\n(one per machine)\nLets pods access the\nagent's socket securely"]
            CC4["OIDC Discovery Service\nPublishes identity info\nso external tools can\nvalidate tokens"]
            CC5["Federation Endpoint\nA public URL where other\nclusters can download\nyour trust list"]
        end

        ZO --> CR1 & CR2 & CR3 & CR4 & CR5
        CR2 --> CC1
        CR3 --> CC2
        CR4 --> CC3
        CR5 --> CC4
        CR2 --> CC5
    end

    subgraph certmgr [cert-manager Operator — The Certificate Automator]
        direction TB
        CMO["You install this operator\nONLY on Cluster 2"]

        subgraph certYouGive [Configuration files you create]
            CM1["Certificate Authority Config\n— Which authority to use\n   (Let's Encrypt)\n— How to prove you own\n   the domain (HTTP challenge)"]
            CM2["Certificate Request\n— Which domain name\n   needs a certificate"]
        end

        subgraph certCreates [What the operator creates for you]
            CMC1["Account Key\nYour identity with\nLet's Encrypt"]
            CMC2["Temporary Proof Page\nA web page Let's Encrypt\nchecks to verify you\nown the domain"]
            CMC3["Signed Certificate\nA publicly trusted\ncertificate for your\nfederation endpoint"]
        end

        CMO --> CM1 & CM2
        CM1 --> CMC1
        CM2 --> CMC2
        CMC2 -->|"Let's Encrypt\nverifies ownership"| CMC3
    end

    CMC3 -->|"Certificate is given to\nthe Identity Server"| CC1
```

### Why Each Operator Matters

| Operator | Installed On | What It Does (Plain English) | Real-World Analogy | What Breaks Without It |
|----------|-------------|-------|--------|----------------------|
| **ZTWIM Operator** | Both clusters | Installs and manages the entire identity system — the server that signs ID cards, the agents that deliver them, and the federation endpoint | Like an **HR department** that issues employee badges, keeps the employee directory updated, and handles badge verification | No identity system exists at all; no pods get ID cards, no federation is possible |
| **cert-manager** | Cluster 2 only | Automatically requests, validates, and renews a publicly trusted certificate from Let's Encrypt | Like hiring a **notary service** that stamps your company's official documents so anyone in the world trusts them | Cluster 2's federation endpoint uses a self-signed certificate; other clusters can't automatically trust it without manual setup |

### What Each Configuration File (CRD) Does

| Configuration File | Which Operator Uses It | What It Does (Plain English) | Analogy |
|-------------------|----------------------|------|---------|
| `ZeroTrustWorkloadIdentityManager` | ZTWIM | Sets the identity domain name for your cluster — this is the "company name" on every ID card | Registering your **company name** with the government |
| `SpireServer` | ZTWIM | Configures the identity server: how to sign cards, where to store data (your RDS database), and how to share trust with other clusters | Setting up the **badge printing machine** and connecting it to the employee database |
| `SpireAgent` | ZTWIM | Configures how each machine verifies that a pod is legitimate before giving it an ID card | Training the **security guard** on each floor to check IDs before handing out badges |
| `SpiffeCSIDriver` | ZTWIM | Lets application pods securely access the identity agent's socket (communication channel) | Installing a **badge reader slot** in every office door |
| `SpireOIDCDiscoveryProvider` | ZTWIM | Publishes identity information so external tools (outside the cluster) can verify tokens | Putting your company's **badge verification hotline number** on a public website |
| `Issuer` | cert-manager | Registers with Let's Encrypt and defines how to prove you own the domain | Opening an **account with the notary service** and agreeing on the verification method |
| `Certificate` | cert-manager | Requests a specific certificate for a specific domain name | Submitting a **notarization request** for a specific document |
| `ClusterSPIFFEID` | ZTWIM (SPIRE) | Rules for which pods get which ID cards, and which other clusters they're allowed to talk to | The **employee directory entry** — "John in Engineering can visit Building B" |
| `ClusterFederatedTrustDomain` | ZTWIM (SPIRE) | Tells your cluster to trust another cluster — provides the other cluster's trust list and how to refresh it | Adding another company to your **list of trusted partner organizations** |

---

## 4. How a Pod Gets Its Identity (Step by Step)

> **What this shows**: When an application pod starts up, it needs an identity certificate (a "digital passport") to prove who it is. This diagram shows the exact sequence of events from pod startup to the app being ready for secure communication.

```mermaid
sequenceDiagram
    participant App as Your Application
    participant Helper as Certificate Writer<br/>(sidecar container)
    participant Agent as Identity Agent<br/>(runs on same machine)
    participant Server as Identity Server<br/>(one per cluster)
    participant DB as Database<br/>(Amazon RDS)

    Note over App,DB: Pod starts — the identity socket is<br/>automatically mounted into the pod

    Helper->>Agent: "Hi, I need an identity<br/>certificate for this pod"
    Agent->>Agent: Checks with Kubernetes:<br/>"Is this pod legitimate?"
    Agent->>Server: "This verified pod needs a<br/>certificate for identity:<br/>spiffe://cluster/ns/app-ns/sa/app-name"
    Server->>DB: Looks up: "Is this identity<br/>registered? Who can it<br/>talk to?"
    DB-->>Server: "Yes, it's registered.<br/>It can talk to Cluster 2."
    Server->>Server: Signs an identity certificate<br/>valid for a few hours
    Server-->>Agent: Sends: certificate + private key<br/>+ list of trusted issuers<br/>(including Cluster 2's issuer)
    Agent-->>Helper: Delivers the certificate package
    Helper->>Helper: Writes 3 files to /certs/:<br/>- svid.pem (identity cert)<br/>- svid_key.pem (private key)<br/>- bundle.pem (trusted issuers)
    App->>App: Reads the files and starts<br/>accepting secure connections
    
    Note over App,DB: Certificate auto-renews before it expires —<br/>the Helper repeats this process automatically
```

---

## 5. How Two Clusters Learn to Trust Each Other

> **What this shows**: Before apps can talk across clusters, each cluster needs the other's "list of trusted issuers." This is called **federation**. Think of it as two companies exchanging their official stamp samples so security guards at each company can verify badges from the other.

```mermaid
sequenceDiagram
    participant S1 as Identity Server<br/>Cluster 1
    participant EP1 as Cluster 1 Public URL<br/>(self-signed certificate)
    participant EP2 as Cluster 2 Public URL<br/>(Let's Encrypt certificate)
    participant S2 as Identity Server<br/>Cluster 2
    participant LE as Let's Encrypt<br/>(Public Certificate Authority)

    Note over S1,S2: ONE-TIME SETUP

    LE->>S2: Issues a publicly trusted certificate<br/>(via cert-manager automation)
    Note over EP1: Cluster 1 publishes its trust list<br/>Anyone can download it, but needs<br/>SPIRE verification to trust it
    Note over EP2: Cluster 2 publishes its trust list<br/>Anyone can download and trust it<br/>(backed by Let's Encrypt)

    Note over S1,S2: ONGOING — Automatic trust list refresh

    S1->>EP2: Downloads Cluster 2's trust list<br/>(trusts it because Let's Encrypt<br/>signed the connection)
    EP2-->>S1: Trust list with Cluster 2's<br/>certificate authority info

    S2->>EP1: Downloads Cluster 1's trust list<br/>(trusts it because SPIRE ID<br/>verification succeeds)
    EP1-->>S2: Trust list with Cluster 1's<br/>certificate authority info

    Note over S1,S2: Now both clusters know how to verify<br/>ID cards issued by the other cluster.<br/>Trust lists refresh automatically as<br/>certificate authorities rotate.
```

### Why Two Different Methods?

| | Cluster 1 (https_spiffe) | Cluster 2 (https_web) |
|--|--------------------------|----------------------|
| **Certificate type** | Self-signed by SPIRE | Signed by Let's Encrypt (publicly trusted) |
| **How others verify it** | Must already have SPIRE's certificate to trust it | Any system trusts it automatically (like HTTPS on websites) |
| **Setup effort** | None (built-in) | Requires cert-manager + Let's Encrypt |
| **Why use it** | Simple, works cluster-to-cluster | Demonstrates production-grade trust with public certificates |

---

## 6. The Secure Connection Test (mTLS Handshake)

> **What this shows**: The final test — a client app on Cluster 2 connects to a server app on Cluster 1. Both sides prove their identity to each other using their SPIFFE certificates. This is called **mutual TLS (mTLS)** because *both* sides authenticate, not just the server.

```mermaid
sequenceDiagram
    participant Client as Test Client App<br/>(on Cluster 2)
    participant Router as Cluster 1 Gateway<br/>(passes traffic through)
    participant Server as Test Server App<br/>(on Cluster 1)

    Note over Client,Server: Both apps already have their<br/>identity certificates from SPIRE

    Client->>Router: "I want to connect to<br/>the server on Cluster 1"
    Router->>Server: Forwards the connection<br/>(does not inspect TLS)

    Server-->>Client: "Here's my ID card"<br/>(signed by Cluster 1's identity system)
    Server->>Client: "Show me YOUR ID card too"

    Client-->>Server: "Here's my ID card"<br/>(signed by Cluster 2's identity system)

    Note over Server: Checks client's certificate against<br/>the combined trust list:<br/>"I trust Cluster 2's issuer — VALID"
    Note over Client: Checks server's certificate against<br/>the combined trust list:<br/>"I trust Cluster 1's issuer — VALID"

    Server-->>Client: "Identity verified — connection secure"
    Client->>Server: Encrypted data flows both ways

    Note over Client,Server: SUCCESS: Verify return code: 0 (ok)<br/>Both apps proved who they are
```

---

## 7. What Gets Stored in the Database

> **What this shows**: SPIRE stores all its data in Amazon RDS PostgreSQL instead of a local file. Here's what ends up in the database and why each piece matters.

| Table Name | What It Stores | Plain English | Example |
|-----------|---------------|---------------|---------|
| `bundles` | Certificate authority certificates | The "master stamps" used to sign ID cards — your own + any trusted partner's | 2 entries: Cluster 1's stamp + Cluster 2's stamp |
| `registered_entries` | Workload identity rules | The list of "who is allowed to get an ID card and what name goes on it" | `mtls-server` pod in the `federation-test` namespace |
| `attested_node_entries` | Verified machines | The list of machines (nodes) that have proven they belong to this cluster | 3 entries: one per worker node, verified via `k8s_psat` |
| `selectors` | Pod matching rules | How SPIRE decides which identity rule applies to which pod (by labels, namespace, service account) | "Any pod with label `spiffe.io/spiffe-id: true` in namespace `federation-test`" |
| `federated_registration_entries` | Cross-cluster permissions | Which workloads are allowed to talk to which other clusters | `mtls-server` is allowed to accept connections from Cluster 2 |
| `federated_trust_domains` | Partner cluster info | Where to download the trust list from each trusted partner cluster | Cluster 2's federation URL + its profile type (`https_web`) |
| `ca_journals` | Key rotation history | A log of when the signing keys were rotated (for security auditing) | Tracks every time SPIRE changed its CA signing key |
| `dns_names` | DNS names for identities | Additional DNS names associated with a workload's identity | DNS names attached to specific registered entries |
| `migrations` | Schema version number | Tracks which database version SPIRE is using (for upgrades) | Version `22` after the latest migration |

---

## 8. What Runs Where — Component Map

> **What this shows**: A bird's-eye view of all the pieces and which cluster or AWS service they run on.

```mermaid
flowchart LR
    subgraph cluster1 [Cluster 1]
        direction TB
        subgraph c1_core [Identity System]
            C1_mgr["Operator\n(manages everything)"]
            C1_srv["Identity Server\n(signs ID cards)"]
            C1_fed["Federation Endpoint\n(publishes trust list)\nSelf-signed certificate"]
        end
        subgraph c1_nodes [On Every Machine]
            C1_agent["Identity Agent\n(delivers ID cards\nto pods)"]
        end
        subgraph c1_test [Test App]
            C1_server["mTLS Server\n(accepts secure\nconnections)"]
        end
        C1_srv --> C1_fed
        C1_agent --> C1_server
    end

    subgraph cluster2 [Cluster 2]
        direction TB
        subgraph c2_core [Identity System]
            C2_mgr["Operator\n(manages everything)"]
            C2_srv["Identity Server\n(signs ID cards)"]
            C2_fed["Federation Endpoint\n(publishes trust list)\nLet's Encrypt certificate"]
        end
        subgraph c2_cert [Certificate Manager]
            C2_cm["cert-manager\n(auto-renews the\nLet's Encrypt cert)"]
        end
        subgraph c2_nodes [On Every Machine]
            C2_agent["Identity Agent\n(delivers ID cards\nto pods)"]
        end
        subgraph c2_test [Test App]
            C2_client["mTLS Client\n(initiates secure\nconnection)"]
        end
        C2_srv --> C2_fed
        C2_cm -->|"provides\ncertificate"| C2_fed
        C2_agent --> C2_client
    end

    subgraph aws [Amazon Web Services]
        RDS1[("Database 1\nStores Cluster 1\nidentities")]
        RDS2[("Database 2\nStores Cluster 2\nidentities")]
    end

    C1_srv --> RDS1
    C2_srv --> RDS2
    C1_fed <-->|"Exchange\ntrust lists"| C2_fed
    C2_client -->|"Secure connection\n(mutual TLS)"| C1_server
```

---

## Quick Reference — Jargon Decoder

| Term You'll See | What It Actually Means |
|----------------|----------------------|
| **SPIFFE** | A standard for giving apps their own identity (like a passport standard) |
| **SPIRE** | The software that implements SPIFFE (the passport-issuing office) |
| **SVID** | The actual identity certificate a pod receives (the passport itself) |
| **Trust Domain** | The identity namespace for a cluster (like a country that issues passports) |
| **Trust Bundle** | A list of certificate authorities you trust (list of countries whose passports you accept) |
| **Federation** | Two clusters agreeing to trust each other's identities |
| **mTLS** | Mutual TLS — both client and server prove their identity (both show passports) |
| **ZTWIM Operator** | Zero Trust Workload Identity Manager — the OpenShift operator that manages SPIRE |
| **cert-manager** | An operator that automatically gets and renews TLS certificates |
| **Let's Encrypt** | A free, public certificate authority (like a government that stamps documents for free) |
| **CRD** | Custom Resource Definition — a YAML configuration file you give to OpenShift |
| **StatefulSet** | A way to run exactly one server pod with persistent storage |
| **DaemonSet** | A way to run one copy of a pod on every machine in the cluster |
| **CSI Driver** | Container Storage Interface — securely mounts files/sockets into pods |
| **k8s_psat** | Kubernetes Projected Service Account Token — how SPIRE verifies a pod's identity with Kubernetes |
| **RDS** | Amazon Relational Database Service — a managed PostgreSQL database in the cloud |
| **VPC** | Virtual Private Cloud — an isolated network on AWS (like a private building) |
| **Security Group** | A firewall rule set on AWS that controls who can connect to what |
| **ACME** | Automatic Certificate Management Environment — the protocol Let's Encrypt uses |
| **https_spiffe** | Federation profile where the endpoint uses SPIRE's own self-signed certificate |
| **https_web** | Federation profile where the endpoint uses a publicly trusted certificate |

---

*Companion to: [SPIFFE Federation RDS PostgreSQL Guide](https://github.com/sayak-redhat/redhat-openshift-onboarding/blob/main/SPIFFE-Federation-RDS-PostgreSQL-Guide-4.19.md)*
