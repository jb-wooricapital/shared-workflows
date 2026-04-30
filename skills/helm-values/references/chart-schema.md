# common-app Chart Schema (v0.2.0)

`playground-deploy/k8s/charts/common-app/values.yaml` 의 입력 스키마. helm-values skill 이 이 표 보고 `values.yaml` 작성.

차트 자체가 source of truth — 이 문서는 skill 편의용 캐시. 차트가 minor bump (v0.3.0 등) 되면 이 문서도 같이 갱신.

## 필드 매핑

| skill 입력 | values.yaml 경로 | 디폴트 / 추정 | 비고 |
|---|---|---|---|
| 앱 이름 | `app.name` | git repo 이름 (모노레포면 `{repo}-{dir}`) | 검증: `^[a-z0-9-]+$`, 영문만 |
| 포트 | `app.port` | Dockerfile EXPOSE 또는 `8080` | |
| replicas | `app.replicas` | `1` | dev/스테이징은 1, 운영은 2+ 권장 |
| 이미지 | `app.image` / `app.tag` | **CI 가 채움** | skill 은 안 건드림. chart 의 default 둠 (`tag: "latest"`) |
| 환경변수 | `app.env` (dict) | `{}` | 평문 OK 한 것만. 시크릿은 secretName 으로 |
| K8s Secret | `app.secretName` | `""` | 미리 K8s 에 생성된 Secret 이름. envFrom 으로 주입됨 |
| resources | `app.resources` | 언어별 추정 | Python: 200m/512Mi req, Java: 500m/1Gi req, Node/Go: 100m/256Mi req |
| probes | `probes.enabled` | `true` (디폴트) | path 디폴트 `/` — chart 가 별도 healthz 엔드포인트 요구 안 함. 앱이 전용 health endpoint 있으면 override |
| securityContext | `securityContext.runAsNonRoot` | `true` | dev 가 root 권한 꼭 필요한 경우 false 로 override (legacy 이미지 등) |
| imagePullSecrets | `imagePullSecrets` | `[]` | ECR OIDC 면 비워둠. cross-account / token 방식이면 secret 이름 |
| Ingress | `ingress.enabled` / `ingress.host` | `enabled: true`, `host: {app-name}.playground.example.com` | dev 가 실제 도메인 알면 override |
| HPA | `hpa.enabled` | `false` | yes 면 minReplicas / maxReplicas / targetCPU 추가 질문 |
| nodeSelector | `nodeSelector` (dict) | `{}` | GPU 노드 등 필요 시: `{gpu: "true"}` 또는 `{kubernetes.io/arch: amd64}` |
| volumes | `volumes` (list) | `[]` | PVC 필요 워커 한정. 각 entry: `{name, mountPath, size, storageClass: gp3, accessMode: ReadWriteOnce}` |
| PDB | `podDisruptionBudget.enabled` | `false` | replicas >= 2 일 때 활성화 권장 |

## 묻지 않고 디폴트로 가는 필드

다음 필드는 **skill 이 사용자에게 안 묻고** chart 의 안전한 디폴트 그대로:
- `securityContext.runAsNonRoot: true`
- `securityContext.runAsUser: 1000`
- `securityContext.allowPrivilegeEscalation: false`
- `securityContext.capabilities.drop: [ALL]`
- `probes.{liveness,readiness}.{initialDelaySeconds, periodSeconds, timeoutSeconds, failureThreshold}` — 운영 일반값
- `app.resources.limits` — requests 의 ~5x

dev 가 의도적으로 override 해야 하는 경우만 chart 의 default 와 다른 값 입력.

## 모노레포 처리

Dockerfile N 개 → 컴포넌트당 위 표를 한 번씩 반복:
- 각 컴포넌트 dir → `{repo}-{dir}` 로 app.name
- 컴포넌트 dir 안의 Dockerfile EXPOSE / package manifest 로 언어별 default 따로 추정
- ingress host 는 컴포넌트별 다르게: backend 가 internal 이면 `enabled: false`, frontend 만 외부 노출 등

각 컴포넌트의 values.yaml 은 별도 파일 (`k8s/apps/{repo}-{dir}/values.yaml`).

## 미지원 / chart 가 못 다루는 케이스

다음 워크로드는 common-app chart 로 안 됨 — platform 팀에 별도 chart 작성 요청 또는 `playground-deploy/k8s/infra/<service>/` 패턴으로:

- 다중 컨테이너 / sidecar (service mesh injection 외)
- StatefulSet (DB, 캐시, 큐)
- DaemonSet
- CronJob
- Custom probe path 외에 다른 종류 (TCP, exec)

skill 이 위 needs 감지하면 (사용자 입력에서) "common-app 로 안 됩니다 — platform 팀에 별도 chart 요청" 안내.
