# 03. Jenkins Installation

CI Pipeline을 실행하기 위한 Jenkins Server를 별도의 VM에 구성합니다.

이 프로젝트에서 Jenkins는 다음 역할을 담당합니다.

```text
GitHub
   │
   ▼
Jenkins
   ├── Source Checkout
   ├── Application Change Detection
   ├── Container Image Build
   ├── Harbor Push
   └── GitOps Repository Update
```

Jenkins가 Kubernetes에 직접 배포하지 않으며, Kubernetes 배포는 Argo CD가 담당합니다.

---

## 1. Jenkins Server

Jenkins는 Kubernetes Cluster 외부의 별도 VM에 설치했습니다.

```text
Hostname: jenkins01
IP:       192.168.10.121
OS:       Rocky Linux 8
```

구조:

```text
Kubernetes Cluster
192.168.10.101 ~ 113

Harbor
192.168.10.114

Jenkins
192.168.10.121
```

Jenkins를 Kubernetes 내부에 배포하지 않고 별도 VM으로 구성하여 CI Server와 Kubernetes Cluster의 역할을 분리했습니다.

---

## 2. Install Java

Jenkins 실행을 위해 Java 21을 설치합니다.

```bash
dnf install -y java-21-openjdk
```

설치 확인:

```bash
java -version
```

---

## 3. Configure Jenkins Repository

Jenkins LTS RPM Repository를 추가합니다.

```bash
curl -fsSL -o /etc/yum.repos.d/jenkins.repo \
  https://pkg.jenkins.io/rpm-stable/jenkins.repo
```

Jenkins Repository Signing Key를 등록합니다.

```bash
rpm --import https://pkg.jenkins.io/rpm-stable/jenkins.io-2026.key
```

Repository Cache를 갱신합니다.

```bash
dnf clean all
dnf makecache
```

> Jenkins Repository URL과 Signing Key는 변경될 수 있으므로 신규 환경에서는 Jenkins 공식 설치 문서를 기준으로 확인합니다.

---

## 4. Install Jenkins

Jenkins를 설치합니다.

```bash
dnf install -y fontconfig java-21-openjdk jenkins
```

서비스를 활성화하고 시작합니다.

```bash
systemctl enable --now jenkins
```

상태 확인:

```bash
systemctl status jenkins
```

---

## 5. Initial Jenkins Login

Jenkins 기본 Web Port는 `8080`입니다.

```text
http://192.168.10.121:8080
```

초기 관리자 Password를 확인합니다.

```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

Web UI에서 Password를 입력하고 초기 설정을 진행합니다.

Plugin 설치 단계에서는:

```text
Install suggested plugins
```

을 선택했습니다.

초기 관리자 계정을 생성한 후 Jenkins Web UI에 로그인합니다.

---

## 6. Jenkins Job Execution User

Jenkins Service는 Linux의 `jenkins` 사용자로 실행됩니다.

확인:

```bash
ps -ef | grep jenkins
```

Pipeline에서 다음 명령을 실행하면:

```groovy
sh 'whoami'
```

결과는 다음과 같습니다.

```text
jenkins
```

따라서 Pipeline에서 실행되는 Git, Podman 등의 명령 역시 기본적으로 `jenkins` 사용자 권한으로 실행됩니다.

```text
Jenkins Service
      │
      ▼
Linux user: jenkins
      │
      ├── git
      ├── podman
      └── shell commands
```

---

# Podman Configuration

Jenkins에서 Container Image를 Build하기 위해 Podman을 사용합니다.

Docker가 반드시 필요한 것은 아니며, Rocky Linux 환경에서 Rootless Container Build를 구성하기 위해 Podman을 선택했습니다.

---

## 7. Install Podman

Podman을 설치합니다.

```bash
dnf install -y podman
```

### containerd.io / runc Conflict

Jenkins VM에 기존 Docker CE 관련 `containerd.io` Package가 설치되어 있는 경우 Podman 설치 과정에서 Rocky Linux의 `runc` Package와 충돌할 수 있습니다.

실제 Lab에서도 다음 계열의 Package Conflict가 발생했습니다.

```text
containerd.io
      ↕ conflict
runc
```

Jenkins VM은 Kubernetes Node가 아니므로 기존 Docker CE 환경이 필요하지 않은 것을 확인한 후 `containerd.io`를 제거하고 Podman을 설치했습니다.

> Kubernetes Node에서 동일한 작업을 수행하면 Container Runtime에 영향을 줄 수 있으므로 Jenkins VM과 Kubernetes Node를 구분해야 합니다.

설치 확인:

```bash
podman --version
```

---

## 8. Rootless Podman

Jenkins Pipeline에서 Host의 `root` 권한을 사용하지 않고 `jenkins` 사용자로 Podman을 실행하도록 구성했습니다.

```text
Jenkins Pipeline
       │
       ▼
Linux user: jenkins
       │
       ▼
Rootless Podman
```

Jenkins 사용자에게 subordinate UID/GID Range를 설정합니다.

```bash
usermod --add-subuids 100000-165535 jenkins
usermod --add-subgids 100000-165535 jenkins
```

설정 확인:

```bash
grep jenkins /etc/subuid
grep jenkins /etc/subgid
```

예:

```text
jenkins:100000:65536
```

`subuid`와 `subgid`는 Jenkins 사용자에게 Host Root 권한을 부여하는 설정이 아닙니다.

Rootless Container 내부의 UID/GID를 Host의 subordinate ID Range와 Mapping하기 위해 사용합니다.

---

## 9. Jenkins User Runtime Directory

Rootless Podman은 사용자별 Runtime Directory를 사용합니다.

Jenkins 사용자의 UID를 확인합니다.

```bash
id jenkins
```

Lab 환경의 Jenkins UID:

```text
995
```

따라서 Runtime Directory는 다음과 같습니다.

```text
/run/user/995
```

Jenkins 사용자의 User Manager를 유지하기 위해 Linger를 활성화합니다.

```bash
loginctl enable-linger jenkins
systemctl start user@995.service
```

Rootless Podman 동작을 테스트합니다.

```bash
runuser -u jenkins -- env \
  XDG_RUNTIME_DIR=/run/user/995 \
  podman info
```

정상적으로 Podman 정보가 출력되면 Jenkins 사용자에서 Rootless Podman을 실행할 수 있습니다.

---

## 10. Configure XDG_RUNTIME_DIR for Jenkins

Jenkins Service에서 Podman을 실행할 때 올바른 Runtime Directory를 사용하도록 systemd Override를 생성합니다.

```text
/etc/systemd/system/jenkins.service.d/override.conf
```

내용:

```ini
[Service]
Environment="XDG_RUNTIME_DIR=/run/user/995"
```

설정을 반영합니다.

```bash
systemctl daemon-reload
systemctl restart jenkins
```

Jenkins Service 환경을 확인할 수 있습니다.

```bash
systemctl show jenkins --property=Environment
```

이제 Jenkins Pipeline에서는 별도의 `runuser` 없이 다음과 같이 Podman을 실행할 수 있습니다.

```groovy
sh 'podman info'
```

---

## 11. Root vs Rootless Podman Storage

Root Podman과 Jenkins 사용자의 Rootless Podman은 서로 다른 Container Storage를 사용합니다.

따라서 Root에서:

```bash
podman images
```

를 실행했을 때 Jenkins가 Build한 이미지가 보이지 않을 수 있습니다.

Jenkins 사용자 기준으로 확인해야 합니다.

```bash
runuser -u jenkins -- env \
  XDG_RUNTIME_DIR=/run/user/995 \
  podman images
```

관리 편의를 위해 Lab에서는 다음과 같은 Wrapper를 사용할 수도 있습니다.

```text
/usr/local/bin/jpodman
```

```bash
#!/bin/bash

exec runuser -u jenkins -- env \
  XDG_RUNTIME_DIR=/run/user/995 \
  podman "$@"
```

이 Wrapper는 관리자가 Jenkins 사용자의 Rootless Podman 상태를 확인하기 위한 편의용이며 Jenkins Pipeline 자체에서는 일반 `podman` 명령을 사용합니다.

---

# Harbor Configuration

Jenkins에서 생성한 이미지를 Private Harbor Registry에 Push할 수 있도록 구성합니다.

---

## 12. Harbor Name Resolution

Harbor:

```text
Hostname: harbor.lab.local
IP:       192.168.10.114
```

Jenkins VM에서 Harbor 이름을 해석할 수 있도록 Lab 환경에서는 `/etc/hosts`에 등록했습니다.

```text
192.168.10.114 harbor.lab.local
```

확인:

```bash
getent hosts harbor.lab.local
```

---

## 13. Harbor TLS Trust

Harbor는 HTTPS를 사용하며 Lab에서 구성한 Private CA Certificate를 사용합니다.

초기 Podman Login 과정에서 다음과 같은 인증서 오류가 발생했습니다.

```text
x509: certificate signed by unknown authority
```

Podman이 Harbor Certificate를 신뢰할 수 있도록 Registry별 CA Certificate를 구성했습니다.

```text
/etc/containers/certs.d/harbor.lab.local/ca.crt
```

구조:

```text
/etc/containers/certs.d/
└── harbor.lab.local/
    └── ca.crt
```

여기에는 Harbor Server Certificate 자체가 아니라 해당 Certificate를 검증할 수 있는 CA Certificate를 사용합니다.

설정 후 Jenkins 사용자의 Podman에서 Harbor Login 및 Push가 정상적으로 동작하는지 확인했습니다.

---

# Jenkins Credentials

Pipeline에 Password나 Token을 직접 작성하지 않기 위해 Jenkins Credentials를 사용합니다.

---

## 14. Harbor Credentials

Jenkins Web UI:

```text
Manage Jenkins
    ↓
Credentials
```

Harbor 인증 정보를 다음과 같이 등록했습니다.

```text
Kind:
Username with password

ID:
harbor-credentials

Username:
Harbor User

Password:
Harbor Password
```

Pipeline에서는 Credential ID를 이용합니다.

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

Harbor Password를 Jenkinsfile에 직접 저장하지 않습니다.

---

## 15. GitHub Credentials

Jenkins가 `server-inventory-k8s` Repository를 변경하고 Push하기 위해 GitHub Credential도 등록했습니다.

```text
Kind:
Username with password

ID:
github-credentials

Username:
GitHub Username

Password:
GitHub Personal Access Token
```

GitHub Personal Access Token에는 GitOps Repository를 변경할 수 있는 필요한 Repository 권한만 부여합니다.

현재 Pipeline에서는 이 Credential을 사용하여 `server-inventory-k8s` Repository를 Clone하고 변경 사항을 Push합니다.

Credential 사용 방식은 향후 더 안전한 Git Credential 방식으로 개선할 수 있습니다.

---

## 16. Jenkins Workspace

Pipeline이 실행되면 Jenkins는 Job별 Workspace를 사용합니다.

일반적인 위치:

```text
/var/lib/jenkins/workspace/<JOB_NAME>
```

예:

```text
/var/lib/jenkins/workspace/server-inventory-api
```

구조:

```text
GitHub
server-inventory-api
      │
      │ Checkout
      ▼
Jenkins Workspace
      │
      ├── main.py
      ├── requirements.txt
      ├── Dockerfile
      └── Jenkinsfile
```

Workspace는 Pipeline 작업을 수행하기 위한 Working Copy이며 Source of Truth는 Git Repository입니다.

---

## 17. Installation Verification

Jenkins Server에서 다음 항목을 확인합니다.

```bash
systemctl status jenkins
java -version
podman --version
```

Jenkins 사용자 Podman:

```bash
runuser -u jenkins -- env \
  XDG_RUNTIME_DIR=/run/user/995 \
  podman info
```

Harbor Name Resolution:

```bash
getent hosts harbor.lab.local
```

Jenkins Web UI:

```text
http://192.168.10.121:8080
```

다음 조건을 만족하면 CI Server 준비가 완료된 것입니다.

```text
Jenkins Service Running
        +
Jenkins Web UI Access
        +
Rootless Podman
        +
Harbor TLS Trust
        +
Harbor Credential
        +
GitHub Credential
```

---

## Troubleshooting

### Jenkins Repository Download Returned HTML

초기 구성에서 오래된 Repository URL을 사용했을 때 Jenkins Repository 파일 대신 HTTP Redirect HTML이 저장되는 문제가 발생했습니다.

Repository 파일이 정상적인 YUM Repository 형식인지 확인하고 현재 Jenkins LTS RPM Repository를 사용하여 해결했습니다.

---

### Podman Installation Conflict

기존 Docker CE의 `containerd.io`와 Rocky Linux의 `runc` Package가 충돌했습니다.

Jenkins VM에서 기존 Docker Runtime이 필요하지 않은 것을 확인한 후 충돌 Package를 제거하고 Podman을 설치했습니다.

---

### Wrong XDG_RUNTIME_DIR

다음과 같이 Jenkins 사용자로 Podman을 실행했을 때:

```bash
runuser -u jenkins -- podman info
```

Root Session의 Runtime Directory가 상속되어 Rootless Podman이 정상적으로 실행되지 않았습니다.

명시적으로:

```bash
XDG_RUNTIME_DIR=/run/user/995
```

를 지정하여 해결했고, 이후 Jenkins systemd Service에도 동일한 환경변수를 설정했습니다.

---

### Harbor Certificate Error

Podman에서 Harbor Login 시:

```text
x509: certificate signed by unknown authority
```

오류가 발생했습니다.

다음 Registry-specific Trust Store에 Harbor CA를 등록하여 해결했습니다.

```text
/etc/containers/certs.d/harbor.lab.local/ca.crt
```

---

## Next

Jenkins Server 준비가 완료되었습니다.

다음 단계에서는 실제 Pipeline을 구성합니다.

```text
GitHub
   ↓
Checkout
   ↓
Detect Changes
   ↓
Build Image
   ↓
Harbor Push
   ↓
GitOps Repository Update
```

다음 문서:

[04. CI Pipeline](04-ci-pipeline.md)