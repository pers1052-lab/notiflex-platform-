# CronJob 수동 실행

## 사전 확인
1. CronJob의 schedule과 lastScheduleTime 확인 (직전 실행과 충돌하지 않는지)
2. 수동 실행 시 영향 범위 파악 (외부 API 호출, 데이터 갱신 등)

## 실행
1. `kubectl create job <name> --from=cronjob/<cronjob-name>` 으로 일회성 Job 생성
2. Job의 Pod 로그 모니터링

## 사후 검증
1. Job 완료 상태 확인 (Completed)
2. 결과가 의도한 대로 처리됐는지 확인 (예: 헬스체크 알림 정상 발송)
3. 수동 실행한 Job 정리(`kubectl delete job <name>`) — CronJob과 별개라 자동 정리 안 됨
