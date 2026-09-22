# Notiflex 현재 상태 — AI 참조용 증류본

> 이력·날짜·트러블슈팅 서사는 `JOURNEY.md`, 상세 다이어그램은 `architecture.md`,
> 결정 전문(이유 불릿)은 `docs/architecture-decisions.md`를 참조. 이 파일은 **지금 이 순간의
> 사실만** 압축해 AI가 세션 시작 시 빠르게 로드하도록 만든 요약본이며, 상태가 바뀌면 갱신한다.

## 클러스터

| 항목 | 값 |
|---|---|
| GCP 프로젝트 | notiflex-09019 |
| 클러스터 | notiflex-cluster (GKE Standard, Zonal) |
| 리전/존 | asia-northeast3 / asia-northeast3-a (**단일 존**) |
| kubectl context | `gke-sysnet4admin_book_gitaiops` (모든 명령에 `--context` 필수) |
| Gateway 외부 IP | 35.216.16.34 |
| Artifact Registry | asia-northeast3-docker.pkg.dev/notiflex-09019/notiflex/api |

## 노드풀

| 풀 | 머신 | 노드 수 | 워크로드 |
|---|---|---|---|
| default-pool | e2-medium | 2 | ArgoCD, Argo Rollouts, Prometheus/Grafana/Alertmanager/Loki, Valkey |
| api-pool | e2-medium | 1 | notiflex-api (smb + enterprise) |
| worker-pool | e2-standard-2 | 1 | Strimzi Operator, Kafka 브로커, kafka-ui |
| ops-pool | e2-small | 1 | Tempo, notiflex-healthcheck CronJob (⚠️ 메모리 여유 적음, NotReady 반복 관측) |

전부 Spot VM.

## 컴포넌트 버전

| 컴포넌트 | 버전 | 비고 |
|---|---|---|
| Notiflex 이미지 | v0.3.6 (`sha-1d4aefd`) | Valkey+Kafka ConsumerGroup+Tempo 전부 활성 |
| Kafka | 4.2.0 (Strimzi 1.2.0, KRaft) | 원 스펙 4.1.0은 Strimzi 1.2.0 미지원 |
| ArgoCD | v3.5.3 | |
| Argo Rollouts | v1.10.0 | |
| Tempo | grafana/tempo:2.9.0 | requests 25m/128Mi, limits 200m/256Mi |
| Grafana | 13.2.2-distroless | API Basic Auth 401 이슈 있음(브라우저 세션 로그인은 정상) |
| Go | 1.25 | |

## GitOps 관리 경계

- **ArgoCD 관리** (`argocd/apps/`): `k8s/smb/`, `k8s/enterprise/` — 코드 변경 → CI 빌드 → 이미지 태그 자동 갱신 → Canary(20→50→80→100%)
- **helm/kubectl 직접 관리** (GitOps 미대상): Kafka(`k8s/kafka/` + `helm-values/strimzi.yaml`), Tempo, Prometheus/Grafana/Loki, kafka-ui — `helm upgrade` 후 JOURNEY.md에 수동 기록
- ArgoCD Application 3개 전부 `Synced`/`Healthy`: root-app, notiflex-smb, notiflex-enterprise

## 네임스페이스

| NS | 핵심 리소스 |
|---|---|
| notiflex | Rollout notiflex-api×2, valkey-primary, CronJob healthcheck |
| enterprise | Rollout notiflex-api×1 (notiflex의 Valkey를 cross-namespace 공유, Kafka/Tempo 미연동) |
| kafka | Strimzi Operator, notiflex-kafka(controller+broker겸임 1노드), notifications 토픽(3 partition), kafka-ui |
| monitoring | kube-prometheus-stack, Loki+Fluent Bit, Tempo |
| argocd / argo-rollouts | 컨트롤 플레인 |

## 접근 (포트포워딩)

```bash
# ArgoCD (admin / kubectl get secret argocd-initial-admin-secret로 조회)
kubectl --context gke-sysnet4admin_book_gitaiops port-forward svc/argocd-server -n argocd 8443:443
# Grafana (admin/admin)
kubectl --context gke-sysnet4admin_book_gitaiops port-forward svc/kube-prometheus-grafana -n monitoring 3000:80
# Kafka UI (인증 없음)
kubectl --context gke-sysnet4admin_book_gitaiops port-forward svc/kafka-ui -n kafka 8081:80
```

## ADR 요약 (전문은 docs/architecture-decisions.md)

| # | 결정 |
|---|---|
| 001 | GitOps — ArgoCD |
| 002 | CI — GitHub Actions |
| 003 | 메트릭 — Prometheus+Grafana |
| 004 | 로그 — Loki+Fluent Bit |
| 005 | 알림 — PrometheusRule |
| 006 | 외부 트래픽 — Gateway API |
| 007 | 배포(1차) — Blue/Green |
| 008 | 캐시 — Valkey |
| 009 | 시크릿 — GKE Secret Manager CSI + WI |
| 010 | 배포(2차) — Canary |
| 011 | 노드 스케줄링 — nodeSelector |
| 012 | 멀티앱 — App of Apps |
| 013 | 멀티테넌시 — Namespace 분리 |
| 014 | 메시징 — Kafka(Strimzi, KRaft, v4.2.0) |
| 015 | 트레이싱 — Tempo |
| 016 | 배치 자동화 — K8s CronJob |
| 017 | 알림 채널 — Slack Webhook |
| 018 | 테넌트 격리 — ResourceQuota+LimitRange |

## 핵심 교훈 (이번 세션에서 실전 검증됨)

1. **Strimzi/Kafka 버전을 고정값으로 믿지 말 것** — `kubectl describe kafka`의 에러 메시지가 현재 지원 버전 목록을 알려준다. sarama의 `cfg.Version`도 브로커 버전과 반드시 맞출 것(`go doc github.com/IBM/sarama.V4_2_0_0`으로 상수 존재 여부 확인 가능).
2. **helm-values 키 경로는 추측 금지** — `helm show values <chart>`로 실제 스키마를 먼저 확인. (Tempo 사례: 최상위 `resources:`가 아니라 `tempo.resources`라 조용히 무시되고 있었음.)
3. **ArgoCD가 추적 중인 리소스는 `kubectl delete` 금지** — `selfHeal: true`면 즉시 git 스펙으로 복원되어 의도치 않은 재배포가 발생한다. 필드만 바꿀 땐 `kubectl apply`.
4. **Spot VM 노드의 일시 NotReady는 대개 자동 복구된다** — kubelet이 상태 보고를 멈춘 것뿐이면 수 분 내 복구. 성급하게 Pod 재생성·노드 강제 삭제 금지, 먼저 `kubectl describe node`로 노드 자체 상태부터 확인.
5. **Consumer Group 없이 파티션만 늘려봐야 소용없다** — `ConsumePartition`으로 개별 구독하면 모든 Pod이 같은 메시지를 중복 수신한다. 파티션 병렬 처리는 `sarama.NewConsumerGroup`으로 그룹 조인해야 실제로 분산된다.
6. **Kafka Consumer는 처음부터 다시 안 읽는다** — 커밋된 오프셋(브로커에 저장, `__consumer_offsets`)부터 재개. `Consumer.Offsets.Initial`은 커밋 이력이 아예 없을 때만 적용됨.
7. **모든 kubectl 명령에 `--context` 필수** — 여러 클러스터를 오갈 때 사고 방지.
8. **`kubectl patch`로 급하게 고친 값은 helm-values 파일에 반영을 잊기 쉽다** — ch6 진입 전 CPU 축소를 patch로만 했다가, helm-values 파일(100m/50m/25m)과 실제 라이브 스펙(5m)이 몇 달째 어긋나 있었음(2026-09-22 발견·수정). 급한 patch 이후엔 반드시 "이 값을 파일에도 옮겨야 한다"를 후속 작업으로 남길 것 — 안 그러면 다음 `helm upgrade -f <file>`이 조용히 리소스를 원복시킨다.

## 알려진 리스크 / 다음 단계 후보

우선순위 순 (근거는 architecture.md의 "다음 단계 후보" 참고):

1. NetworkPolicy 전무 — 네임스페이스 간 트래픽 무제한
2. HPA 없음, Kafka 브로커 단일 노드(SPOF)
3. 단일 zone(asia-northeast3-a) — Regional 전환 필요
4. Tempo 샘플링 미설정(100% AlwaysSample)
5. CronJob 실패 / Kafka Consumer lag에 대한 알림 없음
6. ops-pool NotReady 반복 관측(2026-09-22 하루 6회) — 패턴인지 지속 관찰 필요
