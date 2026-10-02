# 05. CI/CD Integration

Jenkins CI Pipeline과 Argo CD GitOps를 연결하여 애플리케이션 Source 변경부터 Kubernetes 배포까지 자동화합니다.

이 프로젝트의 최종 CI/CD 구조는 다음과 같습니다.

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
    │
    ├── Checkout
    ├── Detect Changes
    ├── Build Image
    ├── Harbor Push
    └── GitOps Repository Update
                │
                ▼
GitHub
server-inventory-k8s
                │
                │ Repository Refresh
                ▼
             Argo CD
                │
                ├── Compare
                ├── Auto Sync
                └── selfHeal
                │
                ▼
        Kubernetes Cluster
                │
                ▼
       server-inventory-api
```

---

## 1. Repository Separation

CI/CD 환경은 역할에 따라 세 개의 Repository로 분리했습니다.

```text
server-inventory-api
        │
        └── Application Source / Jenkinsfile

server-inventory-k8s
        │
        └── Kubernetes Desired State

kubernetes-cicd-lab
        │
        └── CI/CD 및 GitOps 구성 / 문서
```

### Application Repository

```text
server-inventory-api
```

FastAPI Application Source와 실제 Jenkins Pipeline을 관리합니다.

Application Source가 변경되면 Jenkins CI Pipeline이 시작됩니다.

### GitOps Repository

```text
server-inventory-k8s
```

Kubernetes Manifest를 관리합니다.

Argo CD는 이 Repository를 Kubernetes Cluster의 Desired State로 사용합니다.

### CI/CD Lab Repository

```text
kubernetes-cicd-lab
```

Jenkins와 Argo CD 구성, Gateway Manifest 및 구축 문서를 관리합니다.

---

## 2. CI and CD Responsibility

Jenkins와 Argo CD의 역할을 분리했습니다.

```text
                CI
        ┌───────────────────┐
        │      Jenkins      │
        │                   │
        │ Source Checkout   │
        │ Change Detection  │
        │ Image Build       │
        │ Harbor Push       │
        │ GitOps Update     │
        └─────────┬─────────┘
                  │
                  │ Git Push
                  ▼
        server-inventory-k8s
                  │
                  │
        ┌─────────▼─────────┐
        │      Argo CD      │
        │                   │
        │ Compare           │
        │ Auto Sync         │
        │ selfHeal          │
        └─────────┬─────────┘
                  │
                  ▼
             Kubernetes
                  CD
```

Jenkins는 Kubernetes API를 이용하여 Application을 직접 배포하지 않습니다.

즉 Pipeline에서 다음과 같은 방식을 사용하지 않습니다.

```bash
kubectl apply -f ...
```

대신 Jenkins는 GitOps Repository만 변경합니다.

```text
Jenkins
   ↓
GitOps Repository
   ↓
Argo CD
   ↓
Kubernetes
```

이를 통해 CI와 CD의 책임을 분리했습니다.

---

# End-to-End Deployment

## 3. Application Source Change

개발자가 `server-inventory-api` Repository의 Application Source를 수정합니다.

예:

```text
main.py
```

변경 사항을 Commit하고 Push합니다.

```bash
git add main.py
git commit -m "Update application"
git push origin main
```

GitHub에 새로운 Commit이 저장됩니다.

---

## 4. Jenkins Detects Git Change

Jenkins는 Poll SCM을 사용합니다.

```text
H/2 * * * *
```

SCM 변경이 발견되면 Pipeline이 자동으로 실행됩니다.

```text
GitHub
   │
   │ Change detected
   ▼
Jenkins
```

Jenkins Web UI에서 `Build Now`를 직접 실행하지 않아도 Pipeline이 시작되는 것을 확인했습니다.

---

## 5. Application Change Detection

Pipeline은 변경된 파일을 확인합니다.

```bash
git diff --name-only HEAD^ HEAD
```

Application 관련 파일:

```text
main.py
requirements.txt
Dockerfile
```

중 하나가 변경되면:

```text
APP_CHANGED=true
```

가 됩니다.

Pipeline:

```text
Detect Changes
      │
      ▼
APP_CHANGED=true
      │
      ▼
Continue CI
```

---

## 6. Container Image Build

Jenkins는 Rootless Podman을 사용하여 새로운 Container Image를 생성합니다.

```text
harbor.lab.local/server-inventory/server-inventory-api:<BUILD_NUMBER>
```

예:

```text
harbor.lab.local/server-inventory/server-inventory-api:15
```

Build는 Jenkins Linux 사용자의 Rootless Container 환경에서 수행됩니다.

---

## 7. Push Image to Harbor

Jenkins Credentials에 저장된 Harbor 계정을 이용하여 Registry에 Login합니다.

```text
Jenkins
   │
   │ harbor-credentials
   ▼
Harbor
```

Build된 Image를 다음 Registry로 Push합니다.

```text
harbor.lab.local
```

Harbor Project:

```text
server-inventory
```

Repository:

```text
server-inventory-api
```

Harbor에서 새로운 Image Tag가 생성된 것을 확인합니다.

---

## 8. Update GitOps Repository

Image Push가 성공하면 Jenkins는:

```text
server-inventory-k8s
```

Repository를 Clone합니다.

그리고:

```text
api/deployment.yaml
```

의 Image Tag를 새로운 Jenkins `BUILD_NUMBER`로 변경합니다.

예:

```yaml
image: harbor.lab.local/server-inventory/server-inventory-api:14
```

에서:

```yaml
image: harbor.lab.local/server-inventory/server-inventory-api:15
```

로 변경합니다.

Jenkins가 변경 사항을 Commit합니다.

예:

```text
Update server-inventory-api image to 15
```

그리고 `main` Branch에 Push합니다.

이 시점에서 Kubernetes의 Desired State가 변경됩니다.

---

## 9. Argo CD Detects GitOps Change

Argo CD Application은:

```text
server-inventory-k8s
```

Repository를 Source로 사용합니다.

```text
Jenkins
   │
   │ Git Push
   ▼
server-inventory-k8s

image: :15
   │
   ▼
Argo CD Repository Refresh
```

새로운 Git Revision을 발견하면 Argo CD는 Git과 Kubernetes Live State를 비교합니다.

기존 Cluster:

```text
image: :14
```

Git:

```text
image: :15
```

따라서:

```text
OutOfSync
```

상태가 됩니다.

---

## 10. Argo CD Auto Sync

Application에는 Auto Sync가 활성화되어 있습니다.

```yaml
syncPolicy:
  automated:
    selfHeal: true
```

Argo CD가 새로운 Desired State를 확인하면 Kubernetes Resource를 자동으로 변경합니다.

```text
Git

image: :15

    ↓

Argo CD Auto Sync

    ↓

Kubernetes Deployment

image: :15
```

Sync 완료 후:

```text
Synced
Healthy
```

상태가 되는 것을 확인했습니다.

---

## 11. Kubernetes Rollout

Deployment의 Pod Template에 설정된 Image Tag가 변경되기 때문에 Kubernetes Deployment에서 새로운 Rollout이 발생합니다.

```text
Deployment
image :14
    │
    ▼
Argo CD Sync
    │
    ▼
Deployment
image :15
    │
    ▼
New ReplicaSet / Pod
```

현재 Deployment Image는 다음과 같이 확인할 수 있습니다.

```bash
kubectl get deployment \
  -n server-inventory \
  -o custom-columns=NAME:.metadata.name,IMAGE:.spec.template.spec.containers[*].image
```

새로운 Image Tag가 적용되어 있으면 CI/CD Pipeline 전체 과정이 완료된 것입니다.

---

# Documentation-only Changes

## 12. Skip Unnecessary Deployment

초기 Pipeline에서는 `README.md`와 같은 문서만 변경해도 Container Image가 Build되는 문제가 있었습니다.

```text
README.md
    ↓
Jenkins
    ↓
Image Build
    ↓
Harbor
    ↓
GitOps Update
    ↓
Kubernetes Rollout
```

Application에 영향을 주지 않는 변경 때문에 불필요한 CI/CD 작업이 발생하는 구조였습니다.

이를 개선하기 위해 Application Change Detection을 추가했습니다.

---

## 13. Documentation Change Flow

예를 들어:

```text
README.md
```

만 수정합니다.

Jenkins Poll SCM은 Git Commit 자체가 발생했기 때문에 Pipeline을 시작합니다.

하지만 Detect Changes 결과:

```text
Changed files:
README.md

APP_CHANGED=false
```

가 됩니다.

따라서:

```text
Jenkins Pipeline
      │
      ├── Checkout             EXECUTED
      ├── Detect Changes       EXECUTED
      │
      ├── Build Image          SKIPPED
      ├── Push Image           SKIPPED
      └── GitOps Update        SKIPPED
```

Jenkins Build 자체는 기록되지만 Container Image Build 및 Kubernetes Deployment는 발생하지 않습니다.

이 동작 역시 실제 Pipeline에서 검증했습니다.

---

# Argo CD Repository Refresh

## 14. Auto Sync and Refresh

Auto Sync와 Repository Refresh는 서로 다른 역할을 합니다.

```text
GitHub
   │
   │ Repository Refresh
   ▼
Argo CD
   │
   │ New Revision 발견
   ▼
OutOfSync
   │
   │ Auto Sync
   ▼
Synced
```

Auto Sync가 활성화되어 있어도 Argo CD가 아직 새로운 Git Revision을 확인하지 않았다면 기존 Revision 기준으로 `Synced` 상태일 수 있습니다.

UI에서 `Refresh`를 실행하면 Git Repository 상태를 즉시 다시 확인할 수 있습니다.

실제 테스트 과정에서 GitOps Repository가 변경된 후 Refresh를 수행하자 새로운 Revision을 확인하고 즉시 Auto Sync가 수행되는 것을 확인했습니다.

이후 다른 테스트에서는 별도의 수동 Refresh 없이 Repository 변경을 감지하고 Auto Sync가 동작하는 것도 확인했습니다.

---

# End-to-End Verification

## 15. Verified Application Change Flow

실제로 다음 전체 흐름을 검증했습니다.

```text
server-inventory-api
        │
        │ git push
        ▼
     GitHub
        │
        │ Poll SCM
        ▼
     Jenkins
        │
        ├── APP_CHANGED=true
        │
        ├── Podman Build
        │
        ├── Harbor Push
        │
        └── GitOps Update
                │
                ▼
       server-inventory-k8s
                │
                ▼
             Argo CD
                │
                ├── New Revision
                ├── OutOfSync
                └── Auto Sync
                │
                ▼
          Kubernetes
                │
                ▼
          New Image Running
```

결과:

```text
Jenkins Pipeline      SUCCESS
Harbor Image          CREATED
GitOps Commit         CREATED
Argo CD               SYNCED / HEALTHY
Kubernetes Deployment UPDATED
```

---

## 16. Verified Documentation Change Flow

문서 또는 Pipeline 설정만 변경하는 경우도 검증했습니다.

```text
Jenkinsfile / README
        │
        ▼
     Git Push
        │
        ▼
   Jenkins Poll SCM
        │
        ▼
   Detect Changes
        │
        ▼
APP_CHANGED=false
        │
        ├── Build SKIPPED
        ├── Push SKIPPED
        └── GitOps Update SKIPPED
```

따라서 Application에 영향을 주지 않는 변경으로 인해 새로운 Container Image가 생성되거나 Kubernetes Deployment가 발생하지 않습니다.

---

# Failure Boundaries

## 17. Pipeline Failure Points

CI/CD Pipeline은 여러 시스템이 연결되어 있기 때문에 단계별로 장애 지점을 구분할 수 있습니다.

```text
GitHub
  │
  ▼
Jenkins
  │
  ├── Git Checkout Failure
  ├── Image Build Failure
  ├── Harbor Authentication Failure
  ├── Harbor Push Failure
  └── GitOps Push Failure
              │
              ▼
           Argo CD
              │
              ├── Repository Access Failure
              ├── Manifest Error
              └── Sync Failure
                       │
                       ▼
                  Kubernetes
```

어느 단계에서 실패했는지 확인하면 장애 범위를 좁힐 수 있습니다.

---

## 18. Troubleshooting Experienced During Build

구축 과정에서 다음 문제를 확인하고 해결했습니다.

```text
Jenkins Declarative Pipeline Plugin
→ pipeline DSL 인식 문제

Podman Package Installation
→ containerd.io / runc Package Conflict

Rootless Podman
→ XDG_RUNTIME_DIR 문제

Harbor Push
→ Private CA Certificate Trust 문제

GitOps Clone
→ GitHub Credential Username 설정 문제

Argo CD
→ Repository Refresh와 Auto Sync 동작 차이
```

각 문제를 단계별로 확인하면서 CI/CD 전체 연결을 검증했습니다.

---

# Current Limitations

## 19. Polling-based Trigger

현재 Jenkins와 Argo CD 모두 Private Lab Network에 있습니다.

```text
Jenkins
192.168.10.121

Argo CD
192.168.10.117
```

GitHub.com에서 내부 Private IP에 직접 접근할 수 없기 때문에 GitHub Webhook 기반 Trigger를 사용하지 않습니다.

현재:

```text
GitHub
   ↑
   │ Poll SCM
Jenkins
```

및:

```text
GitHub
   ↑
   │ Repository Refresh
Argo CD
```

방식을 사용합니다.

---

## 20. Image Tag

현재 Container Image Tag는:

```text
Jenkins BUILD_NUMBER
```

를 사용합니다.

Jenkins Build와 Container Image Build 횟수가 항상 동일하지 않기 때문에 Image Tag 번호가 연속적이지 않을 수 있습니다.

향후:

```text
Git Commit SHA
```

기반 Image Tag로 변경하면 Source와 Container Image 간 추적성을 높일 수 있습니다.

---

## 21. Change Detection

현재:

```bash
git diff --name-only HEAD^ HEAD
```

을 사용하여 Application 변경 여부를 확인합니다.

이는 바로 이전 Commit과 현재 Commit을 비교하는 방식이므로 여러 Commit이 Pipeline 실행 사이에 누적되는 상황에 대한 개선이 가능합니다.

---

# Future Improvements

현재 End-to-End CI/CD Pipeline은 정상적으로 동작합니다.

향후 다음 항목을 추가로 개선할 수 있습니다.

```text
Git Commit SHA Image Tag
        │
        ├── Source/Image Traceability

Improved Change Detection
        │
        ├── Multiple Commit Handling

Automated Test Stage
        │
        ├── Test before Build/Push

Webhook
        │
        ├── Immediate CI Trigger

Credential Handling
        │
        ├── Safer Git Authentication

Argo CD TLS
        │
        ├── HTTPS Access

Secret Management
        │
        └── External Secrets / Vault etc.
```

현재 Lab의 목표는 Jenkins, Harbor, GitOps Repository, Argo CD, Kubernetes를 연결하여 CI/CD 전체 흐름을 직접 구축하고 검증하는 것입니다.

---

# Final Architecture

```text
                         GitHub
                server-inventory-api
                           │
                           │ Poll SCM
                           ▼
                     ┌───────────┐
                     │  Jenkins  │
                     └─────┬─────┘
                           │
                 Detect Application Change
                           │
                    APP_CHANGED=true
                           │
                           ▼
                     Podman Build
                           │
                           ▼
                  ┌─────────────────┐
                  │     Harbor      │
                  │ 192.168.10.114  │
                  └─────────────────┘
                           │
                           │ Image Push
                           ▼
                     Jenkins
                           │
                           │ Update Manifest
                           ▼
                         GitHub
                 server-inventory-k8s
                           │
                           │ Repository Refresh
                           ▼
                     ┌───────────┐
                     │  Argo CD  │
                     └─────┬─────┘
                           │
                   Auto Sync / selfHeal
                           │
                           ▼
                  Kubernetes Cluster
                           │
                           ▼
                 server-inventory-api
```

---

## Conclusion

이 프로젝트를 통해 다음 CI/CD 및 GitOps 흐름을 구축했습니다.

```text
Source Code
    ↓
Jenkins CI
    ↓
Container Image
    ↓
Harbor Registry
    ↓
GitOps Repository
    ↓
Argo CD
    ↓
Kubernetes
```

Jenkins와 Argo CD의 역할을 분리하고 Git Repository를 Kubernetes Desired State의 기준으로 사용하는 GitOps 기반 배포 구조를 구성했습니다.

Application Source 변경 시 Kubernetes 배포까지 자동화되며, 문서 변경과 같이 Application에 영향을 주지 않는 변경은 Container Build 및 Deployment를 Skip하도록 Pipeline을 구성했습니다.

---

## Documentation

- [01. Argo CD Installation](01-argocd-installation.md)
- [02. GitOps Deployment](02-gitops-deployment.md)
- [03. Jenkins Installation](03-jenkins-installation.md)
- [04. CI Pipeline](04-ci-pipeline.md)