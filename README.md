# Shared Workflows

AI Playground 사내 dev 가 본인 앱 repo 에서 사용하는 **GitHub Actions 워크플로 템플릿 + 컨벤션 + 자동 셋업 skill** 모음.

> 이 레포는 **Private** 입니다. dev 가 skill 또는 수동 복사로 자기 앱 repo 에 inline 으로 가져가서 사용 — reusable workflow 호출 (`uses:`) 패턴 안 씀. 그래서 cross-repo 호출 권한이 필요 없고 사내 정보 (AWS account ID 등) 가 외부 노출 안 됨.
>
> Dev-facing 이 모두 들어있어요 — 사내 dev 는 이 한 레포만 보면 됩니다. 운영 인프라 (Terraform, EKS 설정 등) 는 [playground-infra](https://github.com/jb-wooricapital/playground-infra) 에 있고 그건 platform 팀 전용.

## 무엇이 있나

| 디렉토리 | 무엇 | 어떻게 사용 |
|---|---|---|
| [`templates/`](templates/) | 워크플로 템플릿 (`app-ci-cd.yml`, `safeimport-release.yml`, `safeimport-release-monorepo.yml`) | dev 의 `.github/workflows/` 에 복사 |
| [`docs/`](docs/) | 사내 표준 컨벤션 (`cicd-conventions.md`) | 참고용 — Semgrep 룰셋 / ECR / branch protection |
| [`skills/`](skills/) | Claude Code skills (`safeimport-cicd/`, `helm-values/`) | dev 의 `~/.claude/skills/` 에 복사하면 워크플로 / values.yaml 자동 셋업 |

## 빠르게 시작 (skill 사용 — 권장)

dev PC 에 한 번만 설치:

```bash
git clone https://github.com/jb-wooricapital/shared-workflows.git
cp -r shared-workflows/skills/* ~/.claude/skills/
```

그 다음 본인 앱 repo 안에서 Claude Code 호출:

```
/safeimport-cicd       # CI/CD 워크플로 깔기 (단일/모노레포 자동 감지)
/helm-values           # 배포용 values.yaml 만들기
```

또는 자연어 ("사내 표준 CI/CD 깔아줘", "배포 values 만들어줘"). skill 이 알아서 언어 / Dockerfile / 테스트 명령 감지하고 [docs/cicd-conventions.md](docs/cicd-conventions.md) 의 룰셋 적용.

기존 앱 repo 의 워크플로를 새 버전으로 업그레이드 하려면 같은 명령 다시 호출 — skill 이 diff 보여주고 보강.

## 수동 설치 (skill 없이)

`templates/app-ci-cd.yml` 을 앱 repo 의 `.github/workflows/ci-cd.yml` 로 복사. 상단 env 블록 수정:

```yaml
env:
  APP_NAME: my-ai-app                       # 앱 이름 (ECR repo 이름)
  DOCKERFILE: ./Dockerfile                  # 모노레포면 ./backend/Dockerfile 등
  BUILD_CONTEXT: .                          # 모노레포면 ./backend 등
```

운영 반입 (망연계) 까지 필요한 앱이면 `templates/safeimport-release.yml` 도 `.github/workflows/safeimport-release.yml` 로 복사. 모노레포면 `safeimport-release-monorepo.yml`.

## 두 트랙

dev 가 신경 써야 할 두 가지 흐름:

| 트리거 | 워크플로 | 결과 |
|---|---|---|
| `pull_request: main`<br>`push: main` | `app-ci-cd.yml` | PR check (test + Semgrep) → main 머지 시 ECR push + GitOps values.yaml 갱신 → ArgoCD 자동 배포 (사내 dev/스테이징) |
| `release: published` | `safeimport-release.yml` | 5 자산 (image.tar / image.digest / trivy.json / semgrep.json / sbom.cdx.json) 을 GitHub Release 에 attach → SafeImport 가 fetch → 망연계 → 운영 Harbor |

두 트랙은 **독립** — main 머지 시점과 release 발행 시점이 다른 게 정상.

## 사전 요구사항

### Org-level Variables (platform 팀이 한 번만 설정)

GitHub Org settings → Secrets and variables → Actions → Variables 에서:

```
AWS_REGION       = ap-northeast-2
AWS_ACCOUNT_ID   = <12자리 AWS 계정 ID>
ECR_ROLE         = playground-github-actions-role  (GitHub OIDC trust 한 IAM role 이름)
GITOPS_OWNER     = jb-wooricapital
GITOPS_REPO      = playground-deploy
```

Org-level 로 박으면 사내 모든 dev repo 가 자동 상속 — caller repo 마다 따로 설정 X.

### Org-level Secrets

```
GITOPS_PAT      playground-deploy 쓰기 권한 PAT (values.yaml 자동 갱신용)
```

### 앱 repo 자체

- `Dockerfile` 존재 (단일) 또는 컴포넌트 디렉토리에 (모노레포)
- main branch protection — PR 만 허용, `test` + `semgrep` check 통과 강제 (자세히 [docs/cicd-conventions.md](docs/cicd-conventions.md))

## 흐름

### 사내 환경 (main 머지)
```
앱 repo main push
  → app-ci-cd.yml: test + Semgrep
  → deploy job (inline): AWS OIDC → ECR repo 자동 생성 → Docker build → Trivy 스캔
                          → ECR push → playground-deploy/k8s/apps/<app>/values.yaml 태그 갱신
  → ArgoCD: values.yaml 변경 감지 → EKS 자동 배포
```

### 운영 반입 (release 발행)
```
앱 repo release publish
  → safeimport-release.yml: Docker build → Trivy(vuln+secret+license) + Semgrep + CycloneDX SBOM
  → 5 자산을 release 에 attach
  → DEVELOPER 가 SafeImport 에서 release_tag 입력하여 신청
  → 보안 승인 → 배포 승인 → 망연계 → SafeImport-Agent 가 운영 Harbor 에 push
```

## 관련 레포

| 레포 | 누가 보나 | 무엇 |
|------|---|------|
| [SafeImport](https://github.com/jb-wooricapital/SafeImport) | dev (운영 반입 시) + 보안/배포팀 | 외부 산출물 안전 반입 시스템. 자산 contract 정의 ([docs/release-contract.md](https://github.com/jb-wooricapital/SafeImport/blob/main/docs/release-contract.md)) |
| [playground-deploy](https://github.com/jb-wooricapital/playground-deploy) | dev (values.yaml 작성 시) + platform 팀 | K8s Helm values + ArgoCD 설정. common-app chart |
| [playground-infra](https://github.com/jb-wooricapital/playground-infra) | platform 팀만 | Terraform 인프라, 비용 추정, 아키텍처 docs |
