# Architecture Decision Records — Platform Delivery

Each ADR captures a design decision made during the Argo Rollouts
project. Decisions are numbered and include context, rationale, and
consequences.

---

## ADR-010: Rollout as opt-in, not default

**Context:** The library chart now supports both Deployment and Rollout. We need to decide what happens when a service doesn't explicitly choose.

**Decision:** Rollout is opt-in via `rollout.enabled: true`. The default is `false`, which renders a standard Deployment. Making Rollout the default would be a breaking change (v2.0.0) and would force every service into canary delivery whether they need it or not. Services opt in explicitly, making the choice visible and reviewable in Git.

---

## ADR-011: workloadRef migration pattern

**Context:** Existing services run as Deployments. We need a way to migrate them to Rollouts without downtime and without deleting the running pods.

**Decision:** Use Argo Rollouts' `workloadRef` takeover. The Rollout points at the existing Deployment, reads its pod spec, scales the Deployment down gracefully, and takes over its ReplicaSets with zero downtime. This is migration scaffolding, not a permanent state. Once the Rollout is proven, inline the pod spec into the Rollout and delete the original Deployment. Leaving both permanently creates split-brain — two controllers fighting over the same pods.

---

## ADR-012: Argo Rollouts via ArgoCD, not Terraform

**Context:** Argo Rollouts controller needs to be installed on the cluster. Both Terraform and ArgoCD can manage Helm charts.

**Decision:** Foundation stack (Terraform) is only for things that must exist before ArgoCD can start reconciling — the cluster, ArgoCD itself, identity plumbing like IRSA and OIDC providers. Everything else, including controllers like Argo Rollouts, goes through ArgoCD as a regular Application. Git is the only source of truth even for cluster tooling — upgrades, rollbacks, and audit all flow through PRs, not `terraform apply` from someone's laptop.

---

## ADR-013: nginx canary annotations vs SMI vs mesh

**Context:** We need weight-based traffic splitting for canary rollouts. Three options: nginx canary annotations, SMI (Service Mesh Interface), or a full service mesh like Istio or Linkerd.

**Decision:** Use nginx canary annotations. We already run ingress-nginx, and its canary annotations give us weight-based traffic splitting with zero extra components. SMI requires a mesh adapter we don't have. A full service mesh adds sidecar proxies to every pod for a feature we can get with two annotations. Pick the simplest tool that solves the problem.

---

## ADR-014: ClusterAnalysisTemplate parameterization over per-service copies

**Context:** We need AnalysisTemplates that define "healthy" for canary rollouts. We could create one per service or one shared template parameterized by service name.

**Decision:** Use a single ClusterAnalysisTemplate per metric (success-rate, latency-p99), parameterized with `args.namespace`. Platform owns the definition, services just reference it. If we copied the template per service, changing a threshold would mean editing N files. One parameterized template means one change, and every service gets it on next deploy.

---

## ADR-015: Aware of ingress-nginx retirement

**Context:** The community ingress-nginx controller was retired in March 2026. We are building canary traffic shaping on top of it.

**Decision:** Build on ingress-nginx today because the canary annotation pattern is industry-standard and works the same regardless of which ingress controller you use. Migrating to Gateway API is scoped as a separate future project. The risk is documented here, not ignored.