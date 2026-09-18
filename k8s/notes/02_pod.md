# 02. Pod

- 날짜: 2026-09-18
- 읽기: Concepts > Workloads > Pods
  - https://kubernetes.io/docs/concepts/workloads/pods/

## 실습

- `nginx-pod.yaml` 작성 후 `kubectl apply -f`
- `kubectl get pods -o wide` → 처음 Pending, ~20초 후 Running
- `kubectl describe pod` Events로 생명주기 관찰: Scheduled → Pulling(18s, 72MB) → Pulled → Created → Started
  - Pending 상태의 원인이 "이미지 다운로드 대기"였음을 Events로 직접 확인
- `kubectl logs` → nginx가 8개 worker process로 시작 (원격 장비 8 vCPU와 일치)
- `kubectl exec -it nginx-pod -- /bin/bash` 로 컨테이너 진입 → `curl localhost:80` 정상 응답 확인
- `kubectl port-forward pod/nginx-pod 8080:80` → "Forwarding from 127.0.0.1:8080 -> 80" 확인
- `kubectl delete pod nginx-pod` 로 정리

## YAML 실수 & 교정 (트러블슈팅 기록)

vim의 autoindent 때문에 줄바꿈마다 들여쓰기가 누적되어 YAML 계층이 완전히 깨짐
(`spec:`이 `labels:` 아래로 파고들어감) →
`error converting YAML to JSON: yaml: line 5: mapping values are not allowed in this context`

해결: heredoc(`cat > file << 'EOF' ... EOF`)으로 파일을 다시 생성.
YAML은 들여쓰기가 곧 스키마이므로, 에디터의 자동 들여쓰기 기능이 오히려 방해가 될 수 있음을 확인.
앞으로는 `cat -A file`로 공백/들여쓰기를 눈으로 검증하는 습관을 들이기로 함.

## Q.
- port-forward 상태에서 별도 터미널로 `curl localhost:8080` 실제 응답 확인을 안 하고 종료함 →
  다음 회차(Service) 전에 한 번 더 해보고 "로컬 요청이 Pod까지 전달되는 경로"를 체감할 것
