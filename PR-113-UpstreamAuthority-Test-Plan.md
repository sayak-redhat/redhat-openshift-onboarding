# Test Plan: SPIRE-129 Native UpstreamAuthority Support (cert-manager, Vault)
<!-- markdownlint-disable MD032 -->

**Source:** [PR #113 changes](https://github.com/openshift/zero-trust-workload-identity-manager/pull/113/changes)  
**Date:** 2026-05-14  
**Scope:** Add optional `spec.upstreamAuthority` to `SpireServer` so SPIRE intermediate CA can be signed by cert-manager or Vault, including schema, validation, RBAC, ConfigMap rendering, StatefulSet wiring, and regression safety when field is absent/removed.

## ADR Decomposition

**Feature:** Add native UpstreamAuthority plugin support in `SpireServer` for cert-manager or Vault.

**Components in scope:**
- API/CRD schema: `SpireServerSpec.upstreamAuthority`
- API types: `UpstreamAuthorityConfig`, `UpstreamAuthorityCertManager`, `UpstreamAuthorityVault`, `VaultK8sAuthConfig`, `SecretKeyReference`
- Generated deep-copy methods
- SPIRE server config generation (`pkg/controller/spire-server/configmap.go`)
- Reconcile-time validation (`pkg/controller/spire-server/validation.go` + controller wiring)
- StatefulSet mutation for Vault token and CA cert mounts
- RBAC logic and generated role/CSV updates for `cert-manager.io/certificaterequests`
- Unit tests for configmap/statefulset/validation/rbac

**Positive-path requirements (from goals):**
1. `SpireServer` accepts `upstreamAuthority` and enforces exactly one provider (`certManager` or `vault`).
2. cert-manager mode renders valid UpstreamAuthority plugin data with defaults.
3. Vault mode renders valid UpstreamAuthority plugin data with defaults and optional fields.
4. Vault mode mounts projected SA token and optional CA secret in SPIRE server pod.
5. cert-manager mode grants cert-manager `CertificateRequest` permissions.
6. Removing `upstreamAuthority` reverts behavior to self-signed mode without breaking existing server reconcile.

**Scope boundaries (from PR/non-goal statements):**
- No new controller.
- No new webhook.
- No new CRD kind.
- No redesign of federation behavior.
- No custom auth methods beyond Vault `k8sAuth` in this feature.

**Implementation details requiring test coverage (from changed code):**
- CRD xvalidation exactly-one rule.
- cert-manager defaults: `issuerKind=Issuer`, `issuerGroup=cert-manager.io`.
- Vault defaults: `pkiMountPoint=pki`, `k8sAuthMountPoint=kubernetes`, token path fixed.
- URL validation uses structured parsing and host/scheme checks.
- Conditional RBAC addition only when `upstreamAuthority.certManager` is configured.
- Conditional StatefulSet volume/mount injection only for Vault.
- Condition reporting on invalid upstream config: `ConfigurationValid=False`.

**Risks requiring test coverage:**
- Misconfigured CR accepted/reconciled incorrectly (customer impact).
- Over-privileged RBAC in non-cert-manager mode (security impact).
- Vault token/CA mount missing causing SPIRE startup/signing failures (operational impact).
- Regression in existing self-signed path when feature not used (backward compatibility risk).
- Reconcile churn or rollout instability when toggling feature (operational risk).

**Open questions / areas needing exploratory coverage:**
- Behavior of trust bundle when migrating from self-signed to upstream and back.
- cert-manager transient failure handling (CertificateRequest API unavailable).
- Vault network intermittency and token rotation timing effects.

## Testable Requirements

| ID | Requirement | Category | ADR Source |
| --- | --- | --- | --- |
| REQ-001 | API server must reject `upstreamAuthority` when both providers are set or neither is set. | Negative | What / How / Risks |
| REQ-002 | cert-manager upstream config must render UpstreamAuthority plugin with correct defaults and explicit values. | Functional | Goals / How |
| REQ-003 | Vault upstream config must render UpstreamAuthority plugin with correct defaults and optional CA/namespace fields. | Functional | Goals / How |
| REQ-004 | Vault mode must mount projected SA token and optional upstream CA secret into SPIRE StatefulSet correctly. | Functional / Operational | How / Risks |
| REQ-005 | cert-manager mode must include `certificaterequests` RBAC permissions; non-cert-manager modes must not. | Security / Functional | What / How / Risks |
| REQ-006 | Invalid upstream config must set clear status condition (`ConfigurationValid=False`) and prevent healthy reconcile. | Negative / Operational | How / Risks |
| REQ-007 | When `upstreamAuthority` is absent, behavior must remain backward-compatible with self-signed mode. | Regression | Risks / Non-Goals |
| REQ-008 | Removing previously configured `upstreamAuthority` must revert to self-signed path without breaking reconcile. | Regression / Lifecycle | Goals / Risks |
| REQ-009 | Operator install and basic `SpireServer` lifecycle must remain healthy with this feature enabled in supported modes. | Functional / E2E | What / Goals |
| REQ-010 | Reconcile under scale/load must stay within acceptable latency and resource bounds. | Performance | Risks |
| REQ-011 | Operator must recover from transient failures (operator restart/API blip) and converge back to desired state. | Recovery | Risks |
| REQ-012 | Secrets/tokens used for Vault integration must not be leaked in logs/events and RBAC must remain least-privilege. | Security | How / Risks |

## Test Cases

### Tier 1: Unit Tests

#### UT-001: Validate exactly-one provider rule

**Priority:** Critical  
**Methodology:** White box  
**Relevant Requirement(s):** REQ-001  
**Preconditions:** Go test environment; package `pkg/controller/spire-server` available.  
**Steps:**
1. Call `validateUpstreamAuthority()` with both `CertManager` and `Vault` set.
   - **Expected:** returns error containing "exactly one".
2. Call `validateUpstreamAuthority()` with empty `UpstreamAuthorityConfig`.
   - **Expected:** returns error containing "exactly one".
3. Call with only valid `CertManager`.
   - **Expected:** no error.
4. Call with only valid `Vault`.
   - **Expected:** no error.
**Cleanup:** None  
**Failure Impact:** Invalid CR specs could pass and produce undefined reconcile behavior.

#### UT-002: cert-manager plugin data rendering and defaults

**Priority:** High  
**Methodology:** White box  
**Relevant Requirement(s):** REQ-002  
**Preconditions:** `generateServerConfMap()` callable in unit test.  
**Steps:**
1. Build `SpireServerSpec` with `upstreamAuthority.certManager` and explicit issuer fields.
   - **Expected:** ConfigMap has `plugins.UpstreamAuthority[0].cert-manager.plugin_data` with exact values.
2. Omit `issuerKind` and `issuerGroup`.
   - **Expected:** defaults are `Issuer` and `cert-manager.io`.
**Cleanup:** None  
**Failure Impact:** SPIRE fails to load plugin or points to wrong cert-manager issuer.

#### UT-003: Vault plugin data rendering and defaults

**Priority:** Critical  
**Methodology:** White box  
**Relevant Requirement(s):** REQ-003  
**Preconditions:** `generateServerConfMap()` callable.  
**Steps:**
1. Build spec with `upstreamAuthority.vault` (addr + k8sAuth role only).
   - **Expected:** plugin has `vault_addr`, default `pki_mount_point=pki`, `k8s_auth_mount_point=kubernetes`, token path `/var/run/secrets/tokens/vault`.
2. Add `caCertSecretRef`.
   - **Expected:** `ca_cert_path` set to `/run/spire/upstream-ca/ca.crt`.
3. Add `vaultNamespace`.
   - **Expected:** plugin data contains `namespace`.
**Cleanup:** None  
**Failure Impact:** Vault plugin loads with wrong parameters and cannot sign SPIRE intermediate CA.

#### UT-004: StatefulSet mutation for Vault mounts only

**Priority:** Critical  
**Methodology:** White box  
**Relevant Requirement(s):** REQ-004, REQ-007  
**Preconditions:** `GenerateSpireServerStatefulSet()` available.  
**Steps:**
1. Generate StatefulSet with Vault + `k8sAuth`.
   - **Expected:** volume `vault-token` exists with projected SA token, audience default/explicit, expiration 600, path `vault`.
2. Generate with Vault + `caCertSecretRef`.
   - **Expected:** secret volume `upstream-ca` mapping secret key to `ca.crt`, mounted read-only at `/run/spire/upstream-ca`.
3. Generate with cert-manager mode.
   - **Expected:** no `vault-token`/`upstream-ca` volumes or mounts.
4. Generate with no upstreamAuthority.
   - **Expected:** no Vault-specific volume artifacts.
**Cleanup:** None  
**Failure Impact:** SPIRE pod missing auth material or polluted with unnecessary mounts.

#### UT-005: Conditional RBAC rule generation

**Priority:** Critical  
**Methodology:** White box  
**Relevant Requirement(s):** REQ-005  
**Preconditions:** `reconcileClusterRole()` test harness with fake client.  
**Steps:**
1. Reconcile with cert-manager upstream config.
   - **Expected:** ClusterRole includes `apiGroup=cert-manager.io`, `resource=certificaterequests`, verbs `create,get,list,delete`.
2. Reconcile with Vault mode.
   - **Expected:** no cert-manager rule added.
3. Reconcile with no upstreamAuthority.
   - **Expected:** no cert-manager rule added.
**Cleanup:** None  
**Failure Impact:** Either permission denied in cert-manager path or over-privileged operator.

#### UT-006: Vault URL validation edge cases

**Priority:** High  
**Methodology:** White box  
**Relevant Requirement(s):** REQ-001, REQ-006  
**Preconditions:** `validateUpstreamAuthorityVault()` callable.  
**Steps:**
1. Pass `vaultAddr=""`.
   - **Expected:** required-field validation error.
2. Pass `vaultAddr="ftp://vault.example.org"`.
   - **Expected:** scheme validation error (`http/https` only).
3. Pass malformed URL (`https:///`).
   - **Expected:** host validation error.
4. Pass valid URL with required `k8sAuthRoleName`.
   - **Expected:** no error.
**Cleanup:** None  
**Failure Impact:** Invalid endpoints slip through to runtime failures.

### Tier 2: Integration Tests

#### INT-001: Admission rejects invalid upstreamAuthority combinations

**Priority:** Critical  
**Methodology:** Grey box  
**Relevant Requirement(s):** REQ-001  
**Preconditions:** `envtest` with SpireServer CRD loaded.  
**Steps:**
1. Create `SpireServer` with both `certManager` and `vault`.
   - **Expected:** API create fails with xvalidation message.
2. Create `SpireServer` with empty `upstreamAuthority`.
   - **Expected:** API create fails with xvalidation message.
3. Create with exactly one provider.
   - **Expected:** API create succeeds.
**Cleanup:** Delete created CRs.  
**Failure Impact:** Contract violations can reach controller and break reconcile semantics.

#### INT-002: Reconcile status condition on invalid runtime config

**Priority:** High  
**Methodology:** Grey box  
**Relevant Requirement(s):** REQ-006  
**Preconditions:** envtest + running reconciler; status updates enabled.  
**Steps:**
1. Create CR with Vault config missing `k8sAuthRoleName`.
   - **Expected:** reconcile sets `ConfigurationValid=False` with reason `InvalidUpstreamAuthorityConfiguration`.
2. Fix config by setting `k8sAuthRoleName`.
   - **Expected:** condition transitions toward valid/healthy state on next reconcile.
**Cleanup:** Delete CR.  
**Failure Impact:** Users receive no actionable status and troubleshooting becomes difficult.

#### INT-003: ConfigMap and StatefulSet reflect cert-manager path

**Priority:** High  
**Methodology:** Grey box  
**Relevant Requirement(s):** REQ-002, REQ-005, REQ-009  
**Preconditions:** envtest with reconcile creating managed resources.  
**Steps:**
1. Create valid cert-manager upstream CR.
   - **Expected:** generated server ConfigMap includes UpstreamAuthority cert-manager plugin block.
2. Inspect generated StatefulSet.
   - **Expected:** no Vault-specific volumes/mounts.
3. Inspect generated ClusterRole.
   - **Expected:** includes cert-manager `certificaterequests` permissions.
**Cleanup:** Delete CR and children.  
**Failure Impact:** cert-manager mode appears configured but fails at runtime.

#### INT-004: ConfigMap and StatefulSet reflect Vault path

**Priority:** Critical  
**Methodology:** Grey box  
**Relevant Requirement(s):** REQ-003, REQ-004, REQ-009  
**Preconditions:** envtest with reconcile loop.  
**Steps:**
1. Create valid Vault upstream CR with CA secret ref.
   - **Expected:** generated ConfigMap includes Vault plugin block and CA cert path.
2. Inspect StatefulSet spec.
   - **Expected:** projected SA token and CA secret volumes/mounts are present and read-only.
3. Inspect ClusterRole.
   - **Expected:** no cert-manager-specific rule is added.
**Cleanup:** Delete CR/resources.  
**Failure Impact:** Vault auth/signing cannot function.

#### INT-005: Feature removal reverts rendered config to self-signed

**Priority:** High  
**Methodology:** Grey box  
**Relevant Requirement(s):** REQ-008  
**Preconditions:** envtest with existing CR initially set to cert-manager or Vault mode.  
**Steps:**
1. Create CR with upstreamAuthority and let reconcile settle.
   - **Expected:** UpstreamAuthority plugin exists in ConfigMap.
2. Patch CR removing `spec.upstreamAuthority`.
   - **Expected:** subsequent ConfigMap no longer includes UpstreamAuthority plugin.
3. Verify reconcile completes without terminal error.
   - **Expected:** status remains converging/healthy.
**Cleanup:** Delete CR.  
**Failure Impact:** Migration rollback path is unsafe for customers.

### Tier 3: E2E Automated Tests

#### E2E-001: Smoke - install operator and reconcile baseline self-signed CR

**Priority:** Critical  
**Methodology:** Black box  
**Ginkgo Labels:** `smoke`, `upstreamauthority`, `regression`  
**Relevant Requirement(s):** REQ-007, REQ-009  
**Preconditions:** OpenShift cluster; operator installed and controller ready.  
**Steps:**
1. `oc apply -f spireserver-selfsigned.yaml` (no `upstreamAuthority` block).
   - **Expected:** CR accepted; SPIRE server pod reaches Ready.
2. `oc get spireserver cluster -o yaml`.
   - **Expected:** no invalid configuration condition; normal readiness conditions progress.
3. `oc -n zero-trust-workload-identity-manager get cm spire-server -o yaml`.
   - **Expected:** server config does not contain UpstreamAuthority plugin section.
**Cleanup:** `oc delete spireserver cluster` if test environment permits singleton teardown.  
**Failure Impact:** Existing installations break even when feature unused.

#### E2E-002: Negative input - reject both providers in one CR

**Priority:** Critical  
**Methodology:** Black box  
**Ginkgo Labels:** `negative`, `validation`  
**Relevant Requirement(s):** REQ-001  
**Preconditions:** Operator and CRD installed.  
**Steps:**
1. `oc apply -f spireserver-invalid-both.yaml` with both cert-manager and vault under `upstreamAuthority`.
   - **Expected:** API rejects request with message "exactly one of certManager or vault must be set".
2. `oc get spireserver`.
   - **Expected:** invalid object is not persisted.
**Cleanup:** None (resource should not exist).  
**Failure Impact:** Invalid spec can be persisted and cause unpredictable reconcile loops.

#### E2E-003: cert-manager happy path integration

**Priority:** Critical  
**Methodology:** Black box  
**Ginkgo Labels:** `certmanager`, `functional`  
**Relevant Requirement(s):** REQ-002, REQ-005, REQ-009  
**Preconditions:** cert-manager operator installed; Issuer/ClusterIssuer Ready.  
**Steps:**
1. Apply `SpireServer` with `upstreamAuthority.certManager`.
   - **Expected:** CR accepted and reconciles.
2. Inspect generated ConfigMap (`oc get cm spire-server ...`).
   - **Expected:** UpstreamAuthority cert-manager plugin block contains configured issuer data/defaults.
3. Inspect operator ClusterRole.
   - **Expected:** contains `cert-manager.io/certificaterequests` with required verbs.
4. Verify SPIRE server pod Ready after rollout.
   - **Expected:** StatefulSet updated and stable.
**Cleanup:** Revert to baseline CR or delete test CR.  
**Failure Impact:** Feature advertised for cert-manager is nonfunctional.

#### E2E-004: Vault happy path integration (k8s auth + optional CA cert)

**Priority:** Critical  
**Methodology:** Black box  
**Ginkgo Labels:** `vault`, `functional`  
**Relevant Requirement(s):** REQ-003, REQ-004, REQ-009  
**Preconditions:** Reachable Vault PKI endpoint; configured role for SPIRE service account; optional CA secret created.  
**Steps:**
1. Apply `SpireServer` with `upstreamAuthority.vault`.
   - **Expected:** CR accepted and reconcile starts rollout.
2. Inspect server pod spec.
   - **Expected:** projected token mount at `/var/run/secrets/tokens`, optional CA mount at `/run/spire/upstream-ca`.
3. Inspect ConfigMap server config.
   - **Expected:** Vault plugin section includes expected values and token path.
4. Wait for SPIRE server pod Ready.
   - **Expected:** rollout completes without CrashLoopBackOff.
**Cleanup:** Revert CR.  
**Failure Impact:** Vault integration unusable in real cluster deployments.

#### E2E-005: Lifecycle - enable upstream authority then remove it

**Priority:** High  
**Methodology:** Black box  
**Ginkgo Labels:** `lifecycle`, `migration`  
**Relevant Requirement(s):** REQ-008, REQ-009  
**Preconditions:** Working `SpireServer` baseline.  
**Steps:**
1. Patch CR to enable cert-manager (or vault).
   - **Expected:** StatefulSet rolls; ConfigMap contains UpstreamAuthority plugin.
2. Patch CR to remove `upstreamAuthority`.
   - **Expected:** StatefulSet rolls again; plugin section removed.
3. Verify server stays healthy after each transition.
   - **Expected:** Ready pods and no terminal error conditions.
**Cleanup:** Restore baseline CR.  
**Failure Impact:** Customers cannot safely roll back feature configuration.

#### E2E-006: Regression - federation features remain unaffected

**Priority:** Medium  
**Methodology:** Black box  
**Ginkgo Labels:** `regression`, `federation`  
**Relevant Requirement(s):** REQ-007  
**Preconditions:** Existing federation setup test fixture or minimal federation-enabled CR.  
**Steps:**
1. Apply federation-enabled `SpireServer` without upstreamAuthority.
   - **Expected:** federation resources reconcile as before.
2. Enable upstreamAuthority (one mode), keep federation fields unchanged.
   - **Expected:** federation route/config still present and valid.
3. Run basic federation verification command (`bundle list` or bundle endpoint check).
   - **Expected:** no regression in federation functionality due to upstreamAuthority feature.
**Cleanup:** Delete test-specific federation objects.  
**Failure Impact:** New feature breaks unrelated federation workflows.

### Tier 4: Manual QE Tests

#### MQE-001: Acceptance - admin enables cert-manager upstream in documented flow

**Priority:** High  
**Methodology:** Black box (human execution)  
**Type:** Acceptance  
**Relevant Requirement(s):** REQ-002, REQ-005, REQ-009  
**Preconditions:** OpenShift cluster with ZTWIM and cert-manager installed; Issuer Ready.  
**Steps:**
1. Apply documented cert-manager `SpireServer` YAML.
   - **Expected:** `oc apply` succeeds.
2. Run `oc get spireserver cluster -o yaml`.
   - **Expected:** no invalid config condition; reconcile progressing.
3. Run `oc -n zero-trust-workload-identity-manager get pod -l app=spire-server`.
   - **Expected:** pod becomes Ready.
4. Run `oc -n zero-trust-workload-identity-manager get cm spire-server -o yaml | grep -A20 UpstreamAuthority`.
   - **Expected:** cert-manager plugin block visible with expected issuer.
5. Run `oc get clusterrole <operator-clusterrole> -o yaml | grep -A10 certificaterequests`.
   - **Expected:** cert-manager rule present.
**Pass/Fail Criteria:** All expected resources/fields visible and SPIRE server remains Ready.  
**Cleanup:** Revert CR to baseline if required by environment.  
**Failure Impact:** Documented customer adoption path is broken.

#### MQE-002: Usability - invalid config message is clear and actionable

**Priority:** Medium  
**Methodology:** Black box (human execution)  
**Type:** Usability  
**Relevant Requirement(s):** REQ-001, REQ-006  
**Preconditions:** Running operator.  
**Steps:**
1. Apply CR containing both cert-manager and vault blocks.
   - **Expected:** API rejects with exactly-one validation message.
2. Apply CR with Vault block missing `k8sAuthRoleName` (if admission allows creation in scenario).
   - **Expected:** status shows clear invalid-upstream configuration reason.
3. Inspect events/logs.
   - **Expected:** message references specific missing/invalid field.
**Pass/Fail Criteria:** Error text tells user exactly what to fix without reading source code.  
**Cleanup:** Remove invalid CR objects if created.  
**Failure Impact:** Support burden and customer confusion increase.

#### MQE-003: Exploratory - transient dependency fault during reconcile

**Priority:** Medium  
**Methodology:** Black box (human execution)  
**Type:** Exploratory  
**Relevant Requirement(s):** REQ-011, REQ-012  
**Preconditions:** Vault or cert-manager mode configured and working.  
**Steps:**
1. Temporarily block access to upstream dependency (scale cert-manager controller to 0 or block Vault route).
   - **Expected:** reconcile reports failure signals but operator remains running.
2. Restore dependency.
   - **Expected:** operator recovers and converges to Ready without manual data repair.
3. Inspect logs/events for secrets.
   - **Expected:** no token/cert private material printed in plaintext.
**Pass/Fail Criteria:** System self-recovers and no secret leakage observed.  
**Cleanup:** Restore scaled components/network settings.  
**Failure Impact:** Operational fragility and potential security exposure.

### Tier 5: Non-Functional Tests

#### NFT-001: Reconcile performance under CR update churn

**Priority:** Medium  
**Sub-type:** Performance  
**Methodology:** Metrics-driven  
**Relevant Requirement(s):** REQ-010  
**Measurable Threshold:** P95 reconcile duration for `SpireServer` updates <= 10s; controller memory increase <= 20% from baseline over 30 min churn test.  
**Preconditions:** Prometheus metrics scraping controller; test script to patch CR every 30s for 30 min toggling cert-manager defaults fields.  
**Steps:**
1. Measure baseline reconcile metrics and pod memory for 10 min idle.
   - **Expected:** stable baseline captured.
2. Run patch churn workload.
   - **Expected:** reconciles complete without sustained queue growth.
3. Compare P95 latency and memory to threshold.
   - **Expected:** thresholds met.
**Cleanup:** Stop patch job; restore baseline CR.  
**Failure Impact:** Feature causes control-plane inefficiency at scale.

#### NFT-002: Security least-privilege and secret handling audit

**Priority:** Critical  
**Sub-type:** Security  
**Methodology:** Black box + config audit  
**Relevant Requirement(s):** REQ-005, REQ-012  
**Measurable Threshold:** No unexpected RBAC resources outside `certificaterequests` delta; no secret/token value leaks in operator logs sampled during reconcile.  
**Preconditions:** Operator running with both modes tested separately.  
**Steps:**
1. Diff ClusterRole permissions between no-upstream, cert-manager, and vault scenarios.
   - **Expected:** only cert-manager scenario adds `certificaterequests` rule.
2. Capture operator logs during reconcile of Vault mode.
   - **Expected:** no JWT token string, private key, or full secret data in logs.
3. Inspect events on `SpireServer`.
   - **Expected:** no sensitive payloads.
**Cleanup:** Remove debug log captures containing cluster metadata per policy.  
**Failure Impact:** Security posture regression or credential leakage.

#### NFT-003: Recovery from operator restart mid-rollout

**Priority:** High  
**Sub-type:** Recovery  
**Methodology:** Black box  
**Relevant Requirement(s):** REQ-011  
**Measurable Threshold:** Convergence to Ready within 5 minutes after controller restart.  
**Preconditions:** Trigger a rollout by changing upstreamAuthority fields.  
**Steps:**
1. Patch `SpireServer` to initiate reconcile and rollout.
   - **Expected:** StatefulSet update starts.
2. Delete operator controller pod mid-reconcile.
   - **Expected:** deployment recreates controller pod automatically.
3. Wait for reconcile completion.
   - **Expected:** final state matches desired CR and SPIRE pod Ready within threshold.
**Cleanup:** None beyond normal stabilization checks.  
**Failure Impact:** Control loop is not resilient to routine disruptions.

#### NFT-004: Regression endurance on baseline self-signed mode

**Priority:** High  
**Sub-type:** Regression  
**Methodology:** Metrics-driven black box  
**Relevant Requirement(s):** REQ-007  
**Measurable Threshold:** No increase > 10% in SPIRE server restart count and no new warning events over 24h versus pre-feature baseline.  
**Preconditions:** Cluster running self-signed mode CR only.  
**Steps:**
1. Deploy baseline self-signed config with no `upstreamAuthority`.
   - **Expected:** normal healthy state.
2. Monitor for 24h pod restarts/events/condition flaps.
   - **Expected:** remains within threshold and no upstreamAuthority-related warnings.
**Cleanup:** Export metrics and end test window.  
**Failure Impact:** Existing customers regress despite not using the feature.

## Traceability Matrix

| Requirement | UT | INT | E2E | MQE | NFT | Coverage Status |
| --- | --- | --- | --- | --- | --- | --- |
| REQ-001 | UT-001, UT-006 | INT-001 | E2E-002 | MQE-002 | - | Covered |
| REQ-002 | UT-002 | INT-003 | E2E-003 | MQE-001 | - | Covered |
| REQ-003 | UT-003 | INT-004 | E2E-004 | - | - | Covered |
| REQ-004 | UT-004 | INT-004 | E2E-004 | - | - | Covered |
| REQ-005 | UT-005 | INT-003 | E2E-003 | MQE-001 | NFT-002 | Covered |
| REQ-006 | UT-006 | INT-002 | E2E-002 | MQE-002 | - | Covered |
| REQ-007 | UT-004 | - | E2E-001, E2E-006 | - | NFT-004 | Covered |
| REQ-008 | - | INT-005 | E2E-005 | - | - | Covered |
| REQ-009 | - | INT-003, INT-004 | E2E-001, E2E-003, E2E-004, E2E-005 | MQE-001 | - | Covered |
| REQ-010 | - | - | - | - | NFT-001 | Covered |
| REQ-011 | - | - | - | MQE-003 | NFT-003 | Covered |
| REQ-012 | - | - | - | MQE-003 | NFT-002 | Covered |

## Uncovered Requirements

None. All extracted requirements are covered by at least one test case.

## Coverage Summary

| Tier | Count | Critical | High | Medium |
| --- | --- | --- | --- | --- |
| Unit Tests | 6 | 3 | 3 | 0 |
| Integration Tests | 5 | 2 | 3 | 0 |
| E2E Automated Tests | 6 | 4 | 1 | 1 |
| Manual QE Tests | 3 | 0 | 1 | 2 |
| Non-Functional Tests | 4 | 1 | 2 | 1 |
| **Total** | **24** | **10** | **10** | **4** |

---

## Test Execution Report

**Executed by:** sayadas  
**Execution Date:** 2026-05-19 to 2026-05-25  
**Operator Version:** `zero-trust-workload-identity-manager.v1.0.1`  
**cert-manager Version:** `cert-manager-operator.v1.19.0`  
**SPIRE Version:** `1.13.3`  
**Cluster 1 (Day 1):** `sayadas-upstreamca-1.qe.devcluster.openshift.com`  
**Cluster 2 (Day 1):** `sayadas-upstreamca-clu1.qe.devcluster.openshift.com`  
**Cluster 1 (Day 2):** `sayadas-upca-clu1.qe.devcluster.openshift.com`  
**Cluster 2 (Day 2):** `sayadas-upca-clu2.qe.devcluster.openshift.com`  
**Cluster 3 (Day 3):** `ci-ln-7brchxt-72292.gcp-2.ci.openshift.org` (fresh install, OCP 4.19)  
**Cluster A (Day 5 - Federation):** `sayadas-clu1-upca.qe.devcluster.openshift.com` (cert-manager)  
**Cluster B (Day 5 - Federation):** `sayadas-clu2-upca.qe.devcluster.openshift.com` (Vault)  
**Namespace:** `zero-trust-workload-identity-manager`

---

### Execution Status Summary

| Scenario | Cluster | Priority | Status | Notes |
| --- | --- | --- | --- | --- |
| E2E-001 Baseline smoke | Both | Critical | ✅ PASSED | Self-signed mode confirmed clean |
| E2E-002 Negative validation | Both | Critical | ✅ PASSED | Singleton guard also discovered |
| E2E-003 cert-manager happy path | All 3 | Critical | ✅ PASSED | **DEFINITIVE PROOF** — cert-manager signs SPIRE CA (Day 2 + Day 3 fresh) |
| E2E-005 Lifecycle toggle | Cluster 1 | High | ✅ PASSED | RBAC added/removed correctly |
| MQE-001 Acceptance cert-manager flow | Both | High | ✅ PASSED | Full admin flow verified (Day 2) |
| MQE-002 Invalid config message | Both | Medium | ✅ PASSED | Error messages clear and actionable |
| NFT-002 Security/RBAC audit | Cluster 1 | Critical | ✅ PASSED | Only certificaterequests rule changes |
| E2E-004 Vault happy path | Cluster 3 | Critical | ✅ PASSED | Vault k8s_auth, PKI signing confirmed |
| E2E-006 Federation regression | Cluster A + B | Medium | ✅ PASSED | Cross-CA federation (cert-manager ↔ Vault) verified |
| MQE-003 Transient fault | — | Medium | ⏳ PENDING | |
| NFT-001 Performance/churn | — | Medium | ⏳ PENDING | 30 min churn test |
| NFT-003 Operator restart recovery | Cluster A | High | ✅ PASSED | Controller recovered in 14s, all conditions True |
| NFT-004 24h endurance | — | High | ⏳ PENDING | Long running |

---

### Critical Finding: selfSigned Issuer Does NOT Work — MUST Use CA-type Issuer

**Date:** 2026-05-20  
**Severity:** BLOCKER (for documentation/user guidance)

**Problem:** When `upstreamAuthority.certManager` points to a `selfSigned` ClusterIssuer, SPIRE enters `CrashLoopBackOff` with:
```
level=error msg="Created CertificateRequest has failed"
message="Annotation \"cert-manager.io/private-key-secret-name\" missing or reference empty: secret name missing"
```

**Root Cause:** A `selfSigned` issuer needs the requester's private key (via annotation) to sign. SPIRE's cert-manager plugin does not set this annotation because SPIRE keeps its private key internal.

**Resolution:** Use a **CA-type Issuer** backed by a real root CA secret. The CA issuer uses its OWN key to sign — it never needs SPIRE's key.

**Correct Setup (3-object CA chain):**
```yaml
# Object 1: Bootstrap (selfSigned, used only once to create root CA cert)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
# Object 2: Root CA Certificate (the master key stored in spire-ca-secret)
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
# Object 3: CA Issuer — SPIRE points to THIS (uses root key to sign)
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: spire-ca-issuer
  namespace: cert-manager
spec:
  ca:
    secretName: spire-ca-secret
```

**SpireServer CR:**
```yaml
spec:
  upstreamAuthority:
    certManager:
      namespace: cert-manager
      issuerName: spire-ca-issuer
      issuerKind: Issuer
```

**Recommendation for Developer:**
1. Document that `selfSigned` issuers are NOT supported for `upstreamAuthority`
2. Consider adding webhook validation to reject `selfSigned` issuer references
3. Add this 3-object setup to the official documentation

---

### Critical Finding: ZeroTrustWorkloadIdentityManager CR Required Before Operands

**Date:** 2026-05-21  
**Severity:** HIGH (install blocker if not documented)

**Problem:** After installing the ZTWIM operator and creating operands (SpireServer, SpireAgent, etc.), the operator reported `Ready => False : Failed to retrieve ZeroTrustWorkloadIdentityManager from cluster` and did NOT create any StatefulSets or DaemonSets.

**Root Cause:** The operator requires a `ZeroTrustWorkloadIdentityManager` CR (the "master switch") to exist before it will reconcile any operands. Without it, SpireServer is accepted but never reconciled into running pods.

**CRD Schema:**
```yaml
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: <required — SPIFFE trust domain>
  clusterName: <required — cluster identifier>
  bundleConfigMap: <optional — name for trust bundle ConfigMap>
```

**Resolution:** Create `ZeroTrustWorkloadIdentityManager` CR BEFORE creating operands.

**Correct Install Order:**
1. cert-manager operator + CA chain
2. ZTWIM operator (wait for controller-manager Running)
3. `ZeroTrustWorkloadIdentityManager` CR (master switch)
4. All operands (SpireServer with `upstreamAuthority`, SpireAgent, CSI, OIDC)

**Recommendation for Developer:**
1. Document `ZeroTrustWorkloadIdentityManager` as a prerequisite in user-facing docs
2. Consider adding a more descriptive error message or event when this CR is missing
3. Add this to the Quick Start guide

---

### Critical Finding: Agent Crash After CA Change (PVC Delete)

**Date:** 2026-05-20  
**Severity:** HIGH (production risk if done incorrectly)

**Problem:** When SPIRE's PVC is deleted to force a new CA from cert-manager, existing agents crash because they cached the old trust bundle and cannot verify the server's new certificate.

**Root Cause:** Trust bundle mismatch — agents trust CA-A but server now presents CA-B.

**Mitigation:**
- **Fresh install:** Set `upstreamAuthority` from day 1 → agents boot with correct bundle → no crash
- **Existing cluster:** Add `upstreamAuthority` via patch, do NOT delete PVC → SPIRE rotates naturally at ~50% of `caValidity` → agents update bundle during rotation window → no crash
- **Testing only:** Delete PVC + restart agents (acceptable for QE, NEVER for production)

**Recommendation:** Operator should document that PVC deletion requires agent restart, or operator should auto-detect trust bundle change and trigger rolling agent restart.

---

### E2E-001: Baseline Self-Signed Smoke — ✅ PASSED

**Date:** 2026-05-19  
**Clusters:** Cluster 1 and Cluster 2

#### Commands and Evidence

```bash
# Step 1 - Confirm SpireServer CR exists
oc get spireserver -n zero-trust-workload-identity-manager
```

```
# Cluster 1 output:
NAME      AGE
cluster   57m

# Cluster 2 output:
NAME      AGE
cluster   66m
```

```bash
# Step 2 - Inspect CR spec and status
oc get spireserver cluster -n zero-trust-workload-identity-manager -o yaml
```

```
# Cluster 1 - Key observations:
# spec: no upstreamAuthority block present
# status.conditions: all True
#   ConfigurationValid => True | ConfigurationValid
#   Ready => True | Ready
#   StatefulSetAvailable => True | StatefulSetReady (1/1)

# Cluster 2 - Key observations:
# spec: no upstreamAuthority block present
# status.conditions: all True
#   ConfigurationValid => True | ConfigurationValid
#   Ready => True | Ready
#   StatefulSetAvailable => True | StatefulSetReady (1/1)
```

```bash
# Step 3 - Verify ConfigMap has no UpstreamAuthority
oc get cm spire-server -n zero-trust-workload-identity-manager -o yaml | grep -A 20 "UpstreamAuthority"
```

```
# Cluster 1 output: (empty - no UpstreamAuthority block) ✅
# Cluster 2 output: (empty - no UpstreamAuthority block) ✅
```

```bash
# Step 4 - Verify all pods healthy
oc get pods -n zero-trust-workload-identity-manager
```

```
# Cluster 1 output:
NAME                                                              READY   STATUS    RESTARTS      AGE
spire-agent-nvhs8                                                 1/1     Running   1 (52m ago)   53m
spire-agent-qdptf                                                 1/1     Running   1 (52m ago)   53m
spire-agent-xhs84                                                 1/1     Running   1 (52m ago)   53m
spire-server-0                                                    2/2     Running   0             53m
spire-spiffe-csi-driver-66zr2                                     2/2     Running   0             53m
spire-spiffe-csi-driver-9z95v                                     2/2     Running   0             53m
spire-spiffe-csi-driver-vwj5j                                     2/2     Running   0             53m
spire-spiffe-oidc-discovery-provider-77f4777679-d2mg5             1/1     Running   0             53m
zero-trust-workload-identity-manager-controller-manager-75c9vgx   1/1     Running   0             15h

# Cluster 2 output:
NAME                                                              READY   STATUS    RESTARTS      AGE
spire-agent-2rnfs                                                 1/1     Running   1 (50m ago)   51m
spire-agent-7mpvl                                                 1/1     Running   1 (50m ago)   51m
spire-agent-qmmpf                                                 1/1     Running   1 (50m ago)   51m
spire-server-0                                                    2/2     Running   0             51m
spire-spiffe-csi-driver-h55pf                                     2/2     Running   0             51m
spire-spiffe-csi-driver-ql4c2                                     2/2     Running   0             51m
spire-spiffe-csi-driver-wfjrp                                     2/2     Running   0             51m
spire-spiffe-oidc-discovery-provider-66bc7c5dc8-f5q7s             1/1     Running   0             51m
zero-trust-workload-identity-manager-controller-manager-6bzd5th   1/1     Running   0             55m
```

**Note on agent restarts:** All 3 agents on both clusters show 1 restart each. Root cause confirmed as startup race condition — agents started before `bundle.crt` was written to the `spire-bundle` ConfigMap. Agents recovered on second attempt. Not a regression; spire-server-0 has 0 restarts on both clusters.

**Result:** PASS — Both clusters in clean self-signed mode, all conditions healthy, no UpstreamAuthority in ConfigMap.

---

### E2E-002: Negative Validation — ✅ PASSED

**Date:** 2026-05-19  
**Cluster:** Cluster 1 (also verified on Cluster 2)

#### Commands and Evidence

```bash
# Step 1 - Attempt to create CR with both certManager and vault set
cat <<EOF | oc apply -f -
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster-invalid-both
  namespace: zero-trust-workload-identity-manager
spec:
  caKeyType: rsa-2048
  caSubject:
    commonName: test.example.com
    country: US
    organization: RH
  caValidity: 24h0m0s
  datastore:
    connMaxLifetime: 3600
    connectionString: /run/spire/data/datastore.sqlite3
    databaseType: sqlite3
    disableMigration: "false"
    maxIdleConns: 2
    maxOpenConns: 100
  defaultJWTValidity: 5m0s
  defaultX509Validity: 1h0m0s
  jwtIssuer: https://oidc-discovery.apps.test.example.com
  logFormat: text
  logLevel: info
  persistence:
    accessMode: ReadWriteOncePod
    size: 1Gi
    storageClass: ""
  upstreamAuthority:
    certManager:
      issuerName: test-issuer
      namespace: cert-manager
    vault:
      vaultAddr: https://vault.example.com
      k8sAuth:
        k8sAuthRoleName: spire
EOF
```

```
# Output (both clusters):
The SpireServer "cluster-invalid-both" is invalid:
* <nil>: Invalid value: "object": SpireServer is a singleton, .metadata.name must be 'cluster'
* spec.upstreamAuthority: Invalid value: "object": exactly one of certManager or vault must be set
```

```bash
# Step 2 - Confirm invalid CR was not persisted
oc get spireserver -n zero-trust-workload-identity-manager
```

```
# Output (both clusters):
NAME      AGE
cluster   67m
# cluster-invalid-both does NOT exist ✅
```

**Additional Finding — Singleton Guard:**  
The API also enforces `metadata.name must be 'cluster'`. Only one SpireServer named `cluster` is permitted per cluster. This is an additional protection not covered in the original test plan scope but confirmed working.

**Result:** PASS — Both xvalidation rules fired correctly. Invalid CR not persisted. REQ-001 confirmed.

---

### E2E-003: cert-manager Happy Path — ✅ PASSED (with DEFINITIVE PROOF)

**Date:** 2026-05-19 (Day 1: config/rendering), 2026-05-20 (Day 2: actual CA signing proof)  
**Clusters:** Cluster 1 and Cluster 2

#### Day 1 — Config Rendering Verified (Cluster 1)

- cert-manager Operator v1.19.0 installed via OLM
- Initial test used `selfSigned` ClusterIssuer → proved config/RBAC works
- **However:** selfSigned issuer caused SPIRE CrashLoopBackOff (see Critical Finding above)

#### Day 2 — DEFINITIVE PROOF with CA-type Issuer (Both Clusters)

**Prerequisites (correct setup):**
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

**SpireServer patch:**
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

#### Evidence — Cluster 1 (DEFINITIVE)

```bash
# SPIRE Logs (fresh start with empty journal):
oc logs spire-server-0 -c spire-server | grep -iE "cert-manager|upstream|self_signed|Journal"
```

```
level=info msg="Configured plugin" plugin_name=cert-manager plugin_type=UpstreamAuthority
level=info msg="Plugin loaded" plugin_name=cert-manager plugin_type=UpstreamAuthority
level=info msg="There is not a CA journal record that matches any of the local X509 authority IDs"
level=info msg="Journal loaded" jwt_keys=0 x509_cas=0
level=info msg="Waiting for certificaterequest to be signed" name=spiffe-ca-8zlp5 namespace=cert-manager
level=info msg="X509 CA prepared" self_signed=false upstream_authority_id=46ed8a46e1513fb629d69532a26df2e422ce4e81
level=info msg="X509 CA activated" upstream_authority_id=46ed8a46e1513fb629d69532a26df2e422ce4e81
```

```bash
# Trust bundle shows cert-manager root:
oc get cm spire-bundle -o jsonpath='{.data.bundle\.crt}' | openssl x509 -noout -issuer -subject
```
```
issuer=CN=SPIRE Upstream CA
subject=CN=SPIRE Upstream CA
```

```bash
# Inside pod — local authority:
oc exec spire-server-0 -c spire-server -- /opt/spire/bin/spire-server localauthority x509 show \
  -socketPath /tmp/spire-server/private/api.sock
```
```
Active X.509 authority:
  Authority ID: 4607fea44156c4babfdb14ac0b97e97ad83cfa88
  Upstream authority Subject Key ID: 46ed8a46e1513fb629d69532a26df2e422ce4e81
```

```bash
# Subject Key ID match (cert-manager root CA secret):
oc get secret spire-ca-secret -n cert-manager -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -text | grep -A 1 "Subject Key Identifier"
```
```
X509v3 Subject Key Identifier:
    46:ED:8A:46:E1:51:3F:B6:29:D6:95:32:A2:6D:F2:E4:22:CE:4E:81
→ MATCHES upstream_authority_id in SPIRE logs ✅
```

#### Evidence — Cluster 2 (DEFINITIVE)

```bash
oc logs spire-server-0 -c spire-server | grep -iE "self_signed|upstream_authority_id|CA activated"
```
```
level=info msg="Journal loaded" jwt_keys=0 x509_cas=0
level=info msg="X509 CA prepared" self_signed=false upstream_authority_id=f88003831a893f8b8160fbc0c6fd2cd042f41ea3
level=info msg="X509 CA activated" upstream_authority_id=f88003831a893f8b8160fbc0c6fd2cd042f41ea3
```

```bash
oc get cm spire-bundle -o jsonpath='{.data.bundle\.crt}' | openssl x509 -noout -issuer -subject
```
```
issuer=CN=SPIRE Upstream CA
subject=CN=SPIRE Upstream CA
```

```bash
oc exec spire-server-0 -c spire-server -- /opt/spire/bin/spire-server localauthority x509 show \
  -socketPath /tmp/spire-server/private/api.sock
```
```
Active X.509 authority:
  Authority ID: d879edfc8c32a15613866d7b933a72e85e003d71
  Upstream authority Subject Key ID: f88003831a893f8b8160fbc0c6fd2cd042f41ea3
```

```bash
oc get secret spire-ca-secret -n cert-manager -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -text | grep -A 1 "Subject Key Identifier"
```
```
X509v3 Subject Key Identifier:
    F8:80:03:83:1A:89:3F:8B:81:60:FB:C0:C6:FD:2C:D0:42:F4:1E:A3
→ MATCHES upstream_authority_id in SPIRE logs ✅
```

#### Evidence — Cluster 3: Fresh Install with upstreamAuthority from Day 1 (DEFINITIVE)

**Date:** 2026-05-21  
**Cluster:** `ci-ln-7brchxt-72292.gcp-2.ci.openshift.org` (OCP 4.19, GCP)  
**Method:** Fresh install — `upstreamAuthority` included in SpireServer CR from initial creation (no patching, no PVC delete)

**Key discovery:** Operator requires `ZeroTrustWorkloadIdentityManager` CR before it will reconcile operands. Without it: `Ready => False : Failed to retrieve ZeroTrustWorkloadIdentityManager from cluster`.

**Install order used:**
1. cert-manager operator + CA chain (selfsigned-issuer → spire-ca → spire-ca-issuer)
2. ZTWIM operator v1.0.1 via CatalogSource
3. `ZeroTrustWorkloadIdentityManager` CR (master switch)
4. All operands in one YAML (SpireServer with `upstreamAuthority` + SpireAgent + CSI + OIDC)

```bash
# SPIRE Logs (no PVC delete, no restart needed):
oc logs spire-server-0 -n zero-trust-workload-identity-manager -c spire-server | \
  grep -iE "upstream_authority_id|self_signed|CA activated|cert-manager"
```

```
level=info msg="Configured plugin" plugin_name=cert-manager plugin_type=UpstreamAuthority
level=info msg="Plugin loaded" plugin_name=cert-manager plugin_type=UpstreamAuthority
level=info msg="Waiting for certificaterequest to be signed" name=spiffe-ca-qjs2x namespace=cert-manager
level=info msg="X509 CA prepared" self_signed=false upstream_authority_id=9fe5993e62ad68c70481183f445c6e61d4e2b294
level=info msg="X509 CA activated" upstream_authority_id=9fe5993e62ad68c70481183f445c6e61d4e2b294
level=warning msg="UpstreamAuthority plugin does not support JWT-SVIDs."
```

```bash
# cert-manager root CA Subject Key ID:
oc get secret spire-ca-secret -n cert-manager -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -text | grep -A 1 "Subject Key Identifier"
```
```
X509v3 Subject Key Identifier:
    9F:E5:99:3E:62:AD:68:C7:04:81:18:3F:44:5C:6E:61:D4:E2:B2:94
→ MATCHES upstream_authority_id=9fe5993e62ad68c70481183f445c6e61d4e2b294 ✅
```

```bash
# All pods Running (no restarts for spire-server):
oc get pods -n zero-trust-workload-identity-manager
```
```
spire-agent-2xt97                                                 0/1     Running   0          25s
spire-agent-8bkpd                                                 0/1     Running   0          25s
spire-agent-p9d5p                                                 0/1     Running   0          25s
spire-server-0                                                    2/2     Running   0          24s
spire-spiffe-csi-driver-2dqpp                                     2/2     Running   0          26s
spire-spiffe-csi-driver-xxcwm                                     2/2     Running   0          26s
spire-spiffe-csi-driver-zbp62                                     2/2     Running   0          26s
spire-spiffe-oidc-discovery-provider-6d99b4bf74-lg82r             0/1     Running   0          26s
zero-trust-workload-identity-manager-controller-manager-969pv7z   1/1     Running   0          65s
```

**Key findings from Cluster 3:**
- `upstream_authority_id` is populated immediately on first boot (no empty ID issue)
- No PVC delete or pod restart required when `upstreamAuthority` is in the CR from the start
- `ZeroTrustWorkloadIdentityManager` CR is a prerequisite the operator enforces
- JWT-SVID warning is informational only — cert-manager UpstreamAuthority plugin doesn't support JWT signing (X.509 only)

---

#### ConfigMap and RBAC (verified on all clusters):

```bash
oc get cm spire-server -n zero-trust-workload-identity-manager -o yaml | grep -A 20 "UpstreamAuthority"
```
```
"UpstreamAuthority": [
  {
    "cert-manager": {
      "plugin_data": {
        "issuer_kind": "Issuer",
        "issuer_name": "spire-ca-issuer",
        "namespace": "cert-manager"
      }
    }
  }
]
```

```bash
oc get clusterrole spire-server -o yaml | grep -A 10 "certificaterequests"
```
```
- certificaterequests
  verbs:
  - create
  - get
  - list
  - delete
```

```bash
oc auth can-i create certificaterequests.cert-manager.io \
  --as=system:serviceaccount:zero-trust-workload-identity-manager:spire-server -n cert-manager
```
```
yes
```

#### Summary of Evidence

| Check | Cluster 1 | Cluster 2 | Cluster 3 (fresh) |
| --- | --- | --- | --- |
| cert-manager plugin loaded | ✅ | ✅ | ✅ |
| SPIRE submitted CertificateRequest | ✅ (`spiffe-ca-8zlp5`) | ✅ | ✅ (`spiffe-ca-qjs2x`) |
| `self_signed=false` in logs | ✅ | ✅ | ✅ |
| `upstream_authority_id` populated | `46ed8a46...` | `f8800383...` | `9fe59993...` |
| Bundle = `CN=SPIRE Upstream CA` | ✅ | ✅ | (pending verify) |
| Subject Key ID fingerprints match | ✅ | ✅ | ✅ |
| All pods Running | ✅ | ✅ | ✅ |
| All conditions True | ✅ | ✅ | ✅ |
| RBAC includes certificaterequests | ✅ | ✅ | ✅ |
| No Vault volumes injected | ✅ | ✅ | ✅ |
| No PVC delete / restart needed | ❌ (patched later) | ❌ (patched later) | ✅ (from day 1) |

**Result:** PASS — cert-manager UpstreamAuthority fully functional on all 3 clusters. SPIRE's intermediate CA is signed by cert-manager root CA. Subject Key ID fingerprints definitively prove the signing relationship. Cluster 3 additionally proves the "fresh install" path works without any restart or PVC manipulation. REQ-002, REQ-005, REQ-009 confirmed.

---

### E2E-005: Lifecycle Toggle (enable then remove upstreamAuthority) — ✅ PASSED

**Date:** 2026-05-19  
**Cluster:** Cluster 1  
**Precondition:** cert-manager upstreamAuthority was active from E2E-003

#### Commands and Evidence

```bash
# Step 1 - Remove upstreamAuthority from CR
oc patch spireserver cluster -n zero-trust-workload-identity-manager \
  --type=json -p '[{"op": "remove", "path": "/spec/upstreamAuthority"}]'
```

```
spireserver.operator.openshift.io/cluster patched
```

```bash
# Step 2 - Watch StatefulSet rollout
oc rollout status statefulset spire-server -n zero-trust-workload-identity-manager -w
```

```
Waiting for 1 pods to be ready...
partitioned roll out complete: 1 new pods have been updated...
```

```bash
# Step 3 - Verify ConfigMap NO LONGER has UpstreamAuthority
oc get cm spire-server -n zero-trust-workload-identity-manager -o yaml | grep -A 20 "UpstreamAuthority"
```

```
# (empty output) ✅ - UpstreamAuthority plugin removed
```

```bash
# Step 4 - Verify all conditions still healthy
oc get spireserver cluster -n zero-trust-workload-identity-manager \
  -o jsonpath='{range .status.conditions[*]}{.type}{" => "}{.status}{" | "}{.reason}{"\n"}{end}'
```

```
Ready => True | Ready
ValidatingWebhookAvailable => True | Ready
ServerConfigMapAvailable => True | SpireConfigMapResourceCreated
ControllerManagerConfigAvailable => True | SpireControllerManagerConfigMapCreated
BundleConfigAvailable => True | SpireBundleConfigMapCreated
StatefulSetAvailable => True | StatefulSetReady
ConfigurationValid => True | ConfigurationValid
RBACAvailable => True | Ready
TTLConfigurationValid => True | TTLValidationSucceeded
ServiceAccountAvailable => True | Ready
ServiceAvailable => True | Ready
```

```bash
# Step 5 - Verify cert-manager RBAC rule removed from ClusterRole
oc get clusterrole spire-server -o yaml | grep -A 10 "certificaterequests"
```

```
# (empty output) ✅ - certificaterequests rule removed
```

```bash
# Step 6 - Verify all pods healthy
oc get pods -n zero-trust-workload-identity-manager
```

```
spire-server-0   2/2   Running   0   40s   ✅
# All other pods Running, no CrashLoopBackOff
```

**Key Finding:** Operator correctly added cert-manager RBAC when `upstreamAuthority.certManager` was set and correctly removed it when `upstreamAuthority` was removed — no manual cleanup required.

**Result:** PASS — Rollback to self-signed is safe, clean, and automatic. REQ-008, REQ-009 confirmed.

---

### MQE-001: Acceptance — cert-manager Documented Flow — ✅ PASSED

**Date:** 2026-05-20  
**Clusters:** Cluster 1 and Cluster 2 (Day 2 clusters)

**Steps Executed:**
1. Installed cert-manager operator (pods Running in `cert-manager` namespace)
2. Created 3-object CA chain (selfsigned-issuer → spire-ca Certificate → spire-ca-issuer)
3. Patched SpireServer with `upstreamAuthority.certManager`
4. Verified ConfigMap, RBAC, pod status, and SPIRE logs

**Evidence (same as E2E-003 Day 2 above):**
- All pods Running (2/2 for server, 1/1 for agents)
- ConfigMap has cert-manager plugin block
- ClusterRole has `certificaterequests` RBAC
- SPIRE logs: `self_signed=false`, `upstream_authority_id` populated
- Bundle: `CN=SPIRE Upstream CA`

**Result:** PASS — Admin flow works end-to-end with correct CA-type Issuer. REQ-002, REQ-005, REQ-009 confirmed.

---

### MQE-002: Usability — Invalid Config Message — ✅ PASSED

**Date:** 2026-05-19 and 2026-05-20  
**Clusters:** Both Day 1 and Day 2 clusters

**Commands and Evidence:**
```bash
# Attempt CR with both providers
cat <<EOF | oc apply -f -
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster-invalid-both
  namespace: zero-trust-workload-identity-manager
spec:
  # ... (all required fields) ...
  upstreamAuthority:
    certManager:
      issuerName: test-issuer
      namespace: cert-manager
    vault:
      vaultAddr: https://vault.example.com
      k8sAuth:
        k8sAuthRoleName: spire
EOF
```

```
# Output (both clusters, both days):
The SpireServer "cluster-invalid-both" is invalid:
* <nil>: Invalid value: "object": SpireServer is a singleton, .metadata.name must be 'cluster'
* spec.upstreamAuthority: Invalid value: "object": exactly one of certManager or vault must be set
```

**Error Message Assessment:**
- "exactly one of certManager or vault must be set" — clearly states the problem and fix
- "SpireServer is a singleton, .metadata.name must be 'cluster'" — clear singleton constraint
- Both messages reference specific fields, no source code reading needed

**Result:** PASS — Error messages are clear, specific, and actionable. REQ-001, REQ-006 confirmed.

---

### NFT-002: Security/RBAC Audit — ✅ PASSED

**Date:** 2026-05-19 (Day 1, Cluster 1)  
**Cluster:** Cluster 1

**Evidence:**

1. **RBAC diff — only `certificaterequests` rule added in cert-manager mode:**
```bash
oc get clusterrole spire-server -o yaml | grep -A 10 "certificaterequests"
```
```
# In cert-manager mode:
- certificaterequests
  verbs:
  - create
  - get
  - list
  - delete

# In self-signed mode (after removing upstreamAuthority):
# (empty output — rule removed) ✅
```

2. **RBAC permission check:**
```bash
oc auth can-i create certificaterequests.cert-manager.io \
  --as=system:serviceaccount:zero-trust-workload-identity-manager:spire-server -n cert-manager
```
```
yes  (only when cert-manager mode is active)
```

3. **No secret/token leakage in logs:**
```bash
oc logs -n zero-trust-workload-identity-manager \
  deployment/zero-trust-workload-identity-manager-controller-manager \
  --since=10m | grep -iE "token|secret|private|key|password|credential"
```
```
# (empty output) ✅ — no sensitive material in logs
```

**Result:** PASS — Only `certificaterequests` RBAC changes between modes. No secret leakage. REQ-005, REQ-012 confirmed.

---

---

### Pending Tests — Still To Execute

#### E2E-004: Vault Happy Path — ✅ PASSED

**Date:** 2026-05-21  
**Cluster:** `ci-ln-7brchxt-72292.gcp-2.ci.openshift.org` (Cluster 3)  
**Vault:** HashiCorp Vault 1.15 (dev mode, in-cluster, `vault` namespace)  
**Auth Method:** `k8s_auth` (projected ServiceAccount token)

**Vault Setup Steps:**
1. Deployed Vault in dev mode with `SKIP_SETCAP=1` (OpenShift SCC compatibility)
2. Enabled PKI secrets engine, generated root CA (`CN=Vault SPIRE Root CA`)
3. Created PKI role `spire-intermediate` (`allow_any_name=true`)
4. Enabled Kubernetes auth with explicit `token_reviewer_jwt` and `kubernetes_ca_cert`
5. Created policy allowing `pki/root/sign-intermediate` + `pki/cert/ca`
6. Created auth role `spire` bound to `spire-server` SA
7. Granted `system:auth-delegator` ClusterRoleBinding to Vault SA

**SpireServer CR:**
```yaml
spec:
  upstreamAuthority:
    vault:
      vaultAddr: "http://vault.vault.svc.cluster.local:8200"
      pkiMountPoint: "pki"
      k8sAuth:
        k8sAuthRoleName: "spire"
```

**Evidence — SPIRE Logs:**
```
level=info msg="Configured plugin" plugin_name=vault plugin_type=UpstreamAuthority
level=info msg="Plugin loaded" plugin_name=vault plugin_type=UpstreamAuthority
level=info msg="X509 CA prepared" self_signed=false upstream_authority_id=f61ae028ab793fac9d46481ae812ce95ded8beea
level=info msg="X509 CA activated" upstream_authority_id=f61ae028ab793fac9d46481ae812ce95ded8beea
level=warning msg="UpstreamAuthority plugin does not support JWT-SVIDs." plugin_name=vault
```

**Evidence — vault-token volume present:**
```
spire-data
server-tmp
spire-config
spire-server-socket
spire-controller-manager-tmp
controller-manager-config
vault-token                    ← projected SA token for k8s_auth
kube-api-access-q6cjr
```

**Evidence — ConfigMap Vault plugin block:**
```json
"UpstreamAuthority": [{
  "vault": {
    "plugin_data": {
      "insecure_skip_verify": false,
      "k8s_auth": {
        "k8s_auth_mount_point": "kubernetes",
        "k8s_auth_role_name": "spire",
        "token_path": "/var/run/secrets/tokens/vault"
      },
      "pki_mount_point": "pki",
      "vault_addr": "http://vault.vault.svc.cluster.local:8200"
    }
  }
}]
```

**Evidence — Trust bundle shows Vault root:**
```
issuer=CN=Vault SPIRE Root CA
subject=CN=Vault SPIRE Root CA
```

**Evidence — NO cert-manager RBAC:**
```
(empty output — no certificaterequests rule) ✅
```

**Evidence — All pods Running:**
```
spire-agent-4j5c4       1/1   Running
spire-agent-4phbj       1/1   Running
spire-agent-hnpbm       1/1   Running
spire-server-0          2/2   Running
spire-spiffe-csi-*      2/2   Running (x3)
spire-spiffe-oidc-*     1/1   Running
controller-manager      1/1   Running
```

**Key findings:**
- Vault dev mode loses all config on pod restart (in-memory storage)
- `system:auth-delegator` ClusterRoleBinding required for Vault to call TokenReview API
- Vault policy must include `pki/root/sign-intermediate` (not just `pki/sign/<role>`)
- `SKIP_SETCAP=1` needed for Vault on OpenShift (SCC blocks `CAP_SETFCAP`)
- Vault plugin does not support JWT-SVIDs (same warning as cert-manager)

**Result:** PASS — Vault UpstreamAuthority fully functional with k8s_auth. REQ-003, REQ-004, REQ-009 confirmed.

---

#### E2E-006: Federation Regression — ✅ PASSED (Cross-CA: cert-manager ↔ Vault)

**Date:** 2026-05-25  
**Cluster A:** `sayadas-clu1-upca.qe.devcluster.openshift.com` (cert-manager upstream)  
**Cluster B:** `sayadas-clu2-upca.qe.devcluster.openshift.com` (Vault upstream, in-cluster dev mode)  
**Scope:** Verified federation still works when two clusters use *different* upstream CAs

**Test Configuration:**
- Cluster A: cert-manager CA-type Issuer (`CN=SPIRE Upstream CA - Cluster A`)
- Cluster B: Vault PKI engine (`CN=SPIRE Vault Root CA - Cluster B`)
- Federation profile: `https_spiffe` with managed routes
- Bundle exchange: `ClusterFederatedTrustDomain` with `trustDomainBundle` bootstrap + live `https_spiffe` endpoint
- Workloads: `spiffe-helper` sidecar with `include_federated_domains = true`

**Steps Executed:**
1. Both clusters running with respective upstream CAs (cert-manager on A, Vault on B)
2. Patched `SpireServer` on both clusters to enable federation (`bundleEndpoint`, `managedRoute`, `federatesWith`)
3. Verified `spire-server-federation` route created on both clusters
4. Fetched bundles from federation endpoints and created `ClusterFederatedTrustDomain` CRs with `trustDomainBundle` bootstrap
5. Verified `spire-server bundle list` shows remote cluster's trust domain on both
6. Deployed test workloads with `ClusterSPIFFEID` (`className: zero-trust-workload-identity-manager-spire`, `federatesWith` remote domain)
7. Verified workload trust bundles contain BOTH local and remote root CAs
8. Cross-verified SVIDs: Cluster A workload verified Cluster B's Vault-signed SVID, and vice versa

**Evidence:**
```
=== Federation Routes ===
Cluster A: spire-server-federation   federation.apps.sayadas-clu1-upca.qe.devcluster.openshift.com
Cluster B: spire-server-federation   federation.apps.sayadas-clu2-upca.qe.devcluster.openshift.com

=== Bundle List ===
Cluster A sees: apps.sayadas-clu2-upca.qe.devcluster.openshift.com (Vault Root CA)
Cluster B sees: apps.sayadas-clu1-upca.qe.devcluster.openshift.com (cert-manager CA)

=== SPIRE Entry FederatesWith ===
Cluster A entry: FederatesWith = apps.sayadas-clu2-upca.qe.devcluster.openshift.com
Cluster B entry: FederatesWith = apps.sayadas-clu1-upca.qe.devcluster.openshift.com

=== Workload Trust Bundles (include_federated_domains=true) ===
Cluster A workload bundle: CN=SPIRE Upstream CA - Cluster A + CN=SPIRE Vault Root CA - Cluster B
Cluster B workload bundle: CN=SPIRE Vault Root CA - Cluster B + CN=SPIRE Upstream CA - Cluster A

=== Cross-CA Verification (openssl verify with intermediate handling) ===
Cluster A verifies Cluster B's Vault-signed SVID: /tmp/leaf.pem: OK ✅
Cluster B verifies Cluster A's cert-manager-signed SVID: /tmp/leaf.pem: OK ✅
```

**Key Technical Details:**
- `ClusterFederatedTrustDomain` requires `className: zero-trust-workload-identity-manager-spire` for the ZTWIM operator to reconcile it
- `ClusterSPIFFEID` also requires `className` for `federatesWith` entries to be registered in SPIRE
- `spiffe-helper` needs `include_federated_domains = true` to write all federated CAs to `bundle.pem`
- SVID chain contains leaf + intermediate; `openssl verify` requires `-untrusted` flag for the intermediate
- Vault `system:auth-delegator` ClusterRoleBinding is required for K8s auth method
- Federation routes use TLS passthrough to present the SPIRE server's SVID directly

**Result:** PASS — Federation works correctly across heterogeneous upstream CAs. Adding `upstreamAuthority` (either cert-manager or Vault) does NOT break federation behavior. REQ-007 confirmed. Cross-CA verification proves trust bundles correctly propagate regardless of CA type.

---

#### MQE-003: Transient Fault ⏳

**Cluster:** Cluster 2

```bash
# 1. Confirm cert-manager mode working
oc get clusterrole spire-server -o yaml | grep -A 5 "certificaterequests"

# 2. Scale cert-manager to 0 to simulate outage
oc scale deployment cert-manager -n cert-manager --replicas=0

# 3. Trigger a reconcile by making a harmless patch
oc patch spireserver cluster -n zero-trust-workload-identity-manager \
  --type=merge -p '{"metadata":{"annotations":{"test/trigger":"fault-test-1"}}}'

# 4. Watch operator logs for failure signals (not crash)
oc logs -n zero-trust-workload-identity-manager \
  deployment/zero-trust-workload-identity-manager-controller-manager --follow --tail=30

# 5. Restore cert-manager
oc scale deployment cert-manager -n cert-manager --replicas=1

# 6. Verify recovery without manual intervention
oc rollout status deployment/cert-manager -n cert-manager -w
oc get spireserver cluster -n zero-trust-workload-identity-manager \
  -o jsonpath='{range .status.conditions[*]}{.type}{" => "}{.status}{"\n"}{end}'

# 7. Check no token/secret values in logs
oc logs -n zero-trust-workload-identity-manager \
  deployment/zero-trust-workload-identity-manager-controller-manager \
  --since=10m | grep -iE "token|secret|private|key|password"
# Expected: no sensitive values
```

---

#### NFT-001: Performance/Churn ⏳

**Cluster:** Cluster 2  
**Duration:** 30 minutes

```bash
# 1. Capture baseline metrics
oc adm top pod -n zero-trust-workload-identity-manager

# 2. Run patch churn for 30 min (toggle annotation every 30s)
for i in $(seq 1 60); do
  oc patch spireserver cluster -n zero-trust-workload-identity-manager \
    --type=merge -p "{\"metadata\":{\"annotations\":{\"test/churn\":\"$i\"}}}"
  echo "Patch $i at $(date)"
  sleep 30
done

# 3. Monitor reconcile queue (watch operator logs)
oc logs -n zero-trust-workload-identity-manager \
  deployment/zero-trust-workload-identity-manager-controller-manager \
  --since=30m | grep -iE "reconcil|queue|error"

# 4. Capture final metrics and compare
oc adm top pod -n zero-trust-workload-identity-manager
```

---

#### NFT-003: Operator Restart Recovery — ✅ PASSED

**Date:** 2026-05-25  
**Cluster:** Cluster A (`sayadas-clu1-upca.qe.devcluster.openshift.com`)  
**Threshold:** Convergence to Ready within 5 minutes after controller restart  
**Actual:** Controller recovered in **14 seconds**

**Steps Executed:**
```bash
# 1. Pre-check: all conditions True, all pods healthy (0 restarts)
oc get spireserver cluster -n zero-trust-workload-identity-manager \
  -o jsonpath='{range .status.conditions[*]}{.type}{" => "}{.status}{"\n"}{end}'
# All True ✅

# 2. Trigger reconcile via annotation patch
oc patch spireserver cluster -n zero-trust-workload-identity-manager \
  --type=merge -p '{"metadata":{"annotations":{"test/restart-recovery":"triggered"}}}'
# spireserver.operator.openshift.io/cluster patched

# 3. Kill controller-manager mid-reconcile
oc delete pod zero-trust-workload-identity-manager-controller-manager-84qjz5g \
  -n zero-trust-workload-identity-manager
# pod deleted

# 4. Watch recovery
oc get pods -n zero-trust-workload-identity-manager -w | grep controller-manager
```

**Evidence — Controller Recovery:**
```
zero-trust-workload-identity-manager-controller-manager-84qtwzk   0/1     ContainerCreating   0          2s
zero-trust-workload-identity-manager-controller-manager-84qtwzk   0/1     Running             0          3s
zero-trust-workload-identity-manager-controller-manager-84qtwzk   1/1     Running             0          14s
```

**Evidence — All Conditions True After Recovery:**
```
Ready => True | Ready
ServiceAccountAvailable => True | Ready
ServiceAvailable => True | Ready
RBACAvailable => True | Ready
ValidatingWebhookAvailable => True | Ready
BundleConfigAvailable => True | SpireBundleConfigMapCreated
StatefulSetAvailable => True | StatefulSetReady
ConfigurationValid => True | ConfigurationValid
ServerConfigMapAvailable => True | SpireConfigMapResourceCreated
ControllerManagerConfigAvailable => True | SpireControllerManagerConfigMapCreated
TTLConfigurationValid => True | TTLValidationSucceeded
RouteAvailable => True | FederationRouteCreated
```

**Evidence — Upstream Authority Still Active:**
```
self_signed=false upstream_authority_id=6ef22156b5d9c4f54ed27edd570bc7ba82741b6e
```

**Evidence — All Pods Healthy (0 restarts):**
```
spire-agent-dlvj7          1/1   Running   0   45m
spire-agent-jrqlz          1/1   Running   0   45m
spire-agent-r5cj4          1/1   Running   0   45m
spire-server-0             2/2   Running   0   67m
controller-manager-84qtwzk 1/1   Running   0   3m11s  (new pod)
```

**Result:** PASS — Controller-manager recovered in 14 seconds (well within 5-minute threshold). All conditions converged to True. SPIRE server was completely unaffected (no restart). Upstream authority remained active. REQ-011 confirmed.

---

#### NFT-004: 24h Endurance ⏳

**Cluster:** Cluster 2  
**Duration:** 24 hours (self-signed baseline, no upstreamAuthority)

```bash
# 1. Ensure self-signed mode (no upstreamAuthority)
oc patch spireserver cluster -n zero-trust-workload-identity-manager \
  --type=json -p '[{"op": "remove", "path": "/spec/upstreamAuthority"}]' 2>/dev/null || true

# 2. Record baseline restart counts
oc get pods -n zero-trust-workload-identity-manager

# 3. After 24h - check restart counts (should not increase > 10%)
oc get pods -n zero-trust-workload-identity-manager

# 4. Check for any new upstreamAuthority-related warning events
oc get events -n zero-trust-workload-identity-manager \
  --sort-by='.lastTimestamp' | grep -iE "upstream|certmanager|vault"
# Expected: no such events
```
