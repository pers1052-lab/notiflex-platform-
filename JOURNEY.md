# Notiflex 여정 기록

이 파일은 독자가 실제로 진행한 내용을 기록한다. AI가 각 챕터 완료 시 자동으로 업데이트한다.

## 진행 현황

| 챕터 | 서브챕터 | 상태 | 완료일 | 비고 |
|------|---------|------|--------|------|
| ch2 | 2.2 설치 확인 | ✅ | 2026-04-30 | |
| ch2 | 2.3 gcloud 설정 | ✅ | 2026-04-30 | 2026-09-17 재설정 (아래 트러블슈팅 참고) |
| ch2 | 2.4 GitHub 저장소 | ✅ | 2026-04-30 | 2026-09-17 기존 원격 저장소(origin) 연결 상태 재확인 |
| ch2 | 2.5 GKE 클러스터 | ✅ | 2026-04-30 | 2026-09-17 notiflex-09019 프로젝트에 재생성 (default-pool만 존재, 아래 현재 리소스 참고) |
| ch2 | 2.6 빌드/배포 | ✅ | 2026-04-30 | 2026-09-17 v0.1.0(원본 단순 버전)으로 임시 재배포 — 아래 트러블슈팅 참고 |
| ch2 | 2.7 첫 커밋 | ✅ | 2026-04-30 | |
| ch3 | 3.2 GitOps 도구 | ✅ | 2026-04-30 | 2026-09-18 클러스터 재구축 후 ArgoCD 재설치 (v3.5.3, root-app 적용) — 아래 트러블슈팅 참고 |
| ch3 | 3.3 기능 추가 | ✅ | 2026-04-30 | 2026-09-18 롤링 업데이트 재시연: 임시 Deployment를 v0.1.0→v0.1.1로 `kubectl set image` 교체(Git 미반영, ch2.6 방식과 동일) — 아래 트러블슈팅 참고 |
| ch3 | 3.4 CI | ✅ | 2026-04-30 | 2026-09-18 프로젝트 재생성으로 WIF Pool/Provider·CI SA(`github-ci`) 재생성, GitHub Secrets(GCP_PROJECT_ID/GCP_SA_EMAIL/GCP_WIF_PROVIDER) 재등록, Workflow permissions write로 변경 — 아래 트러블슈팅 참고 |
| ch3 | 3.5 CI-CD 연결 | ✅ | 2026-04-30 | 2026-09-18 `ci.yaml`의 매니페스트 갱신 방식을 sed→yq로 전환 (구 프로젝트 ID 잔존 버그 수정). 엔드투엔드 테스트 완료: 코드 push→CI 빌드→매니페스트 갱신→ArgoCD 감지(커밋 a800b5e)까지 전부 자동 확인. 단 Rollout 실배포는 CRD 부재로 아직 불가(ch5.3 필요) — 아래 트러블슈팅 참고 |
| ch4 | 4.2 메트릭 모니터링 | ✅ | 2026-04-30 | 2026-09-18 클러스터 재구축 후 kube-prometheus-stack 재설치 (Prometheus/Grafana/Alertmanager 등 7 Pod 전부 Running, Pending 없음). Prometheus 타겟 16/18 up(coredns 2개만 down, GKE 환경 특성상 무해) |
| ch4 | 4.3 로그 수집 | ✅ | 2026-04-30 | 2026-09-18 Loki+Fluent Bit 재설치. Grafana Loki 데이터소스 오등록, Fluent Bit output 설정 오류 2건 발견/수정 — 아래 트러블슈팅 참고 |
| ch4 | 4.4 알림 | ✅ | 2026-04-30 | |
| ch5 | 5.2 트래픽 관리 | ✅ | 2026-04-30 | |
| ch5 | 5.3 무중단 배포 | ✅ | 2026-04-30 | |
| ch5 | 5.4 ADR 기록 | ✅ | 2026-04-30 | |
| ch6 | 6.1 캐시 | ✅ | 2026-04-30 | |
| ch6 | 6.2 시크릿 관리 | ✅ | 2026-04-30 | |
| ch6 | 6.3 Canary 전환 | ✅ | 2026-04-30 | |
| ch6 | 6.4 아키텍처 스냅샷 | ✅ | 2026-04-30 | |
| ch7 | 7.2 멀티 노드풀 | ✅ | 2026-04-30 | |
| ch7 | 7.3 App of Apps | ✅ | 2026-04-30 | |
| ch7 | 7.4 멀티테넌시 | ✅ | 2026-04-30 | |
| ch8 | 8.1 메시징 | ✅ | 2026-04-30 | |
| ch8 | 8.2 트레이싱 | ✅ | 2026-04-30 | |
| ch8 | 8.3 CronJob | ✅ | 2026-04-30 | |
| ch9 | 9.1 저장소 분석 | ✅ | 2026-04-30 | |
| ch9 | 9.2 회고 | ✅ | 2026-04-30 | |
| ch9 | 9.3 온보딩 문서 | ✅ | 2026-04-30 | |
| ch9 | 9.4 GitAIOps 분석 | ✅ | 2026-04-30 | |
| ch9 | 9.5 마무리 | ✅ | 2026-04-30 | |

## 도구 선택 기록

독자가 3-프롬프트 패턴(탐색→비교→실행)에서 실제로 선택한 도구와 이유를 기록한다.

| 영역 | 선택 | 검토한 대안 | 선택 이유 |
|------|------|-----------|----------|
| GitOps (ch3.2) | ArgoCD v3.3.8 | Flux, Jenkins X | K8s 네이티브, Web UI, App of Apps 지원 |
| CI (ch3.4) | GitHub Actions | Jenkins, GitLab CI | 저장소 통합, WIF 지원, 무료 |
| 메트릭 (ch4.2) | Prometheus + Grafana | Datadog, New Relic | 오픈소스, kube-prometheus-stack 통합 |
| 로깅 (ch4.3) | Loki + Fluent Bit | ELK, Datadog Logs | Grafana 통합, 경량 인덱싱 |
| 알림 (ch4.4) | PrometheusRule | Grafana Alert | Prometheus 네이티브, PromQL 표현식 |
| 트래픽 관리 (ch5.2) | Gateway API (gke-l7-regional-external-managed) | Ingress, NGINX | GKE 네이티브, K8s 표준, HealthCheckPolicy |
| 배포 전략 (ch5.3) | Argo Rollouts Blue/Green | Flagger, Istio | ArgoCD 통합, 즉각 롤백, autoPromotion |
| 캐시 (ch6.1) | Valkey (Bitnami, standalone) | Redis, Memcached | Redis fork, BSD-3 라이선스, INCR 분산 ID |
| 시크릿 관리 (ch6.2) | GKE Secret Manager CSI + WI | K8s Secret, Vault | GKE 네이티브, SA 키 불필요, 파일 마운트 |
| 배포 전략 전환 (ch6.3) | Argo Rollouts Canary | Blue/Green 유지 | 트래픽 점진 이동, 운영 위험 최소화 |
| 노드 스케줄링 (ch7.2) | nodeSelector (cloud.google.com/gke-nodepool) | nodeAffinity, Taint/Toleration | GKE 자동 라벨, 단순 YAML, 역할별 노드풀 분리 |
| 멀티앱 관리 (ch7.3) | App of Apps (argocd/apps/ 디렉터리) | ApplicationSet, 개별 Application | 파일 추가만으로 앱 등록, Sync Wave 순서 보장 |
| 멀티테넌시 (ch7.4) | Namespace 분리 + per-tenant Rollout | 단일 namespace + 라벨 격리, vCluster | 강한 격리, ArgoCD App of Apps와 자연 결합, 테넌트별 독립 배포 |
| 배치 자동화 (ch8.3) | K8s CronJob | 외부 cron + 쿠버네티스 외부 트리거, Argo Workflows | 쿠버네티스 네이티브, ops-pool 배치, ArgoCD가 매니페스트로 관리 |

## 현재 버전

| 컴포넌트 | 버전 | 변경 이력 |
|---------|------|----------|
| Go | 1.25 | |
| Notiflex 이미지 | v0.3.1 (저장소 코드 기준) / **v0.1.0 (2026-09-17 재구축 클러스터에 실제 배포된 버전)** | v0.1.0→v0.1.1→v0.2.0(Valkey)→v0.2.1(CSI)→v0.3.0(Kafka)→v0.3.1(OTel) |
| ArgoCD | v3.5.3 | 2026-09-18 재설치 (stable manifest 기준 최신) |
| Kafka | 4.1.0 (Strimzi 1.0.0, KRaft) | |
| OTel SDK | - (Tempo 설치, SDK 적용) | |

## 현재 리소스

> ⚠️ 2026-09-17 클러스터 재생성 직후 상태. api-pool/worker-pool/ops-pool은 ch7.2/ch8에서 다시 만들어야 아래 워크로드가 배치된다 (현재는 default-pool만 존재).

| 노드풀 | 머신 타입 | 노드 수 | 주요 워크로드 |
|--------|----------|---------|-------------|
| default-pool | e2-medium | 2 (Spot) | (재생성 직후, Gateway API 활성화됨) |
| api-pool | e2-medium | 1 | notiflex-api (smb + enterprise) — 재생성 필요 (ch7.2) |
| worker-pool | e2-standard-2 | 1 | Kafka (ch8) — 재생성 필요 (ch7.2) |
| ops-pool | e2-small | 1 | Tempo, CronJob (ch8) — 재생성 필요 (ch7.2) |

## 트러블슈팅 이력

독자가 겪은 문제와 해결 방법을 기록한다. 같은 문제를 다시 겪지 않도록 한다.

| 챕터 | 문제 | 해결 |
|------|------|------|
| 2026-09-17 | 기존 GCP 프로젝트(`project-75fce205-dfa5-4975-a56`)가 계정 접근 불가/소멸 상태로 확인됨 (`gcloud projects describe` 권한 오류) | 신규 프로젝트 `notiflex-09019` 생성 후 결제 계정 연결, gcloud 설정(project/zone/region) 재적용, GKE 클러스터(`notiflex-cluster`) 재생성. 저장소 코드/매니페스트는 유지하되 `CLAUDE.md`의 프로젝트 ID를 갱신함. `k8s/*/rollout.yaml`, `k8s/*/secret-provider.yaml`, `claude-context/architecture.md`, `ONBOARDING.md`는 아직 옛 프로젝트 ID를 참조 중 — 해당 챕터(ch2.6, ch6.2) 도달 시 갱신 필요 |
| 2026-09-17 | `app/main.go`가 이미 ch8까지 진화한 v0.3.1(Valkey/Kafka/OTel 의존) 상태라, Valkey 없는 새 클러스터에 그대로 배포하면 크래시 | 2.6 검증 목적으로만 초기 커밋(b46fb1b)의 단순 버전을 스크래치패드에서 빌드(`v0.1.0`, 이미지: `.../notiflex/api:v0.1.0`)해 배포. `app/main.go`는 건드리지 않음(현재도 v0.3.1 그대로). 배포한 Deployment/Service는 저장소 파일이 아니라 `kubectl apply -f -`로 즉석 적용한 임시 리소스 — `k8s/smb/rollout.yaml`(Argo Rollouts)은 아직 미적용. ch3.2(ArgoCD)·ch5.3(Argo Rollouts)·ch6.1(Valkey) 재구축 시 이 임시 Deployment를 정식 Rollout으로 교체해야 함 |
| 2026-09-18 | `argocd/root-app.yaml`, `argocd/apps/notiflex-smb.yaml`, `argocd/apps/notiflex-enterprise.yaml`의 `repoURL`이 예제 저장소(`sysnet4admin/notiflex-platform`)를 가리키고 있어 실제 사용자 저장소(`pers1052-lab/notiflex-platform-`)가 동기화되지 않음 | 세 파일의 repoURL을 사용자 저장소로 수정 후 커밋/push (main). ArgoCD 설치 직후 발견되는 known NetworkPolicy egress 차단 이슈도 선제적으로 `kubectl delete networkpolicy -n argocd --all`로 제거 |
| 2026-09-18 | ArgoCD 재설치 후 `notiflex-smb` Application이 `OutOfSync`/일부 리소스 `SyncFailed` — `Rollout.argoproj.io "" not found`, `SecretProviderClass` CRD 없음 | 예상된 상태. `k8s/smb/rollout.yaml`(Argo Rollouts)과 `k8s/smb/secret-provider.yaml`(Secrets Store CSI)이 요구하는 CRD가 재구축된 클러스터에 아직 없기 때문. ch5.3(Argo Rollouts 설치), ch6.2(Secret Manager CSI 설치) 진행 시 자동 해결됨 (selfHeal이 재시도 중이므로 별도 조치 불필요). root-app 자체는 Synced/Healthy |
| 2026-09-18 | ch3.3 롤링 업데이트 재시연 시점에 `app/main.go`가 이미 v0.3.1(Kafka/Valkey/OTel 의존)이라 실제 GitOps 대상(Rollout)에 그대로 반영 불가 | 초기 커밋(b46fb1b)의 단순 버전을 스크래치패드에서 `version="v0.1.1"`로 수정해 빌드(이미지: `.../notiflex/api:v0.1.1`), 임시 Deployment에 `kubectl set image`로 적용해 롤링 업데이트만 데모. `/version` 응답 확인(`{"version":"v0.1.1"}`). Git에는 미반영 — 실제 Rollout 기반 배포는 ch5.3/ch6.1/ch8.1 완료 후 진행 필요 |
| 2026-09-18 | ch3.4 재설정: 프로젝트 재생성으로 WIF Pool/Provider와 CI SA가 사라져 기존 `ci.yaml`(WIF 인증)이 동작 불가 | `github-ci` SA 재생성(`roles/artifactregistry.writer`), `github-pool`/`github-provider` WIF 재생성(저장소 `pers1052-lab/notiflex-platform-`만 impersonate 허용), GitHub Secrets 3개 재등록. `gh` CLI 로컬 설치 후 `gh auth login`으로 인증. repo Workflow permissions가 기본 `read`라 매니페스트 push 403 예상되어 `write`로 선제 변경 |
| 2026-09-18 | `ci.yaml`의 `Update manifest` 스텝이 `sed`로 이미지 태그 뒤쪽만 치환해, 앞부분의 구 프로젝트 ID(`project-75fce205-dfa5-4975-a56`)가 `k8s/smb/rollout.yaml`에 그대로 남아있던 버그 발견 (CI 실행 시 존재하지 않는 옛 프로젝트 레지스트리 경로로 배포될 뻔함) | `sed`를 `yq eval '.spec.template.spec.containers[0].image = ...' -i`로 교체, 직전 스텝에서 만든 `$IMAGE`(올바른 `GCP_PROJECT_ID` 포함) 재사용. `yq`는 `ubuntu-latest` 러너에 기본 내장이라 추가 설치 불필요 |
| 2026-09-18 | `yq eval -i`가 이미지 필드 외에 파일 전체 들여쓰기 스타일을 재포맷해 diff가 45줄 삭제/45줄 추가로 부풀어짐 (`volumes:` 리스트 인덴트 6→8칸 변경) | 의도한 값(이미지 경로)은 정확히 반영되어 기능상 문제는 없음. yq의 알려진 부작용으로 기록만 해둠 — diff 노이즈를 줄이려면 추후 `yq -P`(원본 스타일 보존) 옵션 검토 가능 |
| 2026-09-18 | Grafana에서 Loki 데이터소스 조회 시 "non-json... <!DOCTYPE html>" 에러 — 사용자가 UI에서 직접 만든 Loki 데이터소스(uid 자동생성)의 URL이 잘못 설정됨 | `k8s/monitoring/loki-datasource.yaml`(ConfigMap, `grafana_datasource: "1"` 라벨)로 올바른 URL(`http://loki.monitoring.svc.cluster.local:3100`, uid=loki)의 데이터소스를 프로비저닝. 사이드카의 자동 reload API 호출이 401로 실패해(아래 항목 참고) Grafana Deployment를 `rollout restart`로 재기동해 확실히 반영 |
| 2026-09-18 | Grafana(v13.2.2-distroless) API에 대한 HTTP Basic Auth가 관리자 계정(`admin`/`admin`, 시크릿 값과 동일)으로도 계속 401 실패 — `curl -u`뿐 아니라 grafana-sc-datasources 사이드카의 reload 호출도 동일하게 401. 반면 브라우저 세션 로그인은 정상 동작 | 원인 미확정(이 Grafana 버전에서 API Basic Auth가 브라우저 세션 로그인과 다르게 동작하는 것으로 추정). 실무 영향: 사이드카의 datasource 자동 reload가 안 먹히므로, 데이터소스/대시보드 ConfigMap을 추가할 때마다 `kubectl rollout restart deployment kube-prometheus-grafana -n monitoring`으로 수동 재기동 필요 |
| 2026-09-18 | Fluent Bit 로그가 Loki에 전혀 도달하지 않음 (namespace 라벨 목록에 notiflex 없음) | `helm-values/fluent-bit.yaml`이 `config.outputs`(raw 텍스트) 방식으로 작성돼 있었는데, 실제 사용 중인 `grafana/fluent-bit`(deprecated) 차트는 이 키를 지원하지 않고 `loki.serviceName`/`config.labelMap`으로 설정해야 함. 미설정 시 기본값 `${RELEASE}-loki`(=`fluent-bit-loki`, 존재하지 않는 서비스)로 렌더링되어 전송 실패. `loki.serviceName: loki` + `config.labelMap`으로 수정 후 `helm upgrade`, Pod 재시작 후 `{namespace="notiflex"}` 쿼리로 실제 로그 수신 확인 완료 |
