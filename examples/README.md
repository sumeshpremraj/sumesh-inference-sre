# Reference Kubernetes manifests

- `fraud-v1.yaml`: v1 Deployment, Service and KEDA ScaledObject
- `fraud-v2.yaml`: independent v2 Deployment, Service and KEDA ScaledObject
- `fraud-route.yaml`: one stable weighted endpoint plus two explicit-version endpoints

**These manifests illustrate an approach; they are not a deployed service.** Prerequisites are an `inference` namespace; EKS/Kubernetes cluster; installed KEDA, Prometheus, a Gateway API-compatible controller and an `inference-gateway`; DNS, TLS and auth controls; a reachable private image registry; a controller that supports `URLRewrite`. Replace `registry.example.com` and the illustrative 64-character image digests with real published images, verify the endpoint paths, and calibrate CPU/memory and RPS per replica with load tests.

**Important production edges:**

1. A separate scaler exists for each version. There is no `.spec.replicas` on either Deployment: HPA owns it and GitOps must not fight it.
2. The v1/v2 metrics require **Gateway instrumentation** that emits `inference_routed_requests_total{model,version}` *after choosing the backend*. This metric name is an intentional platform contract, not an off-the-shelf guarantee. `or vector(0)` accommodates a newly dark-deployed version with zero routed requests; **alert independently if the exporter, Prometheus scrape, or metric freshness fails**, since a missing metric otherwise looks like zero demand.
3. KEDA uses `metricType: AverageValue`; total version-RPS / target-RPS-per-pod provides the approximate desired pod count. Choose thresholds from representative tests. Multiple metric triggers and a concurrency-based scaler may be preferable for variable inference cost.
4. Set a warm minimum before shifting traffic. Neither KEDA nor the node autoscaler magically eliminates cold model startup or GPU provisioning delays.
5. The route's `URLRewrite` is an **Extended** Gateway API feature. Confirm controller conformance and enforce route `Accepted=True` / `ResolvedRefs=True` as release checks. For different internal inference paths across versions, provide a version-specific adapter (or separate rewrite-aware routes) rather than using a single incompatible rewrite.
6. For production resilience, add PodDisruptionBudgets, explicit NetworkPolicies, ServiceAccounts/workload identity, image policy, secrets and full observability from the platform charts. They are described in the architecture but intentionally omitted from the focused example.
7. Model images need a real non-root UID, writable model cache if needed, and correct model-loading probes; update these `securityContext` and probes for the runtime contract.

To run local structural checks: `python -m pip install -r requirements-dev.txt && python tests/validate_examples.py` from repository root.
