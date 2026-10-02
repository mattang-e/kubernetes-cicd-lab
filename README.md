# Kubernetes CI/CD & GitOps Lab

Jenkins, Harbor, Argo CD를 활용하여 Kubernetes 애플리케이션의 CI/CD 및 GitOps 배포 환경을 구성한 프로젝트입니다.

애플리케이션 소스 코드 변경을 Jenkins가 감지하여 컨테이너 이미지를 빌드하고 Harbor Registry에 Push한 뒤, GitOps Repository의 Kubernetes Manifest를 업데이트합니다.

Argo CD는 GitOps Repository를 기준으로 Kubernetes Cluster의 상태를 동기화합니다.

---

## Architecture

```text
Developer
    │
    │ git push
    ▼
GitHub
server-inventory-api
    │
    │ Poll SCM
    ▼
Jenkins
192.168.10.121
    │
    ├── Checkout
    ├── Detect Changes
    ├── Build Image (Podman)
    ├── Push Image
    │       │
    │       ▼
    │    Harbor
    │    harbor.lab.local
    │    192.168.10.114
    │
    └── Update GitOps Repository
                │
                ▼
        server-inventory-k8s
                │
                │ Repository Refresh
                ▼
             Argo CD
                │
                │ Auto Sync
                ▼
         Kubernetes Cluster
                │
                ▼
       server-inventory-api
```

---

## CI/CD Workflow

### 1. Source Code Push

애플리케이션 소스 코드는 다음 Repository에서 관리합니다.

```text
server-inventory-api
```

개발자가 `main` branch에 변경 사항을 Push합니다.

---

### 2. Jenkins Poll SCM

Jenkins는 Private Lab Network에 위치하기 때문에 GitHub Webhook을 직접 받을 수 없습니다.

따라서 Jenkins의 `Poll SCM`을 사용하여 Git Repository의 변경 여부를 확인합니다.

현재 Poll Schedule:

```text
H/2 * * * *
```

SCM 변경이 발견되면 Jenkins Pipeline이 실행됩니다.

---

### 3. Detect Application Changes

모든 Git 변경에 대해 컨테이너 이미지를 새로 생성할 필요는 없습니다.

예를 들어 다음과 같은 문서 변경은 이미지 Build가 필요하지 않습니다.

```text
README.md
Jenkinsfile
.gitignore
```

현재 Pipeline에서는 다음 파일의 변경 여부를 확인합니다.

```text
main.py
requirements.txt
Dockerfile
```

해당 파일 중 하나라도 변경되면:

```text
APP_CHANGED=true
```

그렇지 않으면:

```text
APP_CHANGED=false
```

로 설정합니다.

`APP_CHANGED=false`인 경우 다음 Stage를 Skip합니다.

```text
Build Image
Push Image
Update GitOps Repository
```

---

### 4. Container Image Build

애플리케이션 변경이 확인되면 Jenkins에서 Podman을 사용하여 컨테이너 이미지를 생성합니다.

현재 이미지 Tag는 Jenkins의 `BUILD_NUMBER`를 사용합니다.

```text
harbor.lab.local/server-inventory/server-inventory-api:<BUILD_NUMBER>
```

예:

```text
harbor.lab.local/server-inventory/server-inventory-api:10
```

---

### 5. Harbor Push

생성된 이미지는 Private Container Registry인 Harbor에 Push합니다.

```text
Harbor

harbor.lab.local
192.168.10.114
```

Harbor 인증 정보는 Jenkins Credentials를 통해 관리합니다.

Jenkins Pipeline에 Harbor 계정의 Password를 직접 저장하지 않습니다.

---

### 6. GitOps Repository Update

이미지 Push가 완료되면 Jenkins가 다음 GitOps Repository를 Clone합니다.

```text
server-inventory-k8s
```

그리고 다음 Manifest의 이미지 Tag를 변경합니다.

```text
api/deployment.yaml
```

예:

```yaml
image: harbor.lab.local/server-inventory/server-inventory-api:10
```

변경된 Manifest를 Jenkins가 Commit하고 GitHub에 Push합니다.

---

### 7. Argo CD Sync

Argo CD Application은 다음 GitOps Repository를 Desired State로 사용합니다.

```text
server-inventory-k8s
```

GitOps Repository의 변경 사항을 Argo CD가 감지하면 Kubernetes Cluster와 비교합니다.

Git과 Cluster 상태가 다르면 `OutOfSync` 상태가 되고 Auto Sync를 통해 Git의 상태를 Cluster에 반영합니다.

현재 Application에는 다음 정책을 적용했습니다.

```yaml
syncPolicy:
  automated:
    selfHeal: true
```

`selfHeal`을 활성화하여 Kubernetes Resource가 Git의 Desired State와 다르게 변경된 경우 다시 Git 상태로 복구할 수 있도록 구성했습니다.

현재 `prune`은 활성화하지 않았습니다.

---

## GitOps Design

이 프로젝트에서는 Jenkins가 Kubernetes Cluster에 직접 배포하지 않습니다.

```text
Jenkins
   │
   X kubectl apply
   │
   ▼
GitOps Repository
   │
   ▼
Argo CD
   │
   ▼
Kubernetes
```

Jenkins의 역할은 CI 및 GitOps Repository 업데이트까지입니다.

Kubernetes 배포 및 Desired State 관리는 Argo CD가 담당합니다.

이를 통해 CI와 CD의 역할을 분리했습니다.

---

## Argo CD Access

Argo CD는 Kubernetes 내부에 설치되어 있으며 Envoy Gateway를 통해 접근합니다.

```text
Client
   │
   ▼
192.168.10.117
MetalLB
   │
   ▼
Envoy Gateway
   │
   ▼
argocd-server
```

현재 Lab에서는 HTTP로 접근합니다.

```text
http://192.168.10.117
```

Argo CD Server는 Gateway 뒤에서 HTTP 통신을 사용할 수 있도록 `server.insecure=true`로 구성했습니다.

TLS 적용은 향후 개선 항목으로 남겨두었습니다.

---

## Repository Roles

CI/CD는 세 개의 Repository로 역할을 분리했습니다.

```text
server-inventory-api
        │
        │ Application Source
        ▼
      Jenkins
        │
        ▼
      Harbor


server-inventory-k8s
        │
        │ Kubernetes Desired State
        ▼
      Argo CD
        │
        ▼
    Kubernetes


kubernetes-cicd-lab
        │
        └── CI/CD 및 GitOps 구성/문서 관리
```

### server-inventory-api

FastAPI Application Source 및 실제 Jenkins Pipeline을 관리합니다.

### server-inventory-k8s

Kubernetes Manifest와 Argo CD가 사용하는 Desired State를 관리합니다.

### kubernetes-cicd-lab

Jenkins 및 Argo CD 기반 CI/CD / GitOps 구성 과정을 문서화합니다.

---

## Repository Structure

```text
kubernetes-cicd-lab/
├── README.md
│
├── argocd/
│   ├── README.md
│   ├── applications/
│   │   └── server-inventory.yaml
│   └── gateway/
│       ├── gateway.yaml
│       └── httproute.yaml
│
├── jenkins/
│   ├── README.md
│   └── Jenkinsfile
│
└── docs/
    ├── 01-argocd-installation.md
    ├── 02-gitops-deployment.md
    ├── 03-jenkins-installation.md
    ├── 04-ci-pipeline.md
    └── 05-cicd-integration.md
```

---

## Infrastructure

| Component | Address / Endpoint | Role |
|---|---|---|
| Jenkins | `192.168.10.121:8080` | CI Pipeline |
| Harbor | `192.168.10.114` | Container Registry |
| Argo CD | `192.168.10.117` | GitOps CD |
| GitHub | Public | Source / GitOps Repository |

Argo CD의 `192.168.10.117` 주소는 MetalLB에서 할당하며 Envoy Gateway를 통해 서비스를 제공합니다.

---

## Current Limitations

현재 Lab 환경에는 다음과 같은 제한 사항이 있습니다.

**GitHub Webhook**

Jenkins와 Argo CD가 Private Network에 있기 때문에 GitHub.com에서 직접 Webhook을 전달할 수 없습니다.

현재 Jenkins는 Poll SCM을 사용하고 Argo CD는 Repository Refresh를 통해 변경 사항을 감지합니다.

**Image Tag**

현재 컨테이너 이미지 Tag는 Jenkins `BUILD_NUMBER`를 사용합니다.

따라서 문서 변경 등으로 Jenkins Pipeline만 실행된 경우 실제 이미지 Tag 번호가 연속적이지 않을 수 있습니다.

예:

```text
server-inventory-api:1
server-inventory-api:5
server-inventory-api:11
```

이미지 동작에는 문제가 없지만 Source Commit과 이미지 간 추적성은 제한적입니다.

**Change Detection**

현재 애플리케이션 변경 감지는 다음 비교를 사용합니다.

```bash
git diff --name-only HEAD^ HEAD
```

따라서 현재 Commit과 바로 이전 Commit의 변경 사항을 기준으로 판단합니다.

---

## Future Improvements

향후 다음 항목을 개선할 수 있습니다.

- Jenkins `BUILD_NUMBER` 대신 Git Commit SHA 기반 Image Tag 적용
- 여러 Commit을 고려할 수 있도록 Change Detection 개선
- Application Test Stage 추가
- GitHub Credential 처리 방식 개선
- 외부 HTTPS Endpoint 구성 시 GitHub Webhook 적용
- Argo CD HTTPS/TLS 구성
- Secret Management 개선

---

## Documentation

세부 구축 과정은 다음 문서에서 정리합니다.

1. [Argo CD Installation](docs/01-argocd-installation.md)
2. [GitOps Deployment](docs/02-gitops-deployment.md)
3. [Jenkins Installation](docs/03-jenkins-installation.md)
4. [CI Pipeline](docs/04-ci-pipeline.md)
5. [CI/CD Integration](docs/05-cicd-integration.md)

---

## Related Repositories

- `server-inventory-api` - FastAPI Application Source
- `server-inventory-k8s` - Kubernetes Application / GitOps Repository
- `kubernetes-platform-lab` - Kubernetes Platform Infrastructure