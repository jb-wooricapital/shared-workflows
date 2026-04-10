# Shared Workflows

AI Playground 공통 GitHub Actions 워크플로우

> 이 레포는 **Public**입니다. (GitHub Free 플랜에서 Reusable Workflow를 호출하려면 Public이어야 함)

## 사용 방법

### 1. 앱 레포에 CI/CD 추가

`templates/app-ci-cd.yml` 파일을 앱 레포의 `.github/workflows/ci-cd.yml`에 복사하세요.

### 2. 수정할 부분

```yaml
# app-name: ECR 리포지토리 이름이 됩니다
# developer: k8s/apps/{developer} 디렉토리명

deploy:
  uses: jb-wooricapital/shared-workflows/.github/workflows/build-deploy.yml@main
  with:
    app-name: my-ai-app        # 앱 이름으로 변경
    developer: developer-a     # 본인 이름으로 변경
  secrets: inherit
```

### 3. 사전 요구사항

- Organization Secret: `GITOPS_PAT` (playground-deploy 레포 쓰기 권한)
- 앱 레포에 `Dockerfile` 존재

## 흐름

```
앱 레포 PR merge → test → Docker build → ECR push → playground-deploy values.yaml 태그 업데이트 → ArgoCD 자동 배포
```

## 관련 레포

| 레포 | 설명 |
|------|------|
| [playground-infra](https://github.com/jb-wooricapital/playground-infra) | Terraform 인프라 (관리자 전용) |
| [playground-deploy](https://github.com/jb-wooricapital/playground-deploy) | K8s Helm values + ArgoCD 설정 |
