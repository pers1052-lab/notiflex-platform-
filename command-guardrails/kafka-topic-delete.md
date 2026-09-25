# Kafka Topic 삭제

## 사전 확인
1. Topic에 미처리 메시지가 있는지 확인
2. Consumer가 모두 처리를 완료했는지 확인
3. 이 Topic을 사용하는 Producer 목록 파악

## 실행
1. 관련 Producer를 먼저 중지 (메시지 유입 차단)
2. Consumer가 잔여 메시지를 모두 처리할 때까지 대기
3. KafkaTopic 리소스 삭제

## 사후 검증
1. Topic이 삭제되었는지 확인
2. 관련 Producer/Consumer에 에러가 없는지 로그 확인
