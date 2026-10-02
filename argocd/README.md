# Argo CD

Argo CD를 사용하여 `server-inventory-k8s` Repository를 Kubernetes Cluster의 Desired State로 관리합니다.

## Architecture

```text
server-inventory-k8s
        │
        │ Git Repository
        ▼
     Argo CD
        │
        ├── Compare
        ├── Auto Sync
        └── selfHeal
        │
        ▼
 Kubernetes Cluster
```

Jenkins는 Kubernetes에 직접 배포하지 않습니다.

Jenkins가 GitOps Repository의 Image Tag를 변경하면 Argo CD가 변경된 Desired State를 Kubernetes Cluster에 반영합니다.

## Directory Structure

```text
argocd/
├── README.md
├── applications/
│   └── server-inventory.yaml
└── gateway/
    ├── gateway.yaml
    └── httproute.yaml
```

## Application

Application Manifest:

```text
applications/server-inventory.yaml
```

Source Repository:

```text
server-inventory-k8s
```

Target:

```text
Kubernetes Cluster
└── server-inventory namespace
```

현재 Sync Policy:

```yaml
syncPolicy:
  automated:
    selfHeal: true
```

Auto Sync를 사용하여 Git 변경 사항을 자동으로 Cluster에 반영하며, `selfHeal`을 통해 Live State가 Git의 Desired State와 달라진 경우 복구하도록 구성했습니다.

현재 `prune`은 활성화하지 않았습니다.

## Gateway

Argo CD Web UI는 Envoy Gateway를 통해 접근합니다.

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

Gateway:

```text
gateway/gateway.yaml
```

HTTPRoute:

```text
gateway/httproute.yaml
```

현재 Lab에서는 HTTP를 사용합니다.

```text
http://192.168.10.117
```

TLS 적용은 향후 개선 항목으로 남겨두었습니다.

## GitOps Management

Argo CD가 관리하는 Application Resource의 실제 Manifest는 `server-inventory-k8s` Repository에서 관리합니다.

실제 DB Secret과 환경 종속적인 PostgreSQL PV는 현재 GitOps 관리 대상에서 제외했습니다.

## Documentation

상세 구축 과정:

- [Argo CD Installation](../docs/01-argocd-installation.md)
- [GitOps Deployment](../docs/02-gitops-deployment.md)
- [CI/CD Integration](../docs/05-cicd-integration.md)