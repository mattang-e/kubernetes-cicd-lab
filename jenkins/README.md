# Jenkins

Jenkins를 사용하여 Application Source 변경 감지, Container Image Build, Harbor Push 및 GitOps Repository Update를 자동화합니다.

## Architecture

```text
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
           server-inventory-k8s
```

Jenkins Server:

```text
Hostname: jenkins01
IP:       192.168.10.121
```

## Pipeline

실제 실행되는 Jenkinsfile은 Application Source Repository에서 관리합니다.

```text
server-inventory-api/Jenkinsfile
```

이 Repository의:

```text
jenkins/Jenkinsfile
```

은 CI/CD Lab의 전체 구성을 확인하기 위한 **reference copy**입니다.

실제 Jenkins Job은 `server-inventory-api` Repository의 Jenkinsfile을 사용합니다.

## Pipeline Stages

```text
Checkout
    ↓
Detect Changes
    ↓
APP_CHANGED?
    │
    ├── false
    │     ├── Build Image       SKIPPED
    │     ├── Push Image        SKIPPED
    │     └── GitOps Update     SKIPPED
    │
    └── true
          ↓
      Build Image
          ↓
      Harbor Push
          ↓
      GitOps Repository Update
```

## Change Detection

현재 다음 파일이 변경되었을 때 Application 변경으로 판단합니다.

```text
main.py
requirements.txt
Dockerfile
```

Application과 관련 없는 문서 변경은 Image Build 및 Deployment를 수행하지 않습니다.

예:

```text
README.md
    ↓
APP_CHANGED=false
    ↓
Build / Push / GitOps Update SKIPPED
```

## Container Build

Container Image Build에는 Rootless Podman을 사용합니다.

```text
Jenkins
   ↓
jenkins Linux user
   ↓
Rootless Podman
```

Image:

```text
harbor.lab.local/server-inventory/server-inventory-api:<BUILD_NUMBER>
```

## Harbor

Private Container Registry:

```text
harbor.lab.local
192.168.10.114
```

Harbor 인증 정보는 Jenkins Credentials에서 관리합니다.

```text
Credential ID:
harbor-credentials
```

Private CA는 Jenkins Host의 Podman Registry Trust Store에 등록했습니다.

```text
/etc/containers/certs.d/harbor.lab.local/ca.crt
```

## GitHub Credential

GitOps Repository를 변경하기 위한 GitHub 인증 정보 역시 Jenkins Credentials에서 관리합니다.

```text
Credential ID:
github-credentials
```

Jenkins는 Image Push가 완료되면:

```text
server-inventory-k8s/api/deployment.yaml
```

의 Image Tag를 변경하고 GitHub에 Push합니다.

## Trigger

Jenkins는 Private Network에 있기 때문에 GitHub.com Webhook을 직접 받을 수 없습니다.

따라서 현재 Lab에서는 Poll SCM을 사용합니다.

```text
H/2 * * * *
```

Git 변경이 발견된 경우에만 Pipeline이 실행됩니다.

## Current Limitations

현재 Image Tag는 Jenkins `BUILD_NUMBER`를 사용합니다.

향후 Git Commit SHA 기반 Tag로 변경하여 Source와 Container Image의 추적성을 개선할 수 있습니다.

현재 Change Detection은:

```bash
git diff --name-only HEAD^ HEAD
```

을 사용하므로 여러 Commit이 Pipeline 실행 사이에 누적되는 경우를 고려한 개선이 가능합니다.

## Documentation

상세 구축 과정:

- [Jenkins Installation](../docs/03-jenkins-installation.md)
- [CI Pipeline](../docs/04-ci-pipeline.md)
- [CI/CD Integration](../docs/05-cicd-integration.md)