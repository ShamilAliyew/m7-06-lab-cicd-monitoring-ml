# Rollback Runbook: Vision Moderation Model Service

**Runbook:** https://runbooks.example.com/vision-moderation/rollback  
**Severity:** page / critical  
**Estimated Time:** 5–10 minutes

---

## When to Roll Back

Roll back **immediately** if **any one** of these is true for >5 minutes:

- [ ] Error rate (5xx) > 5%
- [ ] P99 latency > 1 second
- [ ] Model accuracy on canary segment drops >2% vs. baseline
- [ ] `ModelVersionMismatch` alert fires (replica version skew)
- [ ] Canary → production traffic increased but canary metrics degrade further

**Do NOT roll back for:** single failed request, transient network hiccup, or unclear signal. Escalate to on-call manager.

---

## How to Roll Back

**Option 1: Workflow Dispatch (recommended)**
```bash
gh workflow run rollback.yml -f environment=production -f previous_version=<LAST_STABLE_SHA>
```
Replace `<LAST_STABLE_SHA>` with the commit hash of the last production deployment (check `.github/deployments` or ask infra).

**Option 2: Manual kubectl (if workflow is broken)**
```bash
kubectl rollout undo deployment/vision-moderation -n production
kubectl rollout status deployment/vision-moderation -n production --timeout=5m
```

**Option 3: GitHub Actions web UI**
- Go to repo → Actions → `rollback` workflow
- Click "Run workflow"
- Select `environment: production`
- Enter `previous_version` SHA
- Click "Run"

---

## Verify Rollback Succeeded

**Dashboard checks (2 min):**
- [ ] Grafana "Vision Moderation Overview" — error rate returns to <1%
- [ ] P99 latency drops below 500ms
- [ ] Pod restart count is stable (not increasing)

**Alert checks:**
- [ ] `AvailabilityBurnFast` alert clears within 5 minutes
- [ ] `LatencyP99High` alert clears within 5 minutes
- [ ] `ModelVersionMismatch` alert clears (all replicas on same version)

**Traffic check:**
```bash
kubectl get endpoints vision-moderation -n production
# Verify all pods are Ready (not NotReady or Terminating)
```

---

## Notify

**In this order, immediately:**
1. Slack `#ml-incidents` → "Rolling back vision-moderation to `<SHA>` due to [error rate|latency|accuracy]. Standby for confirmation."
2. PagerDuty incident → add note "Rollback initiated at HH:MM UTC."
3. On-call manager (if page went out) → "Rollback executed; metrics recovering. Root cause investigation starting."

---

## What NOT to Do

❌ **Do NOT:**
- Roll forward (re-deploy new version) before root cause is identified
- Dismiss the alert without verifying metrics on dashboards
- Change canary traffic % before checking model predictions
- Assume "restart pod" is the same as "rollback" (it's not)
- Delay notification to team

---

## When to Roll Forward

Roll forward **only after** all of these:

1. [ ] Root cause documented (link in incident ticket)
2. [ ] Fix merged and reviewed
3. [ ] Automated tests pass in CI
4. [ ] Staging environment validated with fix
5. [ ] On-call manager approves re-deploy
6. [ ] Wait 30 minutes post-rollback before re-deploying (stability check)

**Command:**
```bash
gh workflow run deploy-model.yml
```

---

**Questions?** Slack `@ml-platform-oncall` or call +1-xxx-yyy-zzzz (PagerDuty).  
**Last updated:** 2026-06-06
