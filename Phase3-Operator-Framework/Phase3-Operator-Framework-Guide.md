# 📚 Phase 3: Operator Framework Study Guide
## Building and Understanding Kubernetes Operators

---

## Table of Contents

1. [Introduction: What are Operators?](#introduction-what-are-operators)
2. [Module 1: Custom Resource Definitions (CRDs)](#module-1-custom-resource-definitions-crds)
3. [Module 2: The Reconciliation Loop](#module-2-the-reconciliation-loop)
4. [Module 3: Building Your First Operator](#module-3-building-your-first-operator)
5. [Module 4: Operator Lifecycle Manager (OLM)](#module-4-operator-lifecycle-manager-olm)
6. [Module 5: Real-World Operator Examples](#module-5-real-world-operator-examples)
7. [Practice Labs](#practice-labs)
8. [Quiz and Review](#quiz-and-review)

---

## Introduction: What are Operators?

### The Problem Operators Solve

**Simple Explanation**: An Operator is code that automates the work a human expert would do to manage a complex application.

**Real-life Analogy**: 

Think about managing a database like PostgreSQL:
- A human DBA (Database Administrator) knows how to:
  - Install PostgreSQL
  - Configure replication
  - Handle backups
  - Perform upgrades
  - Recover from failures

An Operator is like encoding that DBA's knowledge into code that runs 24/7.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        The Operator Pattern                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   BEFORE OPERATORS                     AFTER OPERATORS                       │
│   ───────────────                      ──────────────                        │
│                                                                              │
│   Human Expert                         Operator (Code)                       │
│   ┌─────────────┐                      ┌─────────────┐                       │
│   │             │                      │             │                       │
│   │  "Install   │                      │  Watches    │                       │
│   │   database" │                      │  for CRs    │                       │
│   │             │                      │             │                       │
│   │  "Scale     │     ═══════►        │  Creates    │                       │
│   │   replicas" │     Encoded as      │  resources  │                       │
│   │             │                      │             │                       │
│   │  "Backup    │                      │  Handles    │                       │
│   │   data"     │                      │  failures   │                       │
│   │             │                      │             │                       │
│   │  "Recover   │                      │  Runs 24/7  │                       │
│   │   failure"  │                      │  automated  │                       │
│   └─────────────┘                      └─────────────┘                       │
│                                                                              │
│   Works 9-5                            Works always                          │
│   Might be sick                        Never sick                            │
│   Manual steps                         Automated                             │
│   Slow reaction                        Instant reaction                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Operator Equation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│        Operator = Custom Resource (CR) + Custom Controller                   │
│                                                                              │
│   ┌────────────────────┐         ┌────────────────────────────────────┐     │
│   │  Custom Resource   │         │      Custom Controller             │     │
│   │                    │         │                                    │     │
│   │  "What you want"   │  ─────► │  "Code that makes it happen"      │     │
│   │                    │         │                                    │     │
│   │  apiVersion: ...   │         │  Watches CRs                       │     │
│   │  kind: PostgreSQL  │         │  Creates Pods, Services, etc.     │     │
│   │  spec:             │         │  Handles updates                   │     │
│   │    replicas: 3     │         │  Manages lifecycle                 │     │
│   │    storage: 100Gi  │         │                                    │     │
│   └────────────────────┘         └────────────────────────────────────┘     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Example: Zero Trust Workload Identity Manager

You've been using operators! Here's how they work:

```yaml
# This is a Custom Resource (CR) - you create this
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  federation:
    bundleEndpoint:
      profile: https_web
    managedRoute: "true"
```

When you apply this:

1. **Operator sees the CR** → "Someone wants a SPIRE Server"
2. **Operator creates resources**:
   - StatefulSet for spire-server pod
   - Service for network access
   - Route for external access (because `managedRoute: true`)
   - ConfigMaps for configuration
   - Secrets for certificates
3. **Operator monitors** → If anything changes or breaks, it fixes it

---

# Module 1: Custom Resource Definitions (CRDs)

## 1.1 What is a CRD?

**Simple Explanation**: A CRD (Custom Resource Definition) extends the Kubernetes API with new resource types. It's like adding a new "kind" to Kubernetes.

**Real-life Analogy**: Kubernetes by default knows about Pods, Services, Deployments. A CRD is like teaching Kubernetes a new word: "PostgreSQL" or "SpireServer".

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       CRD Creates New API Types                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Built-in Kubernetes Resources:                                            │
│   ─────────────────────────────                                              │
│   kubectl get pods                     ← Kubernetes knows this              │
│   kubectl get services                 ← Kubernetes knows this              │
│   kubectl get deployments              ← Kubernetes knows this              │
│                                                                              │
│   After Installing CRD:                                                      │
│   ────────────────────                                                       │
│   kubectl get spireservers             ← NEW! Kubernetes learned this       │
│   kubectl get postgresqls              ← NEW! Kubernetes learned this       │
│   kubectl get certificates             ← NEW! Kubernetes learned this       │
│                                                                              │
│   The CRD defines:                                                           │
│   • What fields are allowed (spec schema)                                   │
│   • What the resource is called (names)                                     │
│   • How it's versioned (v1alpha1, v1beta1, v1)                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 1.2 CRD Structure

```yaml
# This is a CRD - it DEFINES what "SpireServer" looks like
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: spireservers.operator.openshift.io    # plural.group
spec:
  group: operator.openshift.io                 # API group
  names:
    kind: SpireServer                          # What you use in YAML
    plural: spireservers                       # kubectl get spireservers
    singular: spireserver                      # kubectl get spireserver
    shortNames:
      - ss                                     # kubectl get ss
  scope: Cluster                               # Cluster-wide or Namespaced
  versions:
    - name: v1alpha1
      served: true                             # API serves this version
      storage: true                            # Store in this version
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                caSubject:                     # Field definition
                  type: object
                  properties:
                    commonName:
                      type: string
                federation:
                  type: object
                  properties:
                    managedRoute:
                      type: string
                    bundleEndpoint:
                      type: object
                      properties:
                        profile:
                          type: string
                          enum: ["https_web", "https_spiffe"]
```

## 1.3 CR vs CRD

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CRD vs CR - The Difference                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   CRD (Custom Resource Definition)      CR (Custom Resource)                │
│   ────────────────────────────────      ────────────────────                │
│                                                                              │
│   • Defines the SCHEMA                  • An INSTANCE of the schema         │
│   • Created once by operator dev        • Created by users                  │
│   • Like a "class" in programming       • Like an "object" in programming   │
│   • Defines what fields exist           • Has actual values                 │
│                                                                              │
│   Analogy:                                                                   │
│   ─────────                                                                  │
│   CRD = The blueprint for a house       CR = An actual house built          │
│   CRD = Recipe template                 CR = Actual dish you're cooking     │
│   CRD = Form design                     CR = Filled-out form                │
│                                                                              │
│   ┌──────────────────────────────┐     ┌──────────────────────────────┐     │
│   │ CRD: SpireServer definition  │ ──► │ CR: SpireServer "cluster"    │     │
│   │                              │     │                              │     │
│   │ kind: CRD                    │     │ kind: SpireServer            │     │
│   │ name: spireservers...       │     │ name: cluster                │     │
│   │ spec:                        │     │ spec:                        │     │
│   │   schema:                    │     │   federation:                │     │
│   │     federation:              │     │     managedRoute: "true"     │     │
│   │       type: object           │     │     bundleEndpoint:          │     │
│   │       properties:            │     │       profile: https_web     │     │
│   │         managedRoute:        │     │                              │     │
│   │           type: string       │     │                              │     │
│   └──────────────────────────────┘     └──────────────────────────────┘     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 1.4 Hands-on: Working with CRDs

```bash
# List all CRDs in cluster
oc get crd

# List CRDs from a specific operator
oc get crd | grep spire
oc get crd | grep cert-manager

# Describe a CRD (see schema)
oc describe crd spireservers.operator.openshift.io

# Get specific version schema
oc get crd spireservers.operator.openshift.io -o yaml

# List Custom Resources of a type
oc get spireservers
oc get spireserver cluster -o yaml

# Use explain to understand CR fields
oc explain spireserver
oc explain spireserver.spec
oc explain spireserver.spec.federation
```

---

# Module 2: The Reconciliation Loop

## 2.1 What is Reconciliation?

**Simple Explanation**: Reconciliation is the core of how operators work. The controller constantly compares "what you want" (desired state) with "what exists" (actual state) and makes changes to match them.

**Real-life Analogy**: Think of a thermostat. You set it to 72°F (desired state). The thermostat constantly checks the temperature (actual state) and turns heating/cooling on/off to match your setting. That's reconciliation!

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       The Reconciliation Loop                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              ┌──────────────────┐                            │
│                              │                  │                            │
│                              │  DESIRED STATE   │                            │
│                              │  (from your CR)  │                            │
│                              │                  │                            │
│                              │  replicas: 3     │                            │
│                              │  image: v2       │                            │
│                              │                  │                            │
│                              └────────┬─────────┘                            │
│                                       │                                      │
│                                       │ Compare                              │
│                                       ▼                                      │
│   ┌────────────────────────────────────────────────────────────────────┐    │
│   │                         RECONCILER                                  │    │
│   │                                                                     │    │
│   │   if (desired != actual) {                                         │    │
│   │       make changes to reach desired state                          │    │
│   │   }                                                                 │    │
│   │                                                                     │    │
│   └────────────────────────────────────────────────────────────────────┘    │
│                                       │                                      │
│                                       │ Make Changes                         │
│                                       ▼                                      │
│                              ┌──────────────────┐                            │
│                              │                  │                            │
│                              │  ACTUAL STATE    │                            │
│                              │  (in cluster)    │                            │
│                              │                  │                            │
│                              │  replicas: 2     │ ← Different! Fix it!      │
│                              │  image: v1       │                            │
│                              │                  │                            │
│                              └──────────────────┘                            │
│                                                                              │
│   The loop runs continuously:                                               │
│   1. Watch for changes (CR created/updated/deleted)                         │
│   2. Compare desired vs actual                                              │
│   3. Take action to converge                                                │
│   4. Update status                                                          │
│   5. Repeat                                                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 2.2 Reconciliation in Code

```go
// Simplified reconciliation function
func (r *SpireServerReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    
    // Step 1: Get the CR (desired state)
    spireServer := &v1alpha1.SpireServer{}
    err := r.Get(ctx, req.NamespacedName, spireServer)
    if err != nil {
        return ctrl.Result{}, err
    }
    
    // Step 2: Check actual state - Does the StatefulSet exist?
    statefulset := &appsv1.StatefulSet{}
    err = r.Get(ctx, types.NamespacedName{
        Name:      "spire-server",
        Namespace: spireServer.Namespace,
    }, statefulset)
    
    if errors.IsNotFound(err) {
        // Step 3: Actual doesn't exist - CREATE it
        newStatefulSet := r.buildStatefulSet(spireServer)
        err = r.Create(ctx, newStatefulSet)
        return ctrl.Result{}, err
    }
    
    // Step 4: Actual exists - Compare and UPDATE if needed
    if spireServer.Spec.Replicas != *statefulset.Spec.Replicas {
        statefulset.Spec.Replicas = &spireServer.Spec.Replicas
        err = r.Update(ctx, statefulset)
        return ctrl.Result{}, err
    }
    
    // Step 5: Check if Route needed (managedRoute: true)
    if spireServer.Spec.Federation.ManagedRoute == "true" {
        err = r.ensureRouteExists(ctx, spireServer)
        if err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // Everything in sync - requeue after some time to re-check
    return ctrl.Result{RequeueAfter: time.Minute * 5}, nil
}
```

## 2.3 What Triggers Reconciliation?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Reconciliation Triggers                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. CR Created                                                              │
│      User runs: oc apply -f spireserver.yaml                                │
│      → Reconciler: "New SpireServer! Let me create everything!"             │
│                                                                              │
│   2. CR Updated                                                              │
│      User changes: replicas: 1 → replicas: 3                                │
│      → Reconciler: "SpireServer changed! Let me update StatefulSet!"        │
│                                                                              │
│   3. CR Deleted                                                              │
│      User runs: oc delete spireserver cluster                               │
│      → Reconciler: "SpireServer gone! Let me clean up everything!"          │
│                                                                              │
│   4. Owned Resource Changed                                                  │
│      Something deletes the Route that operator created                       │
│      → Reconciler: "Hey! Route is gone! Let me recreate it!"                │
│                                                                              │
│   5. Periodic Requeue                                                        │
│      Operator says: RequeueAfter: 5 minutes                                 │
│      → Reconciler: "Time to check if everything is still good!"             │
│                                                                              │
│   6. External Event                                                          │
│      Secret containing TLS cert is updated by cert-manager                  │
│      → Reconciler: "New cert! Let me update the Route!"                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 2.4 Owner References

**Simple Explanation**: Owner references link resources created by an operator to the CR that caused them. When the CR is deleted, all owned resources are automatically cleaned up.

```yaml
# StatefulSet created by SpireServer operator
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: spire-server
  ownerReferences:                              # ← This links to the CR
    - apiVersion: operator.openshift.io/v1alpha1
      kind: SpireServer
      name: cluster
      uid: abc-123-def
      controller: true
      blockOwnerDeletion: true

# When SpireServer CR is deleted, this StatefulSet is automatically deleted too!
```

---

# Module 3: Building Your First Operator

## 3.1 Operator SDK Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Operator SDK                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   The Operator SDK helps you build operators in three ways:                 │
│                                                                              │
│   1. Go-based Operators (Most powerful)                                     │
│      ────────────────────────────────                                        │
│      • Full programming control                                              │
│      • Best performance                                                      │
│      • Complex logic possible                                                │
│      • Used by: SPIRE Operator, cert-manager                                │
│                                                                              │
│   2. Ansible-based Operators                                                 │
│      ─────────────────────────────                                           │
│      • Write Ansible playbooks                                               │
│      • Good for ops teams                                                    │
│      • Easier if you know Ansible                                           │
│                                                                              │
│   3. Helm-based Operators                                                    │
│      ────────────────────────────                                            │
│      • Wrap existing Helm charts                                             │
│      • Simplest approach                                                     │
│      • Limited customization                                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 3.2 Creating a Go-based Operator

### Step 1: Install Prerequisites

```bash
# Install Operator SDK
brew install operator-sdk   # macOS
# Or download from: https://sdk.operatorframework.io/docs/installation/

# Verify installation
operator-sdk version
```

### Step 2: Initialize Project

```bash
# Create project directory
mkdir my-operator && cd my-operator

# Initialize operator
operator-sdk init \
  --domain=example.com \
  --repo=github.com/myuser/my-operator

# This creates:
# my-operator/
# ├── Dockerfile           # Build container image
# ├── Makefile            # Build commands
# ├── PROJECT             # Project metadata
# ├── go.mod              # Go dependencies
# ├── go.sum
# ├── main.go             # Entry point
# └── config/
#     ├── default/        # Kustomize base
#     ├── manager/        # Operator deployment
#     ├── manifests/      # OLM manifests
#     └── rbac/           # RBAC permissions
```

### Step 3: Create an API (CRD + Controller)

```bash
# Create API for our custom resource
operator-sdk create api \
  --group=myapp \
  --version=v1alpha1 \
  --kind=MyDatabase \
  --resource \
  --controller

# This creates:
# ├── api/v1alpha1/
# │   ├── mydatabase_types.go    # Define your CRD spec here
# │   └── groupversion_info.go
# └── controllers/
#     └── mydatabase_controller.go  # Write reconciliation logic here
```

### Step 4: Define Your CRD (types.go)

```go
// api/v1alpha1/mydatabase_types.go

package v1alpha1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// MyDatabaseSpec defines the desired state of MyDatabase
type MyDatabaseSpec struct {
    // Replicas is the number of database instances
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=5
    Replicas int32 `json:"replicas"`
    
    // StorageSize is the size of storage per instance
    // +kubebuilder:validation:Pattern=^\d+Gi$
    StorageSize string `json:"storageSize"`
    
    // Version is the database version
    // +kubebuilder:default="14"
    Version string `json:"version,omitempty"`
}

// MyDatabaseStatus defines the observed state of MyDatabase
type MyDatabaseStatus struct {
    // ReadyReplicas is the number of ready instances
    ReadyReplicas int32 `json:"readyReplicas"`
    
    // Conditions represent the current state
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Replicas",type="integer",JSONPath=".spec.replicas"
// +kubebuilder:printcolumn:name="Ready",type="integer",JSONPath=".status.readyReplicas"
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"

// MyDatabase is the Schema for the mydatabases API
type MyDatabase struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   MyDatabaseSpec   `json:"spec,omitempty"`
    Status MyDatabaseStatus `json:"status,omitempty"`
}
```

### Step 5: Implement the Controller

```go
// controllers/mydatabase_controller.go

package controllers

import (
    "context"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"
    
    myappv1alpha1 "github.com/myuser/my-operator/api/v1alpha1"
)

type MyDatabaseReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=myapp.example.com,resources=mydatabases,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=myapp.example.com,resources=mydatabases/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups="",resources=services,verbs=get;list;watch;create;update;patch;delete

func (r *MyDatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)
    log.Info("Reconciling MyDatabase", "name", req.Name)

    // Step 1: Fetch the MyDatabase CR
    mydb := &myappv1alpha1.MyDatabase{}
    err := r.Get(ctx, req.NamespacedName, mydb)
    if err != nil {
        if errors.IsNotFound(err) {
            log.Info("MyDatabase not found, must be deleted")
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }

    // Step 2: Create or Update StatefulSet
    statefulset := r.statefulSetForMyDatabase(mydb)
    
    found := &appsv1.StatefulSet{}
    err = r.Get(ctx, client.ObjectKey{Name: statefulset.Name, Namespace: statefulset.Namespace}, found)
    
    if err != nil && errors.IsNotFound(err) {
        log.Info("Creating StatefulSet", "name", statefulset.Name)
        err = r.Create(ctx, statefulset)
        if err != nil {
            return ctrl.Result{}, err
        }
    } else if err == nil {
        // Update if replicas changed
        if *found.Spec.Replicas != mydb.Spec.Replicas {
            found.Spec.Replicas = &mydb.Spec.Replicas
            err = r.Update(ctx, found)
            if err != nil {
                return ctrl.Result{}, err
            }
        }
    }

    // Step 3: Create Service
    service := r.serviceForMyDatabase(mydb)
    err = r.Get(ctx, client.ObjectKey{Name: service.Name, Namespace: service.Namespace}, &corev1.Service{})
    if err != nil && errors.IsNotFound(err) {
        log.Info("Creating Service", "name", service.Name)
        err = r.Create(ctx, service)
        if err != nil {
            return ctrl.Result{}, err
        }
    }

    // Step 4: Update Status
    mydb.Status.ReadyReplicas = found.Status.ReadyReplicas
    err = r.Status().Update(ctx, mydb)

    return ctrl.Result{}, err
}

// Helper to create StatefulSet
func (r *MyDatabaseReconciler) statefulSetForMyDatabase(m *myappv1alpha1.MyDatabase) *appsv1.StatefulSet {
    labels := map[string]string{"app": m.Name}
    replicas := m.Spec.Replicas

    ss := &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      m.Name + "-db",
            Namespace: m.Namespace,
            OwnerReferences: []metav1.OwnerReference{
                *metav1.NewControllerRef(m, myappv1alpha1.GroupVersion.WithKind("MyDatabase")),
            },
        },
        Spec: appsv1.StatefulSetSpec{
            Replicas: &replicas,
            Selector: &metav1.LabelSelector{
                MatchLabels: labels,
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: labels,
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{{
                        Name:  "postgres",
                        Image: "postgres:" + m.Spec.Version,
                        Ports: []corev1.ContainerPort{{
                            ContainerPort: 5432,
                        }},
                    }},
                },
            },
        },
    }
    return ss
}

func (r *MyDatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&myappv1alpha1.MyDatabase{}).        // Watch MyDatabase CRs
        Owns(&appsv1.StatefulSet{}).             // Watch owned StatefulSets
        Owns(&corev1.Service{}).                 // Watch owned Services
        Complete(r)
}
```

### Step 6: Build and Deploy

```bash
# Generate CRD manifests
make manifests

# Install CRD
make install

# Run locally (for development)
make run

# Build image and push
make docker-build docker-push IMG=quay.io/myuser/my-operator:v0.1.0

# Deploy to cluster
make deploy IMG=quay.io/myuser/my-operator:v0.1.0
```

### Step 7: Test Your Operator

```yaml
# config/samples/myapp_v1alpha1_mydatabase.yaml
apiVersion: myapp.example.com/v1alpha1
kind: MyDatabase
metadata:
  name: test-db
spec:
  replicas: 3
  storageSize: "10Gi"
  version: "14"
```

```bash
# Apply the CR
oc apply -f config/samples/myapp_v1alpha1_mydatabase.yaml

# Watch the operator create resources
oc get mydatabase
oc get statefulset
oc get pods

# Check operator logs
oc logs -f deployment/my-operator-controller-manager -n my-operator-system
```

---

# Module 4: Operator Lifecycle Manager (OLM)

## 4.1 What is OLM?

**Simple Explanation**: OLM manages the installation, upgrade, and lifecycle of operators in a cluster. It's like an "App Store" for operators.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        OLM Architecture                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      CatalogSource                                   │   │
│   │         (Index of available operators - like App Store)             │   │
│   │                                                                      │   │
│   │   Examples:                                                          │   │
│   │   • redhat-operators      - Red Hat certified                       │   │
│   │   • certified-operators   - Partner certified                       │   │
│   │   • community-operators   - Community contributed                   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              │ Contains operator bundles                     │
│                              ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                       Subscription                                   │   │
│   │        "I want cert-manager from stable-v1 channel"                 │   │
│   │                                                                      │   │
│   │   apiVersion: operators.coreos.com/v1alpha1                         │   │
│   │   kind: Subscription                                                 │   │
│   │   spec:                                                              │   │
│   │     name: cert-manager                                               │   │
│   │     channel: stable-v1                                               │   │
│   │     source: certified-operators                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              │ Creates                                       │
│                              ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                       InstallPlan                                    │   │
│   │        "Here's what will be installed"                              │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              │ Deploys                                       │
│                              ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                 ClusterServiceVersion (CSV)                          │   │
│   │        "Operator v1.2.3 with these CRDs and RBAC"                   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              │ Creates                                       │
│                              ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    Operator Deployment                               │   │
│   │        (The actual operator pod running)                            │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 4.2 Installing Operators via OLM

```bash
# List available CatalogSources
oc get catalogsource -n openshift-marketplace

# Search for operators
oc get packagemanifest | grep cert-manager
oc get packagemanifest | grep spire

# See operator details
oc describe packagemanifest cert-manager-operator -n openshift-marketplace

# Install an operator
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cert-manager-operator
  namespace: cert-manager-operator
spec:
  upgradeStrategy: Default
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cert-manager-operator
  namespace: cert-manager-operator
spec:
  channel: stable-v1
  name: openshift-cert-manager-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Check installation
oc get subscription -n cert-manager-operator
oc get csv -n cert-manager-operator
oc get installplan -n cert-manager-operator
```

## 4.3 OLM Commands

```bash
# See all installed operators
oc get csv -A

# See operator status
oc describe csv cert-manager.v1.14.0 -n cert-manager-operator

# See what CRDs an operator provides
oc get crd | grep cert-manager

# Uninstall operator
oc delete subscription cert-manager-operator -n cert-manager-operator
oc delete csv cert-manager.v1.14.0 -n cert-manager-operator

# Approve InstallPlan manually (if not automatic)
oc patch installplan install-xxxxx -n my-namespace \
  --type merge -p '{"spec":{"approved":true}}'
```

---

# Module 5: Real-World Operator Examples

## 5.1 Zero Trust Workload Identity Manager Operator

This is the operator you've been using! Let's understand how it works:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│               Zero Trust Workload Identity Manager Flow                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   You Create:                                                                │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ apiVersion: operator.openshift.io/v1alpha1                          │   │
│   │ kind: SpireServer                                                    │   │
│   │ spec:                                                                │   │
│   │   federation:                                                        │   │
│   │     managedRoute: "true"                                            │   │
│   │     bundleEndpoint:                                                  │   │
│   │       profile: https_web                                            │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              │ Operator Reconciles                           │
│                              ▼                                               │
│   Operator Creates:                                                          │
│   ┌───────────────────────────────────────────────────────────────────────┐ │
│   │                                                                        │ │
│   │   StatefulSet: spire-server                                           │ │
│   │   ├── Pod: spire-server-0                                             │ │
│   │   │   ├── Container: spire-server                                     │ │
│   │   │   └── Container: spire-controller-manager                         │ │
│   │   │                                                                    │ │
│   │   Service: spire-server-federation                                    │ │
│   │   ├── Port: 8443                                                      │ │
│   │   │                                                                    │ │
│   │   Route: spire-server-federation (because managedRoute: true)         │ │
│   │   ├── Host: federation.apps.cluster.example.com                       │ │
│   │   ├── TLS: reencrypt                                                  │ │
│   │   │                                                                    │ │
│   │   ConfigMap: spire-server-config                                      │ │
│   │   ├── server.conf                                                     │ │
│   │   │                                                                    │ │
│   │   Secret: spire-server-certs                                          │ │
│   │   │                                                                    │ │
│   │   ClusterFederatedTrustDomain: (for federation)                       │ │
│   │                                                                        │ │
│   └───────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 5.2 cert-manager Operator

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       cert-manager Flow                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   You Create:                                                                │
│   ┌────────────────────────────────────────────────────────────────────┐    │
│   │ kind: Certificate                                                   │    │
│   │ spec:                                                               │    │
│   │   secretName: my-tls                                                │    │
│   │   dnsNames: [my-app.example.com]                                    │    │
│   │   issuerRef:                                                        │    │
│   │     name: letsencrypt                                               │    │
│   └────────────────────────────────────────────────────────────────────┘    │
│                              │                                               │
│                              ▼                                               │
│   cert-manager:                                                              │
│   1. Creates CertificateRequest                                             │
│   2. Creates Order (for ACME)                                               │
│   3. Creates Challenge (HTTP-01 or DNS-01)                                  │
│   4. Verifies domain ownership                                              │
│   5. Gets certificate from Let's Encrypt                                    │
│   6. Creates Secret with tls.crt and tls.key                               │
│   7. Renews automatically before expiry                                     │
│                                                                              │
│   Result:                                                                    │
│   ┌────────────────────────────────────────────────────────────────────┐    │
│   │ kind: Secret                                                        │    │
│   │ name: my-tls                                                        │    │
│   │ data:                                                               │    │
│   │   tls.crt: <Let's Encrypt certificate>                             │    │
│   │   tls.key: <Private key>                                           │    │
│   └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# Practice Labs

## Lab 1: Explore CRDs

```bash
# List all CRDs
oc get crd

# Find SPIRE-related CRDs
oc get crd | grep -E "spire|spiffe"

# Describe a CRD
oc describe crd spireservers.operator.openshift.io

# List CRs of that type
oc get spireservers

# Get CR details
oc get spireserver cluster -o yaml
```

## Lab 2: Watch Reconciliation

```bash
# Watch operator logs
oc logs -f deployment/zero-trust-workload-identity-manager -n openshift-zero-trust

# In another terminal, make a change
oc patch spireserver cluster --type=merge -p '{"spec":{"federation":{"refreshHint": 600}}}'

# Watch the operator reconcile

# See events
oc get events --sort-by='.lastTimestamp' | head -20
```

## Lab 3: Install an Operator via OLM

```bash
# Create namespace
oc new-project operator-lab

# Search for operators
oc get packagemanifest | grep redis

# Install Redis operator
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: operator-lab
  namespace: operator-lab
spec:
  targetNamespaces:
    - operator-lab
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: redis-operator
  namespace: operator-lab
spec:
  channel: stable
  name: redis-operator
  source: community-operators
  sourceNamespace: openshift-marketplace
EOF

# Watch installation
oc get csv -w -n operator-lab

# See what CRDs it installed
oc get crd | grep redis

# Create a Redis CR
oc get packagemanifest redis-operator -o jsonpath='{.status.channels[0].currentCSVDesc.customresourcedefinitions.owned[0].name}'

# Cleanup
oc delete project operator-lab
```

---

# Quiz and Review

## Quiz Questions

1. **What is the relationship between CRD and CR?**

2. **What does the reconciliation loop do?**

3. **What triggers a reconciliation?**

4. **What are owner references used for?**

5. **What does OLM manage?**

6. **Name the three types of operators you can build with Operator SDK.**

## Answers

1. **CRD** defines the schema (blueprint), **CR** is an instance (actual object)

2. Compares desired state (CR) with actual state (cluster) and makes changes to match

3. CR creation/update/delete, owned resource changes, periodic requeue, external events

4. Link created resources to their CR; enables automatic cleanup on CR deletion

5. Installation, upgrade, and lifecycle of operators (like an App Store)

6. Go-based, Ansible-based, Helm-based

---

## What's Next?

After completing Phase 3, you should:

1. ✅ Understand CRDs and CRs
2. ✅ Understand the reconciliation loop
3. ✅ Know how to build a basic operator
4. ✅ Understand OLM and operator installation

**Next Step**: Phase 4 - Advanced Internals

---

*Phase 3 Complete! You now understand how Operators work!*
