# On-call and release runbook (abridged)

## Deployment and promotion

**Before:** confirm pinned image digest, ML contract tests, non-sensitive test fixtures, last staging canary, signing/scans, capacity headroom, 0% production weight and at least two ready v2 replicas. Confirm Prometheus/metrics freshness and route `Accepted`/`ResolvedRefs` conditions.

**At each step:** update desired `HTTPRoute` weights in a protected Git PR, merge, wait until Argo CD is Synced/Healthy and route status accepted; confirm actual traffic shares from gateway metrics. Use 1% -> 5% -> 25% -> 50% -> 100% (example sequence). For each step, observe a minimum sample count and time window; look at 5xx, latency p95/p99, timeouts, rate limits, load, GPU/CPU utilization and model-quality guardrails. Stop early on a failed gate. A time window alone does not produce statistically useful evidence when QPS is low.

**After:** retain old version at a warm minimum for the rollback window, confirm pinned traffic users, then retire by reviewed Git change. No image tag reuse.

## Roll back a broken version

**Trigger:** SLO burn, sustained 5xx/p95 regression versus stable control, a serious model-quality regression, failed probes or explicit incident commander decision.

1. Stop route promotion; page the right SRE/ML owners.
2. Use the emergency GitOps change to route stable endpoint to 100% known-good version, 0% bad version. Verify sync, route status and live metrics; keep `/versions/v2` access limited as appropriate.
3. If GitOps or API is unavailable, perform an audited break-glass route patch, verify and immediately restore Git/live parity when reconciliation resumes.
4. Check whether schema/clients are backward-compatible; if not, coordinate the contract rollback. Preserve logs/traces without request payloads.
5. Record incident, timeline, change that triggered it, contributing factors and follow-up load/failure tests.

**Avoid:** relying on an Argo CD UI historical rollback while automated synchronization is enabled; commit a Git revert/new desired state instead.

## Gateway/Prometheus/scaler failure

- Check Gateway health, route status, backends and upstream 5xx/503 by model/version; if both versions are affected, inspect shared infrastructure before changing models.
- Validate Prometheus target health and availability of `inference_routed_requests_total` for active backends. A zero sample for a newly deployed version may be legitimate; zero for a busy version plus missing scrapes is not.
- If KEDA HPA is stuck, verify `ScaledObject` conditions and external metrics API. Keep warm min replicas; temporarily set manual safe scaling only under documented break-glass ownership to avoid fighting HPA; revert any override and reconcile state after incident.
- If many pods Pending, inspect quota, resource pressure, autoscaler decisions, AZ topology spread, GPU availability and pull failures.

## Regional or control-plane failure

- Control plane unreachable does not immediately stop running data plane; avoid assuming it is healthy indefinitely.
- Restore from Git + Terraform with staged runbooks, registry access and validated secrets. Check DNS/LB failover only if a second region exists and the business has required it.

## Infrastructure updates

Terraform plan and peer approval; stage cloud/IAM changes; upgrade cluster/control plane/addons using supported version order; use node rolling replacement and PDB-safe drains; rehearse in staging and watch 5xx, pending pods and model-start timings. Maintain tested revert/rebuild instructions for irreversible migrations.
