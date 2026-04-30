# CI/CD 컨벤션

사내 dev 가 앱 레포 셋업 시 따라야 하는 표준. [shared-workflows](https://github.com/jb-wooricapital/shared-workflows) 의 두 템플릿 (`app-ci-cd.yml`, `safeimport-release.yml`) 을 그대로 쓰면 거의 다 자동.

## 두 트랙 요약

| 트리거 | 워크플로 | 결과 |
|---|---|---|
| PR open | `app-ci-cd.yml` 의 `test` + `semgrep` | PR check (빌드/테스트/Semgrep finding) |
| `push: main` | `app-ci-cd.yml` → `build-deploy.yml` | Docker build + Trivy + ECR push + GitOps values.yaml 태그 갱신 → ArgoCD 자동 배포 |
| release publish | `safeimport-release.yml` | image.tar / image.digest / trivy.json / semgrep.json / sbom.cdx.json 5개 자산 → SafeImport 가 fetch → 망연계 → 운영 Harbor |

두 트랙은 독립. main push 가 매번 release 만들지는 않음 — release 는 운영 반입 시점에 dev/팀장이 태깅.

## 자산 이름 (★ 강제 — SafeImport contract)

GitHub Release 에 첨부하는 자산 이름은 **반드시** 다음 5개. 이름 / 포맷이 어긋나면 SafeImport-Internet 의 verification 단계에서 자동 거부.

| 자산 | 내용 | 생성 명령 (ref) |
|---|---|---|
| `image.tar` | docker save 결과 | `docker save IMAGE -o image.tar` |
| `image.digest` | config digest 한 줄 (sha256:...) | `docker inspect IMAGE --format '{{.Id}}' > image.digest` |
| `trivy.json` | Trivy image scan, vuln+secret+license | `trivy image --format json --scanners vuln,secret,license` |
| `semgrep.json` | Semgrep SAST | `semgrep --config auto --json --output semgrep.json .` |
| `sbom.cdx.json` | CycloneDX SBOM | `trivy image --format cyclonedx` |

자세한 contract 정의는 [SafeImport/docs/release-contract.md](https://github.com/jb-wooricapital/SafeImport/blob/main/docs/release-contract.md).

## Semgrep 룰셋 사내 표준

`app-ci-cd.yml` 의 `semgrep` job 이 디폴트로 다음 3개 stack:
- `p/security-audit`
- `p/owasp-top-ten`
- `p/secrets`

언어/프레임워크별로 다음을 추가해 워크플로 수정:

| 언어 / 프레임워크 | 룰셋 | 비고 |
|---|---|---|
| Python | `p/python` | + 프레임워크 감지 시 `p/django`, `p/flask` |
| JavaScript / TypeScript | `p/javascript`, `p/typescript` | + `p/react` (React), `p/nodejs` (서버) |
| Go | `p/golang`, `p/gosec` | |
| Java | `p/java`, `p/spring` | Spring 감지 시 |
| Kotlin | `p/kotlin` | |
| Rust | `p/rust` | |
| Ruby | `p/ruby`, `p/rails` | Rails 감지 시 |
| C# / .NET | `p/csharp` | |
| Dockerfile | `p/dockerfile` | container 빌드 있는 모든 repo 에 자동 추가 권장 |
| Terraform / IaC | `p/terraform` | infra repo 만 |

여러 `--config` stack:

```yaml
semgrep \
  --config p/security-audit --config p/owasp-top-ten --config p/secrets \
  --config p/python --config p/django \
  --config p/dockerfile \
  --error
```

repo 의 `.semgrep.yml` 또는 `.semgrep/` 디렉토리도 자동 stack 가능:

```yaml
semgrep --config p/security-audit --config p/python --config .semgrep.yml --error
```

### PR check 정책 강도

| 상황 | 권장 |
|---|---|
| 보안 강도 우선 (디폴트) | `--error` (finding 1개라도 fail) |
| Legacy 코드베이스 finding 많음 | `--severity ERROR --error` (ERROR 만 fail, WARNING 통과) |
| 리포트만 | `--error` 빼면 exit 0, PR 안 깨짐 |

## ECR / 이미지 네이밍

| 컨벤션 | 예시 |
|---|---|
| ECR 레포 이름 | GitHub repo 이름과 동일 (예: `payment-service`) |
| 모노레포 | `{repo}-{component}` (예: `payment-service-frontend`) |
| 태그 (main 머지) | `{timestamp}-{shortsha}` + `latest` (build-deploy.yml 자동) |
| 태그 (운영 반입) | release tag 그대로 (예: `v1.2.3`) — SafeImport 가 그대로 Harbor 에 push |

## Branch protection 권장

- `main` 은 **PR 만 허용** (직접 push 금지)
- PR check (`test`, `semgrep`) 통과 강제
- Review 1+ approval 강제
- Force push 금지

## 앱 레포 네이밍 규칙

[playground-deploy README](https://github.com/jb-wooricapital/playground-deploy) 와 동일:
- 소문자 + 하이픈만
- 영문만
- 형식: `{팀 or 프로젝트}-{서비스 역할}`
- **1 레포 = 1 Dockerfile = 1 앱**

레포 이름 = ECR 레포명 = K8s 리소스명 = `playground-deploy/k8s/apps/<this>/values.yaml` 디렉토리명. 일관성이 자동화의 전제.

## Helm values.yaml 컨벤션

- 위치: `playground-deploy/k8s/apps/<app-name>/values.yaml`
- chart: `playground-deploy/k8s/charts/common-app/` (현재 v0.2.0)
- `app.image` / `app.tag` 는 build-deploy.yml 이 자동으로 채움 — dev 가 작성하지 않음
- 새 values.yaml 만들 때는 [common-app README](https://github.com/jb-wooricapital/playground-deploy/blob/main/README.md#step-3-valuesyaml-작성) 참조. `helm-values` skill 은 별도 (TBD)

## 관련 자료

- [.github/workflows/build-deploy.yml](../.github/workflows/build-deploy.yml) — main 머지 시 Docker 빌드 + Trivy + ECR push + GitOps 자동 갱신 (dev 가 `uses:` 로 호출)
- [templates/](../templates/) — dev 가 본인 repo `.github/workflows/` 에 복사하는 템플릿 (app-ci-cd / safeimport-release)
- [skills/safeimport-cicd/](../skills/safeimport-cicd/) — Claude Code skill, 위 템플릿들을 자동 깔아주는 dev tool
- [playground-deploy](https://github.com/jb-wooricapital/playground-deploy) — common-app chart, ArgoCD ApplicationSet
- [SafeImport](https://github.com/jb-wooricapital/SafeImport) — 외부 반입 시스템, 자산 contract 정의
