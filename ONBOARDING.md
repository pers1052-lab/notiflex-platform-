# Notiflex Platform 온보딩 가이드

> 2026-09-22 기준, ch8까지 완료된 실제 클러스터 상태로 작성됨(재구축 트랙).

## 빠른 시작

### 필수 도구

| 도구 | 버전 | 역할 |
|------|------|------|
| gcloud CLI | 529+ | GCP 인증 및 클러스터 접근 |
| kubectl | 1.35+ | K8s 클러스터 운영 |
| helm | 3.x | Helm 차트 관리 (Strimzi, Tempo, kafka-ui 등 GitOps 미관리 컴포넌트) |
| gh | 2.x | GitHub 저장소 관리 |

### 클러스터 접근

```bash
gcloud container clusters get-credentials notiflex-cluster \
  --zone=asia-northeast3-a \
  --project=notiflex-09019
kubectl config rename-context \
  $(kubectl config current-context) \
  gke-sysnet4admin_book_gitaiops
kubectl --context gke-sysnet4admin_book_gitaiops get nodes
```

> ⚠️ **모든 kubectl 명령에 `--context gke-sysnet4admin_book_gitaiops`를 반드시 지정한다.** 다른 클러스터에 실수로 명령이 나가는 걸 막기 위함.

## 클러스터 실제 상태

### 노드풀 (5개 노드, 역할별 전용 배치)

| 노드풀 | 머신 타입 | 노드 수 | 용도 |
|--------|----------|---------|------|
| default-pool | e2-medium | 2 | ArgoCD, Argo Rollouts, Prometheus/Grafana/Alertmanager/Loki, Valkey |
| api-pool | e2-medium | 1 | notiflex-api (SMB + Enterprise 두 테넌트) |
| worker-pool | e2-standard-2 | 1 | Kafka(Strimzi Operator + 브로커), Kafka UI |
| ops-pool | e2-small | 1 | Tempo, notiflex-healthcheck CronJob |

전부 Spot VM. ops-pool은 e2-small(2GB)로 가장 작아 메모리 여유가 빠듯하다 — 컴포넌트 추가 전 `kubectl top node`로 확인할 것.

### 네임스페이스별 Pod 현황

| 네임스페이스 | Pod 수 | 역할 |
|------------|--------|------|
| kube-system | 57 | GKE 시스템 DaemonSet (CSI, DNS, netd 등) — 조정 불가 |
| monitoring | 23 | Prometheus, Grafana, Alertmanager, Loki, Fluent Bit, Tempo |
| argocd | 7 | ArgoCD 컨트롤 플레인 (server, repo-server, application-controller 등) |
| notiflex | 6 | notiflex-api×2, valkey-primary, healthcheck Job 이력 |
| gmp-system | 6 | GKE 관리형 Prometheus 수집기 |
| kafka | 4 | Strimzi Operator, Kafka 브로커, entity-operator, kafka-ui |
| enterprise | 1 | notiflex-api×1 (Enterprise 테넌트) |
| argo-rollouts | 1 | Argo Rollouts 컨트롤러 |

## 저장소 디렉터리 구조

```
notiflex-platform/
├── app/                    Go 앱 소스 (main.go, go.mod)
├── k8s/                    K8s 매니페스트
│   ├── smb/                SMB 테넌트 (Rollout, Gateway, CronJob 등)
│   ├── enterprise/          Enterprise 테넌트
│   ├── kafka/               Kafka 클러스터 정의
│   └── monitoring/          Grafana 데이터소스, PrometheusRule
├── argocd/                 App of Apps (root-app + apps/)
├── helm-values/             GitOps 미관리 컴포넌트 values (Prometheus, Loki, Strimzi, Tempo, kafka-ui)
├── claude-context/          현재 아키텍처 스냅샷 (architecture.md)
├── command-guardrails/      위험 작업 절차 (Kafka Topic 삭제, CronJob 수동 실행, 테넌트 삭제)
├── docs/architecture-decisions.md   ADR 누적 기록
├── .claude/                 커스텀 스킬(/update-docs), settings
├── CLAUDE.md                 AI 행동 규칙 (매 대화 자동 로드)
└── JOURNEY.md                 진행 기록 (사람이 읽는 히스토리)
```

**GitOps 관리 원칙**: `k8s/smb/`, `k8s/enterprise/`는 ArgoCD가 자동 배포한다. `helm-values/`의 컴포넌트(Kafka, Tempo, 모니터링 스택)는 helm/kubectl로 직접 설치하며 App of Apps 대상이 아니다.

## 접근 방법

### ArgoCD UI

```bash
kubectl --context gke-sysnet4admin_book_gitaiops -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
kubectl --context gke-sysnet4admin_book_gitaiops port-forward svc/argocd-server -n argocd 8443:443
```
→ https://localhost:8443 (ID: `admin`)

### Grafana

```bash
kubectl --context gke-sysnet4admin_book_gitaiops port-forward svc/kube-prometheus-grafana -n monitoring 3000:80
```
→ http://localhost:3000 (ID: `admin` / PW: `admin`)

데이터소스별 용도:
- **Prometheus**: 메트릭. Explore에서 PromQL로 `kube_cronjob_info`, `kube_pod_container_status_restarts_total` 등 조회
- **Loki**: 로그. LogQL로 `{namespace="notiflex"}` 형태 쿼리
- **Tempo**: 트레이스. TraceID 직접 조회 또는 Search 탭에서 span name(`id`, `valkey.incr`, `kafka.produce`)으로 검색

### Kafka UI (조회 전용)

```bash
kubectl --context gke-sysnet4admin_book_gitaiops port-forward svc/kafka-ui -n kafka 8081:80
```
→ http://localhost:8081 (인증 없음) — 토픽, 파티션, Consumer Group lag 확인

### API 엔드포인트

```bash
GATEWAY_IP=$(kubectl --context gke-sysnet4admin_book_gitaiops get gateway -n notiflex -o jsonpath='{.status.addresses[0].value}')
curl http://$GATEWAY_IP/health
curl http://$GATEWAY_IP/id
```

## 배포 플로우

```
코드 변경(app/) → git push main
    ↓
GitHub Actions CI (WIF 인증) → docker build/push → Artifact Registry
    ↓                          → k8s/smb/rollout.yaml 이미지 태그 자동 갱신(yq) → git push
ArgoCD root-app (App of Apps)
    ↓
notiflex-smb / notiflex-enterprise Application → 자동 Sync
    ↓
Argo Rollouts Canary: 20% → 30s → 50% → 30s → 80% → 30s → 100%
```

Kafka/Tempo/모니터링 스택 변경은 이 파이프라인을 안 거친다 — `helm upgrade`로 직접 반영 후 `JOURNEY.md`에 수동 기록한다.

## 자주 묻는 Q&A

**Q1. Canary 배포를 중간에 중단(abort)하고 싶다면?**
```bash
kubectl argo rollouts abort notiflex-api -n notiflex   # CLI 플러그인 설치 시
# 또는 플러그인 없이:
kubectl patch rollout notiflex-api -n notiflex --type merge -p '{"spec":{"paused":true}}'
```
직전 stable ReplicaSet으로 자동 롤백되지 않으므로, 완전 롤백은 `kubectl argo rollouts undo` 또는 이전 이미지 태그로 재배포.

**Q2. 로그를 어떻게 검색해?**
Grafana → Explore → Loki. 예: `{namespace="notiflex", app="notiflex-api"} |= "Kafka"` — 네임스페이스+앱 라벨로 좁힌 뒤 키워드 필터링.

**Q3. 특정 요청의 트레이스를 어떻게 추적해?**
`/health` 응답 헤더나 앱 로그에서 TraceID를 못 얻는 구조라면, Grafana → Explore → Tempo → Search 탭에서 시간 범위 + span name(`id`)으로 검색 후 duration 오름차순/내림차순 정렬로 이상치를 찾는다.

**Q4. Kafka 토픽을 추가하려면?**
`k8s/kafka/kafka-cluster.yaml`에 `KafkaTopic` 리소스를 추가하고 `kubectl apply`(GitOps 미관리이므로 직접 적용). 삭제 시에는 `command-guardrails/kafka-topic-delete.md` 절차를 따른다.

**Q5. 새 테넌트를 추가하려면?**
`k8s/enterprise/`를 템플릿 삼아 `k8s/<tenant>/` 디렉터리 생성(namespace, rollout, secret-provider, resourcequota, service) → `argocd/apps/notiflex-<tenant>.yaml` 추가 → root-app이 자동 인식. 공유 Valkey를 쓸지, 테넌트 전용 캐시를 둘지는 격리 요구 수준에 따라 결정.

**Q6. 알림이 왔는지 어떻게 확인해?**
Grafana → Alerting → Alert rules 에서 PrometheusRule 상태(`firing`/`pending`/`inactive`) 확인. 실제 Slack 수신 여부는 채널을 직접 확인하거나, Alertmanager UI(`kubectl port-forward svc/kube-prometheus-kube-prome-alertmanager -n monitoring 9093:9093`)에서 알림 이력을 조회한다.

**Q7. CronJob이 실패하면?**
`kubectl get jobs -n notiflex`로 실패한 Job 확인 → `kubectl logs job/<name> -n notiflex`로 원인 파악. 수동 재실행은 `command-guardrails/cronjob-manual-run.md` 절차를 따른다.

## 아키텍처 요약

`claude-context/architecture.md` 참조 — ch8 완료 시점 스냅샷(실제 배포 상태 반영).

핵심 흐름:
1. 코드 변경 → GitHub → ArgoCD → Argo Rollouts (Canary)
2. `/id` 호출 → Valkey INCR → Kafka 이벤트 발행(Consumer Group `notiflex-workers`가 파티션 분산 소비) → OTel 트레이스 → Tempo
3. PrometheusRule → Alertmanager → Slack 알림

## 의사결정 이력

`docs/architecture-decisions.md` — ADR-001~018, 18개 결정 기록 (번호가 챕터 순서와 완전히 일치하진 않음 — ADR-017/018은 각각 ch4.4/ch7.4 결정이 뒤늦게 추가된 것)

| 챕터 | ADR 범위 | 핵심 결정 |
|------|----------|----------|
| ch3 | 001~002 | ArgoCD, GitHub Actions |
| ch4 | 003~005, 017 | Prometheus+Grafana, Loki+Fluent Bit, PrometheusRule, Slack 알림 채널 |
| ch5 | 006~007 | Gateway API, Blue/Green |
| ch6 | 008~010 | Valkey, Secret Manager CSI, Canary |
| ch7 | 011~013, 018 | 노드풀 분리, App of Apps, 멀티테넌시, ResourceQuota/LimitRange |
| ch8 | 014~016 | Kafka(KRaft, v4.2.0), Tempo, CronJob |

## 트러블슈팅

자주 발생하는 문제:
- ArgoCD Sync 실패 → `argocd.argoproj.io/refresh=hard` 어노테이션
- Valkey 연결 실패 → `kubectl get pod valkey-primary-0 -n notiflex`
- CSI Secret 마운트 실패 → Workload Identity 바인딩 확인 (전파 지연 1~2분 발생 가능)
- Kafka entity-operator CrashLoop → `userOperator` 섹션 제거(KafkaUser 미사용 시 불필요)
- Kafka 버전 불일치(`UnsupportedKafkaVersionException`) → `kubectl describe kafka`로 Strimzi가 지원하는 버전 목록 확인 후 조정
- ops-pool 등 Spot VM 노드가 일시 `NotReady` → 보통 수 분 내 자동 복구. 성급하게 Pod 재생성/노드 삭제하지 말 것
- helm-values 파일 작성 시 키가 적용 안 됨 → `helm show values <chart>`로 실제 스키마 경로 먼저 확인 (예: Tempo는 최상위 `resources`가 아니라 `tempo.resources`)
- Grafana 데이터소스가 "Failed to connect" → 대상 컴포넌트가 떠 있는 노드의 Ready 상태부터 확인
