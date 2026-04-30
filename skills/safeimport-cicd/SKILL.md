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

**(A) 단일 컴포넌트 (Dockerfile 1개)**:
- 앱 이름 컨펌 — 자동 감지한 repo 이름 → ECR repo 이름 사용 OK?
- 테스트 명령 — 자동 감지 결과 맞는지
- 운영 반입 필요? — yes 면 `safeimport-release.yml` 도 깔기

**(B) 모노레포 (Dockerfile 2개+)**:
- **컴포넌트 매핑 컨펌** — 발견한 Dockerfile 위치를 표로 보여주고 각 컴포넌트의 ECR repo 이름 (`{repo}-{dir_name}` 디폴트) 사용 OK 인지
- **운영 반입 필요한 컴포넌트** — none / one / all 중:
  - `none`: safeimport-release 워크플로 안 깖
  - `one`: 사용자가 컴포넌트 1개 선택 → 단일 `safeimport-release.yml` 을 그 컴포넌트의 build context 로 작성. **현 단계 권장** (SafeImport-Internet phase 2 전엔 이게 동작)
  - `all`: `safeimport-release-monorepo.yml` 사용. 자산은 release 에 attach 되지만 SafeImport-Internet 이 매트릭스 자산 인식하는 건 phase 2 — 지금 신청은 막힘. dev 가 그 사실 인지하고 미래 대비로 깔겠다 하면 OK
- 각 컴포넌트의 테스트 명령 (디폴트: 공통 테스트, 또는 컴포넌트당 분기 — package.json/pyproject.toml 위치 따라)

기본값은 모두 추정하고 진행.

### 4) 템플릿 가져오기 + 채우기

shared-workflows 의 템플릿 3종이 canonical:
- `templates/app-ci-cd.yml` — PR + main 머지 (모든 case). **inline 버전** — reusable workflow 호출 안 함, AWS/ECR step 다 들어있음
- `templates/safeimport-release.yml` — 운영 반입 (단일 컴포넌트)
- `templates/safeimport-release-monorepo.yml` — 운영 반입 (매트릭스, phase 2 의존)

shared-workflows repo 가 **Private** 이라 dev 가 액세스 권한 있어야 함. 보통:
- 로컬 clone (`workspaces/jb-wooricapital/shared-workflows/`) 에서 직접 읽기 (권장)
- 또는 dev 의 GitHub auth 로 raw URL fetch

**단일 컴포넌트 (A)**: 두 템플릿 (app-ci-cd + safeimport-release) 그대로 가져와서:
- 상단 `env:` 블록의 `APP_NAME` → 감지한 앱 이름 (소문자+하이픈)
- `DOCKERFILE` / `BUILD_CONTEXT` → Dockerfile 위치 (단일이면 그대로)
- semgrep job 의 `--config` 들 → 위 2) 에서 결정한 stack 으로 추가
- 테스트 job 의 `echo "테스트를 여기에 추가하세요"` → 감지한 명령

**모노레포 (B)**: app-ci-cd.yml 을 다음과 같이 변환:
- 단일 `deploy:` job 을 컴포넌트당 복제 (`deploy-{component}:`). 각 job 의 step 은 동일하고 `env:` 의 APP_NAME / DOCKERFILE / BUILD_CONTEXT 만 다름:
  ```yaml
  deploy-backend:
    needs: [test, semgrep]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    env:
      APP_NAME: {repo}-backend
      DOCKERFILE: ./backend/Dockerfile
      BUILD_CONTEXT: ./backend
    steps:
      # ... deploy job 의 모든 step 그대로 (Configure AWS / ECR login / build / Trivy / push / GitOps update) ...
  ```
- 운영 반입 모드:
  - `one` → `safeimport-release.yml` 을 가져와 build context 를 `./{선택한_컴포넌트}` 로 수정
  - `all` → `safeimport-release-monorepo.yml` 을 가져와 matrix entries 를 발견한 컴포넌트로 채움
  - `none` → release 워크플로 안 깖

### 4.5) AWS Variables 안내 (★ 첫 셋업 시)

inline 워크플로가 동작하려면 다음 GitHub **Org-level Variables** 가 필요. dev 의 첫 호출 시 안내:

```
vars.AWS_REGION        예: ap-northeast-2
vars.AWS_ACCOUNT_ID    AWS 계정 12자리
vars.ECR_ROLE          OIDC IAM role (예: playground-github-actions-role)
vars.GITOPS_OWNER      jb-wooricapital
vars.GITOPS_REPO       playground-deploy

secrets.GITOPS_PAT     playground-deploy 쓰기 권한 PAT
```

이게 Org level 에 설정되어 있어야 모든 caller repo 가 자동 상속. **신규 repo 라도 platform 팀이 한 번만 Org Variables 설정해두면 dev 별도 작업 X**. 이미 설정된 사내라면 이 단계 skip — 사용자에게 "기존 사내 Variables 확인됐다고 가정한다" 안내만 하고 진행.

### 5) 파일 쓰기

`.github/workflows/` 디렉토리 만들고 두 파일 (또는 사용자가 release 안 깐다고 한 경우 하나) 작성. 이미 같은 이름 파일 있으면 **사용자에게 컨펌 받고** 덮어쓰기.

### 6) 결과 보고 + 다음 단계

- 깔아준 파일 경로 요약
- 사용자 follow-up 작업:
  - **GitHub repo Settings → Branch protection** — main 으로의 PR 만 허용, `test` + `semgrep` check 통과 강제
  - **GitHub Org Secrets** 확인 — `GITOPS_PAT` (build-deploy.yml 이 playground-deploy 에 PR 만들 때 사용). 없으면 platform 팀에 요청
  - **Helm values.yaml** — 운영 / 사내 환경 배포 가려면 `playground-deploy/k8s/apps/<app-name>/values.yaml` 도 만들어야 함 — `helm-values` skill 사용 (별도 skill, 호출 필요)
  - **첫 release 발행** (운영 반입 깐 경우만) — `v0.1.0` 태그 → release publish → SafeImport 가 자동 감지

## 모노레포 컨벤션 요약

자세한 네이밍 규칙은 [../../docs/cicd-conventions.md](../../docs/cicd-conventions.md) 의 "모노레포 네이밍 규칙" 섹션 참조. 핵심:

```
컴포넌트 이름 = {repo}-{component_dir}

GitHub repo:        jb-wooricapital/payment-service
컴포넌트 dir:        backend, frontend, worker
ECR repo:           payment-service-backend, ...
K8s app:            payment-service-backend, ...
values.yaml:        playground-deploy/k8s/apps/payment-service-backend/values.yaml, ...
```

skill 이 컴포넌트 감지하면 ECR repo 이름은 자동으로 `{repo}-{dir}` 로 추정. 사용자가 다르게 가고 싶으면 컨펌 단계에서 override.

## 핵심 보존 — SafeImport contract

`safeimport-release.yml` 이 release 에 첨부하는 자산은 **반드시** 다음 다섯 개 (이름 그대로):

```
image.tar / image.digest / trivy.json / semgrep.json / sbom.cdx.json
```

자세한 contract 는 [SafeImport/docs/release-contract.md](https://github.com/jb-wooricapital/SafeImport/blob/main/docs/release-contract.md). skill 은 이 이름들을 절대 바꾸면 안 됨.

## 호출 인터페이스

### 신규 셋업 모드 (default)

사용자가 `/safeimport-cicd` 또는 자연어 ("playground CI/CD 깔아줘", "SafeImport 호환 CI/CD 셋업해줘") → 위 절차 1)~6) 수행.

기존 `.github/workflows/` 가 비어있거나 없으면 신규 셋업.

### Update 모드 (`/safeimport-cicd update` 또는 "CI/CD 업그레이드")

기존 워크플로가 이미 있는 repo 에서 호출. 이건 inline 워크플로 패턴의 trade-off 를 보완하는 핵심 기능 — shared-workflows 에서 템플릿이 bump 됐을 때 dev 가 한 줄로 자기 repo 도 따라가게 함.

절차:
1. 현재 `.github/workflows/{ci-cd.yml,safeimport-release.yml}` 를 읽음
2. shared-workflows/templates/ 의 최신 템플릿 가져옴
3. **diff 계산** — 어떤 step 이 추가/변경/삭제됐는지
4. 사용자에게 표로 보여줌:
   ```
   변경사항:
   ✓ Trivy step 의 severity 가 CRITICAL,HIGH 에서 CRITICAL,HIGH,MEDIUM 으로 변경
   ✓ semgrep job 에 p/dockerfile 룰셋 추가
   + (신규) PDB lint step
   ```
5. dev 의 커스터마이징 보존 — env 의 APP_NAME, 테스트 명령, 추가 semgrep config 등은 그대로 유지하고 platform 표준 부분만 갱신
6. 머지 PR 생성 (또는 stdout 으로 diff 만 보여주고 dev 가 직접 적용)

모노레포 케이스도 동일하게 동작 — 컴포넌트당 deploy job 보존하고 step 만 갱신.

기존 워크플로가 너무 오래된 (deprecated reusable workflow `uses: ...build-deploy.yml@main` 호출하던 옛 버전) 케이스: skill 이 그 사실 감지하고 "fully replace" 모드 제안 — 옛 워크플로 백업 후 새 inline 버전으로 교체.

## 관련 자료

- 사내 CI/CD 컨벤션 전체: [../../docs/cicd-conventions.md](../../docs/cicd-conventions.md)
- 실제 워크플로 파일 (canonical): [shared-workflows](https://github.com/jb-wooricapital/shared-workflows)
- SafeImport 자산 contract: [SafeImport/docs/release-contract.md](https://github.com/jb-wooricapital/SafeImport/blob/main/docs/release-contract.md)
- common-app Helm chart: [playground-deploy/k8s/charts/common-app/](https://github.com/jb-wooricapital/playground-deploy)
