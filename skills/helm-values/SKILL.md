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

### 3) values.yaml 메모리에 만들기 (★ 아직 디스크 안 씀)

답변 받은 값 + chart 디폴트 합쳐서 values.yaml 컨텐츠를 **변수에 보관**. 이 시점에 파일 쓰지 말 것 — 검증 + 사용자 컨펌 통과한 뒤에만 디스크에 씀.

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
> `probes`, `securityContext`, `imagePullSecrets`, `podDisruptionBudget` 도 chart 디폴트 그대로. dev 가 명시 override 안 했으면 values.yaml 에 적지 말 것 (디폴트가 적용되니 더 깔끔).

### 4) Dry-run + 사용자 컨펌 (★ DX 핵심)

차트의 로컬 clone 이 있으면 (`workspaces/jb-wooricapital/playground-deploy/k8s/charts/common-app/`) 거기 두 명령 실행:

```bash
# (a) lint — values.yaml schema / required-field 검증
helm lint <chart-path> -f - <<<"<메모리의 values.yaml>"

# (b) template — 실제 K8s manifest 가 어떻게 그려지는지
helm template <app-name> <chart-path> -f - <<<"<메모리의 values.yaml>"
```

**(a) 가 fail 하거나 (b) 가 schema error 면**:
- 어느 필드가 잘못됐는지 (`error: ingress.host is required when ingress.enabled=true` 같은) 사용자에게 보여주고 그 필드만 다시 묻기
- 답변 받으면 step 3) 으로 돌아가 메모리 값 갱신 후 다시 dry-run

**둘 다 통과하면**, 사용자에게 다음 박스 출력하고 yes / no / edit 답변 받기:

```
┌─────────────────────────────────────────────────────────┐
│ 작성할 파일:                                              │
│   playground-deploy/k8s/apps/<app-name>/values.yaml      │
│                                                         │
│ ────── values.yaml 핵심 ──────                            │
│ app:                                                    │
│   name: <name>                                          │
│   port: <port>                                          │
│   replicas: <n>                                         │
│   resources: requests cpu=X mem=Y, limits cpu=A mem=B   │
│   env: <key list 또는 (없음)>                             │
│   secretName: <name 또는 (없음)>                          │
│ ingress:                                                │
│   enabled: <bool> · host: <host>                        │
│ hpa: <enabled / off>                                    │
│ volumes: <count 또는 (없음)>                              │
│                                                         │
│ ────── 렌더된 K8s manifest 요약 ──────                    │
│ Deployment "<name>" — <n> replicas                       │
│   container "<name>": <image>:<tag>, port <port>         │
│   probes: liveness path=<path>, readiness path=<path>    │
│   resources: 보존                                         │
│ Service "<name>" — ClusterIP, port <port>                │
│ Ingress "<name>" — host <host>  (ingress.enabled=true 때) │
│ HorizontalPodAutoscaler "<name>" — ...  (hpa.enabled 때)  │
│ PersistentVolumeClaim ... — ...  (volumes 있을 때)        │
│                                                         │
│ ────── 검증 결과 ──────                                    │
│ ✓ helm lint 통과                                          │
│ ✓ helm template error 없음                                │
│                                                         │
│ 이대로 진행할까요? (yes / no / edit)                       │
└─────────────────────────────────────────────────────────┘
```

분기:
- **yes** → step 5) 디스크에 쓰기 + (선택) PR
- **edit** → "어디 고치시겠어요?" 묻기. 답변 받아서 step 3) 으로 돌아가 메모리 값 갱신 후 dry-run 다시
- **no** → 작업 중단. 디스크에 아무것도 안 씀

**helm CLI 가 dev PC 에 없을 경우**: skill 이 helm 설치 안내하거나 (`brew install helm`), 또는 schema-only fallback (chart 의 default values.yaml 과 비교해 "있어야 할 필드 누락 / 알 수 없는 필드" 만 검증). dry-run 자체는 못 보여주지만 lint 비슷한 효과.

**모노레포 케이스**: 컴포넌트당 위 4) 사이클 한 번씩. 각 컴포넌트마다 박스 따로 보여주고 따로 컨펌. 모두 통과하면 N 파일을 한 commit 으로 묶어서 5) 진행.

### 5) 디스크에 쓰기 + PR

컨펌 받은 후 파일 작성:

- **playground-deploy clone 이 로컬에 있으면** (`workspaces/jb-wooricapital/playground-deploy/`) 거기 직접 — `k8s/apps/<app-name>/values.yaml`
- **로컬 clone 없으면** dev 의 현재 repo 안 임시 경로 (예: `deploy/values.yaml`) 에 쓰고 stdout 으로 위치 알려줌. dev 가 알아서 playground-deploy 로 옮김

`gh` CLI 인증 돼있고 사용자가 yes 하면 PR 자동 생성:
```
gh repo fork --clone jb-wooricapital/playground-deploy   # 처음 한 번만
git checkout -b add-<app-name>
mkdir -p k8s/apps/<app-name>
cp <generated values.yaml> k8s/apps/<app-name>/values.yaml
git add . && git commit -m "feat: add <app-name> deployment"
git push -u origin add-<app-name>
gh pr create --base main
```

### 6) 결과 보고

- 작성한 파일 경로 (단일 1개, 모노레포 N개)
- 만들어진 PR URL (`gh` 사용했으면)
- 다음 단계 안내:
  1. PR platform 팀 리뷰 (lint / template 이미 통과한 상태라 빠르게)
  2. 머지 → ArgoCD ApplicationSet 이 1~2분 내 sync → EKS 에 파드 뜸
  3. 확인: `kubectl get pods -n apps -l app=<app-name>`
  4. 이후 dev 의 앱 repo main 머지마다 build-deploy.yml 이 image tag 자동 갱신 → ArgoCD 가 자동 재배포 (수동 작업 0)

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
