# Notiflex Current State — Distilled AI Reference

> For dated history and troubleshooting narrative, see `JOURNEY.md`. For detailed diagrams,
> see `architecture.md`. For full decision rationale, see `docs/architecture-decisions.md`.
> This file compresses **only the facts as of right now**, so an AI session can load it
> quickly at the start of a conversation. Update it whenever the state changes.

## Cluster

| Item | Value |
|---|---|
| GCP project | notiflex-09019 |
| Cluster | notiflex-cluster (GKE Standard, Zonal) |
| Region/zone | asia-northeast3 / asia-northeast3-a (**single zone**) |
| kubectl context | `gke-sysnet4admin_book_gitaiops` (required on every command) |
| Gateway external IP | 35.216.16.34 |
| Artifact Registry | asia-northeast3-docker.pkg.dev/notiflex-09019/notiflex/api |

## Node pools

| Pool | Machine | Nodes | Workload |
|---|---|---|---|
| default-pool | e2-medium | 2 | ArgoCD, Argo Rollouts, Prometheus/Grafana/Alertmanager/Loki, Valkey |
| api-pool | e2-medium | 1 | notiflex-api (smb + enterprise) |
| worker-pool | e2-standard-2 | 1 | Strimzi Operator, Kafka broker, kafka-ui |
| ops-pool | e2-small | 1 | Tempo, notiflex-healthcheck CronJob (⚠️ tight on memory, repeated NotReady observed) |

All Spot VMs.

## Component versions

| Component | Version | Notes |
|---|---|---|
| Notiflex image | v0.3.6 (`sha-1d4aefd`) | Valkey + Kafka ConsumerGroup + Tempo all active |
| Kafka | 4.2.0 (Strimzi 1.2.0, KRaft) | Original spec (4.1.0) unsupported by Strimzi 1.2.0 |
| ArgoCD | v3.5.3 | |
| Argo Rollouts | v1.10.0 | |
| Tempo | grafana/tempo:2.9.0 | requests 25m/128Mi, limits 200m/256Mi |
| Grafana | 13.2.2-distroless | API Basic Auth returns 401 (browser session login works fine) |
| Go | 1.25 | |

## GitOps management boundary

- **ArgoCD-managed** (`argocd/apps/`): `k8s/smb/`, `k8s/enterprise/` — code change → CI build → image tag auto-update → Canary (20→50→80→100%)
- **helm/kubectl direct** (not GitOps-managed): Kafka (`k8s/kafka/` + `helm-values/strimzi.yaml`), Tempo, Prometheus/Grafana/Loki, kafka-ui — apply via `helm upgrade`, then record manually in JOURNEY.md
- All 3 ArgoCD Applications are `Synced`/`Healthy`: root-app, notiflex-smb, notiflex-enterprise

## Namespaces

| NS | Key resources |
|---|---|
| notiflex | Rollout notiflex-api×2, valkey-primary, CronJob healthcheck |
| enterprise | Rollout notiflex-api×1 (shares notiflex's Valkey cross-namespace; Kafka/Tempo not wired) |
| kafka | Strimzi Operator, notiflex-kafka (single node doubling as controller+broker), notifications topic (3 partitions), kafka-ui |
| monitoring | kube-prometheus-stack, Loki+Fluent Bit, Tempo |
| argocd / argo-rollouts | control plane |

## Access (port-forward)

```bash
# ArgoCD (admin / password via kubectl get secret argocd-initial-admin-secret)
kubectl --context gke-sysnet4admin_book_gitaiops port-forward svc/argocd-server -n argocd 8443:443
# Grafana (admin/admin)
kubectl --context gke-sysnet4admin_book_gitaiops port-forward svc/kube-prometheus-grafana -n monitoring 3000:80
# Kafka UI (no auth)
kubectl --context gke-sysnet4admin_book_gitaiops port-forward svc/kafka-ui -n kafka 8081:80
```

## ADR summary (full text in docs/architecture-decisions.md)

| # | Decision |
|---|---|
| 001 | GitOps — ArgoCD |
| 002 | CI — GitHub Actions |
| 003 | Metrics — Prometheus+Grafana |
| 004 | Logging — Loki+Fluent Bit |
| 005 | Alerting — PrometheusRule |
| 006 | External traffic — Gateway API |
| 007 | Deployment (1st) — Blue/Green |
| 008 | Cache — Valkey |
| 009 | Secrets — GKE Secret Manager CSI + WI |
| 010 | Deployment (2nd) — Canary |
| 011 | Node scheduling — nodeSelector |
| 012 | Multi-app — App of Apps |
| 013 | Multi-tenancy — Namespace separation |
| 014 | Messaging — Kafka (Strimzi, KRaft, v4.2.0) |
| 015 | Tracing — Tempo |
| 016 | Batch automation — K8s CronJob |
| 017 | Alert channel — Slack Webhook |
| 018 | Tenant isolation — ResourceQuota+LimitRange |

## Key lessons (verified in practice this session)

1. **Don't trust a pinned Strimzi/Kafka version** — `kubectl describe kafka`'s error message tells you the currently supported version list. Match sarama's `cfg.Version` to the broker version too (check the constant exists with `go doc github.com/IBM/sarama.V4_2_0_0`).
2. **Never guess a helm-values key path** — check the real schema with `helm show values <chart>` first. (Tempo case: `resources:` at the top level was silently ignored; the correct path was `tempo.resources`.)
3. **Never `kubectl delete` a resource ArgoCD is tracking** — with `selfHeal: true`, it gets restored to the git spec immediately, causing an unintended redeploy. Use `kubectl apply` to change fields instead.
4. **A Spot VM node's transient NotReady usually self-heals** — if it's just the kubelet pausing status reports, it typically recovers within a few minutes. Don't rush to recreate the Pod or force-delete the node — check the node's own status with `kubectl describe node` first.
5. **Adding partitions without a Consumer Group does nothing** — subscribing individually via `ConsumePartition` makes every Pod receive the same message redundantly. Real parallel partition processing requires joining a group via `sarama.NewConsumerGroup`.
6. **A Kafka Consumer doesn't re-read from the beginning** — it resumes from the committed offset (stored on the broker in `__consumer_offsets`). `Consumer.Offsets.Initial` only applies when there's no commit history at all.
7. **`--context` is mandatory on every kubectl command** — prevents accidents when juggling multiple clusters.

## Known risks / candidate next steps

In priority order (rationale in architecture.md's "Next step candidates" section):

1. No NetworkPolicy at all — unrestricted cross-namespace traffic
2. No HPA; Kafka broker is a single node (SPOF)
3. Single zone (asia-northeast3-a) — needs Regional conversion
4. Tempo sampling not configured (100% AlwaysSample)
5. No alerting on CronJob failure or Kafka Consumer lag
6. Repeated ops-pool NotReady observed (6 times in one day, 2026-09-22) — needs continued monitoring to see if it's a pattern
