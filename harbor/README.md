# Harbor Private Registry

On-Premise Kubernetes 환경에서 Container Image를 관리하기 위해
Private Harbor Registry를 구성했습니다.

초기 Application 배포 테스트에서는 Apple Silicon Mac에서
Docker Buildx를 이용하여 Image를 직접 Build/Push했습니다.

현재 CI/CD 환경에서는 Jenkins가 Rootless Podman을 사용하여
Application Image Build 및 Harbor Push를 자동으로 수행합니다.

## Architecture

현재 Container Image Build 및 배포 흐름은 다음과 같습니다.

```text
Application Source
server-inventory-api
        |
        | Poll SCM
        v
     Jenkins
        |
        | Rootless Podman
        | Build / Push
        v
Harbor Private Registry
harbor.lab.local
        |
        | Image Pull
        v
Kubernetes Worker
        |
        | containerd
        v
   FastAPI Pod
```

전체 CI/CD 환경에서는 Harbor Push 이후 Jenkins가 GitOps Repository의
Image Tag를 변경하고 Argo CD가 Kubernetes 배포를 담당합니다.

```text
GitHub
server-inventory-api
        |
        v
     Jenkins
        |
        +---- Build / Push ----> Harbor
        |
        +---- GitOps Update ---> server-inventory-k8s
                                      |
                                      v
                                   Argo CD
                                      |
                                      v
                                  Kubernetes
```

---

## Environment

- OS: Rocky Linux 8.10
- Harbor: v2.15.2
- Container Runtime: Docker Engine
- Deployment: Docker Compose
- Hostname: `harbor.lab.local`
- HTTPS: 443
- Harbor Data: `/data`

Harbor는 Kubernetes Cluster와 별도의 Registry Server에서 운영합니다.

---

## Harbor Installation

Harbor 설치 파일을 다운로드하고 압축을 해제합니다.

Harbor 설정 파일은 다음 경로에서 관리합니다.

```text
/opt/harbor/harbor.yml
```

주요 설정:

```yaml
hostname: harbor.lab.local

https:
  port: 443
  certificate: /data/cert/harbor.crt
  private_key: /data/cert/harbor.key

data_volume: /data
```

설정 완료 후 Harbor를 설치합니다.

```bash
./install.sh
```

Harbor Container 상태를 확인합니다.

```bash
docker compose ps
```

---

## DNS

Harbor Registry는 다음 이름을 사용합니다.

```text
harbor.lab.local
```

Harbor를 사용하는 Client, Jenkins Server 및 Kubernetes Worker에서
해당 이름을 Harbor Registry Server IP로 해석할 수 있어야 합니다.

Lab 환경에서는 필요한 Host에 이름 해석 정보를 구성했습니다.

```text
harbor.lab.local
        |
        v
Harbor Registry
192.168.10.114
```

---

# Private CA

Harbor는 HTTPS 통신을 사용하며 Private CA 인증서를 사용합니다.

Private CA를 사용하는 경우 Harbor에 접근하는 Docker, Podman,
containerd 등의 Client가 해당 CA를 신뢰하도록 구성해야 합니다.

```text
                Harbor
           harbor.lab.local
                 |
              HTTPS
                 |
        +--------+--------+
        |        |        |
        v        v        v
      Docker   Podman  containerd
        |        |        |
        +--------+--------+
                 |
          Private CA Trust
```

---

## Docker Client

Docker Client에서 Harbor CA를 신뢰하도록 설정합니다.

```text
~/.docker/certs.d/harbor.lab.local/ca.crt
```

설정 후 Harbor Login을 확인합니다.

```bash
docker login harbor.lab.local
```

---

## Jenkins / Podman

현재 CI Pipeline에서는 Jenkins가 Rootless Podman을 사용하여
Container Image를 Build하고 Harbor에 Push합니다.

Jenkins Server에서도 Harbor Private CA를 신뢰해야 합니다.

CA Certificate를 다음 Registry별 Trust Store에 구성했습니다.

```text
/etc/containers/certs.d/harbor.lab.local/ca.crt
```

초기 구성 과정에서 Podman으로 Harbor에 접근했을 때 다음 TLS 오류가
발생했습니다.

```text
x509: certificate signed by unknown authority
```

Harbor CA를 Podman Trust Store에 등록한 후 Jenkins Pipeline에서
Harbor Login 및 Image Push가 정상적으로 수행되었습니다.

---

## Kubernetes Worker / containerd

Kubernetes Worker의 containerd에서도 Harbor Private CA를
신뢰하도록 설정합니다.

CA Certificate:

```text
/etc/containerd/certs.d/harbor.lab.local/ca.crt
```

Registry 설정:

```text
/etc/containerd/certs.d/harbor.lab.local/hosts.toml
```

예:

```toml
server = "https://harbor.lab.local"

[host."https://harbor.lab.local"]
  capabilities = ["pull", "resolve"]
  ca = "/etc/containerd/certs.d/harbor.lab.local/ca.crt"
```

containerd가 Registry별 설정을 읽을 수 있도록 `config.toml`에서
다음 경로를 사용합니다.

```text
config_path = "/etc/containerd/certs.d"
```

설정 변경 후 containerd를 재시작합니다.

```bash
systemctl restart containerd
```

---

# Harbor Project

Server Inventory API Image는 다음 Harbor Project에서 관리합니다.

```text
server-inventory
```

Image Repository:

```text
harbor.lab.local/server-inventory/server-inventory-api
```

Image Tag를 포함하면 다음과 같은 형태입니다.

```text
harbor.lab.local/server-inventory/server-inventory-api:<tag>
```

초기 수동 배포에서는:

```text
harbor.lab.local/server-inventory/server-inventory-api:v1
```

을 사용했습니다.

현재 Jenkins Pipeline에서는 Jenkins `BUILD_NUMBER`를 Image Tag로
사용합니다.

예:

```text
harbor.lab.local/server-inventory/server-inventory-api:15
```

---

# Manual Build and Push

초기 Kubernetes Application 배포 테스트에서는 개발 환경에서
Container Image를 직접 Build하여 Harbor에 Push했습니다.

개발 환경은 Apple Silicon Mac이고 Kubernetes Worker는
`linux/amd64` 환경이므로 Target Architecture를 명시했습니다.

```text
Apple Silicon Mac
     arm64
       |
       | Docker Buildx
       | --platform linux/amd64
       v
linux/amd64 Image
       |
       v
     Harbor
       |
       v
Kubernetes Worker
    linux/amd64
```

Docker Buildx를 사용하여 Build와 Push를 수행합니다.

```bash
docker buildx build \
  --platform linux/amd64 \
  -t harbor.lab.local/server-inventory/server-inventory-api:v1 \
  --push .
```

`--push`를 사용하면 Build된 Image가 바로 Harbor에 Push됩니다.

이 방식은 초기 Application 배포와 Registry 연결을 검증하기 위해
사용했습니다.

현재는 Jenkins Pipeline이 Image Build와 Push를 자동으로 수행합니다.

---

# Jenkins CI Integration

현재 CI/CD Pipeline에서는 Jenkins가 Application Source 변경을 감지한 후
Container Image Build와 Harbor Push를 수행합니다.

```text
Git Push
   |
   v
GitHub
   |
   | Poll SCM
   v
Jenkins
   |
   | Detect Changes
   v
APP_CHANGED=true
   |
   v
Podman Build
   |
   v
Harbor Push
```

Jenkins Server는 Kubernetes Worker와 동일한 `linux/amd64`
Architecture를 사용하므로 현재 Pipeline에서는 별도의 Cross Platform
Build 없이 Podman으로 Image를 생성합니다.

Image Build:

```bash
podman build \
  -t harbor.lab.local/server-inventory/server-inventory-api:<tag> .
```

Harbor Push:

```bash
podman push \
  harbor.lab.local/server-inventory/server-inventory-api:<tag>
```

---

## Jenkins Harbor Credentials

Harbor Username과 Password를 Jenkinsfile에 직접 저장하지 않습니다.

Jenkins Credentials에 다음 ID로 저장합니다.

```text
harbor-credentials
```

Pipeline에서는 `withCredentials`를 사용하여 필요한 시점에 Credential을
Environment Variable로 전달합니다.

```groovy
withCredentials([
    usernamePassword(
        credentialsId: 'harbor-credentials',
        usernameVariable: 'HARBOR_USER',
        passwordVariable: 'HARBOR_PASSWORD'
    )
]) {
    ...
}
```

Harbor Login:

```bash
printf '%s' "$HARBOR_PASSWORD" | \
  podman login harbor.lab.local \
  --username "$HARBOR_USER" \
  --password-stdin
```

Password를 Git Repository나 Jenkinsfile에 직접 기록하지 않습니다.

---

## Current Image Tag Strategy

현재 Jenkins Pipeline에서는 Jenkins의:

```text
BUILD_NUMBER
```

를 Container Image Tag로 사용합니다.

예:

```text
server-inventory-api:15
```

Jenkins Job 자체의 Build Number를 사용하기 때문에 Application Image가
생성되지 않은 Build도 Jenkins Build Number는 증가할 수 있습니다.

예:

```text
Build #10
README 변경
→ Image Build SKIPPED

Build #11
main.py 변경
→ server-inventory-api:11
```

따라서 Image Tag가 반드시 연속적이지는 않습니다.

향후 Git Commit SHA 기반 Tag를 사용하여 Source Code와 Container Image의
추적성을 개선할 수 있습니다.

예:

```text
server-inventory-api:a84f21c
```

---

# Kubernetes Registry Authentication

Harbor Project가 Private이므로 Kubernetes에서 Image를 Pull할 때
Registry 인증정보가 필요합니다.

`server-inventory` Namespace에 Registry Secret을 생성합니다.

```bash
kubectl create secret docker-registry harbor-secret \
  --docker-server=harbor.lab.local \
  --docker-username=<username> \
  --docker-password=<password> \
  -n server-inventory
```

FastAPI Deployment에서 해당 Secret을 사용합니다.

```yaml
spec:
  imagePullSecrets:
    - name: harbor-secret
```

실제 Harbor Username과 Password는 Git Repository에 저장하지 않습니다.

---

# Image Pull Flow

Jenkins가 새로운 Image를 Harbor에 Push하고 GitOps Repository의
Deployment Image Tag가 변경되면 Argo CD가 Kubernetes Deployment를
업데이트합니다.

이후 Kubernetes Worker가 Harbor에서 새로운 Image를 Pull합니다.

```text
Jenkins
   |
   | podman push
   v
Harbor Private Registry
   |
   | HTTPS
   | Private CA
   v
GitOps Repository
   |
   v
Argo CD
   |
   | Deployment Update
   v
Kubernetes
   |
   v
Kubelet
   |
   | harbor-secret
   | containerd
   v
Harbor Private Registry
   |
   | Image Pull
   v
FastAPI Pod
```

Image Pull 과정에서는 다음 요소가 필요합니다.

```text
Harbor DNS Resolution
        +
Private CA Trust
        +
harbor-secret
        +
containerd Registry Configuration
```

---

# Harbor Role in CI/CD

Harbor는 현재 CI/CD Pipeline에서 Jenkins와 Kubernetes 사이의
Container Image Registry 역할을 담당합니다.

```text
Application Source
       |
       v
    Jenkins
       |
       | Build
       v
Container Image
       |
       | Push
       v
     Harbor
       |
       | Pull
       v
  Kubernetes
```

Jenkins는 새로운 Image를 Harbor에 저장하고 GitOps Repository의
Image Tag를 변경합니다.

Argo CD는 GitOps Repository의 변경을 Kubernetes에 반영합니다.

따라서 각 구성요소의 역할은 다음과 같습니다.

```text
Jenkins
→ CI / Image Build / Image Push / GitOps Update

Harbor
→ Container Image Registry

GitOps Repository
→ Kubernetes Desired State

Argo CD
→ GitOps CD

Kubernetes
→ Application Runtime
```

---

# Verification

Harbor Container 상태:

```bash
docker compose ps
```

Jenkins Server에서 Harbor 이름 확인:

```bash
getent hosts harbor.lab.local
```

Podman Login:

```bash
podman login harbor.lab.local
```

Kubernetes에서 현재 사용 중인 Image 확인:

```bash
kubectl get deployment \
  -n server-inventory \
  -o custom-columns=NAME:.metadata.name,IMAGE:.spec.template.spec.containers[*].image
```

Pod 상태:

```bash
kubectl get pods -n server-inventory
```

정상적인 전체 흐름:

```text
Jenkins Build
     ↓
Harbor Image Created
     ↓
GitOps Image Tag Updated
     ↓
Argo CD Synced
     ↓
Kubernetes Pod Running
```

---

# Related Documentation

Jenkins 구성:

```text
../docs/03-jenkins-installation.md
```

CI Pipeline:

```text
../docs/04-ci-pipeline.md
```

전체 CI/CD Integration:

```text
../docs/05-cicd-integration.md
```

Argo CD:

```text
../argocd/README.md
```

---

# Future Improvements

현재 Harbor와 CI/CD Pipeline은 정상적으로 연동되어 있습니다.

향후 다음 항목을 개선할 수 있습니다.

```text
Git Commit SHA Image Tag
        |
        └── Source / Image Traceability

Harbor Robot Account
        |
        └── CI 전용 Registry Credential

Automated Image Cleanup
        |
        └── Image Retention Policy

TLS / Certificate Management
        |
        └── Certificate Lifecycle Automation
```
## Related Documentation

Harbor와 연동되는 Jenkins, Argo CD 및 전체 CI/CD 구성은 다음 문서에서 확인할 수 있습니다.

- [Jenkins Installation](../docs/03-jenkins-installation.md)
- [CI Pipeline](../docs/04-ci-pipeline.md)
- [CI/CD Integration](../docs/05-cicd-integration.md)
- [Argo CD](../argocd/README.md)