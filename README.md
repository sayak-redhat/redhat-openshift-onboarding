#  Red Hat OpenShift & Operator Framework Onboarding

## Complete Learning Path: From Zero to OpenShift Expert

Welcome! This repository contains a structured learning path to master OpenShift, Kubernetes, and the Operator Framework. Whether you're new to containers or looking to deepen your knowledge, this guide will take you from fundamentals to advanced concepts.

---

##  Learning Path Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                         │
│   🚀 YOUR JOURNEY TO OPENSHIFT MASTERY                                                  │
│                                                                                         │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐             │
│   │   PHASE 1    │   │   PHASE 2    │   │   PHASE 3    │   │   PHASE 4    │             │
│   │              │   │              │   │              │   │              │             │
│   │ Foundations  │──►│  OpenShift   │──►│  Operators   │──►│  Advanced    │             │
│   │              │   │    Core      │   │  Framework   │   │  Internals   │             │
│   │  2-3 weeks   │   │  3-4 weeks   │   │  4-6 weeks   │   │   Ongoing    │             │
│   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘             │
│         │                  │                  │                  │                      │
│         ▼                  ▼                  ▼                  ▼                      │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐             │
│   │ • Linux      │   │ • Routes     │   │ • CRDs       │   │ • API Server │             │
│   │ • Containers │   │ • SCCs       │   │ • Controllers│   │ • etcd       │             │
│   │ • Kubernetes │   │ • RBAC       │   │ • Operator   │   │ • Scheduler  │             │
│   │ • Networking │   │ • Storage    │   │   SDK        │   │ • CNI/CSI    │             │
│   │              │   │ • Networking │   │ • OLM        │   │ • Debugging  │             │
│   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘             │
│                                                                                         │
│   Total Estimated Time: 3-4 months of dedicated learning                                │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

##  Repository Structure

```
REDHAT-OPENSHIFT-ONBOARDING/
│
├── README.md                              # You are here! Start here.
│
├── Phase1-Foundations/
│   └── Phase1-Foundations-Study-Guide.md  # Linux, Containers, Kubernetes basics
│
├── Phase2-OpenShift-Core/
│   └── Phase2-OpenShift-Core-Guide.md     # OpenShift-specific concepts
│
├── Phase3-Operator-Framework/
│   └── Phase3-Operator-Framework-Guide.md # Building and understanding Operators
│
└── Phase4-Advanced-Internals/
    └── Phase4-Advanced-Internals-Guide.md # Deep dive into internals
```

---

##  How to Use This Guide

### Step 1: Assess Your Current Level

| If you... | Start at... |
|-----------|-------------|
| Are new to Linux/containers | **Phase 1** |
| Know Docker/K8s but not OpenShift | **Phase 2** |
| Know OpenShift but not Operators | **Phase 3** |
| Want to understand internals | **Phase 4** |

### Step 2: Follow the Learning Path

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Recommended Daily Schedule                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│    Reading & Theory      : 30-45 minutes                                │
│    Hands-on Practice     : 45-60 minutes                                │
│    Notes & Documentation : 15-30 minutes                                │
│                                                                         │
│   Total: 1.5-2 hours per day                                            │
│                                                                         │
│   Tip: Consistency beats intensity. 1 hour daily > 8 hours on weekend   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Step 3: Practice, Practice, Practice!

Each phase includes:
- Concept explanations with real-life analogies
- Hands-on commands to try
- YAML examples you can apply
- Practice labs
- Quizzes to test understanding

---

##  Phase Summaries

### Phase 1: Foundations (2-3 weeks)
**File**: `Phase1-Foundations/Phase1-Foundations-Study-Guide.md`

| Module | Topics | You'll Learn |
|--------|--------|--------------|
| **1.1 Linux** | Processes, Namespaces, Cgroups, Networking | How the OS works under containers |
| **1.2 Containers** | Images, Layers, Runtime, Registry | What containers REALLY are |
| **1.3 Kubernetes** | Pods, Deployments, Services, RBAC | Container orchestration basics |

**By the end of Phase 1, you'll understand:**
- Why containers are NOT lightweight VMs
- How namespaces and cgroups create isolation
- Kubernetes architecture and core concepts

---

### Phase 2: OpenShift Core (3-4 weeks)
**File**: `Phase2-OpenShift-Core/Phase2-OpenShift-Core-Guide.md`

| Module | Topics | You'll Learn |
|--------|--------|--------------|
| **2.1 OpenShift vs K8s** | What makes OpenShift different | Enterprise features added to K8s |
| **2.2 Routes** | Edge, Passthrough, Reencrypt | How external traffic reaches your apps |
| **2.3 SCCs** | Security Context Constraints | Why pods sometimes fail to start |
| **2.4 Projects** | Namespaces + RBAC | Multi-tenancy in OpenShift |
| **2.5 Builds** | S2I, BuildConfigs | Building images in-cluster |

**By the end of Phase 2, you'll understand:**
- OpenShift's enterprise additions to Kubernetes
- How Routes work vs Ingress
- Security constraints and how to work with them

---

### Phase 3: Operator Framework (4-6 weeks)
**File**: `Phase3-Operator-Framework/Phase3-Operator-Framework-Guide.md`

| Module | Topics | You'll Learn |
|--------|--------|--------------|
| **3.1 What are Operators** | Pattern, Purpose, Benefits | Automating human operations |
| **3.2 CRDs** | Custom Resource Definitions | Extending Kubernetes API |
| **3.3 Controllers** | Reconciliation Loop | The heart of Operators |
| **3.4 Operator SDK** | Building your first Operator | Hands-on development |
| **3.5 OLM** | Operator Lifecycle Manager | Installing/Managing Operators |

**By the end of Phase 3, you'll understand:**
- How Operators automate complex applications
- How to build your own Operator
- How OLM manages Operator lifecycle

---

### Phase 4: Advanced Internals (Ongoing)
**File**: `Phase4-Advanced-Internals/Phase4-Advanced-Internals-Guide.md`

| Module | Topics | You'll Learn |
|--------|--------|--------------|
| **4.1 API Server** | Request flow, Admission | How every API call is processed |
| **4.2 etcd** | Cluster state storage | The database of Kubernetes |
| **4.3 Scheduler** | Pod placement decisions | How pods get assigned to nodes |
| **4.4 Networking** | CNI, OVN-Kubernetes | Deep network internals |
| **4.5 Debugging** | Advanced troubleshooting | Solving complex issues |

**By the end of Phase 4, you'll understand:**
- How Kubernetes components interact internally
- How to debug complex cluster issues
- Deep knowledge of the platform architecture

---

## 🚀 Quick Start

### Prerequisites

1. **A Linux machine or VM** (Fedora, RHEL, or Ubuntu)
2. **Podman or Docker** installed
3. **kubectl/oc CLI** installed
4. **Access to a Kubernetes/OpenShift cluster** (minikube, kind, or real cluster)

### Installation Commands

```bash
# Install kubectl (Linux)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install oc CLI (OpenShift)
# Download from: https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/

# Install Podman (Fedora/RHEL)
sudo dnf install -y podman

# Install minikube (for local K8s cluster)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube-linux-amd64
sudo mv minikube-linux-amd64 /usr/local/bin/minikube

# Start minikube
minikube start
```

### Start Learning

```bash
# Clone or navigate to this directory
cd /home/sayadas/REDHAT-OPENSHIFT-ONBOARDING

# Start with Phase 1
cat Phase1-Foundations/Phase1-Foundations-Study-Guide.md | less

# Or open in your favorite editor
code .
```

---

## Progress Tracking

Use this checklist to track your progress:

### Phase 1 Checklist
- [ ] Linux Fundamentals
  - [ ] Understand processes and PIDs
  - [ ] Understand namespaces
  - [ ] Understand cgroups
  - [ ] Basic networking (IP, ports, DNS)
  - [ ] Filesystem navigation
  - [ ] Systemd basics
- [ ] Container Fundamentals
  - [ ] Understand what containers really are
  - [ ] Work with images (pull, run, inspect)
  - [ ] Understand layers and registries
  - [ ] Practice with Podman
- [ ] Kubernetes Fundamentals
  - [ ] Understand cluster architecture
  - [ ] Work with Pods
  - [ ] Work with Deployments
  - [ ] Work with Services
  - [ ] Understand RBAC
  - [ ] Work with Storage (PV/PVC)
- [ ] Complete Phase 1 Labs
- [ ] Pass Phase 1 Quizzes

### Phase 2 Checklist
- [ ] OpenShift Basics
  - [ ] Understand OpenShift vs Kubernetes
  - [ ] Master Routes (all types)
  - [ ] Understand SCCs
  - [ ] Work with Projects
  - [ ] Understand Builds (S2I)
  - [ ] Work with ImageStreams
- [ ] OpenShift Networking
  - [ ] Understand OVN-Kubernetes
  - [ ] Configure NetworkPolicies
  - [ ] Debug network issues
- [ ] OpenShift Security
  - [ ] Master RBAC
  - [ ] Work with ServiceAccounts
  - [ ] Understand OAuth
- [ ] Complete Phase 2 Labs
- [ ] Pass Phase 2 Quizzes

### Phase 3 Checklist
- [ ] Operator Concepts
  - [ ] Understand the Operator pattern
  - [ ] Understand CRDs
  - [ ] Understand reconciliation loop
- [ ] Operator SDK
  - [ ] Set up development environment
  - [ ] Create a basic Operator
  - [ ] Implement reconciliation logic
  - [ ] Test your Operator
- [ ] OLM
  - [ ] Understand CatalogSources
  - [ ] Understand Subscriptions
  - [ ] Package and deploy an Operator
- [ ] Complete Phase 3 Labs

### Phase 4 Checklist
- [ ] API Server Deep Dive
- [ ] etcd Understanding
- [ ] Scheduler Internals
- [ ] Advanced Debugging
- [ ] Performance Tuning

---

##  Certification Path

After completing this learning path, you'll be prepared for:

| Certification | Description | Phase Alignment |
|---------------|-------------|-----------------|
| **CKA** | Certified Kubernetes Administrator | Phase 1-2 |
| **CKAD** | Certified Kubernetes Application Developer | Phase 1-2 |
| **EX280** | Red Hat Certified Specialist in OpenShift Administration | Phase 2-3 |
| **EX288** | Red Hat Certified OpenShift Application Developer | Phase 2-3 |

---

##  Learning Tips

### Do's 
1. **Practice daily** - Consistency is key
2. **Break things** - Learn from failures
3. **Take notes** - Document your learning
4. **Use `kubectl explain`** - Built-in documentation
5. **Read error messages** - They tell you what's wrong
6. **Join communities** - Ask questions

### Don'ts 
1. **Don't just read** - You must practice
2. **Don't skip phases** - Build on foundations
3. **Don't memorize** - Understand concepts
4. **Don't give up** - It gets easier

---

##  Additional Resources

### Official Documentation
- [Kubernetes Docs](https://kubernetes.io/docs/)
- [OpenShift Docs](https://docs.openshift.com/)
- [Operator SDK Docs](https://sdk.operatorframework.io/)

### Video Courses
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [TechWorld with Nana - Kubernetes](https://www.youtube.com/c/TechWorldwithNana)
- [Red Hat Developer YouTube](https://www.youtube.com/c/RedHatDevelopers)

### Books
- "Kubernetes Up & Running" - O'Reilly
- "Programming Kubernetes" - O'Reilly
- "Kubernetes Patterns" - O'Reilly

### Practice Environments
- [Play with Kubernetes](https://labs.play-with-k8s.com/)
- [Katacoda](https://www.katacoda.com/)
- [Killer Shell](https://killer.sh/) (CKA/CKAD practice)

---

##  Getting Help

If you're stuck:

1. **Check the troubleshooting sections** in each phase guide
2. **Use `kubectl describe`** to see detailed errors
3. **Check events**: `kubectl get events --sort-by='.lastTimestamp'`
4. **Ask in communities**:
   - Kubernetes Slack: kubernetes.slack.com
   - OpenShift Commons: commons.openshift.org
   - Stack Overflow: [kubernetes] tag

---

##  Start Your Journey!

Ready to begin? Open the first phase:

```bash
# Start with Phase 1
less Phase1-Foundations/Phase1-Foundations-Study-Guide.md
```

**Good luck on your OpenShift journey! **

---

*Created for Red Hat OpenShift Learning Path*
*Last Updated: January 2026*
*Author: OpenShift Learning Team*
