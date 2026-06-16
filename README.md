# stockops-gitops

StockOps **CD(GitOps) 전용 레포**. ArgoCD 가 이 레포를 단일 진실(source of truth)로 보고
EKS 에 동기화한다. Terraform(인프라) 과 책임이 분리됨.

```
Stockops-GitOps 레포 생성
apps/stockops/
  base/                 # 리전 무관 워크로드 (api-server, ai-module)
  overlays/seoul/       # 네임스페이스 + 이미지 태그 주입(CI가 SHA 갱신)
argocd/
  stockops-seoul-application.yaml   # ArgoCD Application (서울)
```

## 소유 경계
- **Terraform**: VPC/EKS/ALB/RDS/ECR, ESO, LBC, ArgoCD 설치, aws-auth, IRSA,
  Service, TargetGroupBinding, HPA, Redis, namespace.
- **이 레포(ArgoCD)**: `stockops-api` / `stockops-ai` **Deployment 만**.

> replicas 는 HPA(Terraform)가 소유 → manifest 에 replicas 없음 +
> Application 의 `ignoreDifferences: /spec/replicas`.

---

## 0. 레포 생성 / 푸시
```bash
# 이 디렉토리에서
git init -b main
git add .
git commit -m "init: stockops gitops scaffold (seoul api+ai)"
git remote add origin https://github.com/<ORG>/stockops-gitops.git
git push -u origin main
```
`argocd/stockops-seoul-application.yaml` 의 `repoURL` 을 위 주소로 교체.

## 1. 로컬 렌더 확인(클러스터 영향 없음)
```bash
kubectl kustomize apps/stockops/overlays/seoul   # 에러 없이 두 Deployment 가 나오면 OK
```

## 2. cutover (다음 단계에서 진행) — 요약
> 무중단 핵심: **Terraform 추적만 끊고(live 유지) → ArgoCD 가 입양.**
```powershell
# (a) Terraform 에서 Deployment 추적 해제 (live 파드는 안 죽음)
cd seoul
terraform state rm kubernetes_deployment_v1.api_server
terraform state rm kubernetes_deployment_v1.ai_module
#   → 그리고 kubernetes.tf 에서 두 Deployment 블록 삭제(다음 apply 때 재생성 방지)

# (b) ArgoCD 에 Application 등록 (처음엔 automated 주석 후 수동 sync 로 diff 확인)
kubectl apply -f argocd/stockops-seoul-application.yaml
#   ArgoCD UI 에서 stockops-seoul → SYNC. OutOfSync 차이가 image/replicas 정도면 정상.
#   확인되면 automated(prune/selfHeal) 켜고 재적용.
```

## 3. CI 연결 (다음 단계)
`deploy.yml` 에서 `kubectl rollout restart` 제거 → 빌드/푸시 후 이 레포의
`overlays/seoul/kustomization.yaml` 의 `newTag` 를 빌드 SHA 로 바꿔 commit/push.
ArgoCD 가 감지해 자동 배포.

---
## 오하이오 확장
`overlays/ohio/` 추가(ECR newName=오하이오, 리전 종속 env 패치) +
`argocd/stockops-ohio-application.yaml`(destination.server = 오하이오 클러스터) 또는
ApplicationSet 클러스터 제너레이터로 일괄.

