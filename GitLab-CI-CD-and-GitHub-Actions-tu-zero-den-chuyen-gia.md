# 🚀 Tài liệu GitLab CI/CD & GitHub Actions: Từ Zero đến Chuyên gia

> **Mục tiêu**: Sau khi hoàn thành tài liệu này, bạn có thể thiết kế và vận hành pipeline CI/CD production cho bất kỳ dự án nào.

---

## 📑 Mục lục

| Phần | Nội dung |
|------|----------|
| [P0. Chuẩn bị](#p0) | GitLab/GitHub setup, Runner cơ bản |
| [P1. Pipeline cơ bản](#p1) | .gitlab-ci.yml, workflow, job |
| [P2. GitHub Actions cơ bản](#p2) | Workflow, job, step |
| [P3. Nâng cao GitLab](#p3) | Multi-stage, cache, artifacts, parent-child |
| [P4. Nâng cao GitHub Actions](#p4) | Matrix, reusable workflow, composite |
| [P5. Docker trong CI](#p5) | Build, push, cache layer |
| [P6. Deploy](#p6) | K8s, Ansible, AWS, Blue-Green |
| [P7. Security](#p7) | Secret, SAST, scanning, signing |
| [P8. Best Practice](#p8) | Optimization, monitoring, troubleshooting |

---

<a id="p0"></a>
## P0. Chuẩn bị môi trường

### Bước 1: Cài GitLab Runner local

```bash
# Docker runner (khuyến nghị cho dev)
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest

# Đăng ký với GitLab server
docker exec -it gitlab-runner gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.com/" \
  --registration-token "GR134xxx" \
  --executor "docker" \
  --docker-image "docker:24" \
  --docker-privileged=true \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
  --description "docker-runner" \
  --tag-list "docker,linux" \
  --run-untagged=true
```

### Bước 2: Tạo project đầu tiên

```bash
mkdir ci-cd-lab && cd ci-cd-lab
git init
git remote add origin [email protected]:username/ci-cd-lab.git
```

### Bước 3: GitHub Actions runner self-hosted (optional)

```bash
# Tải Actions Runner
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

# Config
./config.sh --url https://github.com/username/repo --token Axxxxxx

# Chạy
./run.sh
```

---

<a id="p1"></a>
## P1. GitLab CI/CD cơ bản

### Bước 1: Pipeline đầu tiên

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_IMAGE: "myregistry/app"
  VERSION: "1.0.0"

# Định nghĩa job template
.python-setup:
  image: python:3.11
  before_script:
    - pip install -r requirements.txt

# === BUILD STAGE ===
build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: ""
  script:
    - docker build -t $DOCKER_IMAGE:$VERSION .
    - docker save $DOCKER_IMAGE:$VERSION > image.tar
  artifacts:
    paths:
      - image.tar
    expire_in: 1 hour
  only:
    - main
    - develop

# === TEST STAGE ===
unit-tests:
  extends: .python-setup
  stage: test
  script:
    - pytest tests/ --cov=app --cov-report=xml
  coverage: '/(?i)total.*? (100(?:\.0+)?\%|[1-9]?\d(?:\.\d+)?\%)$/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

lint:
  image: node:20
  stage: test
  script:
    - npm install
    - npm run lint
  allow_failure: true

# === DEPLOY STAGE ===
deploy-staging:
  stage: deploy
  script:
    - echo "Deploy to staging"
    - ./scripts/deploy.sh staging
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop
  when: manual

deploy-prod:
  stage: deploy
  script:
    - ./scripts/deploy.sh production
  environment:
    name: production
    url: https://example.com
  only:
    - main
  when: manual
  allow_failure: false
```

### Bước 2: Cú pháp cơ bản

```yaml
# Stages - định nghĩa thứ tự
stages:
  - build
  - test
  - deploy

# Job - đơn vị công việc
job_name:
  stage: test              # Thuộc stage nào
  image: python:3.11       # Docker image
  script:                  # Các lệnh chạy
    - echo "Hello"
    - python test.py
  before_script:           # Trước script
    - apt update
  after_script:            # Sau script (kể cả fail)
    - cleanup.sh
  tags:                    # Runner tags
    - docker
    - linux
  only:                    # Chỉ chạy khi
    - main
    - develop
    - tags
  except:                  # Không chạy khi
    - /^feature\/.*/
  when: manual             # manual | on_success | on_failure | always | delayed
  allow_failure: true      # Cho phép fail
  retry: 2                 # Retry 2 lần
  timeout: 30 minutes
  artifacts:               # Lưu file cho job sau
    paths:
      - dist/
    expire_in: 1 week
  cache:                   # Cache dependencies
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
  needs:                   # Không đợi stage trước (DAG)
    - build_job
  dependencies:
    - build_job
  coverage: '/regex/'      # Coverage regex
  environment:             # Environment tracking
    name: production
    url: https://example.com
    on_stop: stop_prod
```

### Bước 3: Biến & bí mật

```bash
# Trong GitLab UI: Settings > CI/CD > Variables
# Tạo: API_KEY (masked, protected)
```

```yaml
# Dùng biến
deploy:
  script:
    - echo "$API_KEY"          # Trong script shell
    - echo "$CI_COMMIT_SHA"    # GitLab predefined variables
    - echo "$CI_PROJECT_DIR"   # Project directory
    - echo "$CI_REGISTRY_IMAGE" # Registry image path

# Predefined variables thường dùng
# CI_COMMIT_REF_NAME      # Branch name
# CI_COMMIT_TAG           # Tag name (nếu là tag)
# CI_COMMIT_SHA           # Full commit SHA
# CI_PIPELINE_ID          # Pipeline ID
# CI_PROJECT_NAME         # Project name
# CI_ENVIRONMENT_NAME     # Environment name
# CI_RUNNER_SHORT_TOKEN   # Runner token

# Variable types
# - Variable (regular)
# - File (ghi vào file)
# - Masked (ẩn trong log)
# - Protected (chỉ protected branches)
# - Expanded (expand variables)
```

### Bước 4: Chạy pipeline

```bash
git add .gitlab-ci.yml
git commit -m "ci: initial pipeline"
git push origin main

# Xem tại: https://gitlab.com/user/project/-/pipelines
```

---

<a id="p2"></a>
## P2. GitHub Actions cơ bản

### Bước 1: Workflow đầu tiên

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18.x, 20.x]

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install
        run: npm ci

      - name: Test
        run: npm test

      - name: Lint
        run: npm run lint

  build:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4

      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build & Push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://example.com
    steps:
      - name: Deploy
        run: ./scripts/deploy.sh
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
```

### Bước 2: Cú pháp workflow

```yaml
# Trigger events
on:
  push:
    branches: [main]
    tags: ['v*']
    paths: ['src/**', '!*.md']
  pull_request:
    branches: [main]
  workflow_dispatch:           # Manual trigger
    inputs:
      environment:
        description: 'Env to deploy'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]
  schedule:
    - cron: '0 2 * * *'         # Hàng ngày 2h sáng
  workflow_call:                # Reusable workflow

# Permissions
permissions:
  contents: read
  packages: write
  id-token: write             # Cho OIDC

# Defaults cho tất cả jobs
defaults:
  run:
    shell: bash
    working-directory: ./app

# Job
jobs:
  job-name:
    runs-on: ubuntu-latest    # Hoặc self-hosted
    container:                 # Container thay vì VM
      image: node:20
      env:
        NODE_ENV: test
      options: --privileged
    services:                  # Service containers
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
        options: --health-cmd pg_isready

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0        # Full history
          submodules: true

      - name: Setup
        run: echo "Setup"

      - name: Cache
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-npm-

      - name: Run
        env:
          API_KEY: ${{ secrets.API_KEY }}
        run: ./run.sh
        if: github.event_name == 'push'

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
          retention-days: 7
```

### Bước 3: Context & Expressions

```yaml
jobs:
  example:
    runs-on: ubuntu-latest
    steps:
      - name: Echo context
        run: |
          echo "Branch: ${{ github.ref }}"
          echo "Commit: ${{ github.sha }}"
          echo "Actor: ${{ github.actor }}"
          echo "Event: ${{ github.event_name }}"
          echo "Job: ${{ github.job }}"

      # Functions
      - name: Always
        if: always()
        run: echo "always runs"

      - name: On success
        if: success()
        run: echo "if previous succeeded"

      - name: On failure
        if: failure()
        run: echo "if previous failed"

      - name: Custom condition
        if: github.event_name == 'push' && contains(github.ref, 'main')
        run: echo "push to main"

      - name: Hash
        run: echo "${{ hashFiles('package-lock.json') }}"
```

---

<a id="p3"></a>
## P3. Nâng cao GitLab CI/CD

### Bước 1: Multi-project pipeline (parent-child)

```yaml
# Trong project con (.gitlab-ci.yml)
build:
  stage: build
  trigger:
    include: 
      - artifact: ci/cicd.yml
        job: generate-config
```

```yaml
# Trong project cha (downstream)
# config/cicd.yml
stages:
  - test
  - build
  - deploy

test:
  stage: test
  script: echo "test"

deploy:
  stage: deploy
  script: echo "deploy"
  when: manual
```

### Bước 2: Include & extends

```yaml
# .gitlab-ci.yml (main)
include:
  - local: 'ci/build.yml'
  - local: 'ci/test.yml'
  - local: 'ci/deploy.yml'
  - template: 'Security/SAST.gitlab-ci.yml'
  - project: 'shared/ci-templates'
    ref: main
    file: '/templates/docker.yml'
```

```yaml
# ci/build.yml
.build-base:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

docker-build:
  extends: .build-base
  script:
    - docker build -t $IMAGE:$TAG .
    - docker push $IMAGE:$TAG
```

### Bước 3: Cache thông minh

```yaml
# Global cache
cache:
  key:
    files:
      - package-lock.json
  paths:
    - node_modules/
  policy: pull-push   # Mặc định, pull-push | pull | push

# Per-job cache
job:
  cache:
    key: $CI_COMMIT_REF_SLUG
    paths:
      - vendor/
    policy: pull

# Cache nhiều key
job:
  cache:
    - key:
        files:
          - Gemfile.lock
      paths:
        - vendor/bundle
    - key:
        files:
          - yarn.lock
      paths:
        - .yarn-cache
    - key: $CI_JOB_NAME
      paths:
        - build/cache
```

### Bước 4: DAG (needs)

```yaml
stages:
  - build
  - test
  - deploy

# Bình thường: build -> test -> deploy
# Với needs: chạy song song

build-api:
  stage: build
  script: ./build-api.sh
  artifacts:
    paths: [api/]

build-web:
  stage: build
  script: ./build-web.sh
  artifacts:
    paths: [web/]

test-api:
  stage: test
  needs: [build-api]   # Đợi build-api, không đợi build-web
  script: ./test-api.sh

test-web:
  stage: test
  needs: [build-web]
  script: ./test-web.sh

deploy:
  stage: deploy
  needs: [test-api, test-web]
  script: ./deploy.sh
```

### Bước 5: Matrix build

```yaml
.test:
  stage: test
  image: python:3.11
  script:
    - pip install -r requirements.txt
    - pytest tests/

test-py3.10:
  extends: .test
  image: python:3.10

test-py3.11:
  extends: .test
  image: python:3.11

test-py3.12:
  extends: .test
  image: python:3.12
```

### Bước 6: Review Apps

```yaml
review:
  stage: deploy
  script:
    - deploy-review.sh
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_COMMIT_REF_SLUG.review.example.com
    on_stop: stop-review
  only:
    - branches
  except:
    - main

stop-review:
  stage: deploy
  script:
    - delete-review.sh
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  when: manual
  only:
    - branches
```

---

<a id="p4"></a>
## P4. Nâng cao GitHub Actions

### Bước 1: Reusable workflow

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy Workflow

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string
    secrets:
      DEPLOY_KEY:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        run: ./deploy.sh ${{ inputs.environment }} ${{ inputs.image-tag }}
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
```

```yaml
# .github/workflows/deploy-prod.yml
name: Deploy Production
on:
  push:
    tags: ['v*']

jobs:
  deploy:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
      image-tag: ${{ github.ref_name }}
    secrets:
      DEPLOY_KEY: ${{ secrets.PROD_DEPLOY_KEY }}
```

### Bước 2: Composite action

```yaml
# .github/actions/setup-app/action.yml
name: 'Setup Application'
description: 'Setup Node.js and install deps'

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20'

runs:
  using: 'composite'
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'

    - name: Install dependencies
      shell: bash
      run: npm ci

    - name: Build
      shell: bash
      run: npm run build
```

```yaml
# .github/workflows/ci.yml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-app
      - name: Test
        run: npm test
```

### Bước 3: Matrix strategy

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false        # Không cancel matrix khác khi 1 fail
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node: [18, 20, 22]
        exclude:
          - os: windows-latest
            node: 18
        include:
          - os: ubuntu-latest
            node: 20
            coverage: true

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}

      - name: Test
        run: npm test

      - name: Coverage
        if: ${{ matrix.coverage }}
        run: npm run coverage
```

### Bước 4: OIDC với AWS

```yaml
# GitHub OIDC -> AWS không cần access key
permissions:
  id-token: write
  contents: read

steps:
  - name: Configure AWS
    uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/github-actions-role
      aws-region: us-east-1

  - name: Deploy to ECS
    run: |
      aws ecs update-service --cluster prod --service app --force-new-deployment
```

### Bước 5: Conditional steps

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build only if changed
        if: contains(github.event.head_commit.modified, 'src/')
        run: npm run build

      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {"text": "Build failed: ${{ github.workflow }}#${{ github.run_number }}"}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

      - name: Cleanup
        if: always()
        run: ./cleanup.sh
```

---

<a id="p5"></a>
## P5. Docker trong CI/CD

### Bước 1: Build & Push image (GitLab)

```yaml
# .gitlab-ci.yml
build-and-push:
  stage: build
  image: docker:24
  services:
    - name: docker:24-dind
      alias: docker
  variables:
    DOCKER_TLS_CERTDIR: ""
    IMAGE_NAME: $CI_REGISTRY_IMAGE
    IMAGE_TAG: $CI_COMMIT_SHORT_SHA
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin $CI_REGISTRY
  script:
    - docker build
        --cache-from $IMAGE_NAME:latest
        --tag $IMAGE_NAME:$IMAGE_TAG
        --tag $IMAGE_NAME:latest
        .
    - docker push $IMAGE_NAME:$IMAGE_TAG
    - docker push $IMAGE_NAME:latest
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

### Bước 2: Layer caching (GitHub Actions)

```yaml
- uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ${{ env.IMAGE }}:${{ github.sha }}
    cache-from: type=gha    # GitHub Actions cache
    cache-to: type=gha,mode=max
    provenance: true        # SLSA attestation
```

### Bước 3: Multi-arch build

```yaml
build-multiarch:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker buildx create --use
    - docker buildx build
        --platform linux/amd64,linux/arm64
        --tag $IMAGE:$TAG
        --push
        .
```

### Bước 4: Cosign signing

```yaml
sign-image:
  stage: sign
  image: ghcr.io/sigstore/cosign:v2.2.3
  script:
    - cosign sign --yes $IMAGE:$SHA
  variables:
    COSIGN_EXPERIMENTAL: "1"
```

---

<a id="p6"></a>
## P6. Deploy tự động

### Bước 1: Deploy K8s từ GitLab

```yaml
deploy-k8s:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - echo "$KUBECONFIG_DATA" > kubeconfig.yaml
    - export KUBECONFIG=$PWD/kubeconfig.yaml
    
    # Update image
    - kubectl set image deployment/app app=$IMAGE:$SHA -n production
    
    # Hoặc dùng Kustomize
    - kubectl apply -k overlays/production
    
    # Hoặc Helm
    - helm upgrade app ./chart --set image.tag=$SHA -n production
    
    # Đợi rollout
    - kubectl rollout status deployment/app -n production --timeout=5m
  environment:
    name: production
  only:
    - main
```

### Bước 2: Blue-Green deployment (GitHub Actions)

```yaml
deploy-blue-green:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4

    - name: Setup kubectl
      uses: azure/setup-kubectl@v4

    - name: Configure kubeconfig
      run: |
        mkdir -p ~/.kube
        echo "${{ secrets.KUBECONFIG }}" > ~/.kube/config

    - name: Deploy to green
      run: |
        kubectl patch service app -p '{"spec":{"selector":{"version":"green"}}}'
        kubectl set image deployment/app-green app=$IMAGE:$SHA

    - name: Test green
      run: ./scripts/smoke-test.sh green

    - name: Switch traffic
      run: |
        kubectl patch service app -p '{"spec":{"selector":{"version":"green"}}}'
        kubectl scale deployment/app-blue --replicas=0

    - name: Rollback
      if: failure()
      run: |
        kubectl patch service app -p '{"spec":{"selector":{"version":"blue"}}}'
        kubectl scale deployment/app-green --replicas=0
```

### Bước 3: Canary deployment (Argo Rollouts)

```yaml
deploy-canary:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl apply -f rollouts.yaml
    - kubectl argo rollouts set image app app=$IMAGE:$SHA
    - kubectl argo rollouts promote app
```

### Bước 4: Deploy bằng Ansible

```yaml
deploy-ansible:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client ansible
    - echo "$ANSIBLE_SSH_KEY" > /tmp/key
    - chmod 600 /tmp/key
  script:
    - ansible-playbook -i inventory/production/hosts deploy.yaml
      --private-key=/tmp/key
      --extra-vars "image_tag=$SHA"
```

### Bước 5: AWS Deploy (ECS)

```yaml
deploy-ecs:
  stage: deploy
  image: amazon/aws-cli:latest
  script:
    - aws ecs update-service
        --cluster production
        --service app
        --force-new-deployment
```

---

<a id="p7"></a>
## P7. Security

### Bước 1: Secret management

```yaml
# GitLab - dùng masked & protected variables
deploy:
  script:
    - echo "$DB_PASSWORD" | mysql -h $DB_HOST -u $DB_USER -p$DB_PASSWORD
  variables:
    DB_PASSWORD:
      value: "secret123"
      description: "DB password"
      mask: true
      protected: true

# Vault integration
secrets:
  DB_PASSWORD:
    vault: production/db/password@secrets
    file: false
```

```yaml
# GitHub - dùng secrets
steps:
  - name: Deploy
    env:
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
      API_KEY: ${{ secrets.API_KEY }}
    run: ./deploy.sh
```

### Bước 2: SAST (Static Analysis)

```yaml
# GitLab
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml
```

```yaml
# GitHub
name: Security Scan
on: [push, pull_request]

jobs:
  codeql:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript, python
      - name: Perform Analysis
        uses: github/codeql-action/analyze@v3
```

### Bước 3: Trivy - scan image & code

```yaml
# GitLab
trivy:
  stage: test
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE:$TAG
    - trivy fs --exit-code 1 --severity HIGH,CRITICAL .
  allow_failure: false
```

```yaml
# GitHub
- name: Trivy scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.IMAGE }}:${{ github.sha }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'

- name: Upload Trivy scan results
  if: always()
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

### Bước 4: License compliance

```yaml
license-check:
  stage: test
  script:
    - npm install
    - npx license-checker --failOn 'GPL;AGPL' --onlyAllow 'MIT;Apache-2.0;BSD-3-Clause'
```

### Bước 5: OIDC thay secret

```yaml
# .github/workflows/deploy.yml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: AWS login qua OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-deploy
          aws-region: us-east-1
```

---

<a id="p8"></a>
## P8. Best Practice & Production

### Bước 1: Tối ưu pipeline

```yaml
# Cache dependencies
cache:
  key:
    files:
      - package-lock.json
  paths:
    - node_modules/

# Parallel jobs
test:
  parallel: 5
  script: pytest tests/

# Smaller images
variables:
  DOCKER_BUILDKIT: 1

# Skip trên draft PR
rules:
  - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    when: always
  - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    when: always
  - when: manual
    allow_failure: true
```

### Bước 2: Monitoring pipeline

```yaml
# GitLab CI metrics qua Prometheus
# Sử dụng GitLab API
script:
  - |
    curl --header "PRIVATE-TOKEN: $ADMIN_TOKEN" \
      "https://gitlab.com/api/v4/projects/$CI_PROJECT_ID/pipelines/$CI_PIPELINE_ID"
```

### Bước 3: Pipeline as Code - testing

```yaml
# Test CI/CD bằng `gitlab-ci-local`
# Hoặc GitHub Actions: `act`
```

```bash
# act - test GitHub Actions local
brew install act
act                           # Mặc định push event
act -j build                   # Job cụ thể
act --secret-file .secrets     # Dùng secret file
```

### Bước 4: Top 10 lỗi thường gặp

```yaml
# 1. Quên cache dependencies -> pipeline chậm
# Fix: thêm cache key đúng

# 2. Hardcode secret trong code -> leak
# Fix: dùng masked variable / Vault

# 3. Không tag image -> dùng "latest" -> khó rollback
# Fix: tag theo commit SHA

# 4. Không test pipeline -> fail production
# Fix: dùng act/gitlab-ci-local

# 5. Không giới hạn trigger -> pipeline spam
# Fix: dùng only/except hoặc rules

# 6. Không set timeout -> job treo vô tận
# Fix: timeout: 30 minutes

# 7. Không giới hạn resource -> runner quá tải
# Fix: tags + dedicated runner

# 8. Deploy tự động không có manual gate
# Fix: when: manual cho production

# 9. Không có rollback plan
# Fix: helm rollback / kubectl rollout undo

# 10. Không scan lỗ hổng
# Fix: tích hợp Trivy/Snyk
```

### Bước 5: Migration từ Jenkins sang GitLab/GitHub

```bash
# Mapping:
# Jenkinsfile stage -> .gitlab-ci.yml stages / GitHub jobs
# Jenkins credentials -> GitLab/GitHub secrets
# Jenkins plugins -> Marketplace actions / GitLab templates
# Jenkinsfile agent -> image / runs-on
```

### 🎯 Bài tập P0-P8
1. Tạo project với pipeline build + test + deploy lên K3s
2. Setup reusable workflow cho 2 services
3. Tích hợp Trivy scan image
4. Deploy blue-green lên K8s cluster
5. Test pipeline bằng `act`

---

> **💡 Tip cuối**: Pipeline càng đơn giản càng tốt. Dùng caching đúng cách giảm 50% thời gian. Luôn test pipeline local trước khi commit.

---

*Tạo bởi tài liệu học CI/CD - Chúc bạn thành công! 🚀*
