# 04. CI Pipeline

Jenkins Pipeline을 구성하여 애플리케이션 Source 변경 시 Container Image를 Build하고 Harbor에 Push한 뒤 GitOps Repository의 Image Tag를 자동으로 변경합니다.

전체 Pipeline 흐름:

```text
GitHub
server-inventory-api
        │
        ▼
     Checkout
        │
        ▼
  Detect Changes
        │
        ├── APP_CHANGED=false
        │         │
        │         └── Build / Push / GitOps Update SKIP
        │
        └── APP_CHANGED=true
                  │
                  ▼
             Build Image
                  │
                  ▼
              Harbor Push
                  │
                  ▼
        Update GitOps Repository
```

---

## 1. Pipeline as Code

Jenkins Job을 UI에서 직접 구성하는 대신 Pipeline을 `Jenkinsfile`로 관리합니다.

실제 Jenkinsfile은 Application Source Repository에 위치합니다.

```text
server-inventory-api/
├── main.py
├── requirements.txt
├── Dockerfile
└── Jenkinsfile
```

Jenkins Job에서는 다음 방식을 사용합니다.

```text
Definition:
Pipeline script from SCM

SCM:
Git

Branch:
*/main

Script Path:
Jenkinsfile
```

따라서 Jenkins는 Git Repository에서 Jenkinsfile을 읽어 Pipeline을 실행합니다.

```text
GitHub
   │
   ├── Application Source
   └── Jenkinsfile
          │
          ▼
       Jenkins
```

---

## 2. Pipeline Environment

Pipeline에서 반복적으로 사용하는 값을 Environment Variable로 정의합니다.

```groovy
environment {
    HARBOR_REGISTRY = 'harbor.lab.local'
    HARBOR_PROJECT = 'server-inventory'
    IMAGE_NAME = 'server-inventory-api'
    APP_CHANGED = 'false'
}
```

각 변수의 역할:

```text
HARBOR_REGISTRY
→ Harbor Registry 주소

HARBOR_PROJECT
→ Harbor Project

IMAGE_NAME
→ Container Image 이름

APP_CHANGED
→ Application 관련 파일 변경 여부
```

`APP_CHANGED`는 기본적으로 `false`입니다.

Application 코드 변경이 확인된 경우에만 `true`로 변경합니다.

---

## 3. Checkout

Pipeline 실행 시 Jenkins는 Source Repository를 Workspace에 Checkout합니다.

```groovy
stage('Checkout') {
    steps {
        sh 'echo "Git checkout successful"'
        sh 'whoami'
        sh 'pwd'
        sh 'ls -al'
    }
}
```

`Pipeline script from SCM`을 사용하는 경우 Jenkins가 Pipeline 실행 전에 Repository를 Checkout합니다.

이 Stage에서는 Checkout 결과와 실행 환경을 확인합니다.

예:

```text
whoami
→ jenkins

pwd
→ /var/lib/jenkins/workspace/server-inventory-api
```

Jenkins Workspace는 Source의 Working Copy이며 Source of Truth는 GitHub Repository입니다.

---

# Change Detection

## 4. Why Change Detection Is Needed

초기 Pipeline에서는 Git Commit이 발생할 때마다 Container Image를 생성했습니다.

따라서 다음과 같은 문서 변경도 Image Build로 이어졌습니다.

```text
README.md 변경
      ↓
Jenkins 실행
      ↓
Container Image Build
      ↓
Harbor Push
      ↓
GitOps Update
      ↓
Kubernetes Rollout
```

문서 변경은 Application Container에 영향을 주지 않기 때문에 불필요한 Build와 Deployment가 발생합니다.

이를 방지하기 위해 Application 관련 파일의 변경 여부를 먼저 판단합니다.

---

## 5. Detect Changes Stage

```groovy
stage('Detect Changes') {
    steps {
        script {
            def changedFiles = sh(
                script: 'git diff --name-only HEAD^ HEAD',
                returnStdout: true
            ).trim()

            echo "Changed files:\n${changedFiles}"

            if (changedFiles.split('\n').any {
                it == 'main.py' ||
                it == 'requirements.txt' ||
                it == 'Dockerfile'
            }) {
                env.APP_CHANGED = 'true'
            }

            echo "APP_CHANGED=${env.APP_CHANGED}"
        }
    }
}
```

---

## 6. Find Changed Files

핵심 명령:

```bash
git diff --name-only HEAD^ HEAD
```

의미:

```text
HEAD
→ 현재 Commit

HEAD^
→ 현재 Commit의 Parent Commit

--name-only
→ 변경된 파일 이름만 출력
```

예를 들어:

```text
main.py
README.md
```

가 변경되었다면 다음과 같이 출력됩니다.

```text
main.py
README.md
```

---

## 7. Capture Shell Output

```groovy
def changedFiles = sh(
    script: 'git diff --name-only HEAD^ HEAD',
    returnStdout: true
).trim()
```

`returnStdout: true`를 사용하면 Shell Command의 Standard Output을 Groovy 변수로 받을 수 있습니다.

개념적으로:

```text
Shell

git diff --name-only HEAD^ HEAD

        ↓ stdout

main.py
README.md

        ↓

Groovy

changedFiles
```

`.trim()`은 문자열 앞뒤의 불필요한 공백 및 개행을 제거합니다.

---

## 8. Determine Application Changes

```groovy
if (changedFiles.split('\n').any {
    it == 'main.py' ||
    it == 'requirements.txt' ||
    it == 'Dockerfile'
}) {
    env.APP_CHANGED = 'true'
}
```

먼저:

```groovy
changedFiles.split('\n')
```

으로 변경 파일 목록을 줄 단위로 분리합니다.

예:

```text
[
  "main.py",
  "README.md"
]
```

`any`는 목록 중 하나라도 조건을 만족하는지 확인합니다.

현재 Application Build에 영향을 주는 파일:

```text
main.py
requirements.txt
Dockerfile
```

하나라도 발견되면:

```text
APP_CHANGED=true
```

로 변경합니다.

---

## 9. Change Detection Example

README만 변경:

```text
Changed files:
README.md

APP_CHANGED=false
```

결과:

```text
Build Image                 SKIPPED
Push Image                  SKIPPED
Update GitOps Repository    SKIPPED
```

Application 변경:

```text
Changed files:
main.py

APP_CHANGED=true
```

결과:

```text
Build Image
     ↓
Push Image
     ↓
Update GitOps Repository
```

---

# Conditional Pipeline

## 10. when Condition

Application 변경이 있는 경우에만 Image 관련 Stage를 실행합니다.

```groovy
when {
    environment name: 'APP_CHANGED', value: 'true'
}
```

예:

```groovy
stage('Build Image') {
    when {
        environment name: 'APP_CHANGED', value: 'true'
    }

    steps {
        ...
    }
}
```

동작:

```text
APP_CHANGED=true
      ↓
Stage 실행


APP_CHANGED=false
      ↓
Stage SKIPPED
```

동일한 조건을 다음 Stage에 적용합니다.

```text
Build Image
Push Image
Update GitOps Repository
```

---

# Container Image Build

## 11. Build Image

Jenkins 사용자에서 Rootless Podman을 사용하여 Container Image를 Build합니다.

```groovy
stage('Build Image') {
    when {
        environment name: 'APP_CHANGED', value: 'true'
    }

    steps {
        sh '''
            podman build \
                -t ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${BUILD_NUMBER} .
        '''
    }
}
```

생성되는 Image:

```text
harbor.lab.local/server-inventory/server-inventory-api:<BUILD_NUMBER>
```

예:

```text
harbor.lab.local/server-inventory/server-inventory-api:15
```

현재는 Jenkins `BUILD_NUMBER`를 Image Tag로 사용합니다.

---

# Harbor Push

## 12. Harbor Credentials

Harbor Username과 Password는 Jenkins Credentials에 저장합니다.

Credential ID:

```text
harbor-credentials
```

Pipeline에서는 `withCredentials`를 사용합니다.

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

Pipeline 실행 중 Credential 값이 Environment Variable로 전달됩니다.

```text
HARBOR_USER
HARBOR_PASSWORD
```

---

## 13. Harbor Login

```bash
printf '%s' "$HARBOR_PASSWORD" | \
    podman login ${HARBOR_REGISTRY} \
    --username "$HARBOR_USER" \
    --password-stdin
```

Password를 Command Argument로 직접 전달하지 않고 Standard Input으로 전달합니다.

```text
Jenkins Credential
       │
       ▼
HARBOR_PASSWORD
       │
       │ stdin
       ▼
podman login
```

---

## 14. Push Image

Login 후 Image를 Harbor로 Push합니다.

```bash
podman push \
    ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${BUILD_NUMBER}
```

전체 Stage:

```groovy
stage('Push Image') {
    when {
        environment name: 'APP_CHANGED', value: 'true'
    }

    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'harbor-credentials',
                usernameVariable: 'HARBOR_USER',
                passwordVariable: 'HARBOR_PASSWORD'
            )
        ]) {
            sh '''
                printf '%s' "$HARBOR_PASSWORD" | \
                    podman login ${HARBOR_REGISTRY} \
                    --username "$HARBOR_USER" \
                    --password-stdin

                podman push \
                    ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${BUILD_NUMBER}
            '''
        }
    }
}
```

---

# GitOps Repository Update

## 15. Why Jenkins Updates Git

Harbor에 새로운 Image가 생성되어도 Kubernetes Deployment가 자동으로 새로운 Image를 사용하는 것은 아닙니다.

예:

```text
Harbor

server-inventory-api:15
```

하지만 GitOps Repository가:

```yaml
image: harbor.lab.local/server-inventory/server-inventory-api:14
```

라면 Argo CD의 Desired State는 여전히 `:14`입니다.

따라서 Jenkins가 GitOps Repository의 Image Tag를 변경합니다.

```text
Harbor
Image :15
    │
    ▼
Jenkins
    │
    ▼
server-inventory-k8s

image: ...:14
       ↓
image: ...:15
```

---

## 16. GitHub Credentials

GitOps Repository에 Push하기 위한 GitHub Credential:

```text
Credential ID:
github-credentials
```

Pipeline:

```groovy
withCredentials([
    usernamePassword(
        credentialsId: 'github-credentials',
        usernameVariable: 'GITHUB_USER',
        passwordVariable: 'GITHUB_TOKEN'
    )
]) {
    ...
}
```

---

## 17. Clone GitOps Repository

기존 Working Directory가 남아 있을 수 있으므로 먼저 제거합니다.

```bash
rm -rf gitops
```

GitOps Repository를 `gitops` Directory에 Clone합니다.

```bash
git clone \
    "https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/mattang-e/server-inventory-k8s.git" \
    gitops
```

구조:

```text
Jenkins Workspace
│
├── server-inventory-api Source
│
└── gitops/
     └── server-inventory-k8s
```

> 현재 Lab에서는 Pipeline 구현을 단순화하기 위해 Credential을 Clone URL에 전달합니다. Credential 노출 가능성을 줄이기 위해 향후 Jenkins Git Credential Binding 또는 별도의 인증 방식을 적용할 수 있습니다.

---

## 18. Update Image Tag

GitOps Repository로 이동합니다.

```bash
cd gitops
```

Deployment Manifest의 Image Tag를 변경합니다.

```bash
sed -i \
  "s|image: harbor.lab.local/server-inventory/server-inventory-api:.*|image: harbor.lab.local/server-inventory/server-inventory-api:${BUILD_NUMBER}|" \
  api/deployment.yaml
```

예:

```text
Before

image: harbor.lab.local/server-inventory/server-inventory-api:14


After

image: harbor.lab.local/server-inventory/server-inventory-api:15
```

---

## 19. Commit Identity

Jenkins가 Git Commit을 생성하기 위해 Commit Author를 설정합니다.

```bash
git config user.name "jenkins"
git config user.email "jenkins@lab.local"
```

이 설정은 GitHub 인증 정보가 아닙니다.

```text
git config user.name / user.email
→ Commit 작성자 정보

GitHub PAT
→ GitHub Push 인증
```

두 역할은 서로 다릅니다.

---

## 20. Commit and Push

변경된 Manifest를 Commit합니다.

```bash
git add api/deployment.yaml

git commit \
  -m "Update server-inventory-api image to ${BUILD_NUMBER}"
```

GitHub에 Push합니다.

```bash
git push origin main
```

이 시점부터 GitOps Repository의 Desired State가 변경됩니다.

```text
Jenkins
   ↓
Git Push
   ↓
server-inventory-k8s
   ↓
Argo CD
```

---

## 21. Update GitOps Repository Stage

전체 Stage:

```groovy
stage('Update GitOps Repository') {
    when {
        environment name: 'APP_CHANGED', value: 'true'
    }

    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'github-credentials',
                usernameVariable: 'GITHUB_USER',
                passwordVariable: 'GITHUB_TOKEN'
            )
        ]) {
            sh '''
                rm -rf gitops

                git clone \
                    "https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/mattang-e/server-inventory-k8s.git" \
                    gitops

                cd gitops

                sed -i \
                    "s|image: harbor.lab.local/server-inventory/server-inventory-api:.*|image: harbor.lab.local/server-inventory/server-inventory-api:${BUILD_NUMBER}|" \
                    api/deployment.yaml

                git config user.name "jenkins"
                git config user.email "jenkins@lab.local"

                git add api/deployment.yaml
                git commit \
                    -m "Update server-inventory-api image to ${BUILD_NUMBER}"

                git push origin main
            '''
        }
    }
}
```

---

# Poll SCM

## 22. Automatic Pipeline Trigger

초기 테스트에서는 Jenkins Web UI의 `Build Now`를 사용했습니다.

최종 Lab 구성에서는 Source 변경 시 자동으로 Pipeline을 실행하기 위해 Poll SCM을 사용합니다.

Jenkins Job:

```text
Configure
    ↓
Build Triggers
    ↓
Poll SCM
```

Schedule:

```text
H/2 * * * *
```

Jenkins가 SCM 변경을 주기적으로 확인합니다.

중요한 점은 Poll SCM이 단순히 2분마다 Build를 실행하는 것이 아니라는 것입니다.

```text
Poll
  ↓
Git 변경 없음
  ↓
Build 실행 안 함
```

```text
Poll
  ↓
Git 변경 발견
  ↓
Pipeline 실행
```

---

## 23. Why Poll SCM Instead of GitHub Webhook

Jenkins는 Private Lab Network에 있습니다.

```text
Jenkins
192.168.10.121:8080
```

GitHub.com에서는 RFC1918 Private IP인 `192.168.10.121`에 직접 접근할 수 없습니다.

따라서 현재 Lab에서는:

```text
GitHub Webhook
     X
     │
Private Jenkins

대신

Jenkins
   │
   └── Poll GitHub
```

구조를 사용합니다.

외부에서 접근 가능한 HTTPS Endpoint가 구성된다면 향후 Webhook 방식으로 변경할 수 있습니다.

---

# Current Pipeline

## 24. Pipeline Flow

현재 Pipeline의 최종 흐름:

```text
Developer
    │
    │ git push
    ▼
server-inventory-api
    │
    │ Poll SCM
    ▼
Jenkins Pipeline
    │
    ├── Checkout
    │
    └── Detect Changes
            │
            ├── APP_CHANGED=false
            │        │
            │        ├── Build Image       SKIPPED
            │        ├── Push Image        SKIPPED
            │        └── GitOps Update     SKIPPED
            │
            └── APP_CHANGED=true
                     │
                     ▼
                Build Image
                     │
                     ▼
                 Harbor Push
                     │
                     ▼
             GitOps Repository
                     │
                     ▼
                   Argo CD
```

---

## 25. Current Image Tag Strategy

현재 Image Tag는:

```text
BUILD_NUMBER
```

를 사용합니다.

이 값은 Container Image Build 횟수가 아니라 Jenkins Job 실행 횟수입니다.

따라서 다음과 같은 상황이 가능합니다.

```text
Build #1
main.py 변경
→ Image :1

Build #2 ~ #10
README / Jenkinsfile 변경
→ Image Build SKIPPED

Build #11
main.py 변경
→ Image :11
```

결과적으로 Harbor에는:

```text
:1
:11
```

처럼 Tag 번호가 연속적이지 않을 수 있습니다.

Container Image 동작에는 문제가 없지만 Source와 Image 간 추적성을 개선하기 위해 향후 Git Commit SHA 기반 Tag를 적용할 수 있습니다.

예:

```text
server-inventory-api:a84f21c
```

---

## 26. Current Change Detection Limitation

현재:

```bash
git diff --name-only HEAD^ HEAD
```

을 사용하므로 현재 Commit과 바로 이전 Commit만 비교합니다.

여러 Commit이 Jenkins 실행 사이에 누적된 경우 모든 변경을 완벽하게 표현하지 못할 수 있습니다.

현재 Lab에서는 Pipeline의 조건부 실행 원리를 구현하기 위한 방식으로 사용하고 있으며 향후 Jenkins SCM Changelog 또는 이전 Build Revision 기준 비교 방식으로 개선할 수 있습니다.

---

## Verification

### Documentation-only Change

예:

```text
README.md
```

Jenkins Console:

```text
Changed files:
README.md

APP_CHANGED=false
```

예상 결과:

```text
Build Image                 SKIPPED
Push Image                  SKIPPED
Update GitOps Repository    SKIPPED
```

### Application Change

예:

```text
main.py
```

예상:

```text
APP_CHANGED=true
```

Pipeline:

```text
Build Image
    ↓
Harbor Push
    ↓
GitOps Update
```

Harbor에서 새로운 Image Tag가 생성되고 `server-inventory-k8s/api/deployment.yaml`의 Image Tag가 변경되는지 확인합니다.

---

## Next

CI Pipeline 구성이 완료되었습니다.

마지막 단계에서는 Jenkins와 Argo CD를 연결한 전체 End-to-End CI/CD 흐름을 정리하고 실제 자동 배포 결과를 검증합니다.

```text
Developer
   ↓
GitHub
   ↓
Jenkins
   ↓
Harbor
   ↓
GitOps Repository
   ↓
Argo CD
   ↓
Kubernetes
```

다음 문서:

[05. CI/CD Integration](05-cicd-integration.md)