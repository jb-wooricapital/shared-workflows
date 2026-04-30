---
name: safeimport-cicd
description: Set up GitHub Actions CI/CD for an internal app repo so it integrates with the playground platform. Wires up app-ci-cd.yml (PR check + Semgrep + main-merge build/scan/ECR push via shared-workflows) and optionally safeimport-release.yml (release-publish 5-asset attach for SafeImport intake into the closed network). Handles language detection, Semgrep ruleset stacking, and monorepo confirmation.
---

# safeimport-cicd

You are setting up the standard playground CI/CD for the current repo. Two trigger tracks; the dev gets two workflow files dropped into `.github/workflows/`.

## What you produce

| 워크플로 | 트리거 | 파일 |
|---|---|---|
| **PR + main 머지** | `pull_request` (PR check) + `push: main` (ECR + GitOps 자동 배포) | `app-ci-cd.yml` (shared-workflows 의 build-deploy reusable 호출) |
| **운영 반입 (선택)** | `release: published` | `safeimport-release.yml` — SafeImport 가 5개 자산 fetch 해서 망연계 → 운영 Harbor |

운영 반입 안 할 거면 두 번째 워크플로는 안 깔아도 됨. SafeImport 통과 필요한 앱 (운영 단말망에 배포될 거) 만 같이 깐다.

## 절차

### 1) 프로젝트 구조 파악

먼저 다음을 자동 감지:

1. **언어/프레임워크** — `package.json` (Node), `pyproject.toml` / `requirements.txt` (Python), `go.mod` (Go), `pom.xml` / `build.gradle` (Java), `Cargo.toml` (Rust). 두 개 이상이면 모노레포 가능성.
2. **Dockerfile 위치** — `find . -name Dockerfile -not -path "*/node_modules/*"`. 1개면 단일, 2개+ 면 모노레포.
3. **테스트 명령** — 언어별 컨벤션. Makefile / package.json scripts / pyproject.toml `[tool.pytest...]` 우선.
4. **앱 이름** — git remote 의 repo 이름 (사내 컨벤션: 소문자+하이픈, 영문만, `{팀}-{역할}` 형식)

### 2) Semgrep 룰셋 결정

[../../docs/cicd-conventions.md](../../docs/cicd-conventions.md) 의 "Semgrep 룰셋 사내 표준" 표 따라 stack:

- 공통 (항상): `p/security-audit`, `p/owasp-top-ten`, `p/secrets`
- 언어별: 감지 결과에 매칭 (Python→`p/python`, Node→`p/javascript`+`p/typescript`, Go→`p/golang`+`p/gosec`, ...)
- Dockerfile 있으면: `p/dockerfile` 추가
- 사내 커스텀: repo 의 `.semgrep.yml` 또는 `.semgrep/` 존재 시 같이 stack
- 매핑 안 되는 언어 (Lua, Crystal 등): fallback `--config auto`

### 3) 사용자에게 한 묶음 질문

대화 첫 답변에서 묶어서 질문:
- **모노레포 컨펌** — 발견한 Dockerfile 위치 나열, 컴포넌트별 빌드/푸시 매트릭스로 갈지. (현재 build-deploy.yml 은 단일 컴포넌트 가정. 모노레포면 dev 가 deploy job 을 컴포넌트당 하나씩 분기 — 이미 app-ci-cd.yml 의 주석에 가이드 있음)
- **앱 이름 컨펌** — 자동 감지한 repo 이름 보여주고 ECR repo 이름으로 사용할지
- **테스트 명령** — 자동 감지한 거 보여주고 맞는지
- **운영 반입 필요?** — yes 면 safeimport-release.yml 도 같이 깔기

기본값은 모두 추정하고 진행.

### 4) 템플릿 가져오기

shared-workflows 의 두 템플릿이 canonical:
- `https://github.com/jb-wooricapital/shared-workflows/blob/main/templates/app-ci-cd.yml`
- `https://github.com/jb-wooricapital/shared-workflows/blob/main/templates/safeimport-release.yml`

또는 로컬 clone 이 있으면 거기서 직접 (`workspaces/jb-wooricapital/shared-workflows/templates/`).

가져온 후 다음 치환:
- `app-name: my-ai-app` → 감지/확정한 앱 이름
- semgrep job 의 `--config` 들 → 위 2) 에서 결정한 stack 으로 추가
- 테스트 job 의 `echo "테스트를 여기에 추가하세요"` → 감지한 명령

### 5) 파일 쓰기

`.github/workflows/` 디렉토리 만들고 두 파일 (또는 사용자가 release 안 깐다고 한 경우 하나) 작성. 이미 같은 이름 파일 있으면 **사용자에게 컨펌 받고** 덮어쓰기.

### 6) 결과 보고 + 다음 단계

- 깔아준 파일 경로 요약
- 사용자 follow-up 작업:
  - **GitHub repo Settings → Branch protection** — main 으로의 PR 만 허용, `test` + `semgrep` check 통과 강제
  - **GitHub Org Secrets** 확인 — `GITOPS_PAT` (build-deploy.yml 이 playground-deploy 에 PR 만들 때 사용). 없으면 platform 팀에 요청
  - **Helm values.yaml** — 운영 / 사내 환경 배포 가려면 `playground-deploy/k8s/apps/<app-name>/values.yaml` 도 만들어야 함 — `helm-values` skill 사용 (별도 skill, 호출 필요)
  - **첫 release 발행** (운영 반입 깐 경우만) — `v0.1.0` 태그 → release publish → SafeImport 가 자동 감지

## 모노레포 처리

`app-ci-cd.yml` 자체가 deploy job 을 모노레포 시 분기하는 패턴을 주석으로 안내함:

```yaml
deploy-api:
  needs: [test, semgrep]
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  uses: jb-wooricapital/shared-workflows/.github/workflows/build-deploy.yml@main
  with:
    app-name: myproject-api
    dockerfile: ./backend/Dockerfile
    context: ./backend
  secrets: inherit

deploy-web:
  needs: [test, semgrep]
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  uses: jb-wooricapital/shared-workflows/.github/workflows/build-deploy.yml@main
  with:
    app-name: myproject-web
    dockerfile: ./frontend/Dockerfile
    context: ./frontend
  secrets: inherit
```

skill 이 모노레포 감지 시 단일 `deploy:` 를 위처럼 컴포넌트당 분기로 자동 변환. SafeImport 의 release 자산은 phase 2 까지는 단일 컴포넌트만 — 모노레포라도 운영 반입 하나만 우선 cover.

## 핵심 보존 — SafeImport contract

`safeimport-release.yml` 이 release 에 첨부하는 자산은 **반드시** 다음 다섯 개 (이름 그대로):

```
image.tar / image.digest / trivy.json / semgrep.json / sbom.cdx.json
```

자세한 contract 는 [SafeImport/docs/release-contract.md](https://github.com/jb-wooricapital/SafeImport/blob/main/docs/release-contract.md). skill 은 이 이름들을 절대 바꾸면 안 됨.

## 호출 인터페이스

사용자가 `/safeimport-cicd` 또는 자연어 ("playground CI/CD 깔아줘", "SafeImport 호환 CI/CD 셋업해줘") → 이 skill 이 위 절차 수행.

이미 워크플로 있는 repo 면 "현재 워크플로 분석해서 사내 표준 호환 검토" 모드로 — 누락된 것 (Semgrep, safeimport-release 등) 보강 PR 제안.

## 관련 자료

- 사내 CI/CD 컨벤션 전체: [../../docs/cicd-conventions.md](../../docs/cicd-conventions.md)
- 실제 워크플로 파일 (canonical): [shared-workflows](https://github.com/jb-wooricapital/shared-workflows)
- SafeImport 자산 contract: [SafeImport/docs/release-contract.md](https://github.com/jb-wooricapital/SafeImport/blob/main/docs/release-contract.md)
- common-app Helm chart: [playground-deploy/k8s/charts/common-app/](https://github.com/jb-wooricapital/playground-deploy)
