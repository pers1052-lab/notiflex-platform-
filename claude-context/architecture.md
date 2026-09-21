# Notiflex 아키텍처 스냅샷 — 클러스터 재구축분, ch6.3 완료 시점

> ⚠️ **재구축 트랙 안내**: 이 클러스터(`notiflex-09019` 프로젝트)는 2026-09-17에 재생성되었고,
> 독자는 ch2부터 순차적으로 재시연 중이다. 이 문서는 **원래 ch9까지 완료했던 이전 스냅샷을
> 대체**하며, 현재 실제로 클러스터에 떠 있는 상태(ch6.3까지)만을 반영한다. ch7~9에서 다룬
> 컴포넌트(멀티 노드풀, Kafka, Tempo, CronJob 등)는 git 매니페스트(`k8s/`)에는 이미 존재하지만
> **아직 이 클러스터에는 배포되지 않았다.**

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
| 노드풀 | **default-pool만 존재** (e2-medium × 2, Spot). api-pool/worker-pool/ops-pool은 ch7.2에서 재생성 예정 |
| Workload Identity | 활성화됨 (`notiflex-09019.svc.id.goog`), default-pool에 `GKE_METADATA` 적용 완료 |
| Secret Manager CSI | 활성화됨 (`--enable-secret-manager`, GKE managed addon, provider=`gke`) |
| Gateway API | 활성화됨, GatewayClass `gke-l7-regional-external-managed`, 외부 IP `35.216.16.34` |
| Artifact Registry | `asia-northeast3-docker.pkg.dev/notiflex-09019/notiflex/api` |

## 컴포넌트 다이어그램

```
외부 클라이언트
    │
    ▼ HTTP :80  (http://35.216.16.34)
GKE Gateway: notiflex-gateway (gke-l7-regional-external-managed)
    │ HTTPRoute: notiflex-route
    ▼
Service: notiflex-api (stable, ClusterIP)
Service: notiflex-api-preview (canary, ClusterIP)
    │
    ▼
Rollout: notiflex-api  (namespace: notiflex)
  전략: Canary  20% → 30s pause → 50% → 30s pause → 80% → 30s pause → 100%
  이미지: .../notiflex/api:v0.3.3
  ⚠️ 이 Rollout은 스크래치 적용본이다 — git의 k8s/smb/rollout.yaml은 이미
     api-pool nodeSelector + Kafka + OTel까지 반영된 "미래" 스펙이라
     아직 이 클러스터에는 그대로 적용할 수 없다 (아래 배포 파이프라인 참고)
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
    └── Valkey StatefulSet (valkey-primary, namespace: notiflex)
          VALKEY_ADDR=valkey-primary.notiflex.svc.cluster.local:6379
          INCR 명령으로 Pod 간 공유 분산 ID 생성 (/id 엔드포인트)

※ Kafka·Tempo는 아직 미설치 — 앱 코드(main.go)는 KAFKA_BROKER/
  OTEL_EXPORTER_OTLP_ENDPOINT 미설정 시 non-fatal로 건너뛰도록 되어 있어
  현재는 두 연동 모두 비활성 상태로 정상 동작한다 (ch8.1/ch8.2에서 활성화 예정)
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
ArgoCD (root-app → notiflex-smb Application)
  ⚠️ notiflex-smb는 현재 automated sync 비활성화 상태 (syncPolicy: {})
     이유: k8s/smb/rollout.yaml(미래 스펙)이 요구하는 api-pool·Kafka·Tempo가
     아직 없어, selfHeal이 이를 강제 적용하면 Pod가 계속 Pending에 빠짐.
     ch7.2(노드풀)·ch8.1(Kafka)·ch8.2(Tempo) 완료 후 automated로 복원 예정.
     → 그때까지는 CI가 매니페스트를 갱신해도 실배포는 kubectl로 수동 적용.
    │
    ▼
Argo Rollouts 컨트롤러 (namespace: argo-rollouts)
  Canary: 20% → 30s → 50% → 30s → 80% → 30s → 100%
```

**참고**: `notiflex-enterprise` Application은 여전히 automated selfHeal 상태라 git의
`k8s/enterprise`를 자동 적용 중이지만, `enterprise` 네임스페이스에 `notiflex-sa`가 없어
Pod 생성이 실패(`Degraded`)한다 — ch7.4(멀티테넌시) 재시연 시 해결 예정.

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
| notiflex | Rollout/notiflex-api (Canary, v0.3.3, default-pool), valkey-primary (StatefulSet), SecretProviderClass/notiflex-secrets, CronJob/notiflex-healthcheck | CronJob은 ops-pool 대상이라 Pending (ch7.2 대기) |
| enterprise | Rollout/notiflex-api (1 replica 목표) | notiflex-sa 없어 Degraded — ch7.4 대기 |
| argocd | ArgoCD v3.5.3 (root-app + notiflex-smb + notiflex-enterprise) | notiflex-smb는 수동 동기화 모드 |
| argo-rollouts | Argo Rollouts 컨트롤러 | server-side apply로 설치됨 |
| monitoring | kube-prometheus-stack, Loki, Fluent Bit, Alertmanager | Tempo 미포함 |
| default | (워크로드 없음) | GKE 기본 네임스페이스 |

**아직 존재하지 않는 네임스페이스**: `kafka`(ch8.1), `tempo` 전용 네임스페이스 없음(Tempo는 monitoring에 설치 예정, ch8.2). `api-pool`/`worker-pool`/`ops-pool` 노드풀도 ch7.2에서 재생성 필요.
