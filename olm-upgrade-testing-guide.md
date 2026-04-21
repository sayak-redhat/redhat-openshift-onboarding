# OLM Upgrade Testing Guide: v1.0.0 → v1.0.1

End-to-end guide for building operator images locally, pushing them to quay.io,
installing v1.0.0 via OLM, and testing the upgrade path to v1.0.1 with operand deployment.

Tested with PR [#104](https://github.com/openshift/zero-trust-workload-identity-manager/pull/104) (controller-manager-0.6.4).

---

## Prerequisites

| Tool | Purpose |
|------|---------|
| `podman` | Build and push container images |
| `operator-sdk` | Generate bundle manifests |
| `opm` | Build catalog images |
| `oc` | Interact with OpenShift cluster |
| `git` | Switch between branches |
| `jq` | Parse JSON output |

You also need:
- A quay.io account with push access
- Access to `registry.ci.openshift.org` (for the builder image)
- An OpenShift cluster (e.g., via cluster-bot)

---

## Phase 1 — Build and Push All Images to quay.io

### 1.1 Login to registries

```bash
podman login quay.io
```

```
Authenticating with existing credentials for quay.io
Existing credentials are valid. Already logged in to quay.io
```

Login to the OpenShift CI registry (needed for the builder base image):

```bash
oc login --token=<your-token> --server=https://api.ci.l2s4.p1.openshiftapps.com:6443
podman login -u <your-username> -p $(oc whoami -t) registry.ci.openshift.org
```

```
Login Succeeded!
```

> Get your CI token from: https://oauth-openshift.apps.ci.l2s4.p1.openshiftapps.com/oauth/token/request

### 1.2 Checkout the PR branch and build the operator image

```bash
gh repo set-default openshift/zero-trust-workload-identity-manager
gh pr checkout 104
```

```
Switched to branch 'controller-manager-0.6.4'
```

Build and push:

```bash
make generate manifests CONTAINER_TOOL=podman
make docker-build docker-push IMG=quay.io/<your-namespace>/zero-trust-workload-identity-manager:latest CONTAINER_TOOL=podman
```

```
Successfully tagged quay.io/<your-namespace>/zero-trust-workload-identity-manager:latest
...
Writing manifest to image destination
```

### 1.3 Build and push bundle v1.0.0

Switch to the release branch:

```bash
git checkout release-1.0.0
```

Generate the bundle. **Important:** Pass `IMG=` pointing to your quay.io operator image,
otherwise the CSV will embed the default image (`openshift.io/...`) which doesn't exist on any registry.

```bash
make bundle VERSION=1.0.0 CHANNELS=stable-v1 DEFAULT_CHANNEL=stable-v1 \
  IMG=quay.io/<your-namespace>/zero-trust-workload-identity-manager:latest \
  CONTAINER_TOOL=podman
```

```
INFO[0000] Bundle metadata generated successfully
INFO[0000] All validation tests have completed successfully
```

Verify the generated metadata:

```bash
grep "bundle.package.v1" bundle/metadata/annotations.yaml
grep "bundle.channels.v1" bundle/metadata/annotations.yaml
```

```
  operators.operatorframework.io.bundle.package.v1: zero-trust-workload-identity-manager
  operators.operatorframework.io.bundle.channels.v1: stable-v1
```

Build and push (use `podman` directly because the Makefile's `bundle-build` target hardcodes `docker`):

```bash
podman build -f bundle.Dockerfile -t quay.io/<your-namespace>/zero-trust-workload-identity-manager-bundle:v1.0.0 .
podman push quay.io/<your-namespace>/zero-trust-workload-identity-manager-bundle:v1.0.0
```

```
Successfully tagged quay.io/<your-namespace>/zero-trust-workload-identity-manager-bundle:v1.0.0
...
Writing manifest to image destination
```

### 1.4 Build and push bundle v1.0.1

Switch to the PR branch and reset the bundle folder:

```bash
git stash
git checkout controller-manager-0.6.4
git checkout -- bundle/
```

Generate the bundle:

```bash
make bundle VERSION=1.0.1 CHANNELS=stable-v1 DEFAULT_CHANNEL=stable-v1 \
  IMG=quay.io/<your-namespace>/zero-trust-workload-identity-manager:latest \
  CONTAINER_TOOL=podman
```

**Add the upgrade edge (CRITICAL):** Edit `bundle/manifests/zero-trust-workload-identity-manager.clusterserviceversion.yaml`
and add the `replaces` field at the end of the file, just before the `version` line:

```yaml
  replaces: zero-trust-workload-identity-manager.v1.0.0
  version: 1.0.1
```

> Without this field, OLM cannot build the upgrade graph and won't offer v1.0.1 as an upgrade.

Verify all metadata:

```bash
grep "bundle.package.v1" bundle/metadata/annotations.yaml
grep "replaces" bundle/manifests/zero-trust-workload-identity-manager.clusterserviceversion.yaml
```

```
  operators.operatorframework.io.bundle.package.v1: zero-trust-workload-identity-manager
  replaces: zero-trust-workload-identity-manager.v1.0.0
```

Build and push:

```bash
podman build -f bundle.Dockerfile -t quay.io/<your-namespace>/zero-trust-workload-identity-manager-bundle:v1.0.1 .
podman push quay.io/<your-namespace>/zero-trust-workload-identity-manager-bundle:v1.0.1
```

### 1.5 Verify bundle versions

```bash
opm render quay.io/<your-namespace>/zero-trust-workload-identity-manager-bundle:v1.0.0 | grep -E '"version": "1\.'
opm render quay.io/<your-namespace>/zero-trust-workload-identity-manager-bundle:v1.0.1 | grep -E '"version": "1\.'
```

```
                "version": "1.0.0"
                "version": "1.0.1"
```

### 1.6 Build and push catalog (with both versions)

```bash
opm index add \
  --container-tool podman \
  --mode semver \
  --tag quay.io/<your-namespace>/zero-trust-workload-identity-manager-catalog:latest \
  --bundles quay.io/<your-namespace>/zero-trust-workload-identity-manager-bundle:v1.0.0,quay.io/<your-namespace>/zero-trust-workload-identity-manager-bundle:v1.0.1

podman push quay.io/<your-namespace>/zero-trust-workload-identity-manager-catalog:latest
```

### 1.7 Validate the catalog

```bash
opm render quay.io/<your-namespace>/zero-trust-workload-identity-manager-catalog:latest \
  | jq -c 'select(.schema == "olm.package" or .schema == "olm.channel")'
```

Expected output (formatted for readability):

```json
{
  "schema": "olm.package",
  "name": "zero-trust-workload-identity-manager",
  "defaultChannel": "stable-v1"
}
{
  "schema": "olm.channel",
  "name": "stable-v1",
  "package": "zero-trust-workload-identity-manager",
  "entries": [
    { "name": "zero-trust-workload-identity-manager.v1.0.0" },
    { "name": "zero-trust-workload-identity-manager.v1.0.1",
      "replaces": "zero-trust-workload-identity-manager.v1.0.0" }
  ]
}
```

### 1.8 Clean up local temp files

```bash
rm -rf bundle_tmp* index.Dockerfile*
```

### 1.9 Make repositories public on quay.io

Go to quay.io and set all three repositories to **public**:

- `<your-namespace>/zero-trust-workload-identity-manager`
- `<your-namespace>/zero-trust-workload-identity-manager-bundle`
- `<your-namespace>/zero-trust-workload-identity-manager-catalog`

> Settings → Repository Visibility → Make Public

If they're private, the cluster won't be able to pull the images.

### Images summary

| Repository | Tag(s) | Purpose |
|-----------|--------|---------|
| `zero-trust-workload-identity-manager` | `latest` | Operator binary |
| `zero-trust-workload-identity-manager-bundle` | `v1.0.0`, `v1.0.1` | OLM bundle metadata |
| `zero-trust-workload-identity-manager-catalog` | `latest` | OLM catalog (contains both versions) |

---

## Phase 2 — Get a Cluster

Request a cluster via cluster-bot (Slack) and login with the admin kubeconfig:

```bash
export KUBECONFIG=~/Downloads/<cluster-bot-kubeconfig-file>
oc whoami
```

```
system:admin
```

---

## Phase 3 — Install v1.0.0 via OLM

### 3.1 Create the namespace

```bash
oc create namespace zero-trust-workload-identity-manager
```

```
namespace/zero-trust-workload-identity-manager created
```

### 3.2 Create CatalogSource

```bash
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ztwim-test-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/<your-namespace>/zero-trust-workload-identity-manager-catalog:latest
  displayName: ZTWIM Test Catalog
  updateStrategy:
    registryPoll:
      interval: 2m
EOF
```

```
catalogsource.operators.coreos.com/ztwim-test-catalog created
```

Wait ~30 seconds and verify the catalog is healthy:

```bash
oc get catalogsource ztwim-test-catalog -n openshift-marketplace \
  -o jsonpath='{.status.connectionState.lastObservedState}' && echo
```

```
READY
```

> If it shows `TRANSIENT_FAILURE`, check if the catalog pod is running:
> `oc get pods -n openshift-marketplace | grep ztwim`
> and check logs: `oc logs -n openshift-marketplace -l olm.catalogSource=ztwim-test-catalog`

### 3.3 Create OperatorGroup

```bash
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: zero-trust-workload-identity-manager-og
  namespace: zero-trust-workload-identity-manager
spec:
  upgradeStrategy: Default
EOF
```

```
operatorgroup.operators.coreos.com/zero-trust-workload-identity-manager-og created
```

### 3.4 Create Subscription (pinned to v1.0.0, Manual approval)

```bash
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: zero-trust-workload-identity-manager
  namespace: zero-trust-workload-identity-manager
spec:
  source: ztwim-test-catalog
  sourceNamespace: openshift-marketplace
  name: zero-trust-workload-identity-manager
  channel: stable-v1
  installPlanApproval: Manual
  startingCSV: zero-trust-workload-identity-manager.v1.0.0
EOF
```

```
subscription.operators.coreos.com/zero-trust-workload-identity-manager created
```

### 3.5 Approve v1.0.0 InstallPlan

```bash
oc get installplan -n zero-trust-workload-identity-manager
```

```
NAME            CSV                                           APPROVAL   APPROVED
install-xxxxx   zero-trust-workload-identity-manager.v1.0.0   Manual     false
```

Approve it (replace the plan name with your actual value):

```bash
oc patch installplan install-xxxxx \
  -n zero-trust-workload-identity-manager \
  --type merge \
  -p '{"spec":{"approved":true}}'
```

```
installplan.operators.coreos.com/install-xxxxx patched
```

### 3.6 Verify v1.0.0 is installed

Wait ~30 seconds:

```bash
oc get csv -n zero-trust-workload-identity-manager
```

```
NAME                                          DISPLAY                                VERSION   REPLACES   PHASE
zero-trust-workload-identity-manager.v1.0.0   Zero Trust Workload Identity Manager   1.0.0                Succeeded
```

```bash
oc get pods -n zero-trust-workload-identity-manager
```

```
NAME                                                              READY   STATUS    RESTARTS   AGE
zero-trust-workload-identity-manager-controller-manager-xxxxx     1/1     Running   0          30s
```

---

## Phase 4 — Test Upgrade to v1.0.1

### 4.1 Wait for OLM to detect v1.0.1

The catalog poll interval is 2 minutes. Wait and then check:

```bash
oc get installplan -n zero-trust-workload-identity-manager
```

```
NAME            CSV                                           APPROVAL   APPROVED
install-xxxxx   zero-trust-workload-identity-manager.v1.0.0   Manual     true
install-yyyyy   zero-trust-workload-identity-manager.v1.0.1   Manual     false
```

### 4.2 Approve the v1.0.1 InstallPlan

```bash
oc patch installplan install-yyyyy \
  -n zero-trust-workload-identity-manager \
  --type merge \
  -p '{"spec":{"approved":true}}'
```

```
installplan.operators.coreos.com/install-yyyyy patched
```

### 4.3 Verify the upgrade

```bash
oc get csv -n zero-trust-workload-identity-manager
```

During upgrade:

```
NAME                                          DISPLAY                                VERSION   REPLACES                                      PHASE
zero-trust-workload-identity-manager.v1.0.0   Zero Trust Workload Identity Manager   1.0.0                                                   Replacing
zero-trust-workload-identity-manager.v1.0.1   Zero Trust Workload Identity Manager   1.0.1     zero-trust-workload-identity-manager.v1.0.0   Installing
```

After ~30 seconds:

```
NAME                                          DISPLAY                                VERSION   REPLACES                                      PHASE
zero-trust-workload-identity-manager.v1.0.1   Zero Trust Workload Identity Manager   1.0.1     zero-trust-workload-identity-manager.v1.0.0   Succeeded
```

The v1.0.0 CSV is removed and v1.0.1 shows `Succeeded` with `REPLACES` pointing to v1.0.0.

---

## Phase 5 — Deploy Operands and Verify

### 5.1 Set environment variables

```bash
export APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
export JWT_ISSUER_ENDPOINT=oidc-discovery.${APP_DOMAIN}
export CLUSTER_NAME=test01
```

### 5.2 Deploy the full ZTWIM stack

```bash
envsubst < fixtures/crds/ztwim-stack.yaml | oc apply -f -
```

This creates:
1. `ZeroTrustWorkloadIdentityManager` (cluster)
2. `SpireServer` (cluster)
3. `SpireAgent` (cluster)
4. `SpiffeCSIDriver` (cluster)
5. `SpireOIDCDiscoveryProvider` (cluster)

### 5.3 Verify all operands are running

```bash
oc get pods -n zero-trust-workload-identity-manager
```

```
NAME                                                              READY   STATUS    RESTARTS   AGE
spire-agent-62q56                                                 1/1     Running   0          5m
spire-agent-l56jd                                                 1/1     Running   0          5m
spire-agent-x7jl7                                                 1/1     Running   0          5m
spire-server-0                                                    2/2     Running   0          5m
spire-spiffe-csi-driver-cmjk6                                     2/2     Running   0          5m
spire-spiffe-csi-driver-f9j66                                     2/2     Running   0          5m
spire-spiffe-csi-driver-rjjmt                                     2/2     Running   0          5m
spire-spiffe-oidc-discovery-provider-86d666bc7-czdhc              1/1     Running   0          5m
zero-trust-workload-identity-manager-controller-manager-xxxxx     1/1     Running   0          10m
```

All pods should be `Running` with the expected `READY` count:

| Component | Expected READY | Count |
|-----------|---------------|-------|
| Controller Manager | 1/1 | 1 |
| SpireServer | 2/2 | 1 |
| SpireAgent | 1/1 | 1 per node |
| SpiffeCSIDriver | 2/2 | 1 per node |
| SpireOIDCDiscoveryProvider | 1/1 | 1 |

---

## Troubleshooting

### CatalogSource stuck in TRANSIENT_FAILURE

Check if the catalog pod is running and its logs:

```bash
oc get pods -n openshift-marketplace | grep ztwim
oc logs -n openshift-marketplace -l olm.catalogSource=ztwim-test-catalog
```

If `ImagePullBackOff`, make sure the catalog repo is **public** on quay.io, then delete the pod to retry:

```bash
oc delete pods -n openshift-marketplace -l olm.catalogSource=ztwim-test-catalog
```

### Operator pod stuck in ImagePullBackOff

This happens when `make bundle` was run without the `IMG=` parameter. The CSV embeds the
default image (`openshift.io/zero-trust-workload-identity-manager:latest`) which doesn't exist.

Fix: rebuild bundles with the correct `IMG`:

```bash
make bundle VERSION=1.0.0 CHANNELS=stable-v1 DEFAULT_CHANNEL=stable-v1 \
  IMG=quay.io/<your-namespace>/zero-trust-workload-identity-manager:latest
```

Then rebuild and re-push the bundle and catalog images.

### InstallPlan not appearing

If the Subscription was created before the CatalogSource was ready, delete and recreate it:

```bash
oc delete subscription zero-trust-workload-identity-manager -n zero-trust-workload-identity-manager
# Then re-apply the Subscription YAML
```

### Makefile `bundle-build` uses docker instead of podman

The `bundle-build` target in the Makefile hardcodes `docker` instead of using `$(CONTAINER_TOOL)`.
Use `podman build` directly:

```bash
podman build -f bundle.Dockerfile -t <image-tag> .
```

---

## Quick Reference — Full Command Sequence

```bash
# Login
podman login quay.io
oc login --token=<token> --server=https://api.ci.l2s4.p1.openshiftapps.com:6443
podman login -u <user> -p $(oc whoami -t) registry.ci.openshift.org

# Operator image (from PR branch)
gh pr checkout 104
make generate manifests CONTAINER_TOOL=podman
make docker-build docker-push IMG=quay.io/<ns>/zero-trust-workload-identity-manager:latest CONTAINER_TOOL=podman

# Bundle v1.0.0 (from release branch)
git checkout release-1.0.0
make bundle VERSION=1.0.0 CHANNELS=stable-v1 DEFAULT_CHANNEL=stable-v1 IMG=quay.io/<ns>/zero-trust-workload-identity-manager:latest
podman build -f bundle.Dockerfile -t quay.io/<ns>/zero-trust-workload-identity-manager-bundle:v1.0.0 .
podman push quay.io/<ns>/zero-trust-workload-identity-manager-bundle:v1.0.0

# Bundle v1.0.1 (from PR branch)
git stash && git checkout controller-manager-0.6.4 && git checkout -- bundle/
make bundle VERSION=1.0.1 CHANNELS=stable-v1 DEFAULT_CHANNEL=stable-v1 IMG=quay.io/<ns>/zero-trust-workload-identity-manager:latest
# *** Edit CSV: add "replaces: zero-trust-workload-identity-manager.v1.0.0" before "version: 1.0.1" ***
podman build -f bundle.Dockerfile -t quay.io/<ns>/zero-trust-workload-identity-manager-bundle:v1.0.1 .
podman push quay.io/<ns>/zero-trust-workload-identity-manager-bundle:v1.0.1

# Catalog (both versions)
opm index add --container-tool podman --mode semver \
  --tag quay.io/<ns>/zero-trust-workload-identity-manager-catalog:latest \
  --bundles quay.io/<ns>/zero-trust-workload-identity-manager-bundle:v1.0.0,quay.io/<ns>/zero-trust-workload-identity-manager-bundle:v1.0.1
podman push quay.io/<ns>/zero-trust-workload-identity-manager-catalog:latest

# Clean up
rm -rf bundle_tmp* index.Dockerfile*
# Make all 3 repos public on quay.io!

# --- On cluster ---
export KUBECONFIG=~/Downloads/<kubeconfig>
oc create namespace zero-trust-workload-identity-manager
# Apply CatalogSource, OperatorGroup, Subscription (see Phase 3 above)
# Approve v1.0.0 InstallPlan
# Wait for v1.0.1 InstallPlan, approve it
# Deploy operands with envsubst < fixtures/crds/ztwim-stack.yaml | oc apply -f -
```

> Replace `<ns>` with your quay.io namespace (e.g., `rh-ee-sayadas`).
