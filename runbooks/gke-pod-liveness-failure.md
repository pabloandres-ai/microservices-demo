# Runbook: GKE Pod Liveness Probe Failures / Insufficient Resource Limits

**Last updated:** 2026-03-12
**Default severity:** P1 (if preventing application startup) or P2 (if only affecting single service)
**Owner:** Platform / SRE team
**Related incidents:** AICR-246 (productcatalogservice CPU starvation)

---

## When to use this runbook

Pod creation is failing with one or more of these symptoms:

1. **Liveness probe failures:**
   ```
   Warning FailedScheduling ... Readiness probe failed:
   Warning BackOff ... Back-off restarting failed container
   ```

2. **Pod stuck in CrashLoopBackOff or Pending:**
   ```
   kubectl get pods -n <namespace>
   NAME                       READY   STATUS             RESTARTS   AGE
   myservice-6d9dc48f59-xyz   0/1     CrashLoopBackOff   5          2m
   ```

3. **Liveness probe timeout errors in logs:**
   ```
   rpc error: code = DeadlineExceeded desc = context deadline exceeded
   ```

4. **Multiple services cascading into failure** — one service fails to start, blocking dependent services that try to reach it

---

## Escalate to P0 if

- **All pods in a critical path service are failing** (e.g., frontend, checkout service)
- **Application is completely unavailable** to end users
- **Error rate is 100%** across the affected service
- **This is not resolved within 10 minutes**

---

## Diagnosis Steps

### Step 1: Confirm pod status and events

```bash
# Check the pod status
kubectl get pods -n <namespace> -l app=<service>

# Get detailed pod events (shows liveness probe failures)
kubectl describe pod -n <namespace> -l app=<service>

# Look for these patterns:
# - "Liveness probe failed"
# - "Back-off restarting failed container"
# - "Failed to perform a kubernetes.io/startup probe"
```

**Expected output if this runbook applies:**
```
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Warning  FailedScheduling  2m   default-scheduler  ...
  Warning  BackOff    90s    kubelet            Back-off restarting failed container server in pod ...
  Warning  Unhealthy  45s    kubelet            Liveness probe failed: ...
```

### Step 2: Check resource allocation and limits

```bash
# Get the Deployment spec
kubectl get deployment <service> -n <namespace> -o yaml | grep -A 10 "resources:"

# Expected to see:
# resources:
#   requests:
#     cpu: 100m
#     memory: 64Mi
#   limits:
#     cpu: 500m
#     memory: 128Mi
```

**Red flags:**
- `limits.cpu < 100m` (e.g., 1m, 10m, 50m)
- `limits.memory < 128Mi`
- `requests.cpu > limits.cpu` (impossible constraint)

### Step 3: Check liveness probe configuration

```bash
# Extract liveness probe settings
kubectl get deployment <service> -n <namespace> -o yaml | grep -A 8 "livenessProbe:"

# Expected reasonable values:
# livenessProbe:
#   periodSeconds: 10        (check interval)
#   timeoutSeconds: 5        (timeout per check)
#   failureThreshold: 3      (failures before restart)
```

**Red flags:**
- `periodSeconds < 5` (too aggressive, pod restarts too quickly)
- `timeoutSeconds < 3` (may timeout even with adequate resources)
- `failureThreshold: 1` (pod dies on first failure, no tolerance for transient slowness)

### Step 4: Correlate with recent deployments or config changes

```bash
# Check recent commits to the manifest
git log --oneline kubernetes-manifests/<service>.yaml | head -5

# Look for commits that mention:
# - resource limits
# - CPU, memory, or liveness probe changes
# - environment: staging, test, debug (these should never reach prod)
```

---

## Remediation

### Option A: Quick resource fix (most common — 2–5 minutes)

**Use this when:** Liveness probe failures are happening immediately, and resource limits are obviously too low (< 100m CPU).

1. **Identify the problematic manifest:**
   ```bash
   # Find which manifest has the bad config
   grep -r "cpu: [0-9]m$" kubernetes-manifests/ release/
   # Look for cpu values less than 100m (e.g., "1m", "10m", "50m")
   ```

2. **Revert to the last known stable version:**
   ```bash
   # Check the git history for the last stable resource config
   git log --oneline -n 10 -- kubernetes-manifests/<service>.yaml

   # Check out the last stable version of that file
   git show <last-stable-commit>:kubernetes-manifests/<service>.yaml > /tmp/<service>.yaml

   # Review the resources section
   cat /tmp/<service>.yaml | grep -A 10 "resources:"
   ```

3. **Apply the corrected manifest:**
   ```bash
   # Dry run first
   kubectl apply -f kubernetes-manifests/<service>.yaml -n <namespace> --dry-run=client

   # Apply if dry-run looks good
   kubectl apply -f kubernetes-manifests/<service>.yaml -n <namespace>

   # Or, if release manifest is being used
   kubectl apply -f release/kubernetes-manifests.yaml -n <namespace>
   ```

4. **Verify pod recovery:**
   ```bash
   # Watch pod startup
   kubectl rollout status deployment/<service> -n <namespace> --timeout=2m

   # Check for healthy pods
   kubectl get pods -n <namespace> -l app=<service>

   # Verify liveness probe is passing
   kubectl get events -n <namespace> -f --sort-by='.lastTimestamp' | tail -20
   ```

   **Expected result:**
   ```
   NAME                       READY   STATUS    RESTARTS   AGE
   <service>-6d9dc48f59-xyz   1/1     Running   0          1m
   ```

**Risk:** Low — reverting to a known stable configuration.
**Revert:** If needed:
   ```bash
   git revert HEAD --no-edit
   kubectl apply -f kubernetes-manifests/<service>.yaml -n <namespace>
   ```

### Option B: Adjust liveness probe only (if resource limits are adequate but probe is too aggressive — 5–10 minutes)

**Use this when:** Pod has > 100m CPU allocated but is still timing out on the liveness probe (e.g., heavy initialization, GC pauses, slow startup).

1. **Edit the liveness probe timeout and period:**
   ```bash
   kubectl patch deployment <service> -n <namespace> -p \
     '{"spec":{"template":{"spec":{"containers":[{"name":"server","livenessProbe":{"timeoutSeconds":10,"periodSeconds":15,"failureThreshold":3}}]}}}}'
   ```

2. **Or edit the manifest directly:**
   ```bash
   kubectl edit deployment <service> -n <namespace>
   # Update livenessProbe section:
   # - Increase timeoutSeconds (e.g., 5 → 10)
   # - Increase periodSeconds (e.g., 10 → 15)
   # - Increase failureThreshold (e.g., 3 → 5)
   ```

3. **Watch the pod restart:**
   ```bash
   kubectl rollout status deployment/<service> -n <namespace> --timeout=2m
   ```

**Risk:** Medium — probe timing changes could mask a real performance issue. Use if confident that the pod just needs more time to respond.
**Revert:** Edit the deployment again with the original probe timings.

---

## Verification

Confirm full resolution:

1. **Pod reaches Running:**
   ```bash
   kubectl get pods -n <namespace> -l app=<service> -o wide
   # All pods should show STATUS=Running, READY=1/1
   ```

2. **No liveness failures for 5+ minutes:**
   ```bash
   kubectl logs -n <namespace> -l app=<service> -c server --tail=100 | grep -i "liveness\|failed\|deadline"
   # Should return no matches
   ```

3. **Service is reachable (for HTTP services):**
   ```bash
   kubectl exec -it -n <namespace> <pod-name> -- curl -w "\nHTTP %{http_code}\n" http://localhost:<port>/<health-path>
   # Expected: HTTP 200
   ```

4. **Dependent services can reach this service (gRPC or HTTP call):**
   ```bash
   kubectl logs -n <namespace> -l app=<dependent-service> -c server --tail=50 | grep -i "connection refused\|unavailable"
   # Should return no matches if communication is working
   ```

---

## Post-Incident Actions

After resolution:

1. **File a Jira ticket** (reference this runbook with the ticket number)
2. **Update manifests in git** if the fix was a quick kubectl patch — commit the corrected YAML to prevent regression
3. **Review the commit that introduced the bad config** — determine if it was:
   - A debug change accidentally merged
   - A misunderstanding of GKE Autopilot minimum requirements
   - An overly aggressive performance tuning attempt
4. **Add CI/CD validation** to prevent recurrence:
   ```bash
   # Add to pre-deployment checks:
   grep -r "cpu: [0-9]*m$" kubernetes-manifests/ && grep -rE "cpu: [0-9]m|cpu: [0-9]{2}m" kubernetes-manifests/ && echo "FAIL: CPU limit < 100m detected" && exit 1 || true
   ```
5. **Update deployment runbook** with GKE Autopilot minimum resource requirements
6. **Schedule postmortem** if incident was P0 or P1

---

## Related Documentation

- GKE Autopilot resource requirements: https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-resource-requests
- Kubernetes liveness probes: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- Boutique microservices deployment guide: `kubernetes-manifests/README.md`

---

## Escalation

**If pod still fails after 10 minutes:**
- Page on-call SRE via PagerDuty / incident-response Slack channel
- Escalate to GKE support if cluster-level issue is suspected (node pressure, kubelet issues)
- Check GKE cluster logs for control plane errors: `gcloud container clusters describe <cluster> --zone=<zone>`
