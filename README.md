# Shared Workflows

AI Playground 사내 dev 가 본인 앱 repo 에서 사용하는 **GitHub Actions reusable workflow + 템플릿 + 컨벤션 + 자동 셋업 skill** 모음.

> 이 레포는 **Public** 입니다. (GitHub Free 플랜에서 Reusable Workflow 를 호출하려면 Public 이어야 함)
>
> Dev-facing 이 모두 들어있어요 — 사내 dev 는 이 한 레포만 보면 됩니다. 운영 인프라 (Terraform, EKS 설정 등) 는 [playground-infra](https://github.com/jb-wooricapital/playground-infra) 에 있고 그건 platform 팀 전용.

## 무엇이 있나

| 디렉토리 | 무엇 | 어떻게 사용 |
|---|---|---|
| [`.github/workflows/`](.github/workflows/) | Reusable workflow (`build-deploy.yml`) | dev 의 앱 repo 가 `uses:` 로 호출 |
| [`templates/`](templates/) | 워크플로 템플릿 (`app-ci-cd.yml`, `safeimport-release.yml`) | dev 의 `.github/workflows/` 에 복사 |
| [`docs/`](docs/) | 사내 표준 컨벤션 (`cicd-conventions.md`) | 참고용 — Semgrep 룰셋 / ECR / branch protection |
| [`skills/`](skills/) | Claude Code skill (`safeimport-cicd/`) | dev 의 `~/.claude/skills/` 에 복사하면 위 템플릿/컨벤션 자동 셋업 |

## 빠르게 시작 (skill 사용 — 권장)

dev PC 에 한 번만 설치:

```bash
git clone https://github.com/jb-wooricapital/shared-workflows.git
cp -r shared-workflows/skills/safeimport-cicd ~/.claude/skills/
```

그 다음 본인 앱 repo 안에서 Claude Code 호출:

```
/safeimport-cicd
또는 자연어: "사내 표준 CI/CD 깔아줘"
```

skill 이 알아서 언어 / Dockerfile / 테스트 명령 감지하고 [docs/cicd-conventions.md](docs/cicd-conventions.md) 의 룰셋 적용해서 `.github/workflows/` 채워줌.

## 수동 설치 (skill 없이)

`templates/app-ci-cd.yml` 을 앱 repo 의 `.github/workflows/ci-cd.yml` 로 복사. 한 곳만 수정:

```yaml
deploy:
  uses: jb-wooricapital/shared-workflows/.github/workflows/build-deploy.yml@main
  with:
    app-name: my-ai-app        # 앱 이름으로 변경
  secrets: inherit
```

운영 반입 (망연계 통과) 까지 필요한 앱이면 `templates/safeimport-release.yml` 도 같이 `.github/workflows/safeimport-release.yml` 로 복사.

## 두 트랙

dev 가 신경 써야 할 두 가지 흐름:

| 트리거 | 워크플로 | 결과 |
|---|---|---|
| `pull_request: main`<br>`push: main` | `app-ci-cd.yml` | PR check (test + Semgrep) → main 머지 시 ECR push + GitOps values.yaml 갱신 → ArgoCD 자동 배포 (사내 dev/스테이징) |
| `release: published` | `safeimport-release.yml` | 5 자산 (image.tar / image.digest / trivy.json / semgrep.json / sbom.cdx.json) 을 GitHub Release 에 attach → SafeImport 가 fetch → 망연계 → 운영 Harbor |

두 트랙은 **독립** — main 머지 시점과 release 발행 시점이 다른 게 정상.

## 사전 요구사항

- Organization Secret: `GITOPS_PAT` (build-deploy.yml 이 playground-deploy 의 values.yaml 갱신할 때 사용)
- 앱 repo 에 `Dockerfile` 존재
- main branch protection — PR 만 허용, `test` + `semgrep` check 통과 강제 (자세히 [docs/cicd-conventions.md](docs/cicd-conventions.md))

## 흐름

### 사내 환경 (main 머지)
```
앱 repo main push
  → app-ci-cd.yml: test + Semgrep
  → build-deploy.yml: Docker build → Trivy 스캔 → ECR push → playground-deploy/k8s/apps/<app>/values.yaml 태그 갱신
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
