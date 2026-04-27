# Platform Delivery — Progressive Delivery with Argo Rollouts

Canary rollouts driven by Prometheus analysis, with automatic rollback on SLO breach.
Connects Project 1 (Golden Path library chart) and Project 2 (SLO observability) into a closed loop.

## What this project does

When a service deploys a new version:

1. Argo Rollouts creates a canary pod alongside the stable pods
2. Ingress-nginx routes a small percentage of traffic to the canary (weight-based, not replica-based)
3. Two ClusterAnalysisTemplates query Prometheus continuously:
   - **success-rate** — 5xx error rate must stay below 5%
   - **latency-p99** — p99 response time must stay below 500ms
4. If metrics are healthy, the rollout auto-promotes through each step (20% → 50% → 100%)
5. If metrics fail twice, the rollout auto-aborts. Stable keeps serving. No human needed.

## Architecture

```
Git push (new image tag)
    │
    ▼
ArgoCD syncs Rollout
    │
    ▼
Argo Rollouts creates canary ReplicaSet + canary Service
    │
    ▼
Ingress-nginx splits traffic (stable Ingress + canary Ingress with weight annotation)
    │
    ▼
Prometheus scrapes nginx metrics (request count, latency histogram)
    │
    ▼
AnalysisRun queries Prometheus every 30s
    │
    ├── Healthy → auto-promote to next step
    └── Failed (2x) → auto-abort, stable untouched
```

## Repo structure

```
platform-delivery/
├── apps/
│   └── argo-rollouts-app.yaml       # ArgoCD Application for the controller
├── manifests/
│   └── analysis-templates/
│       ├── success-rate.yaml         # ClusterAnalysisTemplate: 5xx rate < 5%
│       └── latency-p99.yaml          # ClusterAnalysisTemplate: p99 < 500ms
├── chaos-test/
│   ├── Dockerfile                    # Broken image for chaos drill (returns 503)
│   └── default.conf                  # nginx config: 200 on /healthz, 503 on /
├── docs/
│   └── decisions.md                  # 6 ADRs
└── README.md
```

## How to onboard a new service

1. In the service's `values.yaml`, enable rollout:

```yaml
rollout:
  enabled: true
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: {duration: 60s}
        - setWeight: 50
        - pause: {duration: 60s}
        - setWeight: 100
  analysis:
    templates:
      - templateName: success-rate
        clusterScope: true
      - templateName: latency-p99
        clusterScope: true
    args:
      - name: namespace
        value: <service-namespace>
```

2. The library chart (platform-golden-path) renders the Rollout with analysis config automatically.
3. Push to Git. ArgoCD syncs. Next image change triggers a canary with analysis.

## Key decisions

| # | Decision | Reason |
|---|----------|--------|
| ADR-010 | Rollout is opt-in, not default | Avoids breaking change for services that don't need canary |
| ADR-011 | workloadRef for migration | Zero-downtime Deployment-to-Rollout migration, then remove scaffolding |
| ADR-012 | Argo Rollouts via ArgoCD, not Terraform | Only bootstrap deps go in Terraform; everything else through Git |
| ADR-013 | nginx canary annotations over SMI/mesh | Simplest option, zero extra components |
| ADR-014 | ClusterAnalysisTemplate, parameterized | One template per metric, reusable across all services |
| ADR-015 | Build on ingress-nginx despite retirement | Pattern is portable; Gateway API migration is a separate project |

## Chaos drill results

Deployed a broken image (returns 503 on all traffic, 200 on /healthz):
- Canary started, passed health checks, received 20% traffic
- AnalysisRun detected error rate > 5% after 3 failed checks
- Rollout auto-aborted. Stable pods kept serving. Zero user impact.
- Abort message: `Metric "error-rate" assessed Failed due to failed (3) > failureLimit (2)`

## Dependencies

- **Argo Rollouts** v2.40.0 — controller + CRDs
- **ingress-nginx** — canary annotations for traffic shaping
- **kube-prometheus-stack** — Prometheus for analysis queries
- **platform-golden-path** — library chart with Rollout template support (v1.4.0+)
- **platform-observability** — traffic generator producing metrics