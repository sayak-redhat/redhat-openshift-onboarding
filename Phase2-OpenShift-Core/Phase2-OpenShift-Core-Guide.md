# 📚 Phase 2: OpenShift Core Study Guide
## From Kubernetes to Enterprise-Grade Platform

---

## Table of Contents

1. [Introduction: OpenShift vs Kubernetes](#introduction-openshift-vs-kubernetes)
2. [Module 1: Routes - Exposing Applications](#module-1-routes---exposing-applications)
3. [Module 2: Security Context Constraints (SCCs)](#module-2-security-context-constraints-sccs)
4. [Module 3: Projects and Multi-Tenancy](#module-3-projects-and-multi-tenancy)
5. [Module 4: Builds and ImageStreams](#module-4-builds-and-imagestreams)
6. [Module 5: OpenShift Networking Deep Dive](#module-5-openshift-networking-deep-dive)
7. [Module 6: OpenShift CLI (oc) Mastery](#module-6-openshift-cli-oc-mastery)
8. [Practice Labs](#practice-labs)
9. [Quiz and Review](#quiz-and-review)

---

## Introduction: OpenShift vs Kubernetes

### What is OpenShift?

**Simple Explanation**: OpenShift is Kubernetes with enterprise features added. It's like comparing a regular car (Kubernetes) to a luxury car (OpenShift) - same engine, but with added comfort, safety, and features.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      OpenShift = Kubernetes + Enterprise Features           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                     OpenShift Additions                              │  │
│   │                                                                      │  │
│   │  • Routes (better than Ingress)        • Built-in Monitoring        │  │
│   │  • SCCs (security controls)            • Logging (EFK stack)        │  │
│   │  • Projects (namespaces + RBAC)        • Developer Console          │  │
│   │  • BuildConfigs (S2I)                  • Operator Hub               │  │
│   │  • ImageStreams                        • Service Mesh               │  │
│   │  • OAuth integration                   • Registry                   │  │
│   │  • Web Console                         • Templates                  │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    │ Built on top of                        │
│                                    ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                         Kubernetes                                   │  │
│   │                                                                      │  │
│   │  Pods • Deployments • Services • ConfigMaps • Secrets               │  │
│   │  RBAC • Storage • Networking • Scheduling • API                     │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Side-by-Side Comparison

| Feature | Kubernetes | OpenShift |
|---------|------------|-----------|
| **External Access** | Ingress | Routes (more powerful) |
| **Security** | PodSecurityStandards | SCCs (more granular) |
| **Namespaces** | Plain namespaces | Projects (ns + RBAC) |
| **Builds** | External CI/CD | Built-in BuildConfigs |
| **Images** | External registry | Integrated registry + ImageStreams |
| **CLI** | kubectl | oc (kubectl + more) |
| **Console** | Basic dashboard | Full-featured console |
| **Auth** | Manual setup | Built-in OAuth |
| **Monitoring** | Manual setup | Built-in Prometheus |

---

# Module 1: Routes - Exposing Applications

## 1.1 What are Routes?

**Simple Explanation**: A Route exposes your application to the outside world with a URL. It's OpenShift's more powerful alternative to Kubernetes Ingress.

**Real-life Analogy**: If your pod is an office inside a building, a Route is like putting a sign outside the building that says "Entrance here" and directing visitors to the right floor.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          How Routes Work                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Internet User                                                              │
│        │                                                                     │
│        │ https://my-app.apps.cluster.example.com                            │
│        ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        OpenShift Router                              │   │
│   │                       (HAProxy pods)                                 │   │
│   │                                                                      │   │
│   │   Listens on ports 80/443 on all cluster nodes                      │   │
│   │   Routes requests based on hostname                                  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                     │
│        │ Looks at Route: "my-app.apps.cluster..." → Service "my-app"        │
│        ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         Service                                      │   │
│   │                    my-app:80 (ClusterIP)                            │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                     │
│        │ Load balances to pods                                              │
│        ▼                                                                     │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                     │
│   │    Pod 1    │    │    Pod 2    │    │    Pod 3    │                     │
│   └─────────────┘    └─────────────┘    └─────────────┘                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 1.2 TLS Termination Types

This is CRITICAL to understand - especially for your SPIRE federation work!

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       TLS Termination Types                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. EDGE TERMINATION                                                        │
│   ───────────────────                                                        │
│   TLS ends at the router. Traffic inside cluster is unencrypted.            │
│                                                                              │
│   Client ──HTTPS──► Router ──HTTP──► Service ──HTTP──► Pod                  │
│                       │                                                      │
│                       └─ Router has the TLS certificate                     │
│                                                                              │
│   Use when: You trust the internal network                                  │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   2. PASSTHROUGH TERMINATION                                                 │
│   ──────────────────────────                                                 │
│   TLS goes all the way to the pod. Router just forwards.                    │
│                                                                              │
│   Client ──HTTPS──► Router ──HTTPS──► Service ──HTTPS──► Pod                │
│                       │                          │                           │
│                       └─ Router doesn't          └─ Pod has the              │
│                          see traffic                TLS certificate          │
│                                                                              │
│   Use when: End-to-end encryption required, app manages its own TLS         │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   3. REENCRYPT TERMINATION  ⭐ (Used in SPIRE federation!)                  │
│   ────────────────────────                                                   │
│   TLS ends at router, then NEW TLS connection to pod.                       │
│                                                                              │
│   Client ──HTTPS──► Router ──HTTPS──► Service ──HTTPS──► Pod                │
│                       │         │                 │                          │
│                       │         └─────────────────┘                          │
│                       │         Different certificates!                      │
│                       │                                                      │
│                       └─ Router has external cert (Let's Encrypt)           │
│                          Pod has internal cert (service CA)                 │
│                                                                              │
│   Use when: Need different certs for external and internal                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 1.3 Creating Routes

### Basic Route (Edge)

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-app
  namespace: my-project
spec:
  host: my-app.apps.cluster.example.com    # Optional - auto-generated if omitted
  to:
    kind: Service
    name: my-app-service
  port:
    targetPort: 8080
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect  # HTTP → HTTPS redirect
```

### Passthrough Route

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-secure-app
spec:
  host: secure.apps.cluster.example.com
  to:
    kind: Service
    name: my-secure-service
  port:
    targetPort: 8443
  tls:
    termination: passthrough
```

### Reencrypt Route (What you used for SPIRE federation!)

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: spire-server-federation
  namespace: zero-trust-workload-identity-manager
spec:
  host: federation.apps.cluster.example.com
  to:
    kind: Service
    name: spire-server-federation
  port:
    targetPort: 8443
  tls:
    termination: reencrypt
    certificate: |
      -----BEGIN CERTIFICATE-----
      ... (Let's Encrypt cert - presented to external clients)
      -----END CERTIFICATE-----
    key: |
      -----BEGIN PRIVATE KEY-----
      ... (Let's Encrypt private key)
      -----END PRIVATE KEY-----
    destinationCACertificate: |
      -----BEGIN CERTIFICATE-----
      ... (Service CA - to verify the pod's cert)
      -----END CERTIFICATE-----
```

## 1.4 Hands-on: Route Commands

```bash
# List all routes
oc get routes

# Get route details
oc get route my-app -o yaml

# Create a simple route
oc expose service my-service

# Create route with specific hostname
oc expose service my-service --hostname=myapp.apps.cluster.example.com

# Create edge route with TLS
oc create route edge my-route --service=my-service

# Create passthrough route
oc create route passthrough my-route --service=my-service

# Create reencrypt route
oc create route reencrypt my-route --service=my-service \
  --cert=server.crt --key=server.key \
  --dest-ca-cert=service-ca.crt

# Debug route issues
oc describe route my-route
oc get events | grep -i route

# Test route externally
curl -v https://my-app.apps.cluster.example.com
```

---

# Module 2: Security Context Constraints (SCCs)

## 2.1 What are SCCs?

**Simple Explanation**: SCCs define what a pod is allowed to do security-wise. They control things like running as root, using host network, or accessing special devices.

**Real-life Analogy**: SCCs are like security clearance levels at a government building. Level 1 employees can only access the lobby. Level 5 can go anywhere including restricted areas.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Why SCCs Exist                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Problem: Containers can request dangerous capabilities                     │
│                                                                              │
│   Pod YAML says:                                                             │
│   ┌─────────────────────────────────────────┐                               │
│   │ securityContext:                         │                               │
│   │   runAsUser: 0          # Run as root!  │                               │
│   │   privileged: true      # Full access!  │                               │
│   │   hostNetwork: true     # Use host net! │                               │
│   └─────────────────────────────────────────┘                               │
│                                                                              │
│   Without SCCs: Kubernetes would allow this! 😱                             │
│   With SCCs: OpenShift says "DENIED - you don't have permission" ✅        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 2.2 SCC Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SCC Hierarchy (Most → Least Restrictive)                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  restricted-v2  (DEFAULT for all pods)                              │   │
│   │                                                                      │   │
│   │  • Cannot run as root                                               │   │
│   │  • Cannot use privileged mode                                       │   │
│   │  • Cannot use host network/PID/IPC                                  │   │
│   │  • Must drop all capabilities                                       │   │
│   │  • Read-only root filesystem                                        │   │
│   │                                                                      │   │
│   │  This is what most pods should use!                                 │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  nonroot                                                             │   │
│   │                                                                      │   │
│   │  • Cannot run as root (UID 0)                                       │   │
│   │  • Can use any other UID                                            │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  anyuid                                                              │   │
│   │                                                                      │   │
│   │  • Can run as any UID including root (0)                            │   │
│   │  • Used when app requires specific UID                              │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  hostnetwork                                                         │   │
│   │                                                                      │   │
│   │  • Can use host network namespace                                   │   │
│   │  • Pod gets node's IP address                                       │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  privileged  ⚠️  DANGER - Use only when absolutely necessary!       │   │
│   │                                                                      │   │
│   │  • Full access to everything                                        │   │
│   │  • Can access all host resources                                    │   │
│   │  • Can load kernel modules                                          │   │
│   │  • Essentially same access as host                                  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 2.3 How SCCs are Assigned

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      SCC Assignment Flow                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. Pod is created with ServiceAccount                                      │
│                              │                                               │
│                              ▼                                               │
│   2. OpenShift looks at ServiceAccount's SCCs                               │
│      (via RoleBindings/ClusterRoleBindings)                                 │
│                              │                                               │
│                              ▼                                               │
│   3. Tries each SCC in priority order                                       │
│      - Can pod requirements be satisfied?                                    │
│      - If yes, assign this SCC and continue                                 │
│      - If no, try next SCC                                                  │
│                              │                                               │
│                              ▼                                               │
│   4. If no SCC matches → Pod is REJECTED                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 2.4 Hands-on: Working with SCCs

```bash
# List all SCCs
oc get scc

# Describe an SCC (see what it allows)
oc describe scc restricted-v2
oc describe scc anyuid
oc describe scc privileged

# See which SCC your pod is using
oc get pod my-pod -o yaml | grep "openshift.io/scc"
# or
oc describe pod my-pod | grep "scc"

# See who can use an SCC
oc adm policy who-can use scc privileged

# Grant SCC to a ServiceAccount
oc adm policy add-scc-to-user anyuid -z my-service-account -n my-namespace
# -z = ServiceAccount (zervice account)
# -n = namespace

# Remove SCC from ServiceAccount
oc adm policy remove-scc-from-user anyuid -z my-service-account -n my-namespace

# Check if pod would be admitted
oc adm policy scc-subject-review -f pod.yaml
```

## 2.5 Common SCC Issues and Solutions

### Issue 1: Pod fails with "unable to validate against any SCC"

```bash
# Error message:
# "unable to validate against any security context constraint"

# Solution 1: Check what the pod needs
oc describe pod my-pod | grep -A 20 "Events:"

# Solution 2: Grant appropriate SCC
oc adm policy add-scc-to-user anyuid -z default -n my-namespace
```

### Issue 2: Container runs as wrong user

```yaml
# Your pod needs to run as specific user
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-sa  # Must have anyuid SCC
  securityContext:
    runAsUser: 1000          # Specific UID
  containers:
  - name: app
    image: my-app:latest
```

---

# Module 3: Projects and Multi-Tenancy

## 3.1 What are Projects?

**Simple Explanation**: A Project is a Kubernetes namespace with extra features - automatic RBAC, resource quotas, and network isolation.

**Real-life Analogy**: If a cluster is an office building, Projects are like different company offices on different floors. Each company has their own space, their own locks, and can't access other companies.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Project = Namespace + Extras                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      Project "team-alpha"                            │   │
│   │                                                                      │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │                    Namespace                                 │   │   │
│   │   │         (Pods, Services, ConfigMaps, etc.)                  │   │   │
│   │   └─────────────────────────────────────────────────────────────┘   │   │
│   │                              +                                       │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │                  Automatic RBAC                              │   │   │
│   │   │     Creator gets "admin" role automatically                  │   │   │
│   │   └─────────────────────────────────────────────────────────────┘   │   │
│   │                              +                                       │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │                  Network Policies                            │   │   │
│   │   │     Can isolate network from other projects                  │   │   │
│   │   └─────────────────────────────────────────────────────────────┘   │   │
│   │                              +                                       │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │                  Resource Quotas                             │   │   │
│   │   │     Limit CPU, memory, storage per project                   │   │   │
│   │   └─────────────────────────────────────────────────────────────┘   │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 3.2 Project Commands

```bash
# List projects
oc projects

# Create a project
oc new-project my-project --description="My project" --display-name="My Project"

# Switch to a project
oc project my-project

# See current project
oc project

# Delete a project (CAREFUL!)
oc delete project my-project

# See who has access to a project
oc describe rolebinding -n my-project

# Grant access to another user
oc adm policy add-role-to-user edit user@example.com -n my-project
oc adm policy add-role-to-user view user@example.com -n my-project
oc adm policy add-role-to-user admin user@example.com -n my-project

# Remove access
oc adm policy remove-role-from-user edit user@example.com -n my-project
```

## 3.3 Resource Quotas

```yaml
# Limit resources in a project
apiVersion: v1
kind: ResourceQuota
metadata:
  name: project-quota
  namespace: my-project
spec:
  hard:
    requests.cpu: "4"           # Total CPU requests
    requests.memory: "8Gi"      # Total memory requests
    limits.cpu: "8"             # Total CPU limits
    limits.memory: "16Gi"       # Total memory limits
    pods: "20"                  # Max pods
    persistentvolumeclaims: "10" # Max PVCs
```

```bash
# Check quota usage
oc get resourcequota -n my-project
oc describe resourcequota project-quota -n my-project
```

---

# Module 4: Builds and ImageStreams

## 4.1 What are BuildConfigs?

**Simple Explanation**: BuildConfig tells OpenShift how to build container images from your source code, directly in the cluster.

**Real-life Analogy**: BuildConfig is like a recipe card. It tells the kitchen (OpenShift) what ingredients (source code) to use and how to cook (build) them into a dish (container image).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Build Strategies                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. Source-to-Image (S2I)  - Most common                                   │
│   ─────────────────────────                                                  │
│   Source Code + Builder Image → Application Image                           │
│                                                                              │
│   ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐         │
│   │ Your Code    │ +  │ python:3.9       │ →  │ Your App Image   │         │
│   │ (app.py)     │    │ (builder image)  │    │ (ready to run)   │         │
│   └──────────────┘    └──────────────────┘    └──────────────────┘         │
│                                                                              │
│   2. Dockerfile Build                                                        │
│   ───────────────────                                                        │
│   Uses your Dockerfile to build the image                                   │
│                                                                              │
│   3. Pipeline Build                                                          │
│   ─────────────────                                                          │
│   Uses Jenkins/Tekton for CI/CD pipeline                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 4.2 S2I Example

```yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: my-python-app
spec:
  source:
    type: Git
    git:
      uri: https://github.com/myuser/my-app.git
      ref: main
  strategy:
    type: Source
    sourceStrategy:
      from:
        kind: ImageStreamTag
        namespace: openshift
        name: python:3.9
  output:
    to:
      kind: ImageStreamTag
      name: my-python-app:latest
  triggers:
    - type: GitHub              # Trigger build on GitHub webhook
    - type: ImageChange         # Trigger when base image changes
    - type: ConfigChange        # Trigger when BuildConfig changes
```

## 4.3 What are ImageStreams?

**Simple Explanation**: ImageStream is a pointer to container images. It tracks image versions and can trigger deployments when images change.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ImageStream Concept                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Without ImageStream:                                                       │
│   ────────────────────                                                       │
│   Deployment uses: docker.io/library/nginx:1.21                             │
│   - You must manually update tag when image changes                         │
│   - No automatic redeployment                                               │
│                                                                              │
│   With ImageStream:                                                          │
│   ──────────────────                                                         │
│   ImageStream: my-nginx                                                      │
│       │                                                                      │
│       ├── latest → sha256:abc123 (nginx:1.21)                               │
│       ├── stable → sha256:def456 (nginx:1.20)                               │
│       └── v1.21  → sha256:abc123                                            │
│                                                                              │
│   Deployment uses: my-nginx:latest                                          │
│   - When image changes, ImageStream updates                                 │
│   - Deployment automatically triggered                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

```yaml
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: my-nginx
spec:
  lookupPolicy:
    local: true
  tags:
    - name: latest
      from:
        kind: DockerImage
        name: docker.io/library/nginx:latest
      importPolicy:
        scheduled: true    # Auto-import new versions
```

## 4.4 Build Commands

```bash
# Start a build
oc start-build my-app

# Start build and follow logs
oc start-build my-app --follow

# See build history
oc get builds

# See build logs
oc logs build/my-app-1

# Create app from Git (auto-creates BuildConfig + ImageStream + Deployment)
oc new-app https://github.com/myuser/my-app.git

# Create app with specific builder
oc new-app python:3.9~https://github.com/myuser/my-app.git

# Import image into ImageStream
oc import-image my-image --from=docker.io/library/nginx:latest --confirm

# See ImageStream tags
oc get imagestream my-image -o yaml
```

---

# Module 5: OpenShift Networking Deep Dive

## 5.1 OpenShift Network Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OpenShift Network Architecture                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                     External Traffic                                 │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    Load Balancer                                     │   │
│   │              (AWS ELB / GCP LB / MetalLB)                           │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                     Ingress Router                                   │   │
│   │                    (HAProxy Pods)                                    │   │
│   │                                                                      │   │
│   │   Routes requests based on hostname to Services                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    OVN-Kubernetes CNI                                │   │
│   │                                                                      │   │
│   │   • Pod-to-Pod networking                                           │   │
│   │   • Service load balancing                                          │   │
│   │   • Network policies                                                │   │
│   │   • Egress IPs                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│         ┌────────────────────┼────────────────────┐                         │
│         ▼                    ▼                    ▼                         │
│   ┌───────────┐        ┌───────────┐        ┌───────────┐                   │
│   │  Node 1   │        │  Node 2   │        │  Node 3   │                   │
│   │           │        │           │        │           │                   │
│   │ Pod A     │        │ Pod C     │        │ Pod E     │                   │
│   │ Pod B     │        │ Pod D     │        │ Pod F     │                   │
│   └───────────┘        └───────────┘        └───────────┘                   │
│                                                                              │
│   All pods can communicate across nodes via overlay network                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 5.2 DNS in OpenShift

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DNS Resolution                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Service DNS Format:                                                        │
│   ───────────────────                                                        │
│   <service-name>.<namespace>.svc.cluster.local                              │
│                                                                              │
│   Examples:                                                                  │
│   ─────────                                                                  │
│   my-service.my-project.svc.cluster.local                                   │
│   postgres.database.svc.cluster.local                                       │
│   spire-server.zero-trust-workload-identity-manager.svc.cluster.local       │
│                                                                              │
│   Short forms (within same namespace):                                       │
│   ─────────────────────────────────────                                      │
│   my-service                           ← Same namespace                     │
│   my-service.my-project                ← Different namespace                │
│   my-service.my-project.svc            ← Explicit                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 5.3 Network Policies

```yaml
# Allow only specific pods to access this pod
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: my-project
spec:
  podSelector:
    matchLabels:
      app: backend           # Apply to pods with label app=backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend  # Only allow from pods with label app=frontend
      ports:
        - protocol: TCP
          port: 8080
```

## 5.4 Network Debugging Commands

```bash
# Check DNS resolution
oc exec -it my-pod -- nslookup kubernetes.default.svc.cluster.local

# Check service endpoints
oc get endpoints my-service

# Test connectivity between pods
oc exec -it pod-a -- curl http://my-service:8080

# Debug node networking
oc debug node/node-1
# Then: chroot /host
# Then: ip addr, ss -tulpn, tcpdump, etc.

# Check NetworkPolicies
oc get networkpolicy -n my-project
oc describe networkpolicy my-policy

# Check route backend
oc get route my-route -o jsonpath='{.spec.to.name}'
```

---

# Module 6: OpenShift CLI (oc) Mastery

## 6.1 Essential oc Commands

```bash
# ═══════════════════════════════════════════════════════════════════════════
# AUTHENTICATION & CONTEXT
# ═══════════════════════════════════════════════════════════════════════════

# Login to cluster
oc login https://api.cluster.example.com:6443 -u admin -p password

# Login with token
oc login --token=sha256~xxxxxx --server=https://api.cluster.example.com:6443

# See current user
oc whoami

# See current context
oc whoami --show-context

# See current server
oc whoami --show-server

# ═══════════════════════════════════════════════════════════════════════════
# PROJECT MANAGEMENT
# ═══════════════════════════════════════════════════════════════════════════

# List projects
oc projects

# Switch project
oc project my-project

# Create project
oc new-project my-project

# Delete project
oc delete project my-project

# ═══════════════════════════════════════════════════════════════════════════
# RESOURCE MANAGEMENT
# ═══════════════════════════════════════════════════════════════════════════

# Get resources
oc get pods
oc get pods -o wide
oc get pods -o yaml
oc get pods -o json
oc get all

# Describe resources
oc describe pod my-pod
oc describe svc my-service

# Create/update from YAML
oc apply -f manifest.yaml
oc create -f manifest.yaml

# Delete resources
oc delete -f manifest.yaml
oc delete pod my-pod

# Edit resources
oc edit deployment my-deployment

# ═══════════════════════════════════════════════════════════════════════════
# DEBUGGING
# ═══════════════════════════════════════════════════════════════════════════

# View logs
oc logs my-pod
oc logs my-pod -c container-name    # Specific container
oc logs my-pod -f                    # Follow
oc logs my-pod --previous            # Previous instance

# Execute commands in pod
oc exec my-pod -- ls /
oc exec -it my-pod -- /bin/bash      # Interactive shell

# Port forwarding
oc port-forward my-pod 8080:80
oc port-forward svc/my-service 8080:80

# Debug node
oc debug node/my-node

# Get events
oc get events --sort-by='.lastTimestamp'

# ═══════════════════════════════════════════════════════════════════════════
# APPLICATION DEPLOYMENT
# ═══════════════════════════════════════════════════════════════════════════

# Create app from image
oc new-app nginx:latest

# Create app from Git
oc new-app https://github.com/user/repo.git

# Create app with builder
oc new-app python:3.9~https://github.com/user/repo.git

# Expose service as route
oc expose service my-service

# Scale deployment
oc scale deployment my-deployment --replicas=3

# Rollout status
oc rollout status deployment my-deployment

# Rollback
oc rollout undo deployment my-deployment

# ═══════════════════════════════════════════════════════════════════════════
# SECURITY & RBAC
# ═══════════════════════════════════════════════════════════════════════════

# Grant SCC
oc adm policy add-scc-to-user anyuid -z my-sa -n my-project

# Grant role
oc adm policy add-role-to-user edit user@example.com -n my-project

# Check permissions
oc auth can-i create pods
oc auth can-i --list

# Who can do something
oc adm policy who-can create pods
```

---

# Practice Labs

## Lab 1: Deploy Application with Route

```bash
# Create project
oc new-project lab-routes

# Deploy nginx
oc new-app nginx:latest --name=web

# Expose with route
oc expose service web

# Get route URL
oc get route web -o jsonpath='{.spec.host}'

# Test
curl http://$(oc get route web -o jsonpath='{.spec.host}')

# Create edge route with TLS
oc delete route web
oc create route edge web --service=web

# Test HTTPS
curl -k https://$(oc get route web -o jsonpath='{.spec.host}')

# Cleanup
oc delete project lab-routes
```

## Lab 2: Working with SCCs

```bash
# Create project
oc new-project lab-scc

# Create a pod that needs to run as specific user
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: test
    image: nginx:latest
    securityContext:
      runAsUser: 1000
EOF

# Check if pod is running
oc get pods

# Check which SCC it's using
oc get pod test-pod -o yaml | grep scc

# Now try to create privileged pod (will fail)
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: priv-pod
spec:
  containers:
  - name: test
    image: nginx:latest
    securityContext:
      privileged: true
EOF

# See the error
oc describe pod priv-pod

# Cleanup
oc delete project lab-scc
```

## Lab 3: Build from Source

```bash
# Create project
oc new-project lab-build

# Create app from GitHub (uses S2I)
oc new-app python:3.9~https://github.com/sclorg/django-ex.git

# Watch build
oc logs -f buildconfig/django-ex

# Expose
oc expose service django-ex

# Get URL
oc get route

# Cleanup
oc delete project lab-build
```

---

# Quiz and Review

## Quiz Questions

1. **What is the difference between edge, passthrough, and reencrypt routes?**

2. **What SCC is assigned to pods by default?**

3. **How do you grant the 'anyuid' SCC to a service account?**

4. **What is an ImageStream?**

5. **What DNS name would you use to reach service 'db' in namespace 'prod'?**

6. **What command shows which SCC a pod is using?**

## Answers

1. **Edge**: TLS terminates at router, unencrypted to pod.
   **Passthrough**: TLS goes all the way to pod.
   **Reencrypt**: TLS terminates at router, NEW TLS to pod.

2. **restricted-v2** (or restricted on older versions)

3. `oc adm policy add-scc-to-user anyuid -z my-sa -n my-namespace`

4. A pointer to container images that tracks versions and can trigger deployments

5. `db.prod.svc.cluster.local` or just `db.prod` from another namespace

6. `oc get pod <name> -o yaml | grep scc` or `oc describe pod <name> | grep scc`

---

## What's Next?

After completing Phase 2, you should:

1. ✅ Understand Routes and TLS termination types
2. ✅ Know how SCCs work and how to grant them
3. ✅ Be comfortable with Projects and multi-tenancy
4. ✅ Understand BuildConfigs and ImageStreams
5. ✅ Master the `oc` CLI

**Next Step**: Phase 3 - Operator Framework

---

*Phase 2 Complete! You now understand OpenShift enterprise features.*
