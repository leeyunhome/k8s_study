# 쿠버네티스 학습 (공식 자료 기반)

PyTorch/HuggingFace 딥러닝 스터디([jd_deeplearning](https://github.com/leeyunhome/pytorch_basics))와
병행하는 학습 트랙입니다. 계획은 [KUBERNETES.md](KUBERNETES.md) 참고.

- 실습 환경: 원격 GPU 장비(Ubuntu, RTX 3080) + kind
- 자료: kubernetes.io 공식 문서 기준
- 목표: Phase 5에서 딥러닝 트랙의 학습 모델을 GPU 추론 서비스로 배포

## 구조

```
KUBERNETES.md   # 학습 계획 (Phase 0~6, 공식 문서 링크)
k8s/
  notes/        # 회차별 학습 메모
  01_pods/ ...  # 회차별 매니페스트
```

## 진행 현황

- [x] 01. 클러스터 기동 (kind) — [메모](k8s/notes/01_cluster_bootstrap.md)
- [ ] 02. Pod
- [ ] 03. kubectl 사용법 & 오브젝트 모델
