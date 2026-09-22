# Notiflex 아키텍처 스냅샷 — 클러스터 재구축분, ch8 완료 시점

> ⚠️ **재구축 트랙 안내**: 이 클러스터(`notiflex-09019` 프로젝트)는 2026-09-17에 재생성되었고,
> 독자는 ch2부터 순차적으로 재시연 중이다. 이 문서는 **재구축 클러스터의 현재 실제 상태
> (ch8까지)**를 반영한다. ch8에서 다룬 Kafka·Tempo·CronJob이 전부 실제로 배포·검증됐다.
> ch9(회고, 온보딩 문서, GitAIOps 분석, 마무리)만 아직 이 재구축 트랙에서 다시 진행되지 않았다.

## 3층 지식 구조

| 문서 | 역할 | 업데이트 주기 |
|------|------|------------|
| **CLAUDE.md** | 프로젝트 메타데이터 (GCP 프로젝트, 리전, Artifact Registry, 가드레일 매칭 규칙) — 매 대화 자동 로드 | 초기 설정 시 |
| **claude-context/** | 현재 아키텍처 스냅샷 (토폴로지·컴포넌트·파이프라인) — AI가 참조 요청 시 | 챕터 완료 시 |
| **docs/architecture-decisions.md** | 결정 누적 기록 (왜 이 도구를 선택했는가) — 사람·AI가 결정 근거 검토 시 | 결정 시점마다 |

이 세 층은 서로 다른 목적을 가진다: CLAUDE.md는 "AI가 이 프로젝트에서 어떻게 행동해야 하는가", claude-context는 "지금 클러스터가 실제로 어떻게 생겼는가", ADR은 "왜 지금 이 모습이 되었는가"를 담는다. 섞이면 AI가 과거 결정 이유를 현재 상태로 착각하거나, 현재 상태를 미래 목표와 혼동하게 된다.

## 클러스터 토폴로지

| 항목 | 값 |
|------|-----|
| 클러스터 | notiflex-cluster |
| GCP 프로젝트 | notiflex-09019 (2026-09-17 재생성 — 이전 프로젝트 `project-75fce205-dfa5-4975-a56`는 소멸) |
| 리전/존 | asia-northeast3 / asia-northeast3-a |
| 노드풀 | **5개 노드 전부 존재, 전부 워크로드 배치 완료**: default-pool(e2-medium×2, 플랫폼 컴포넌트), api-pool(e2-medium×1, notiflex-api), worker-pool(e2-standard-2×1, Kafka), ops-pool(e2-small×1, Tempo+CronJob) — 모두 Spot, `GKE_METADATA` 적용 |
| Workload Identity | 활성화됨 (`notiflex-09019.svc.id.goog`), 전체 노드풀 `GKE_METADATA` 적용 완료 |
| Secret Manager CSI | 활성화됨 (`--enable-secret-manager`, GKE managed addon, provider=`gke`) |
| Gateway API | 활성화됨, GatewayClass `gke-l7-regional-external-managed`, 외부 IP `35.216.16.34` (notiflex/smb 테넌트만 외부 노출, enterprise는 클러스터 내부 전용) |
| Artifact Registry | `asia-northeast3-docker.pkg.dev/notiflex-09019/notiflex/api` |

## 컴포넌트 다이어그램

```
외부 클라이언트
    │
    ▼ HTTP :80  (http://35.216.16.34)
GKE Gateway: notiflex-gateway (gke-l7-regional-external-managed)
    │ HTTPRoute: notiflex-route
    ▼
Service: notiflex-api (stable, ClusterIP, ns: notiflex)
Service: notiflex-api-preview (canary, ClusterIP, ns: notiflex)
    │
    ▼
Rollout: notiflex-api  (namespace: notiflex, nodeSelector: api-pool)
  전략: Canary  20% → 30s pause → 50% → 30s pause → 80% → 30s pause → 100%
  이미지: .../notiflex/api:v0.3.6 (sha-1d4aefd, CI가 sha-<commit> 태그로 자동 갱신)
    │
    ├── serviceAccountName: notiflex-sa (Workload Identity)
    │     └── GCP SA: notiflex-secrets@notiflex-09019.iam.gserviceaccount.com
    │           (roles/secretmanager.secretAccessor)
    │
    ├── Secret 볼륨 (CSI → Google Secret Manager)
    │     driver: secrets-store-gke.csi.k8s.io
    │     SecretProviderClass: notiflex-secrets (provider: gke)
    │     valkey-password → /mnt/secrets/valkey-password (VALKEY_PASSWORD_FILE)
    │
    ├── ResourceQuota/notiflex-quota + LimitRange/notiflex-limits (ns: notiflex)
    │     requests.cpu 500m / requests.memory 512Mi / limits.cpu 1 / limits.memory 1Gi / pods 15
    │
    ├── Valkey StatefulSet (valkey-primary, namespace: notiflex, default-pool)
    │     VALKEY_ADDR=valkey-primary.notiflex.svc.cluster.local:6379
    │     INCR 명령으로 Pod 간 공유 분산 ID 생성 (/id 엔드포인트)
    │     span: valkey.incr
    │
    ├── Kafka Producer (KAFKA_BROKER=notiflex-kafka-kafka-bootstrap.kafka.svc.cluster.local:9092)
    │     /id 요청마다 notifications 토픽에 {"id":N,"pod":"..."} 발행
    │     span: kafka.produce
    │
    ├── Kafka Consumer (ConsumerGroup: notiflex-workers, sarama)
    │     notifications 토픽(3 파티션)을 Pod 간 자동 분배 소비 (파티션당 Consumer 1개)
    │     at-least-once, 커밋된 오프셋부터 재개 (Pod 재시작 시 처음부터 재읽기 없음)
    │
    ├── OTel SDK → Tempo (OTEL_EXPORTER_OTLP_ENDPOINT=tempo.monitoring.svc.cluster.local:4317)
    │     요청당 span 구조: id(root) → valkey.incr, kafka.produce (자식 span)
    │
    └── CronJob: notiflex-healthcheck (ops-pool, */5 * * * *, curl /health, restartPolicy: OnFailure)

Rollout: notiflex-api  (namespace: enterprise, nodeSelector: api-pool)  ← 별도 테넌트
  전략: Canary  20% → 30s pause → 100%  (replicas: 1)
  같은 이미지(v0.3.6), 같은 CSI Secret 패턴, 별도 notiflex-sa(WI 바인딩 별도)
    │
    ├── ResourceQuota/enterprise-quota + LimitRange/enterprise-limits (ns: enterprise)
    │     requests.cpu 200m / requests.memory 128Mi / limits.cpu 400m / limits.memory 256Mi / pods 5
    │
    └── VALKEY_ADDR=valkey-primary.notiflex.svc.cluster.local:6379  ← cross-namespace로
          notiflex 네임스페이스의 같은 Valkey를 공유 (테넌트 간 분산 ID 카운터 연속)
    외부 노출 없음 — Service(ClusterIP)만 존재, Gateway/HTTPRoute 미연결
    ⚠️ enterprise 테넌트는 KAFKA_BROKER/OTEL 엔드포인트가 아직 미설정 — Kafka/Tempo 연동은
       notiflex(smb) 테넌트에만 적용되어 있다 (enterprise 확장은 ch8 범위 밖)

namespace: kafka  (worker-pool)
├── strimzi-cluster-operator (Strimzi 1.2.0)
├── KafkaNodePool/controller + Kafka/notiflex-kafka (KRaft, v4.2.0, 단일 브로커 controller+broker 겸임)
│     ⚠️ 원 가이드는 4.1.0을 지정했으나 Strimzi 1.2.0이 4.2.0부터 지원해 4.2.0으로 조정
├── KafkaTopic/notifications (3 partitions, replicas: 1)
└── kafka-ui (helm, 별도 설치 — Kafka UI로 토픽/Consumer Group/lag 조회, localhost:8081 포트포워딩)

namespace: monitoring  (default-pool + ops-pool)
├── kube-prometheus-stack (Prometheus + Grafana + Alertmanager + kube-state-metrics)
├── Loki + Fluent Bit (DaemonSet, 전 노드)
└── tempo-0 (ops-pool, grafana/tempo:2.9.0, OTLP gRPC :4317 수신, 조회 API :3200)
      Grafana 데이터소스 등록 완료 (k8s/monitoring/tempo-datasource.yaml)
      ⚠️ ops-pool은 Spot VM — kubelet 순단으로 노드가 일시 NotReady 되면
         Tempo Service Endpoints가 비어 Grafana 연결이 끊길 수 있음(수 분 내 자동 복구됨)
```

## 배포 파이프라인

```
코드 변경 (app/)
    │
    ▼ git push → main
GitHub Actions CI (WIF 인증, github-ci SA)
    │ docker build + push (yq로 매니페스트 이미지 태그 갱신)
    ▼
Artifact Registry
  asia-northeast3-docker.pkg.dev/notiflex-09019/notiflex/api
    │
    ▼
ArgoCD App of Apps
  root-app (path: argocd/apps, directory.recurse: true, automated)
  ├── notiflex-smb (sync-wave "1", path: k8s/smb, ns: notiflex) — automated, Synced/Healthy
  └── notiflex-enterprise (sync-wave "2", path: k8s/enterprise, ns: enterprise) — automated, Synced/Healthy
  smb가 먼저 Healthy가 된 뒤 enterprise가 뒤따라 설치되도록 wave로 순서 지정
    │
    ▼
Argo Rollouts 컨트롤러 (namespace: argo-rollouts)
  각 테넌트별 독립 Canary: 20% → 30s → 50%(smb만) → 30s → 80%(smb만) → 30s → 100%
```

Kafka(`k8s/kafka/`)와 Tempo(helm-values 기반)는 ArgoCD App of Apps 관리 대상이 아니라
**helm/kubectl로 직접 설치**한다 — 모니터링 스택(Prometheus/Grafana/Loki)과 동일한 패턴이다.
notiflex-smb/enterprise 애플리케이션 워크로드만 GitOps로 관리하고, 클러스터 인프라 컴포넌트는
직접 설치가 이 프로젝트의 일관된 원칙이다.

## 관측 가능성

| 도구 | 역할 | 네임스페이스 | 상태 |
|------|------|------------|------|
| Prometheus | 메트릭 수집 (kube-prometheus-stack) | monitoring | Running |
| Grafana | 대시보드·데이터소스 통합 (Prometheus+Loki+Tempo) | monitoring | Running |
| Alertmanager | 알림 라우팅 (PrometheusRule 연동, Slack Incoming Webhook) | monitoring | Running |
| Loki | 로그 저장 (SingleBinary, filesystem) | monitoring | Running |
| Fluent Bit | 로그 수집 DaemonSet → Loki | monitoring | Running |
| Tempo | 분산 트레이싱 (OTLP gRPC :4317, 조회 :3200) | monitoring | Running (ops-pool) |
| Kafka UI | Kafka 토픽/Consumer Group/lag 조회 (조회 전용, GitOps 미관리) | kafka | Running (worker-pool) |

## 주요 네임스페이스

| 네임스페이스 | 주요 워크로드 | 비고 |
|------------|-------------|------|
| notiflex | Rollout/notiflex-api (Canary, v0.3.6, api-pool, Kafka+Tempo 연동), valkey-primary (StatefulSet, default-pool), SecretProviderClass/notiflex-secrets, ResourceQuota/notiflex-quota, CronJob/notiflex-healthcheck(ops-pool, 5분마다) | 전부 Healthy, ArgoCD 정식 관리 |
| enterprise | Rollout/notiflex-api (1 replica, api-pool), SecretProviderClass/notiflex-secrets, ResourceQuota/enterprise-quota, notiflex-sa | Healthy — notiflex 네임스페이스의 Valkey를 cross-namespace로 공유. Kafka/Tempo 미연동 |
| kafka | Strimzi Operator, Kafka(KRaft, v4.2.0, worker-pool), notifications 토픽, kafka-ui | 2026-09-22 설치, 전부 Running |
| argocd | ArgoCD v3.5.3 (root-app + notiflex-smb + notiflex-enterprise, App of Apps) | 세 Application 모두 Synced/Healthy |
| argo-rollouts | Argo Rollouts 컨트롤러 (v1.10.0) | default-pool, server-side apply로 설치됨 |
| monitoring | kube-prometheus-stack, Loki, Fluent Bit, Alertmanager, Tempo | 관측 가능성 3축(메트릭/로그/트레이스) 전부 갖춤 |
| default | (워크로드 없음) | GKE 기본 네임스페이스 |

**남은 재구축 항목**: ch9(회고, 온보딩 문서, GitAIOps 분석, 마무리)만 재구축 트랙에서 아직
다시 진행되지 않았다. 인프라·애플리케이션 컴포넌트는 ch8까지 전부 실제 배포 완료.
