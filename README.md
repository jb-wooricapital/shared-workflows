# Shared Workflows

AI Playground 공통 GitHub Actions 워크플로우

> 이 레포는 **Public**입니다. (GitHub Free 플랜에서 Reusable Workflow를 호출하려면 Public이어야 함)

## 두 종류의 워크플로

이 레포가 제공하는 두 트랙:

| 트랙 | 트리거 | 목적지 | 언제 |
|---|---|---|---|
| **Build & Deploy** | `push: main` | ECR + GitOps values.yaml 태그 갱신 → ArgoCD 자동 배포 | 사내 dev/스테이징 |
| **SafeImport Release** | `release: published` | image.tar / image.digest / trivy.json / semgrep.json / sbom.cdx.json 5개 자산을 GitHub Release 에 attach → SafeImport 가 fetch → 망연계 → 운영 Harbor | 운영 (중요 단말망) 반입 |

두 트랙은 **독립적으로 동시 운용**됩니다 — main 머지로 사내 환경 배포되고, 운영 반입은 release tag 발행하는 시점에만.

## 사용 방법

### 1. 앱 레포에 CI/CD 추가

`templates/app-ci-cd.yml` 파일을 앱 레포의 `.github/workflows/ci-cd.yml` 에 복사. 이 파일이 PR check (test + Semgrep) + main 머지 시 build-deploy reusable 호출까지 다 처리.

```yaml
# app-name: ECR 리포지토리 이름이 됩니다

deploy:
  uses: jb-wooricapital/shared-workflows/.github/workflows/build-deploy.yml@main
  with:
    app-name: my-ai-app        # 앱 이름으로 변경
  secrets: inherit
```

### 2. SafeImport 호환 release 워크플로 (운영 반입)

`templates/safeimport-release.yml` 을 앱 레포의 `.github/workflows/safeimport-release.yml` 에 복사. `app-ci-cd.yml` 과 별개 파일 — release publish 시 자동 실행되어 SafeImport 가 받아갈 수 있도록 5개 자산 attach.

운영 반입 안 할 거면 안 깔아도 됨 (사내 환경만 쓸 거면 build-deploy 만으로 충분).

### 3. Semgrep 룰셋 사내 표준

`templates/app-ci-cd.yml` 의 `semgrep` job 은 공통 보안 룰만 디폴트로 stack:
- `p/security-audit`, `p/owasp-top-ten`, `p/secrets`

언어/프레임워크별 추가 룰은 사내 컨벤션 ([playground-infra/docs/cicd-conventions.md](https://github.com/jb-wooricapital/playground-infra/blob/main/docs/cicd-conventions.md)) 참조해 dev 가 추가.

### 4. 사전 요구사항

- Organization Secret: `GITOPS_PAT` (playground-deploy 레포 쓰기 권한 — main 머지의 values.yaml 자동 업데이트용)
- 앱 레포에 `Dockerfile` 존재
- main 머지 시 build-deploy 의 Trivy 가 CRITICAL/HIGH 발견하면 ECR push 자체 실패 → 코드 고치고 다시 push

## 흐름

### 사내 환경 (main 머지)
```
앱 레포 main push
  → app-ci-cd.yml: test + Semgrep
  → build-deploy.yml: Docker build → Trivy 스캔 → ECR push → playground-deploy/k8s/apps/<app>/values.yaml 태그 갱신
  → ArgoCD: values.yaml 변경 감지 → EKS 자동 배포
```

### 운영 반입 (release 발행)
```
앱 레포 release publish
  → safeimport-release.yml: Docker build → Trivy(vuln+secret+license) + Semgrep + CycloneDX SBOM
  → 5개 자산을 release 에 attach
  → 사용자 (DEVELOPER) 가 SafeImport 에서 release_tag 입력하여 신청
  → 보안 승인 → 배포 승인 → 망연계 → SafeImport-Agent 가 운영 Harbor 에 push
```

## 관련 레포

| 레포 | 설명 |
|------|------|
| [playground-infra](https://github.com/jb-wooricapital/playground-infra) | Terraform 인프라, dev 가이드, CI/CD 컨벤션 / skill |
| [playground-deploy](https://github.com/jb-wooricapital/playground-deploy) | K8s Helm values + ArgoCD 설정 (common-app chart 포함) |
| [SafeImport](https://github.com/jb-wooricapital/SafeImport) | 외부 산출물 안전 반입 시스템 (망연계). 이 레포의 `safeimport-release.yml` 자산을 받아서 검증 → 운영 Harbor 배포 |
