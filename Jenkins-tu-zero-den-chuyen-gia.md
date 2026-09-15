# 🔧 Tài liệu Jenkins: Từ Zero đến Chuyên gia

> **Mục tiêu**: Xây dựng hệ thống CI/CD production với Jenkins, bao gồm pipelines, plugins, security, scaling.

---

## 📑 Mục lục

| Phần | Nội dung |
|------|----------|
| [P0. Chuẩn bị](#p0) | Cài Jenkins, plugins |
| [P1. Freestyle Job](#p1) | Job cơ bản, build trigger |
| [P2. Pipeline Scripted](#p2) | Groovy DSL |
| [P3. Pipeline Declarative](#p3) | Jenkinsfile chuẩn |
| [P4. Shared Library](#p4) | Tái sử dụng code |
| [P5. Agents & Scaling](#p5) | Distributed builds |
| [P6. Security](#p6) | RBAC, credentials, secrets |
| [P7. K8s Integration](#p7) | Jenkins trên K8s |
| [P8. Best Practice](#p8) | Backup, monitoring, performance |
| [P9. Migration](#p9) | Từ freestyle sang pipeline |

---

<a id="p0"></a>
## P0. Chuẩn bị

### Bước 1: Cài Jenkins

```bash
# Docker (khuyến nghị)
docker run -d \
  --name jenkins \
  --restart=on-failure \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts

# Lấy initial password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

# Linux
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update && sudo apt-get install jenkins

# macOS
brew install jenkins-lts
brew services start jenkins-lts
```

### Bước 2: Setup ban đầu

```
1. Truy cập http://localhost:8080
2. Nhập initial password
3. Cài plugins (suggested hoặc custom)
4. Tạo admin user
5. Cấu hình Jenkins URL: http://jenkins.example.com:8080
```

### Bước 3: Cấu trúc Jenkins

```bash
# Jenkins Home
$JENKINS_HOME = /var/jenkins_home
├── jobs/                  # Job configs
│   └── my-job/
│       ├── config.xml     # Job definition
│       ├── builds/        # Build history
│       └── workspace/     # Working directory
├── plugins/               # Plugins
├── users/                 # User accounts
├── credentials.xml        # Encrypted credentials
├── secrets/               # Secret files
│   └── master.key         # Encryption key
├── nodes/                 # Agent configs
└── jenkins.xml            # Main config
```

### Bước 4: Jenkins CLI

```bash
# Download jenkins-cli.jar
wget http://localhost:8080/jnlpJars/jenkins-cli.jar

# Sử dụng
java -jar jenkins-cli.jar -s http://localhost:8080/ \
  -auth admin:apitoken who-am-i

# Common commands
java -jar jenkins-cli.jar list-jobs
java -jar jenkins-cli.jar build my-job
java -jar jenkins-cli.jar console my-job
java -jar jenkins-cli.jar install-plugin git
java -jar jenkins-cli.jar safe-restart
```

### Bước 5: Plugins cần thiết

```
# Essentials
- Git
- Pipeline
- Blue Ocean
- Credentials
- Workspace Cleanup
- Timestamper
- AnsiColor

# CI/CD
- Docker Pipeline
- Kubernetes CLI
- Ansible
- SSH Agent
- HTTP Request

# Security
- Role-based Authorization Strategy
- Credentials Binding
- OWASP Dependency-Check
- SonarQube Scanner

# Notification
- Slack Notification
- Email Extension
- Jira

# Build tools
- Maven
- NodeJS
- Gradle
- Python
```

---

<a id="p1"></a>
## P1. Freestyle Job

### Bước 1: Tạo Job

```
Dashboard > New Item > Freestyle project
- Name: my-first-job
- Description: My first Jenkins job
```

### Bước 2: Cấu hình Source Code

```
Source Code Management > Git
- Repository URL: https://github.com/myorg/myapp.git
- Credentials: github-token
- Branch Specifier: */main
```

### Bước 3: Build Triggers

```
Build Triggers:
☐ Trigger builds remotely (e.g., from scripts)
  - Authentication Token: my-secret-token

☐ Build after other projects are built
  - Projects to watch: upstream-job

☐ Build periodically
  - Schedule: H 2 * * *     # H = hash (spread load)

☐ GitHub hook trigger for GITScm polling
☐ Poll SCM
  - Schedule: H/15 * * * *  # Mỗi 15 phút
```

### Bước 4: Build Steps

```bash
# Execute shell
#!/bin/bash
set -e
echo "Building..."
npm install
npm run build
npm test
```

```bash
# Invoke Ant/Maven/Gradle
# - Task: clean install

# Invoke top-level Maven targets
Maven Version: Maven 3.9
Goals: clean package
```

### Bước 5: Post-build Actions

```
Post-build Actions:
- Archive the artifacts
  - Files to archive: dist/*.jar, build/output/**
- Publish JUnit test result report
  - Test report XMLs: **/test-results/**/*.xml
- Send build artifacts over SSH
- Trigger parameterized build on other projects
- Git Publisher
- Email notification
```

---

<a id="p2"></a>
## P2. Pipeline Scripted

### Bước 1: Scripted Pipeline cơ bản

```groovy
// Jenkinsfile (Scripted)
node('agent') {
    stage('Checkout') {
        git url: 'https://github.com/myorg/myapp.git',
            branch: 'main',
            credentialsId: 'github-token'
    }

    stage('Build') {
        sh 'npm install'
        sh 'npm run build'
    }

    stage('Test') {
        sh 'npm test'
        junit '**/test-results/**/*.xml'
    }

    stage('Deploy') {
        if (env.BRANCH_NAME == 'main') {
            sh './deploy.sh production'
        } else if (env.BRANCH_NAME == 'develop') {
            sh './deploy.sh staging'
        }
    }
}
```

### Bước 2: Groovy DSL cơ bản

```groovy
// Variables
def name = "World"
def version = "1.0.0"

// String interpolation
echo "Hello, ${name}!"
echo "Version: ${version}"

// Lists
def fruits = ['apple', 'banana', 'cherry']
echo fruits[0]
echo fruits.size()

// Maps
def config = [
    name: 'myapp',
    port: 8080,
    debug: false
]
echo config.name

// Closure
def greet = { name -> "Hello, ${name}!" }
echo greet('Alice')

// Conditionals
if (env.BRANCH_NAME == 'main') {
    echo 'Production build'
} else {
    echo 'Non-production build'
}

// Loops
for (int i = 0; i < 5; i++) {
    echo "Iteration ${i}"
}

['apple', 'banana'].each { fruit ->
    echo "Fruit: ${fruit}"
}

// Try-catch
try {
    sh 'risky-command'
} catch (Exception e) {
    echo "Failed: ${e.message}"
    currentBuild.result = 'UNSTABLE'
}
```

### Bước 3: Pipeline nâng cao

```groovy
// Parallel stages
stage('Parallel Tests') {
    parallel(
        'unit': {
            sh 'npm run test:unit'
        },
        'integration': {
            sh 'npm run test:integration'
        },
        'e2e': {
            sh 'npm run test:e2e'
        },
        failFast: true    # Dừng nếu 1 fail
    )
}

// Try-finally cleanup
node {
    try {
        stage('Build') {
            sh 'make build'
        }
        stage('Test') {
            sh 'make test'
        }
    } finally {
        stage('Cleanup') {
            sh 'make clean'
        }
    }
}

// Properties
properties([
    parameters([
        string(name: 'VERSION', defaultValue: '1.0.0', description: 'Version'),
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod']),
        booleanParam(name: 'DEPLOY', defaultValue: false)
    ]),
    buildDiscarder(logRotator(numToKeepStr: '30')),
    pipelineTriggers([
        githubPush(),
        pollSCM('H/15 * * * *')
    ])
])
```

### Bước 4: Environment & secrets

```groovy
node {
    // Environment variables
    env.BUILD_NUMBER = env.BUILD_NUMBER
    env.JOB_NAME = env.JOB_NAME
    env.WORKSPACE = pwd()

    // Set custom env
    env.APP_VERSION = '1.2.3'
    env.DEPLOY_ENV = 'production'

    // Use credentials
    withCredentials([string(credentialsId: 'api-key', variable: 'API_KEY')]) {
        sh 'curl -H "Authorization: Bearer $API_KEY" https://api.example.com'
    }

    // Username/password
    withCredentials([usernamePassword(
        credentialsId: 'docker-registry',
        usernameVariable: 'USERNAME',
        passwordVariable: 'PASSWORD'
    )]) {
        sh 'docker login -u $USERNAME -p $PASSWORD'
    }

    // File
    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
        sh 'kubectl apply -f deployment.yaml'
    }
}
```

### Bước 5: Post actions

```groovy
node {
    try {
        // Stages
    } catch (Exception e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        // Always run
        cleanWs()
    }
}

// More granular post
stage('Build') {
    steps {
        sh 'make'
    }
    post {
        always {
            echo 'Always run'
        }
        success {
            echo 'Build success'
            archiveArtifacts artifacts: '**/target/*.jar'
        }
        failure {
            echo 'Build failed'
            mail to: '[email protected]',
                 subject: "Build Failed: ${env.JOB_NAME}",
                 body: "Check ${env.BUILD_URL}"
        }
        unstable {
            echo 'Build unstable'
        }
        changed {
            echo 'Build status changed'
        }
    }
}
```

---

<a id="p3"></a>
## P3. Pipeline Declarative (Jenkinsfile chuẩn)

### Bước 1: Structure cơ bản

```groovy
// Jenkinsfile (Declarative)
pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        ansiColor('xterm')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        disableConcurrentBuilds()
        skipDefaultCheckout()
    }

    parameters {
        string(name: 'VERSION', defaultValue: '1.0.0', description: 'Version to deploy')
        choice(name: 'ENV', choices: ['dev', 'staging', 'prod'])
        booleanParam(name: 'DEPLOY', defaultValue: false)
    }

    triggers {
        pollSCM('H/15 * * * *')
        githubPush()
    }

    environment {
        APP_NAME = 'myapp'
        VERSION = "${params.VERSION}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/myorg/myapp.git',
                        credentialsId: 'github-token'
                    ]]
                )
            }
        }

        stage('Build') {
            steps {
                sh 'npm ci'
                sh 'npm run build'
            }
        }

        stage('Test') {
            parallel {
                stage('Unit') {
                    steps {
                        sh 'npm run test:unit'
                    }
                    post {
                        always {
                            junit '**/test-results/unit/*.xml'
                        }
                    }
                }
                stage('Integration') {
                    steps {
                        sh 'npm run test:integration'
                    }
                    post {
                        always {
                            junit '**/test-results/integration/*.xml'
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                allOf {
                    branch 'main'
                    expression { params.DEPLOY == true }
                }
            }
            steps {
                sh "./deploy.sh ${params.ENV}"
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            slackSend(channel: '#deploys',
                      color: 'good',
                      message: "Build ${env.JOB_NAME} #${env.BUILD_NUMBER} succeeded")
        }
        failure {
            slackSend(channel: '#alerts',
                      color: 'danger',
                      message: "Build ${env.JOB_NAME} #${env.BUILD_NUMBER} failed")
        }
    }
}
```

### Bước 2: Stages & Steps

```groovy
pipeline {
    agent any

    stages {
        // Build Docker image
        stage('Build Image') {
            steps {
                script {
                    def image = docker.build("myregistry/myapp:${env.BUILD_NUMBER}")
                    docker.withRegistry('https://myregistry.com', 'registry-creds') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }

        // Deploy to K8s
        stage('Deploy') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        sh '''
                            kubectl set image deployment/myapp \
                                myapp=myregistry/myapp:${BUILD_NUMBER} \
                                -n production
                            kubectl rollout status deployment/myapp -n production
                        '''
                    }
                }
            }
        }

        // Run ansible
        stage('Ansible') {
            steps {
                ansiblePlaybook(
                    playbook: 'deploy.yml',
                    inventory: 'inventory/production',
                    credentialsId: 'ssh-key'
                )
            }
        }

        // Send notification
        stage('Notify') {
            steps {
                mail to: '[email protected]',
                     subject: "Build ${env.BUILD_NUMBER}",
                     body: "Deployed successfully"
            }
        }
    }
}
```

### Bước 3: When conditions

```groovy
stage('Deploy Prod') {
    when {
        allOf {
            branch 'main'
            environment name: 'DEPLOY_PROD', value: 'true'
        }
    }
    steps {
        echo 'Deploying to production'
    }
}

stage('Deploy Staging') {
    when {
        anyOf {
            branch 'develop'
            branch 'feature/*'
        }
    }
    steps {
        echo 'Deploying to staging'
    }
}

stage('Nightly Tests') {
    when {
        triggeredBy 'TimerTrigger'
    }
    steps {
        sh 'npm run test:nightly'
    }
}

stage('Tag Build') {
    when {
        buildingTag()
    }
    steps {
        echo "Tagging: ${TAG_NAME}"
    }
}
```

### Bước 4: Matrix builds

```groovy
pipeline {
    agent none

    stages {
        stage('Test Matrix') {
            matrix {
                agent any
                axes {
                    axis {
                        name 'OS'
                        values 'linux', 'macos', 'windows'
                    }
                    axis {
                        name 'NODE_VERSION'
                        values '18', '20', '22'
                    }
                }
                excludes {
                    exclude {
                        axis {
                            name 'OS'
                            values 'windows'
                        }
                        axis {
                            name 'NODE_VERSION'
                            values '18'
                        }
                    }
                }
                stages {
                    stage('Build') {
                        steps {
                            echo "Building on ${OS} with Node ${NODE_VERSION}"
                            sh 'npm install'
                            sh 'npm run build'
                        }
                    }
                    stage('Test') {
                        steps {
                            sh 'npm test'
                        }
                    }
                }
            }
        }
    }
}
```

### Bước 5: Multi-branch pipeline

```groovy
// Jenkinsfile (root) - Multi-branch
pipeline {
    agent any

    options {
        disableConcurrentBuilds()
    }

    stages {
        stage('Detect Branch') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        env.DEPLOY_ENV = 'production'
                    } else if (env.BRANCH_NAME.startsWith('release/')) {
                        env.DEPLOY_ENV = 'staging'
                    } else {
                        env.DEPLOY_ENV = 'dev'
                    }
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm ci && npm run build'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test'
            }
        }

        stage('Deploy') {
            when {
                not { branch 'PR-*' }
            }
            steps {
                sh "./deploy.sh ${env.DEPLOY_ENV}"
            }
        }
    }
}
```

---

<a id="p4"></a>
## P4. Shared Library

### Bước 1: Cấu trúc

```bash
# jenkins-shared-library/ (Git repo)
├── src/                  # Groovy source
│   └── org/
│       └── mycompany/
│           └── Pipeline.groovy
├── vars/                 # Global variables (functions)
│   ├── buildDocker.groovy
│   ├── deployK8s.groovy
│   └── sendSlack.groovy
├── resources/            # Static resources
└── README.md
```

### Bước 2: vars/buildDocker.groovy

```groovy
#!/usr/bin/env groovy
// vars/buildDocker.groovy

def call(Map config = [:]) {
    def imageName = config.image ?: error("image is required")
    def tag = config.tag ?: env.BUILD_NUMBER
    def dockerfile = config.dockerfile ?: 'Dockerfile'
    def context = config.context ?: '.'
    def registry = config.registry ?: 'docker.io'
    def credentials = config.credentials ?: 'docker-registry'

    echo "Building ${imageName}:${tag}"

    def image = docker.build("${imageName}:${tag}",
        "-f ${dockerfile} ${context}")

    if (config.push != false) {
        docker.withRegistry("https://${registry}", credentials) {
            image.push()
            if (config.latest != false) {
                image.push('latest')
            }
        }
    }

    return image
}
```

### BƯớc 3: Dùng trong Jenkinsfile

```groovy
// Jenkinsfile
@Library('jenkins-shared-library@main') _

pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                buildDocker(
                    image: 'myregistry/myapp',
                    tag: env.BUILD_NUMBER,
                    credentials: 'myregistry-creds'
                )
            }
        }

        stage('Deploy') {
            steps {
                deployK8s(
                    deployment: 'myapp',
                    namespace: 'production',
                    image: "myregistry/myapp:${env.BUILD_NUMBER}"
                )
                sendSlack(
                    channel: '#deploys',
                    message: "Deployed myapp:${env.BUILD_NUMBER}"
                )
            }
        }
    }
}
```

### Bước 4: Global config

```groovy
// jenkins.yaml (cấu hình shared lib trong Jenkins)
/var/jenkins_home/jenkins.yaml

jenkins:
  globalLibraries:
    libraries:
      - name: "jenkins-shared-library"
        defaultVersion: "main"
        implicit: false
        allowVersionOverride: true
        includeInChangesets: true
        retriever:
          modernSCM:
            scm:
              git:
                remote: "https://github.com/myorg/jenkins-shared-library.git"
                credentialsId: "github-token"
```

### Bước 5: Steps classes

```groovy
// src/org/mycompany/Build.groovy
package org.mycompany

class Build implements Serializable {
    private static final long serialVersionUID = 1L

    def script
    String name

    Build(script, name) {
        this.script = script
        this.name = name
    }

    def mvn(args) {
        script.sh "./mvnw ${args}"
    }

    def test() {
        script.sh './mvnw test'
        script.junit '**/target/surefire-reports/*.xml'
    }
}

// Dùng
import org.mycompany.Build
def build = new Build(this, 'myapp')
build.test()
```

---

<a id="p5"></a>
## P5. Agents & Scaling

### Bước 1: Static agent

```
Manage Jenkins > Nodes > New Node
- Permanent Agent
- Name: build-agent-1
- Number of executors: 2
- Remote root directory: /home/jenkins/agent
- Labels: linux docker
- Usage: Use this node as much as possible
- Launch method: Launch agents via SSH
  - Host: 192.168.1.100
  - Credentials: ssh-key
```

### Bước 2: Docker agent (ephemeral)

```groovy
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '-v /tmp:/tmp'
            label 'docker-host'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'npm ci && npm run build'
            }
        }
    }
}
```

### BƯớc 3: Docker agent với multiple services

```groovy
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-21'
            args '-v $HOME/.m2:/root/.m2'
        }
    }

    stages {
        stage('Test') {
            steps {
                script {
                    docker.image('postgres:16').withRun('-e POSTGRES_PASSWORD=test') { c ->
                        docker.image('redis:7').withRun() { r ->
                            sh '''
                                sleep 5
                                mvn test
                            '''
                        }
                    }
                }
            }
        }
    }
}
```

### Bước 4: K8s agent (Jenkins on K8s)

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: agent
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest
    resources:
      requests:
        cpu: 500m
        memory: 1Gi
      limits:
        cpu: 1
        memory: 2Gi
  - name: docker
    image: docker:24-dind
    securityContext:
      privileged: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["/bin/sh"]
    args: ["-c", "sleep infinity"]
'''
        }
    }

    stages {
        stage('Build') {
            steps {
                container('docker') {
                    sh 'docker build -t myapp:${BUILD_NUMBER} .'
                }
            }
        }
        stage('Deploy') {
            steps {
                container('kubectl') {
                    sh 'kubectl apply -f deployment.yaml'
                }
            }
        }
    }
}
```

### Bước 5: Agent scaling với K8s plugin

```yaml
# JCasC (Configuration as Code)
jenkins:
  clouds:
    - kubernetes:
        name: "k8s-cloud"
        serverUrl: "https://kubernetes.default.svc"
        namespace: "jenkins"
        jenkinsUrl: "http://jenkins.jenkins.svc:8080"
        jenkinsTunnel: "jenkins-jnlp.jenkins.svc:50000"
        templates:
          - name: "default-agent"
            label: "k8s"
            namespace: "jenkins"
            nodeUsageMode: NORMAL
            serviceAccount: "jenkins-agent"
            slaveConnectTimeout: 100
            idleMinutes: 1
            instanceCap: 50
            containers:
              - name: "jnlp"
                image: "jenkins/inbound-agent:latest"
                resourceRequestCpu: "500m"
                resourceRequestMemory: "1Gi"
                resourceLimitCpu: "1"
                resourceLimitMemory: "2Gi"
```

---

<a id="p6"></a>
## P6. Security

### Bước 1: Security realm

```
Manage Jenkins > Security:
- Security Realm: Jenkins' own user database (hoặc LDAP/AD)
- Authorization: Role-Based Strategy (khuyến nghị)
```

### Bước 2: Role-Based Strategy

```
Manage Jenkins > Manage and Assign Roles > Manage Roles:
- Global roles:
  - admin: full permissions
  - developer: read, build, configure jobs
  - viewer: read only

- Project roles:
  - dev-*: match dev/* projects
    Permissions: build, read, workspace
  - prod-*:
    Permissions: build, read, configure
```

### BƯớc 3: Credentials

```groovy
// Trong Jenkinsfile
withCredentials([
    string(credentialsId: 'my-secret', variable: 'SECRET'),
    usernamePassword(
        credentialsId: 'docker-creds',
        usernameVariable: 'USER',
        passwordVariable: 'PASS'
    ),
    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
    sshUserPrivateKey(
        credentialsId: 'ssh-key',
        keyFileVariable: 'SSH_KEY',
        usernameVariable: 'SSH_USER'
    )
]) {
    // Use credentials
}
```

### Bước 4: Credentials trong JCasC

```yaml
jenkins:
  credentials:
    system:
      domainCredentials:
        - credentials:
            - usernamePassword:
                scope: GLOBAL
                id: "github-token"
                username: "myuser"
                password: "${GITHUB_TOKEN}"
                description: "GitHub access token"

            - string:
                scope: GLOBAL
                id: "api-key"
                secret: "${API_KEY}"
                description: "Production API key"

            - file:
                scope: GLOBAL
                id: "kubeconfig"
                fileName: "kubeconfig"
                secretBytes:
                  base64: "${KUBECONFIG_BASE64}"
```

### Bước 5: Disable security risks

```groovy
// Script Console (Manage Jenkins > Script Console)
Jenkins.instance.setCrumbIssuer(false)  // NOT recommended for prod
Jenkins.instance.setAgentProtocols(['JNLP4-connect'])

# Disable old protocols
System.setProperty('jenkins.slave.JnlpAgentEndpointProtocols.disabled', 'true')

# Remoting
Manage Jenkins > Security > Agent > Agent protocols:
- Disable JNLP1-connect, JNLP2-connect, JNLP3-connect
- Keep JNLP4-connect

# CSRF
Manage Jenkins > Security > CSRF Protection:
- Enable CSRF (recommended)
- Default crumb issuer
```

### Bước 6: Backup & restore

```bash
# Backup script
#!/bin/bash
JENKINS_HOME=/var/jenkins_home
BACKUP_DIR=/var/backups/jenkins
DATE=$(date +%Y%m%d-%H%M%S)

tar czf "$BACKUP_DIR/jenkins-backup-$DATE.tar.gz" \
  --exclude="$JENKINS_HOME/war" \
  --exclude="$JENKINS_HOME/workspace" \
  --exclude="$JENKINS_HOME/caches" \
  -C "$JENKINS_HOME" .

# Cron
0 2 * * * /opt/backup-jenkins.sh

# Restore
systemctl stop jenkins
mv /var/jenkins_home /var/jenkins_home.old
tar xzf jenkins-backup.tar.gz -C /var/jenkins_home
systemctl start jenkins
```

---

<a id="p7"></a>
## P7. Jenkins trên K8s

### Bước 1: Cài Jenkins bằng Helm

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update

helm install jenkins jenkins/jenkins \
  --namespace jenkins --create-namespace \
  --set controller.serviceType=NodePort \
  --set controller.nodePort=30808 \
  --set controller.adminPassword=admin123 \
  --set persistence.size=20Gi \
  --set agent.enabled=false
```

### Bước 2: ServiceAccount cho agents

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-agent
  namespace: jenkins

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-agent
subjects:
- kind: ServiceAccount
  name: jenkins-agent
  namespace: jenkins
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

### Bước 3: Jenkins Configuration as Code

```yaml
# jenkins.yaml
jenkins:
  systemMessage: "Jenkins managed by JCasC"
  numExecutors: 0
  mode: EXCLUSIVE
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "${JENKINS_ADMIN_PASSWORD}"
  authorizationStrategy:
    loggedInUsersCanDoAnything:
      allowAnonymousRead: false

unclassified:
  location:
    url: "https://jenkins.example.com/"
  scmGit:
    userRemoteConfigs:
      - url: "https://github.com/myorg/jenkins-shared-library.git"
        credentialsId: "github-token"

tool:
  git:
    installations:
      - name: "Default"
        home: "git"
  docker:
    installations:
      - name: "docker"
        home: "/usr/bin/docker"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "github-token"
              username: "myuser"
              password: "${GITHUB_TOKEN}"
```

```bash
# Mount vào pod
helm install jenkins jenkins/jenkins \
  --set controller.JCasC.configMapName=jenkins-config

# Tạo configmap
kubectl create configmap jenkins-config --from-file=jenkins.yaml -n jenkins
```

### Bước 4: Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jenkins
  namespace: jenkins
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
spec:
  ingressClassName: nginx
  tls:
  - hosts: [jenkins.example.com]
    secretName: jenkins-tls
  rules:
  - host: jenkins.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jenkins
            port:
              number: 8080
```

### Bước 5: Persistent storage

```yaml
persistence:
  enabled: true
  size: 50Gi
  storageClass: "fast-ssd"
  accessMode: ReadWriteOnce
```

---

<a id="p8"></a>
## P8. Best Practice

### Bước 1: Monitoring

```yaml
# Prometheus scrape Jenkins
- job_name: 'jenkins'
  metrics_path: '/prometheus/'
  static_configs:
    - targets: ['jenkins:8080']
```

```groovy
// Jenkinsfile - Send metrics
def buildDuration = currentBuild.duration / 1000
echo "Build duration: ${buildDuration}s"
```

### Bước 2: Performance tuning

```bash
# JVM options
JAVA_OPTS="-Djava.awt.headless=true \
  -XX:+UseG1GC \
  -XX:MaxRAMPercentage=75 \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/jenkins_home/heapdump.hprof \
  -Djenkins.install.runSetupWizard=false"
```

```
# Manage Jenkins > System Configuration:
- # of executors: 2-5 per agent
- SCM checkout retry count: 3
- Quiet period: 5 seconds
- Label string: linux docker
```

### Bước 3: Build time optimization

```groovy
// Sử dụng cache
pipeline {
    agent any

    options {
        // Tăng tốc checkout
        skipDefaultCheckout()

        // Giữ workspace
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            options {
                timeout(time: 5, unit: 'MINUTES')
            }
            steps {
                // Shallow clone
                checkout([$class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: 'https://github.com/myorg/myapp.git']],
                    extensions: [
                        [$class: 'CloneOption', depth: 1, shallow: true, noTags: false],
                        [$class: 'SubmoduleOption', disableSubmodules: false]
                    ]
                ])
            }
        }

        stage('Cache') {
            steps {
                // Restore cache
                cache(path: 'node_modules', key: 'npm-cache') {
                    sh 'npm ci'
                }
            }
        }
    }
}
```

### Bước 4: Logging

```
Manage Jenkins > System Log:
- Log Levels: Add logger
- com.cloudbees.jenkins.plugins.bitbucket: WARNING
- jenkins.security: WARNING

# System properties
java.util.logging.config.file=/var/jenkins_home/log.properties
```

```properties
# log.properties
.level=INFO
handlers=java.util.logging.FileHandler
java.util.logging.FileHandler.pattern=/var/log/jenkins/jenkins.%g.log
java.util.logging.FileHandler.count=10
java.util.logging.FileHandler.limit=10000000
java.util.logging.FileHandler.append=true
```

### Bước 5: Maintenance

```bash
# Cron cleanup
0 2 * * 0 find /var/jenkins_home/jobs/*/builds -maxdepth 1 -name "^[0-9]*$" -mtime +30 -exec rm -rf {} \;

# Plugin update
java -jar jenkins-cli.jar install-plugin <plugin>:<version>

# Backup before update
ansible-playbook jenkins-backup.yml
ansible-playbook jenkins-upgrade.yml
```

---

<a id="p9"></a>
## P9. Migration từ Freestyle sang Pipeline

### Bước 1: Freestyle cũ

```
Project: legacy-app
Build Steps:
1. Execute shell:
   - npm install
   - npm test
   - npm run build
2. Archive artifacts: dist/
3. Send email notification
```

### Bước 2: Convert sang Declarative Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh '''
                    npm install
                    npm test
                    npm run build
                '''
            }
            post {
                success {
                    archiveArtifacts artifacts: 'dist/**'
                }
            }
        }
    }

    post {
        failure {
            emailext subject: "Build failed: ${env.JOB_NAME}",
                      body: "Check ${env.BUILD_URL}",
                      to: '[email protected]'
        }
    }
}
```

### BƯớc 3: Plugin tự động convert

```
# Plugin: pipeline-migration
# Plugin: declarative-pipeline-migration-assistant

# Convert UI:
1. Mở freestyle job
2. Cấu hình > Convert to Pipeline
3. Review generated Jenkinsfile
4. Commit to repo
```

### Bước 4: Multi-job → Pipeline

```groovy
// Trước: 3 freestyle jobs
// job1: build
// job2: test (triggered after job1)
// job3: deploy (triggered after job2)

// Sau: 1 pipeline
pipeline {
    agent any
    stages {
        stage('Build') { steps { sh 'make build' } }
        stage('Test') { steps { sh 'make test' } }
        stage('Deploy') { steps { sh './deploy.sh' } }
    }
}
```

### Bước 5: Build monitor / dashboard

```
Plugins:
- Build Monitor View
- Blue Ocean (UI mới đẹp)
- Dashboard View

Manage Jenkins > Blue Ocean:
- Mở Blue Ocean UI: $JENKINS_URL/blue
```

---

## 🎯 Bài tập P0-P9

1. Cài Jenkins bằng Docker, cài plugins cơ bản
2. Tạo freestyle job build + test 1 Node.js app
3. Tạo declarative pipeline cho ứng dụng của bạn
4. Setup shared library với buildDocker, deployK8s
5. Cấu hình JCasC + K8s + agent ephemeral
6. Setup RBAC, roles cho 3 nhóm: dev, qa, ops
7. Backup Jenkins, restore lên cluster khác

---

> **💡 Tip cuối**: Jenkins đang dần chuyển sang declarative pipeline. Tránh scripted pipeline trừ khi cần logic phức tạp. Dùng shared library để DRY. Tích hợp với K8s qua plugin để scale auto.

---

*Tạo bởi tài liệu học Jenkins - Chúc bạn thành công! 🚀*
