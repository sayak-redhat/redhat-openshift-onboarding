# Catalog Image Quick Reference

**One-page guide for pulling CI catalogs and pushing to Quay.io**

---

## Prerequisites

- `podman` installed
- Quay.io account (https://quay.io)
- Access to cluster-bot (Slack)

---

## Step 1: Request Build from Cluster-Bot

In Slack, message cluster-bot:

```
catalog build 4.20,openshift/zero-trust-workload-identity-manager#92 zero-trust-workload-identity-manager-bundle
```

**Format:** `catalog build <VERSION>,<REPO>#<PR> <BUNDLE_NAME>`

Wait for build to complete.

---

## Step 2: Extract Info from Logs

From the cluster-bot response logs, find:

| Info | Example | Look For |
|------|---------|----------|
| **Build Cluster** | `build10` | `apps.build10.ci.devcluster` in URL |
| **Namespace** | `ci-ln-fxpvcz2` | `Using namespace .../projects/ci-ln-xxxxx` |

**Construct these:**
```
Secret URL:  https://console-openshift-console.apps.build10.ci.devcluster.openshift.com/k8s/ns/ci-ln-fxpvcz2/secrets/registry-pull-credentials
Catalog:     registry.build10.ci.openshift.org/ci-ln-fxpvcz2/pipeline:catalog
```

---

## Step 3: Get Auth & Pull Catalog

### 3.1 Get the Secret
1. Open the **Secret URL** in browser
2. Login with **RedHat_Internal_SSO**
3. Click **Reveal values** or go to **YAML** tab
4. Copy the `.dockerconfigjson` base64 value (starts with `eyJ...`)

### 3.2 Save & Decode
```bash
# Save to file (paste the base64 inside)
nano /tmp/dockerconfig.b64
# Paste, save (Ctrl+X, Y, Enter)

# Decode
base64 -d /tmp/dockerconfig.b64 > /tmp/ci-registry-auth.json

# Verify
cat /tmp/ci-registry-auth.json | grep -o '"registry.build10.ci.openshift.org"'
```

### 3.3 Pull
```bash
podman pull --authfile /tmp/ci-registry-auth.json \
    registry.build10.ci.openshift.org/ci-ln-fxpvcz2/pipeline:catalog
```

---

## Step 4: Push to Quay.io

```bash
# Login to Quay
podman login quay.io

# Tag
podman tag \
    registry.build10.ci.openshift.org/ci-ln-fxpvcz2/pipeline:catalog \
    quay.io/rh-ee-sayadas/zero-trust-workload-identity-manager-catalog:v1.0.0

# Push
podman push quay.io/rh-ee-sayadas/zero-trust-workload-identity-manager-catalog:v1.0.0
```

---

## Step 5: Make Public

1. Open: https://quay.io/repository/rh-ee-sayadas/zero-trust-workload-identity-manager-catalog?tab=settings
2. Click **Settings** → **Repository Visibility** → **Make Public**

---

## Step 6: Deploy to OpenShift (Optional)

```bash
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: zero-trust-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/rh-ee-sayadas/zero-trust-workload-identity-manager-catalog:v1.0.0
  displayName: Zero Trust Catalog
  publisher: Red Hat QE
EOF

# Verify
oc get catalogsource -n openshift-marketplace -w
```

---

## Copy-Paste Template

Replace the values in `< >` with your actual values:

```bash
# === VARIABLES (update these) ===
BUILD_CLUSTER="<build10>"
CI_NAMESPACE="<ci-ln-fxpvcz2>"
QUAY_USER="<rh-ee-sayadas>"
CATALOG_NAME="<zero-trust-workload-identity-manager-catalog>"
VERSION="v1.0.0"

# === COMMANDS (just run) ===
# Decode auth
base64 -d /tmp/dockerconfig.b64 > /tmp/ci-registry-auth.json

# Pull
podman pull --authfile /tmp/ci-registry-auth.json \
    registry.${BUILD_CLUSTER}.ci.openshift.org/${CI_NAMESPACE}/pipeline:catalog

# Tag & Push
podman login quay.io
podman tag \
    registry.${BUILD_CLUSTER}.ci.openshift.org/${CI_NAMESPACE}/pipeline:catalog \
    quay.io/${QUAY_USER}/${CATALOG_NAME}:${VERSION}
podman push quay.io/${QUAY_USER}/${CATALOG_NAME}:${VERSION}

# Make public: https://quay.io/repository/${QUAY_USER}/${CATALOG_NAME}?tab=settings
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "authentication required" | Use `--authfile /tmp/ci-registry-auth.json` |
| "manifest unknown" | CI namespace expired, request new build |
| Push fails | Run `podman login quay.io` |
| CatalogSource FAILED | Make Quay repo public |

---

## Automation Script

For fully automated workflow, use:

```bash
cd ~/Documents/My-Notes/catalog-automation
python catalog_manager.py \
    --url "https://console-openshift-console.apps.build10.ci.devcluster.openshift.com/k8s/cluster/projects/ci-ln-fxpvcz2" \
    --quay-user rh-ee-sayadas \
    --quay-repo zero-trust-workload-identity-manager-catalog \
    --dockerconfig-file /tmp/dockerconfig.b64
```

---

**Last Updated:** February 2026  
**Full Guide:** [Catalog-Management-Complete-Guide.md](./Catalog-Management-Complete-Guide.md)
