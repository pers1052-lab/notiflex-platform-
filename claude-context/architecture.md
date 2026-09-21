# Notiflex 아키텍처 스냅샷 — 클러스터 재구축분, ch7.4 완료 시점

> ⚠️ **재구축 트랙 안내**: 이 클러스터(`notiflex-09019` 프로젝트)는 2026-09-17에 재생성되었고,
> 독자는 ch2부터 순차적으로 재시연 중이다. 이 문서는 **원래 ch9까지 완료했던 이전 스냅샷을
> 대체**하며, 현재 실제로 클러스터에 떠 있는 상태(ch7.4까지)만을 반영한다. ch8~9에서 다룬
> 컴포넌트(Kafka, Tempo, CronJob 결과 등)는 git 매니페스트(`k8s/`)에는 이미 존재하지만
> **Kafka·Tempo는 아직 이 클러스터에 배포되지 않았다.**

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
| 노드풀 | **5개 노드 전부 존재**: default-pool(e2-medium×2), api-pool(e2-medium×1), worker-pool(e2-standard-2×1), ops-pool(e2-small×1) — 모두 Spot, `GKE_METADATA` 적용 |
| Workload Identity | 활성화됨 (`notiflex-09019.svc.id.goog`), 4개 노드풀 전체 `GKE_METADATA` 적용 완료 |
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
  이미지: .../notiflex/api:v0.3.3 (CI가 sha-<commit> 태그로 자동 갱신)
  ⚠️ ch7.2부터 스크래치 적용본이 아니라 git(k8s/smb/rollout.yaml)을
     ArgoCD가 그대로 배포하는 정식 GitOps 경로로 복귀함
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
    │     컨테이너 기본값(미지정 시 자동 주입): request 50m/64Mi, limit 200m/256Mi
    │
    └── Valkey StatefulSet (valkey-primary, namespace: notiflex, default-pool)
          VALKEY_ADDR=valkey-primary.notiflex.svc.cluster.local:6379
          INCR 명령으로 Pod 간 공유 분산 ID 생성 (/id 엔드포인트)

Rollout: notiflex-api  (namespace: enterprise, nodeSelector: api-pool)  ← 별도 테넌트
  전략: Canary  20% → 30s pause → 100%  (replicas: 1)
  같은 이미지(v0.3.3), 같은 CSI Secret 패턴, 별도 notiflex-sa(WI 바인딩 별도)
    │
    ├── ResourceQuota/enterprise-quota + LimitRange/enterprise-limits (ns: enterprise)
    │     requests.cpu 200m / requests.memory 128Mi / limits.cpu 400m / limits.memory 256Mi / pods 5
    │
    └── VALKEY_ADDR=valkey-primary.notiflex.svc.cluster.local:6379  ← cross-namespace로
          notiflex 네임스페이스의 같은 Valkey를 공유 (테넌트 간 분산 ID 카운터 연속)
    외부 노출 없음 — Service(ClusterIP)만 존재, Gateway/HTTPRoute 미연결

※ Kafka·Tempo는 아직 미설치 — 앱 코드(main.go)는 KAFKA_BROKER/
  OTEL_EXPORTER_OTLP_ENDPOINT 미설정 시 non-fatal로 건너뛰도록 되어 있어
  현재는 두 연동 모두 비활성 상태로 정상 동작한다(OTel은 재시도 로그만 남김).
  worker-pool은 아직 비어있음 (ch8.1에서 Kafka 배치 예정)
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

ch6.3~ch7.1까지는 `notiflex-smb`의 automated sync를 임시로 꺼두고 스크래치 Rollout으로
데모했지만, **ch7.2에서 api-pool이 생기면서 git 스펙이 실제로 스케줄 가능해져 정식 GitOps
경로로 완전히 복귀했다.** 현재 두 Application 모두 수동 개입 없이 git push만으로 배포된다.

## 관측 가능성

| 도구 | 역할 | 네임스페이스 | 상태 |
|------|------|------------|------|
| Prometheus | 메트릭 수집 (kube-prometheus-stack) | monitoring | Running |
| Grafana | 대시보드·데이터소스 통합 | monitoring | Running |
| Alertmanager | 알림 라우팅 (PrometheusRule 연동, Slack Incoming Webhook) | monitoring | Running |
| Loki | 로그 저장 (SingleBinary, filesystem) | monitoring | Running |
| Fluent Bit | 로그 수집 DaemonSet → Loki | monitoring | Running |
| Tempo | 분산 트레이싱 (OTLP gRPC) | — | **미설치** (ch8.2 예정) |

## 주요 네임스페이스

| 네임스페이스 | 주요 워크로드 | 비고 |
|------------|-------------|------|
| notiflex | Rollout/notiflex-api (Canary, v0.3.3, api-pool), valkey-primary (StatefulSet, default-pool), SecretProviderClass/notiflex-secrets, ResourceQuota/notiflex-quota, CronJob/notiflex-healthcheck(ops-pool) | 전부 Healthy, ArgoCD 정식 관리 |
| enterprise | Rollout/notiflex-api (1 replica, api-pool), SecretProviderClass/notiflex-secrets, ResourceQuota/enterprise-quota, notiflex-sa | Healthy — notiflex 네임스페이스의 Valkey를 cross-namespace로 공유 |
| argocd | ArgoCD v3.5.3 (root-app + notiflex-smb + notiflex-enterprise, App of Apps) | 세 Application 모두 Synced/Healthy |
| argo-rollouts | Argo Rollouts 컨트롤러 | default-pool, server-side apply로 설치됨 |
| monitoring | kube-prometheus-stack, Loki, Fluent Bit, Alertmanager | Tempo 미포함 |
| default | (워크로드 없음) | GKE 기본 네임스페이스 |

**아직 없는 것**: `kafka` 네임스페이스(ch8.1), Tempo(ch8.2, monitoring에 설치 예정). worker-pool 노드는 존재하지만 워크로드가 없음(Kafka 배치 예정).
