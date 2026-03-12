# Postmortem: GKE Boutique App Complete Outage — Productcatalogservice CPU Starvation

**Date:** 2026-03-12
**Severity:** P1
**Duration:** ~5 minutes (06:50–06:55 UTC-6)
**Authors:** Pablo Perez Quevedo, Platform Team
**Status:** Draft
**Jira:** AICR-246

---

## Executive Summary

A misconfigured Kubernetes manifest (commit 3a293029) set the productcatalogservice pod CPU limit to 1 millicore (1m) with an aggressive liveness probe (2-second period, 1-second timeout, 1-failure threshold). This caused immediate pod failures, cascading to all dependent services (frontend, checkout, payment, etc.), resulting in a complete application outage lasting approximately 5 minutes. Root cause was identified and remediated within 5 minutes of incident onset.

---

## Impact

**Users affected:** All active users (100% of traffic)
**Features unavailable:** All microservices; frontend HTTP 500 on every request
**Error rate:** 100% for duration of incident
**Duration:** 2026-03-12 06:50–06:55 UTC-6 (5 minutes total)
**SLA impact:** P1 incident — 30-minute response SLA (requirement: resolve within 30 min; actual: 5 min)
**Revenue impact:** To be assessed in separate financial review
**Data integrity:** No — no data loss or corruption; purely availability issue

---

## Timeline

| Time (UTC-6) | Event |
|---|---|
| 06:50 | Commit 3a293029 deployed to prod GKE cluster; productcatalogservice Deployment applied |
| 06:50:30 | First productcatalogservice pod created; liveness probe fails within 2 seconds (timeout: 1s, pod cannot execute gRPC health check with only 1m CPU) |
| 06:51 | Pods begin rapid restart loop; kubelet marks pod as CrashLoopBackOff |
| 06:51:15 | Frontend pod creation blocked (cannot reach productcatalogservice); HTTP 500 begins |
| 06:52 | All 12 microservices unable to start due to cascading dependency failures |
| 06:52:30 | Incident reported / alert fired (estimated time) |
| 06:53 | Root cause diagnosed: CPU limit set to 1m in commit 3a293029 |
| 06:53:45 | kubernetes-manifests/productcatalogservice.yaml corrected (CPU 1m → 500m, liveness probe restored) |
| 06:54 | Corrected manifest applied via kubectl apply |
| 06:54:30 | Productcatalogservice pod reaches Running state; liveness probe passing |
| 06:55 | All 12 services reached Running; frontend HTTP 200 verified |

**Total incident duration:** ~5 minutes
**Total response time:** ~3 minutes (detection to mitigation)

---

## Root Cause

**Primary cause:** Commit 3a293029 introduced a misconfigured resource specification to the productcatalogservice Deployment:

```yaml
# BROKEN (commit 3a293029)
resources:
  limits:
    cpu: 1m       # 1 millicore — far too low
    memory: 128Mi
livenessProbe:
  periodSeconds: 2       # Check every 2 seconds
  timeoutSeconds: 1      # Timeout after 1 second
  failureThreshold: 1    # Restart pod on first failure
```

**Mechanism:**
1. Container allocated 1m CPU = 1/1000th of a CPU core
2. Kubelet starts gRPC health check at 2-second intervals
3. gRPC health check cannot complete within 1-second window with 1m CPU (context switches, GC, scheduling delays)
4. Probe times out → failure count = 1
5. failureThreshold: 1 is met → pod immediately restarts
6. Pod enters infinite CrashLoopBackOff

**Cascade:**
- Productcatalogservice cannot start → cannot accept traffic
- Frontend tries to call productcatalogservice → connection refused
- Frontend pod creation blocked (dependent service failure)
- All transitive dependencies fail to start
- Entire application unavailable

---

## Contributing Factors

### 1. No manifest validation in CI/CD pipeline

Commit 3a293029 was merged and deployed without any pre-deployment checks to validate:
- CPU limit >= 100m (GKE Autopilot minimum)
- Liveness probe timeouts are realistic relative to CPU allocation
- Resource limits haven't regressed from prior commits

**Why:** Manifest validation was not enforced in the deploy pipeline at the time of incident.

### 2. Insufficient code review process for infrastructure changes

The Deployment manifest change (1m CPU, aggressive liveness probe) did not receive explicit acknowledgment from a reviewer. It appears to have been a debug configuration that was never intended for production.

**Why:** Code review checklist does not explicitly require sign-off on resource limit or health probe changes.

### 3. Lack of pre-prod validation / staging environment

The corrected manifest was not tested in a staging environment before applying to production.

**Why:** Deployment process does not enforce `kubectl diff` or dry-run validation against a staging cluster.

### 4. Absence of documentation on GKE Autopilot resource minimums

The productcatalogservice manifest and deployment runbooks did not explicitly document that GKE Autopilot has minimum resource requirements (CPU >= 100m, memory >= 128Mi for most workloads).

**Why:** No runbook or documentation existed for this failure class.

---

## What Went Well

1. **Rapid detection:** Incident was detected immediately upon pod startup failure (within ~30 seconds of deployment)
2. **Root cause diagnosis:** The correlation between 1m CPU limit and liveness probe timeout was identified within 3 minutes
3. **Low-risk fix:** Remediation was a simple configuration revert to a known stable state (no code changes required)
4. **Fast mitigation:** Pods recovered to healthy state within 1 minute of manifest correction
5. **No data loss:** Purely an availability issue; no persistent data was corrupted or lost
6. **Clear error messages:** Kubernetes events and pod logs clearly indicated liveness probe failures (good observability)

---

## What Went Poorly

1. **No pre-deployment manifest validation** — The broken configuration was deployed to production without pipeline checks
2. **Aggressive liveness probe timing** — 2-second period + 1-second timeout + 1-failure threshold is too aggressive for any workload
3. **Lack of awareness of GKE constraints** — Insufficient documentation that 1m CPU is not viable for gRPC services
4. **Missing code review** — Infrastructure change (CPU limit, probe timing) did not receive explicit acknowledgment
5. **No staging validation step** — Manifest was not tested in a pre-prod environment before applying to prod
6. **Insufficient runbook coverage** — No existing runbook for "pod liveness probe failures" to accelerate diagnosis

---

## Action Items

| Action | Owner | Due | Priority | Status |
|---|---|---|---|---|
| Add manifest validation to CI/CD: reject CPU < 100m | Platform / DevOps | 2026-03-14 | P1 | Pending |
| Create Kubernetes manifest linting policy (kubeval/Kyverno) | Platform | 2026-03-15 | P1 | Pending |
| Add code review checklist item: resource limit sign-off | Engineering Lead | 2026-03-13 | P2 | Pending |
| Document GKE Autopilot minimum resource requirements | Platform / Docs | 2026-03-14 | P2 | Pending |
| Add `kubectl diff` dry-run validation to deployment pipeline | DevOps | 2026-03-16 | P2 | Pending |
| Create runbook: GKE pod liveness probe failures | SRE | 2026-03-12 | P1 | DONE |
| Schedule post-incident review meeting | Incident Commander | 2026-03-13 | P1 | Pending |
| Audit other service manifests for similar resource constraints | Platform | 2026-03-14 | P2 | Pending |

---

## Lessons Learned

### 1. Infrastructure changes require the same rigor as code changes

Resource limits, health probe timings, and other Kubernetes configuration changes have direct impact on availability. These must go through the same review, testing, and validation pipeline as application code.

**Transferable insight:** Treat manifest changes as code. Require explicit sign-off, automated validation, and staging environment testing.

### 2. Liveness probe timings must scale with resource allocation

A liveness probe with a 1-second timeout is only viable for services with generous CPU allocation (>= 500m). Services with constrained resources (< 200m) need longer probe windows.

**Transferable insight:** Create a policy matrix: CPU allocation → minimum viable probe timeout. Document in deployment guide.

### 3. GKE Autopilot enforces stricter constraints than standard GKE

GKE Autopilot has implicit minimum resource requirements. Violating these constraints causes pods to fail immediately, with no graceful degradation.

**Transferable insight:** Maintain a runbook of GKE Autopilot-specific requirements. Review this during architecture reviews for new services.

### 4. Debug/test configurations should never reach main branch

Commit 3a293029 appears to be a debug configuration that was never intended for production. The presence of a 1m CPU limit and 2-second probe interval indicates testing-only code.

**Transferable insight:** Enforce a branch protection rule: any commit modifying resource limits or health probes must be reviewed and approved before merging to main. Consider automated alerts for "suspicious" values (e.g., CPU < 100m, timeout < 3s).

### 5. Cascading failure patterns are predictable — leverage runbooks

The symptom pattern (one service fails → dependent services also fail) is a classic microservice failure mode. A pre-written runbook for this pattern could have accelerated diagnosis from 3 minutes to < 1 minute.

**Transferable insight:** Build a library of failure patterns and corresponding runbooks. Classify by symptom (pod won't start, connection refused, timeout, high error rate) rather than by service name.

---

## Prevention Summary

**Short-term (< 1 week):**
1. Add manifest validation gate in CI/CD pipeline (reject CPU < 100m)
2. Create runbook for pod liveness probe failures
3. Audit all service manifests for CPU < 100m constraints

**Medium-term (< 2 weeks):**
1. Integrate Kubernetes manifest linting (kubeval or Kyverno)
2. Add code review checklist for infrastructure changes
3. Document GKE Autopilot minimum requirements in deployment guide
4. Implement `kubectl diff` dry-run in deployment pipeline

**Long-term (< 4 weeks):**
1. Evaluate policy-as-code framework (Kyverno, OPA/Rego) for runtime enforcement
2. Build comprehensive runbook library for common failure patterns
3. Create e2e test suite that validates manifest deployment in staging GKE cluster
4. Establish SLO for infrastructure change review/approval cycle

---

## Stakeholder Communication

- **Engineering:** Manifest validation now required in CI/CD; all new Deployments must specify CPU >= 100m
- **Product:** Incident resolved with no data loss; all features restored within 5 minutes
- **Customers:** Brief service interruption (5 minutes) due to infrastructure misconfiguration; now fully resolved
- **Platform Team:** Review GKE Autopilot constraints; update runbooks; implement new validation

---

## Appendix: Reproduction Steps

To reproduce this incident in a lab environment:

1. Deploy a simple gRPC service with:
   ```yaml
   resources:
     limits:
       cpu: 1m       # Too low
   livenessProbe:
     periodSeconds: 2
     timeoutSeconds: 1
     failureThreshold: 1
     grpc:
       port: 3550
   ```

2. Observe pod reaching CrashLoopBackOff within ~5 seconds

3. Correct the manifest:
   ```yaml
   resources:
     limits:
       cpu: 500m
   livenessProbe:
     periodSeconds: 10
     timeoutSeconds: 5
     failureThreshold: 3
   ```

4. Pod reaches Running state within ~30 seconds

---

**Prepared by:** Incident Response Team
**Date:** 2026-03-12 07:30 UTC-6
**Status:** Draft — awaiting final review and sign-off
