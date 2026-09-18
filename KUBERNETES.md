# 쿠버네티스 학습 계획 (공식 자료 기반)

딥러닝 스터디(`README.md`)와 병행하는 별도 트랙입니다.
JD의 **"Cloud 환경의 Architecture 설계 기술 및 경험"** 항목에 대응하고,
최종적으로 **Phase 1~4에서 학습한 PyTorch 모델을 쿠버네티스 위에 GPU 추론 서비스로 배포**하는 것을 목표로 합니다.

원칙: **자료는 공식 문서(kubernetes.io / CNCF / 각 프로젝트 공식 docs)만 사용**합니다.
블로그·강의 요약본은 막혔을 때 보조로만 쓰고, 기준은 항상 공식 문서 해당 페이지입니다.

---

## 0. 최종 산출물 (Definition of Done)

1. 매니페스트(YAML)로 정의된 재현 가능한 클러스터 워크로드 세트 (`k8s/` 디렉터리)
2. 원격 GPU 장비에서 `nvidia.com/gpu` 리소스를 요청하는 Pod 실행 성공
3. FashionMNIST/CIFAR-10 분류 모델을 FastAPI(또는 TorchServe/KServe)로 감싸 Deployment + Service + Ingress 로 노출
4. HPA, probe, ConfigMap/Secret, 리소스 제한이 적용된 "운영 가능한" 형태
5. 각 회차 학습 메모 (`k8s/notes/NN_*.md`)

---

## 1. 실습 환경 — **원격 GPU 장비 단독으로 진행**

검토 결과(2026-09-18), 실습은 **전 구간을 원격 Ubuntu 장비(`deeplearning-H110-D3`, 10.10.237.222)에서** 합니다.
로컬 Windows PC는 보조(SSH 터미널 + 매니페스트 편집 + git)로만 씁니다.

### 왜 로컬이 아닌가

| 항목 | 로컬 (Windows 10 Home) | 원격 (Ubuntu) |
|------|------------------------|---------------|
| 디스크 여유 | **C: 11.6GB / E: 4.2GB — 사실상 불가** | 딥러닝 데이터셋을 이미 쌓고 있음 (여유 확인 필요) |
| CPU | i5-7400 4코어 4스레드 (HT 없음) | 학습용 장비 |
| RAM | 16GB (WSL2가 절반 점유) | 〃 |
| GPU | Intel HD 630 (CUDA 불가) | RTX 3080 10GB |
| 컨테이너 런타임 | Docker 미설치 (WSL2 Ubuntu-22.04만 있음) | containerd/Docker |

결정적인 건 **디스크**입니다. Docker Desktop 설치만 ~3GB, kind 노드 이미지 ~1GB,
여기에 ingress-nginx·Calico·metrics-server·Prometheus 스택·직접 빌드한 PyTorch 이미지(수 GB)가 쌓이면
C: 11.6GB로는 Phase 3을 넘기지 못하고 막힙니다. 중간에 환경을 옮기면 그게 더 손해라 처음부터 원격으로 갑니다.

부수 효과로 좋은 점: **Phase 5(GPU 스케줄링·모델 서빙)까지 환경 이전 없이 이어집니다.**
또 "원격 리눅스 서버에 SSH로 붙어 클러스터를 다룬다"는 것 자체가 실제 운영 환경과 같은 모양입니다.

### 사전 점검 (Step 0) — 원격에서 가장 먼저 실행

```bash
ssh deeplearning@10.10.237.222
```
붙은 뒤:
```bash
df -h /                 # ★ 최소 40GB 이상 여유 권장 (Phase 5 이미지 빌드 포함)
free -g
nproc
docker version || echo NO_DOCKER
nvidia-smi
cat /etc/os-release | head -2
```

### 배포판 선택

- **Docker가 없다 → k3s 권장**: `curl -sfL https://get.k3s.io | sh -` 한 줄. containerd 내장, 경량, 단일 노드.
  단점은 기본 컴포넌트가 일부 교체(traefik, servicelb)되어 있어 공식 문서와 다른 부분이 생깁니다 — 설치 시 `--disable traefik`로 끄고 ingress-nginx를 직접 올리면 문서와 맞습니다.
- **Docker가 있다 → kind 권장**: 공식 문서 기본값에 가장 가깝고, 멀티노드 클러스터를 만들 수 있어
  Phase 3의 스케줄링(taint/affinity) 실습이 제대로 됩니다. 클러스터를 통째로 지웠다 만들기도 쉽습니다.

> 딥러닝 Jupyter 서버가 같은 장비에서 돌고 있습니다. 쿠버네티스가 GPU/메모리를 잡아먹지 않도록
> **Phase 1~4는 GPU를 전혀 쓰지 않는 워크로드로만** 진행하고, Phase 5의 GPU Job은
> 노트북 학습을 돌리지 않는 시간에 실행합니다.

### 로컬에서 하는 일

- `k8s/` 아래 매니페스트 YAML 편집 (VS Code) + git 커밋 — 딥러닝 노트북과 동일한 "편집은 로컬, 실행은 원격" 구조
- 원격 파일 동기화는 VS Code **Remote-SSH** 확장을 쓰면 가장 편합니다 (`.ipynb` 커널 연결과 별개로 설치 가능)
- 대시보드(Grafana 등) 접근은 딥러닝 Jupyter와 같은 방식의 SSH 포트포워딩:
  `ssh -N -L 3000:127.0.0.1:3000 deeplearning@10.10.237.222`

설치 문서:
- k3s: https://docs.k3s.io/quick-start
- kind: https://kind.sigs.k8s.io/docs/user/quick-start/
- kubectl: https://kubernetes.io/docs/tasks/tools/

> 버전은 고정하지 말고 설치 시점의 stable 을 씁니다. 다만 한 트랙 안에서는 같은 마이너 버전을 유지하세요.

## 2. 공식 자료 목록

**주 교재 (이 순서로 소비)**
- Learn Kubernetes Basics (대화형 튜토리얼): https://kubernetes.io/docs/tutorials/kubernetes-basics/
- Concepts (개념 레퍼런스, 이 계획의 축): https://kubernetes.io/docs/concepts/
- Tasks (하고 싶은 일별 레시피): https://kubernetes.io/docs/tasks/
- Tutorials (스테이트풀/컨피그맵 등 종합 실습): https://kubernetes.io/docs/tutorials/

**레퍼런스 (항상 열어두기)**
- kubectl Quick Reference: https://kubernetes.io/docs/reference/kubectl/quick-reference/
- API Reference: https://kubernetes.io/docs/reference/kubernetes-api/
- 용어집: https://kubernetes.io/docs/reference/glossary/

**보조 공식 자료**
- Linux Foundation 무료 강의 LFS158 "Introduction to Kubernetes": https://training.linuxfoundation.org/training/introduction-to-kubernetes/
- CNCF 자격 커리큘럼(CKA/CKAD 범위 = 학습 체크리스트로 유용): https://github.com/cncf/curriculum
- Helm 공식 docs: https://helm.sh/docs/
- Kustomize(kubectl 내장): https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/

**Phase 5용**
- GPU 스케줄링: https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/
- NVIDIA GPU Operator: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html
- KServe: https://kserve.github.io/website/latest/
- Kubeflow: https://www.kubeflow.org/docs/

---

## 3. 커리큘럼

각 회차는 **(1) 공식 문서 해당 섹션 읽기 → (2) 실습 → (3) `k8s/notes/`에 요약 + 막힌 점 "Q." 기록** 순서로 진행합니다.

### Phase 1 — 개념과 클러스터 (1~5)

1. **컨테이너 복습 & 클러스터 기동**
   - 읽기: Concepts > Overview, Cluster Architecture
   - 실습: `kind create cluster`, `kubectl cluster-info`, `kubectl get nodes -o wide`, 컨트롤 플레인 컴포넌트 Pod 확인(`kubectl -n kube-system get pods`)
2. **Pod**
   - 읽기: Concepts > Workloads > Pods
   - 실습: nginx Pod를 YAML로 작성 → `apply` / `describe` / `logs` / `exec` / `port-forward`
3. **kubectl 사용법 & 오브젝트 모델**
   - 읽기: Kubernetes Objects, Object Management, Labels and Selectors
   - 실습: `-o yaml`, `--dry-run=client`, `explain`, label/selector로 조회 필터링
4. **ReplicaSet / Deployment**
   - 읽기: Workloads > Deployments
   - 실습: 스케일 인/아웃, 롤링 업데이트, `rollout history` / `undo`, Pod 강제 삭제 후 자가치유 관찰
5. **Namespace / 리소스 요청과 제한**
   - 읽기: Namespaces, Resource Management for Pods and Containers
   - 실습: 네임스페이스 분리, `requests`/`limits` 설정 후 스케줄링 실패(Pending) 재현 및 `describe`로 원인 읽기

### Phase 2 — 워크로드와 설정 (6~10)

6. **Service (ClusterIP / NodePort / LoadBalancer)**
   - 읽기: Services, Networking > Service
   - 실습: Deployment 앞에 Service 붙이고 클러스터 내부 DNS(`svc.namespace.svc.cluster.local`)로 호출
7. **Ingress**
   - 읽기: Ingress, Ingress Controllers
   - 실습: kind에 ingress-nginx 설치 후 경로 기반 라우팅 (공식 kind 문서의 ingress 가이드 사용)
8. **ConfigMap / Secret**
   - 읽기: Configuration > ConfigMaps, Secrets
   - 실습: 환경변수 주입 vs 볼륨 마운트 두 방식 비교, Secret이 base64일 뿐임을 직접 확인
9. **Probe & 라이프사이클**
   - 읽기: Tasks > Configure Liveness, Readiness and Startup Probes
   - 실습: 일부러 실패하는 readiness probe로 트래픽 차단 관찰, `terminationGracePeriodSeconds` 실험
10. **Job / CronJob / DaemonSet / StatefulSet 개관**
    - 읽기: Workloads > Jobs, CronJob, DaemonSet, StatefulSet
    - 실습: 배치 학습 작업을 Job으로, 주기적 리포트를 CronJob으로 모사

### Phase 3 — 스토리지 · 스케줄링 · 보안 (11~15)

11. **Volume / PV / PVC / StorageClass**
    - 읽기: Storage > Volumes, Persistent Volumes
    - 실습: PVC를 붙인 Pod에 체크포인트 파일 쓰기 → Pod 삭제 후 재생성해서 데이터 유지 확인
12. **스케줄링 제어**
    - 읽기: Scheduling > Assigning Pods to Nodes, Taints and Tolerations, Affinity
    - 실습: nodeSelector / taint로 특정 노드(=GPU 노드 모사)에만 배치
13. **RBAC & ServiceAccount**
    - 읽기: Security > Authorization > RBAC, ServiceAccounts
    - 실습: 읽기 전용 Role을 가진 SA 생성 후 `kubectl auth can-i`로 검증
14. **네트워크 정책 & 클러스터 네트워킹 모델**
    - 읽기: Services/Networking > Network Policies, Cluster Networking
    - 실습: 네임스페이스 간 통신 차단 정책 적용 (CNI가 지원하는 환경에서 — kind 기본 CNI는 미지원이므로 Calico 설치 후 진행)
15. **트러블슈팅**
    - 읽기: Tasks > Monitor, Log, and Debug 전체
    - 실습: CrashLoopBackOff / ImagePullBackOff / Pending / OOMKilled 4가지를 **의도적으로 재현**하고 각각의 진단 절차 정리 ← 실무에서 가장 값어치 있는 회차

### Phase 4 — 패키징과 운영 (16~19)

16. **Kustomize** — base/overlay 로 dev/prod 분리
17. **Helm** — 앞서 만든 매니페스트를 차트로 패키징, values로 환경 분리
18. **오토스케일링** — HPA(metrics-server 설치), 부하 생성 후 스케일 아웃 관찰
    - 읽기: Tasks > Run Applications > Horizontal Pod Autoscale Walkthrough
19. **관측성** — kube-prometheus-stack 설치, Pod/노드 메트릭과 대시보드 확인

### Phase 5 — ML 워크로드 배포 (20~23) ★ 딥러닝 트랙과 결합

20. **GPU 노드 구성**
    - 읽기: Scheduling GPUs + NVIDIA GPU Operator 설치 가이드
    - 실습: `resources.limits."nvidia.com/gpu": 1` 을 요청한 Pod에서 `nvidia-smi` 성공
21. **학습 Job 컨테이너화**
    - 실습: Phase 2 미니 프로젝트(CNN 학습 스크립트)를 Dockerfile로 이미지화 → GPU Job으로 실행, 결과 체크포인트를 PVC에 저장
22. **추론 서비스 배포**
    - 실습: 저장된 체크포인트(또는 ONNX)를 로드하는 FastAPI 추론 서버 → Deployment + Service + Ingress + probe + HPA
23. **모델 서빙 프레임워크 맛보기**
    - 읽기: KServe First InferenceService
    - 실습: 같은 모델을 KServe `InferenceService`로 배포해 직접 만든 Deployment와 비교 (운영 부담 vs 제어권)

---

## 4. 저장소 구조 (추가 예정)

```
k8s/
  01_pods/              # 회차별 매니페스트
  02_workloads/
  ...
  app/                  # 추론 서비스 소스 + Dockerfile
  charts/               # Helm 차트
  overlays/             # Kustomize dev/prod
  notes/                # NN_주제.md  학습 메모 + "Q." 기록
```

---

## 5. 진행 방식

- 한 회차 = 공식 문서 1~2 섹션 + 매니페스트 실습 + 메모. 분량이 크면 쪼갭니다.
- **매니페스트는 반드시 손으로 YAML을 씁니다.** `kubectl run/create`는 뼈대 생성(`--dry-run=client -o yaml`)용으로만.
- 클러스터는 주기적으로 `kind delete cluster` 후 처음부터 재구축 — 재현성 확인용.
- 딥러닝 트랙과의 진행 비율은 자유롭게(예: DL 2회차 : K8s 1회차). Phase 5는 DL Phase 2가 끝난 뒤 시작해야 의미가 있습니다.

## 6. (선택) 자격증 연계

Phase 1~4를 마치면 CKA/CKAD 커리큘럼의 대부분을 다루게 됩니다.
목표로 삼는다면 https://github.com/cncf/curriculum 의 최신 도메인 체크리스트를 내려받아
빠진 항목만 보충 회차로 추가하세요. (CKA는 클러스터 구축/업그레이드·etcd 백업 비중이 있어 kubeadm 실습이 추가로 필요)
