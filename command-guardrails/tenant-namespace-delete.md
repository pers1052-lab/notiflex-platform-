# 테넌트 Namespace 삭제

## 사전 확인
1. 해당 namespace의 모든 워크로드 식별
2. PVC·Secret·ConfigMap 등 영구 자원 백업 필요성 판단
3. 다른 namespace에서 cross-namespace로 참조하는 리소스가 있는지 확인 (예: 공유 Valkey DNS)
4. ArgoCD Application(`notiflex-<tenant>`)이 이 namespace를 관리하는지 확인

## 실행
1. ArgoCD Application 먼저 제거 (`argocd/apps/notiflex-<tenant>.yaml` 삭제 → git push)
2. ArgoCD가 Application과 그 안의 리소스를 정리하기를 대기
3. 잔여 리소스가 있으면 매니페스트에서 제거하고 git push (kubectl delete 직접 사용 금지)

## 사후 검증
1. ArgoCD UI에서 Application이 사라졌는지 확인
2. `kubectl get all -n <namespace>`로 리소스가 모두 정리됐는지 확인
3. 남은 namespace에서 cross-namespace 참조가 끊겼는지 로그 확인
