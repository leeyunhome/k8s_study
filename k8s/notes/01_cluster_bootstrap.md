# 01. 클러스터 기동 (kind, 원격 GPU 장비)

- 날짜: 2026-09-18
- 환경: 원격 `deeplearning-H110-D3` (10.10.237.222), Ubuntu 24.04.4, Docker 29.6.2
- 읽기: Concepts > Overview, Cluster Architecture
  - https://kubernetes.io/docs/concepts/overview/
  - https://kubernetes.io/docs/concepts/architecture/

## 사전 점검 (Step 0) 결과

| 항목 | 값 |
|---|---|
| 디스크(`/`) | 1.8T, 사용 628G, 여유 1.1T |
| RAM | 15GB total / 10GB available |
| CPU | 8 vCPU |
| Docker | v29.6.2 (docker group, sudo 불필요) |
| GPU | RTX 3080 10GB, driver 580.126.09, CUDA 13.0 |
| OS | Ubuntu 24.04.4 LTS |

→ 계획서(`KUBERNETES.md`) 기준대로 Docker가 있으므로 **kind** 선택.
→ 같은 장비에 Hasura/SonarQube/Postgres 컨테이너가 이미 떠 있음 — 포트(5432, 8081, 9000, 33333, 8888 Jupyter)와 겹치지 않게 주의.

## 설치

- kubectl v1.37.0, kind v0.33.0 → `~/bin`에 설치 (sudo 불필요), `~/.bashrc`에 PATH 추가
- 클러스터: `kind create cluster --name jd-study`

## 확인

```
kubectl cluster-info
kubectl get nodes -o wide
kubectl -n kube-system get pods
```

- 노드 1개(`jd-study-control-plane`), kind 기본값이라 컨트롤 플레인 노드가 워커 역할도 겸함(단일 노드 클러스터)
- kube-system: etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kube-proxy, kindnet(CNI), coredns 2개 — 컨트롤 플레인 컴포넌트가 정적 Pod로 떠 있음을 실제로 확인
- node NotReady → CNI(kindnet) 준비 전에는 NotReady였다가, kindnet Running 이후 Ready로 전환되는 걸 관찰함 (CNI가 노드 Ready 조건인 이유를 체감)

## Q.
- 멀티노드 kind 클러스터로 바꾸면 Phase 3의 taint/affinity 실습이 더 명확해질 것 — 필요 시 `kind create cluster --config`로 재생성 예정
- Phase 5에서 kind 컨테이너 안에서 GPU를 어떻게 노출할지(NVIDIA Container Toolkit + kind의 extraMounts)는 별도 확인 필요
