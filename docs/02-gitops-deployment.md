# 02. GitOps Deployment

Argo CD와 `server-inventory-k8s` Repository를 연결하여 Git을 Kubernetes의 Desired State로 사용하는 GitOps 배포 환경을 구성합니다.

전체 구조는 다음과 같습니다.

```text
server-inventory-k8s
        │
        │ Git Repository
        ▼
     Argo CD
        │
        │ Compare / Sync
        ▼
 Kubernetes Cluster
```

---

## 1. GitOps Repository

애플리케이션 Kubernetes Manifest는 별도의 Repository에서 관리합니다.

```text
server-inventory-k8s
```

애플리케이션 Source Repository와 Kubernetes Manifest Repository를 분리했습니다.

```text
server-inventory-api
        │
        └── Application Source

server-inventory-k8s
        │
        └── Kubernetes Desired State
```

Argo CD는 `server-inventory-api`가 아니라 `server-inventory-k8s`를 감시합니다.

---

## 2. Kustomize Configuration

`server-inventory-k8s` Repository Root에 `kustomization.yaml`을 추가하여 Argo CD가 관리할 Resource를 명시했습니다.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - namespace/namespace.yaml

  - api/deployment.yaml
  - api/service.yaml

  - postgres/configmap.yaml
  - postgres/service.yaml
  - postgres/statefulset.yaml

  - gateway/gateway.yaml
  - gateway/httproute.yaml
```

Argo CD Application의 `path`는 Repository Root인 `.`을 사용합니다.

Repository Root에 `kustomization.yaml`이 존재하기 때문에 Argo CD는 Kustomize를 사용하여 Desired Manifest를 생성합니다.

```text
server-inventory-k8s
        │
        ├── kustomization.yaml
        │
        ▼
     Kustomize
        │
        ▼
Desired Kubernetes Resources
```

---

## 3. GitOps Management Scope

모든 Kubernetes Resource를 무조건 Argo CD에서 관리하도록 구성하지 않았습니다.

현재 GitOps 관리 대상은 다음과 같습니다.

```text
Namespace
API Deployment
API Service
PostgreSQL ConfigMap
PostgreSQL Service
PostgreSQL StatefulSet
Gateway
HTTPRoute
```

### PostgreSQL Secret

실제 DB Password가 포함된 Secret은 Git Repository에 저장하지 않습니다.

Repository에는 Example Manifest만 유지합니다.

```text
postgres/secret.example.yaml
```

실제 Secret은 Cluster에 별도로 생성되어 있으며 Argo CD 관리 대상에서 제외했습니다.

### PostgreSQL PV

PostgreSQL PV Manifest에는 Lab 환경의 NFS Server 및 Export Path와 같은 환경 종속 정보가 필요합니다.

따라서 PV도 현재 Argo CD 관리 대상에서 제외했습니다.

```text
postgres/postgres-pv.yaml
```

즉 현재 구성은 다음과 같습니다.

```text
GitOps Managed
├── Application Resources
├── PostgreSQL ConfigMap
├── PostgreSQL StatefulSet / Service
└── Gateway Resources

GitOps Unmanaged
├── Actual DB Secret
└── Environment-specific PV
```

---

## 4. Create Argo CD Application

`kubernetes-cicd-lab` Repository에 다음 Application Manifest를 작성했습니다.

```text
argocd/applications/server-inventory.yaml
```

Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: server-inventory
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/mattang-e/server-inventory-k8s.git
    targetRevision: main
    path: .

  destination:
    server: https://kubernetes.default.svc
    namespace: server-inventory

  syncPolicy:
    automated:
      selfHeal: true
```

Application Resource는 `argocd` Namespace에 생성합니다.

```text
Application
name: server-inventory
namespace: argocd
```

하지만 실제 Application Resource가 배포하는 대상 Namespace는:

```text
server-inventory
```

입니다.

---

## 5. Application Source

Application의 Source 설정:

```yaml
source:
  repoURL: https://github.com/mattang-e/server-inventory-k8s.git
  targetRevision: main
  path: .
```

각 항목의 의미는 다음과 같습니다.

```text
repoURL
→ GitOps Repository

targetRevision
→ 사용할 Branch

path
→ Manifest를 읽을 Repository 내부 경로
```

현재는 Repository Root에 `kustomization.yaml`이 있으므로:

```yaml
path: .
```

을 사용합니다.

---

## 6. Application Destination

```yaml
destination:
  server: https://kubernetes.default.svc
  namespace: server-inventory
```

`https://kubernetes.default.svc`는 Argo CD가 설치된 현재 Kubernetes Cluster의 API Server를 의미합니다.

따라서 구조는 다음과 같습니다.

```text
Argo CD
   │
   ├── Source
   │     └── GitHub / server-inventory-k8s
   │
   └── Destination
         └── Current Kubernetes Cluster
                 │
                 └── server-inventory namespace
```

---

## 7. Apply Application

Application Manifest를 Kubernetes에 적용합니다.

Control Plane에서:

```bash
kubectl apply -f server-inventory.yaml
```

Application 확인:

```bash
kubectl get application -n argocd
```

초기에는 Git과 Cluster의 상태가 다르기 때문에 다음과 같이 표시될 수 있습니다.

```text
NAME               SYNC STATUS   HEALTH STATUS
server-inventory   OutOfSync     Healthy
```

---

## 8. First Manual Sync

초기 구성에서는 Auto Sync를 바로 활성화하지 않고 Argo CD UI에서 Diff를 확인한 후 Manual Sync를 수행했습니다.

```text
Git Desired State
        │
        │ Compare
        ▼
Kubernetes Live State
        │
        ▼
      Diff
        │
        ▼
  Manual Sync
```

첫 Sync 후:

```text
SYNC STATUS     Synced
HEALTH STATUS   Healthy
```

상태를 확인했습니다.

이 과정을 통해 Argo CD가 Git Repository의 Manifest를 정상적으로 읽고 Kubernetes Resource와 비교할 수 있는지 먼저 검증했습니다.

---

## 9. Desired State and Live State

Argo CD는 Git의 Manifest를 Desired State로 사용합니다.

```text
Git
Desired State
      │
      │ Compare
      ▼
Kubernetes
Live State
```

두 상태가 같으면:

```text
Synced
```

다르면:

```text
OutOfSync
```

상태가 됩니다.

Sync의 기준은 항상 Git입니다.

```text
Git → Kubernetes
```

Kubernetes의 상태를 Git으로 가져오는 방식이 아닙니다.

---

## 10. Auto Sync

Manual Sync 동작을 검증한 후 Auto Sync를 활성화했습니다.

```yaml
syncPolicy:
  automated:
    selfHeal: true
```

이후 GitOps Repository가 변경되고 Argo CD가 새로운 Revision을 감지하면 자동으로 Kubernetes Cluster에 반영합니다.

```text
Git Push
   ↓
Argo CD Repository Refresh
   ↓
New Revision
   ↓
OutOfSync
   ↓
Auto Sync
   ↓
Synced
```

---

## 11. selfHeal

`selfHeal`은 Kubernetes의 Live Resource가 Git의 Desired State와 다르게 변경된 경우 다시 Git 상태로 복구하기 위해 사용합니다.

예:

```text
Git

replicas: 3

       ↓

kubectl edit

       ↓

Kubernetes

replicas: 1

       ↓

Argo CD detects drift

       ↓

selfHeal

       ↓

replicas: 3
```

따라서 Git Repository가 Desired State의 기준이 됩니다.

---

## 12. Prune Policy

현재 Application에서는 `prune`을 활성화하지 않았습니다.

현재 설정:

```yaml
syncPolicy:
  automated:
    selfHeal: true
```

다음 설정은 아직 적용하지 않았습니다.

```yaml
prune: true
```

`prune`을 활성화하면 Git에서 제거된 Resource를 Kubernetes에서도 자동으로 삭제할 수 있습니다.

Lab에서는 자동 삭제 범위를 명확하게 확인한 후 적용할 수 있도록 현재는 비활성 상태로 유지했습니다.

---

## 13. Repository Refresh

Auto Sync가 활성화되어 있다고 해서 Argo CD가 GitHub의 변경을 매 순간 실시간으로 전달받는 것은 아닙니다.

현재 Lab의 Argo CD는 외부 GitHub Webhook을 사용하지 않습니다.

```text
GitHub
   │
   │ Repository Refresh
   ▼
Argo CD
   │
   │ New Revision 발견
   ▼
Auto Sync
```

따라서 Git Push와 실제 Sync 사이에 약간의 지연이 발생할 수 있습니다.

Argo CD UI에서 `Refresh`를 수행하면 Repository 상태를 즉시 다시 확인할 수 있습니다.

현재 Argo CD 역시 Private Network에 있기 때문에 GitHub.com에서 내부 IP로 직접 Webhook을 전달하는 구조는 사용하지 않았습니다.

---

# Argo CD Gateway Access

초기에는 Port Forward와 SSH Tunnel을 사용했지만 최종적으로 Envoy Gateway와 MetalLB를 통해 Argo CD Web UI에 접근하도록 구성했습니다.

```text
Client
   │
   ▼
192.168.10.117
   │
   ▼
MetalLB
   │
   ▼
Envoy Gateway
   │
   ▼
HTTPRoute
   │
   ▼
argocd-server
```

---

## 14. Configure Argo CD HTTP Mode

Envoy Gateway에서 HTTP로 Argo CD Server에 요청을 전달하기 위해 Argo CD Server의 insecure mode를 활성화했습니다.

```bash
kubectl patch configmap argocd-cmd-params-cm \
  -n argocd \
  --type merge \
  -p '{"data":{"server.insecure":"true"}}'
```

설정을 반영하기 위해 Deployment를 Restart합니다.

```bash
kubectl rollout restart deployment argocd-server -n argocd
```

상태 확인:

```bash
kubectl rollout status deployment/argocd-server -n argocd
```

이 설정 이후 Gateway에서 `argocd-server`의 HTTP Port로 요청을 전달할 수 있습니다.

---

## 15. Argo CD Gateway

다음 위치에 Gateway Manifest를 관리합니다.

```text
argocd/gateway/gateway.yaml
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: argocd-gateway
  namespace: argocd
spec:
  gatewayClassName: envoy

  listeners:
    - name: http
      protocol: HTTP
      port: 80

      allowedRoutes:
        namespaces:
          from: Same
```

기존 Kubernetes Cluster에 설치된 Envoy Gateway의 `GatewayClass`를 사용합니다.

```text
gatewayClassName: envoy
```

---

## 16. HTTPRoute

다음 위치에 HTTPRoute Manifest를 관리합니다.

```text
argocd/gateway/httproute.yaml
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: argocd
  namespace: argocd
spec:
  parentRefs:
    - name: argocd-gateway

  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /

      backendRefs:
        - name: argocd-server
          port: 80
```

모든 `/` 하위 요청을 `argocd-server` Service로 전달합니다.

```text
/
├── /applications
├── /api
└── ...

        ↓

argocd-server:80
```

---

## 17. MetalLB Address

Gateway 생성 후 Envoy Gateway가 LoadBalancer Service를 생성하고 MetalLB가 External IP를 할당합니다.

확인:

```bash
kubectl get gateway -n argocd
```

구성된 Argo CD Gateway 주소:

```text
192.168.10.117
```

최종 접근:

```text
http://192.168.10.117
```

따라서 초기 접근 방식이었던:

```text
kubectl port-forward
+
SSH Tunnel
```

은 더 이상 일반적인 Argo CD 접근에 필요하지 않습니다.

---

## 18. Final GitOps Architecture

최종 Argo CD 구성은 다음과 같습니다.

```text
GitHub
server-inventory-k8s
        │
        ▼
  Argo CD repo-server
        │
        ▼
     Kustomize
        │
        ▼
   Desired State
        │
        ▼
Application Controller
        │
        ├── Compare
        ├── Auto Sync
        └── selfHeal
        │
        ▼
Kubernetes Cluster
```

Argo CD Web UI 접근:

```text
Client
   ↓
192.168.10.117
   ↓
MetalLB
   ↓
Envoy Gateway
   ↓
HTTPRoute
   ↓
argocd-server
```

---

## Verification

Application 상태:

```bash
kubectl get application server-inventory -n argocd
```

예상 정상 상태:

```text
SYNC STATUS     Synced
HEALTH STATUS   Healthy
```

Gateway:

```bash
kubectl get gateway -n argocd
```

Application에서 사용 중인 Revision 확인:

```bash
kubectl get application server-inventory \
  -n argocd \
  -o jsonpath='{.status.sync.status}{"\n"}{.status.sync.revision}{"\n"}'
```

---

## Next

Argo CD를 통한 GitOps 배포 구성이 완료되었습니다.

다음 단계에서는 별도의 VM에 Jenkins를 설치하고 CI 환경을 구성합니다.

```text
GitHub
   ↓
Jenkins
   ↓
Podman
   ↓
Harbor
```

다음 문서:

[03. Jenkins Installation](03-jenkins-installation.md)