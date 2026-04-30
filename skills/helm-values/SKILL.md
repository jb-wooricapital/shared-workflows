---
name: helm-values
description: Generate values.yaml file(s) for the playground common-app Helm chart. Reads the canonical chart from playground-deploy, introspects the current app repo (Dockerfile / EXPOSE / language), asks the user only for the bits it can't infer (ingress host, env vars, secrets, replicas), and writes ready-to-deploy values.yaml under playground-deploy/k8s/apps/<app>/values.yaml. Handles single-component and monorepo (N values.yaml at once) layouts.
---

# helm-values

Playground 의 `common-app` Helm chart 에 들어가는 `values.yaml` 을 dev 가 손으로 안 채우게 자동 생성. 7개 필드만 자동/질문으로 파악해 chart schema 에 맞춰 작성.

## 차트 위치 (canonical)

`https://github.com/jb-wooricapital/playground-deploy/tree/main/k8s/charts/common-app/`

또는 로컬 clone: `workspaces/jb-wooricapital/playground-deploy/k8s/charts/common-app/`

skill 은 `Chart.yaml` + `values.yaml` (default) + `templates/deployment.yaml` 을 읽어서 schema 파악. 차트 version 은 [references/chart-schema.md](references/chart-schema.md) 에 정리되어 있음 (skill 이 stale 하면 chart 의 values.yaml 이 source of truth).

## 절차

### 1) 현재 앱 repo 컨텍스트 introspect

자동 감지:
- **앱 이름** ← 현재 git repo 이름 (소문자+하이픈, 영문). 모노레포면 컴포넌트당 `{repo}-{component_dir}`
- **포트** ← Dockerfile 의 `EXPOSE` 첫 번째 값. 없으면 8080 디폴트
- **언어** ← `package.json` (Node), `pyproject.toml` (Python), `go.mod` (Go), `pom.xml` (Java) 등 → 언어별 기본 resources 추정:
  - Python: `requests: {cpu: 200m, memory: 512Mi}`, `limits: {cpu: 1000m, memory: 1Gi}`
  - Java: `requests: {cpu: 500m, memory: 1Gi}`, `limits: {cpu: 2000m, memory: 2Gi}`
  - Node: `requests: {cpu: 100m, memory: 256Mi}`, `limits: {cpu: 500m, memory: 512Mi}`
  - Go / Rust: `requests: {cpu: 100m, memory: 128Mi}`, `limits: {cpu: 500m, memory: 256Mi}`
- **모노레포 여부** ← Dockerfile 갯수 (1개 → 단일, 2개+ → 모노레포)

### 2) 사용자에게 한 묶음 질문

**(A) 단일 컴포넌트**:
1. 앱 이름 컨펌 — 자동 감지 OK?
2. 포트 컨펌
3. replicas (default 1)
4. 환경변수 — `key=value` 형태로 한 줄에 하나씩 (빈 답이면 skip):
   ```
   DB_HOST=playground-postgresql.xxx.rds.amazonaws.com
   DB_NAME=my_db
   ```
5. K8s Secret 이름 — 평문 안 되는 시크릿. K8s 에 미리 생성된 Secret 의 이름. (없으면 skip)
6. Ingress host — `{app-name}.playground.example.com` 디폴트 추정
7. HPA 활성화? (default off — 활성화 시 minReplicas/maxReplicas/targetCPU 추가 질문)
8. GPU / 특수 노드 필요? (default no — yes 면 nodeSelector 입력)
9. 영구 볼륨 필요? (default no — yes 면 mountPath/size/storageClass 입력)

**(B) 모노레포**:
- 위 질문을 컴포넌트당 한 번씩 반복
- 컴포넌트별 디폴트 값을 다르게 추정 (backend → port 8080, frontend → port 80 등 흔한 값으로)

### 3) values.yaml 작성

각 컴포넌트당 한 파일. 위치:

```
playground-deploy/k8s/apps/{app-name}/values.yaml
```

playground-deploy clone 이 로컬에 있으면 (`workspaces/jb-wooricapital/playground-deploy/`) 거기 직접 작성. 없으면 사용자에게 어디로 만들지 묻고 stdout 으로 출력 후 dev 가 직접 파일 둘 곳 결정.

생성 예시:

```yaml
app:
  name: payment-service-backend
  port: 8080
  replicas: 2
  env:
    DB_HOST: playground-postgresql.xxx.rds.amazonaws.com
    DB_NAME: payment
  secretName: payment-service-backend-secret
  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi

ingress:
  enabled: true
  host: payment-service-backend.playground.example.com
```

> `app.image` / `app.tag` 는 build-deploy.yml 이 자동으로 채움 — skill 은 안 채움 (chart 의 default `tag: "latest"` 둠).

### 4) 검증

가능하면 `helm template` 로 dry-run:

```bash
cd playground-deploy/k8s/charts/common-app
helm template <app-name> . -f ../../apps/<app-name>/values.yaml
```

출력에 K8s 리소스가 정상 렌더되면 OK. lint 통과 못하면 사용자에게 어떤 필드 잘못 됐는지 보고하고 재질문.

### 5) 결과 보고 + 다음 단계

- 작성한 파일 경로 (단일 1개, 모노레포 N개)
- 다음 단계:
  1. **playground-deploy 에 PR** — `gh repo clone jb-wooricapital/playground-deploy` 로 clone, 새 브랜치 + commit + push + PR. (gh CLI 인증 돼있으면 자동, 아니면 사용자에게 경로 알려주고 dev 가 직접)
  2. 머지 후 ArgoCD ApplicationSet 이 1~2분 내 sync → EKS 에 파드 뜸
  3. `kubectl get pods -n apps -l app=<app-name>` 로 확인

## 모노레포 처리 흐름

Dockerfile 갯수에 따라:

```
Dockerfile 1개:
  → app-name = repo 이름
  → values.yaml 1개 작성 → k8s/apps/<repo>/values.yaml

Dockerfile 2개+ (예: backend/Dockerfile, frontend/Dockerfile):
  → 컴포넌트당 app-name = {repo}-{dir} (예: payment-service-backend)
  → values.yaml N개 작성 → k8s/apps/<repo>-<dir>/values.yaml × N
  → 각 컴포넌트의 PR 은 한 번에 묶어서 (한 commit 에 N파일)
```

`safeimport-cicd` skill 의 모노레포 처리와 **동일한 네이밍 규칙** 사용 — `{repo}-{dir}` 을 K8s app 이름 / values.yaml 디렉토리 이름으로.

## 호출 인터페이스

사용자가 `/helm-values` 또는 자연어 ("배포 values.yaml 만들어줘", "Helm 값 채워줘") → 이 skill 실행.

이미 `k8s/apps/<app-name>/values.yaml` 이 있는 경우는 "현재 파일 분석해서 누락 / 시맨틱 잘못된 필드 검토" 모드 — 보강 PR 제안.

## 차트 변경 추적

playground-deploy 의 common-app chart 가 v0.2.0 → v0.3.0 같이 bump 되면 새 필드 (예: `app.tolerations`, `app.affinity`) 가 추가될 수 있음. 그때 [references/chart-schema.md](references/chart-schema.md) 도 같이 업데이트해야 skill 이 새 필드 인식. chart 만 업데이트하고 skill schema 안 따라가면 dev 가 새 필드 못 쓰게 됨 (질문 안 하니까).

## 관련 자료

- 차트 (canonical): [playground-deploy/k8s/charts/common-app/](https://github.com/jb-wooricapital/playground-deploy/tree/main/k8s/charts/common-app)
- 차트 schema 정리: [references/chart-schema.md](references/chart-schema.md)
- 사내 CI/CD 컨벤션 (네이밍): [../../docs/cicd-conventions.md](../../docs/cicd-conventions.md)
- 짝 skill: [../safeimport-cicd/](../safeimport-cicd/) — CI/CD 워크플로 자동 셋업
