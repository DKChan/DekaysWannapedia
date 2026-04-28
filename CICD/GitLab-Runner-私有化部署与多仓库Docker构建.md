# GitLab Runner 私有化部署与多仓库 Docker 构建指南

> 适用场景：私有 GitLab + 多仓库个性化 Docker 镜像构建 + 自建 Docker 仓库

---

## 目录

1. [架构概览](#架构概览)
2. [Runner 安装](#runner-安装)
3. [Runner 注册](#runner-注册)
4. [Docker-in-Docker 配置](#docker-in-docker-配置)
5. [.gitlab-ci.yml 脚本编写](#gitlab-ciyml-脚本编写)
6. [权限管理](#权限管理)
7. [高级技巧](#高级技巧)
8. [故障排查](#故障排查)

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                      私有 GitLab 服务器                       │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   Repo A    │    │   Repo B    │    │   Repo C    │     │
│  │  (Go项目)    │    │ (Node项目)  │    │ (Java项目)  │     │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘     │
│         │                  │                  │             │
│         └──────────────────┼──────────────────┘             │
│                            ▼                                │
│                   ┌─────────────────┐                       │
│                   │  GitLab Runner  │  ← 可部署多台          │
│                   │   (Docker模式)   │                       │
│                   └────────┬────────┘                       │
└────────────────────────────┼────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                   自建 Docker Registry                       │
│              (Harbor/Nexus/Registry)                        │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │  project-a  │    │  project-b  │    │  project-c  │     │
│  │  /app:tag   │    │  /app:tag   │    │  /app:tag   │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

---

## Runner 安装

### 方式一：Docker 部署（推荐）

**注意：需要根据宿主机架构选择镜像标签**

| 宿主机架构 | 镜像标签 |
|-----------|---------|
| AMD64 (x86_64) | `gitlab/gitlab-runner:latest` 或 `gitlab/gitlab-runner:amd64` |
| ARM64 (aarch64) | `gitlab/gitlab-runner:arm64` |

#### AMD64 部署

```bash
# 创建配置目录
mkdir -p /srv/gitlab-runner/config

# 运行 Runner 容器（AMD64）
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

#### ARM64 部署

```bash
# 创建配置目录
mkdir -p /srv/gitlab-runner/config

# 运行 Runner 容器（ARM64）
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:arm64
```

**验证架构：**
```bash
# 检查宿主机架构
uname -m
# 输出: aarch64 = ARM64, x86_64 = AMD64

# 检查运行的容器架构
docker exec gitlab-runner uname -m
```

### 方式二：二进制安装

#### AMD64 (x86_64)

```bash
# 下载
curl -LJO "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64"

# 安装
chmod +x gitlab-runner-linux-amd64
sudo mv gitlab-runner-linux-amd64 /usr/local/bin/gitlab-runner
```

#### ARM64 (aarch64)

```bash
# 下载
curl -LJO "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-arm64"

# 安装
chmod +x gitlab-runner-linux-arm64
sudo mv gitlab-runner-linux-arm64 /usr/local/bin/gitlab-runner
```

#### 通用安装步骤（AMD64/ARM64 相同）

```bash
# 创建用户
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash

# 安装并启动服务
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start
```

---

## Runner 注册

### 获取注册 Token

在 GitLab 页面：
- **Group 级别**: `Group → Settings → CI/CD → Runners → New group runner`
- **Project 级别**: `Project → Settings → CI/CD → Runners → New project runner`

### 执行注册

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.yourcompany.com" \
  --registration-token "GR1348941xxxxxxxxxxxx" \
  --executor "docker" \
  --docker-image "docker:24-dind" \
  --description "Docker Runner for Multi-Repo Builds" \
  --tag-list "docker,dind,multi-repo" \
  --run-untagged="false" \
  --locked="false" \
  --access-level="not_protected"
```

### 关键配置说明

| 参数 | 说明 |
|------|------|
| `--executor docker` | 使用 Docker 执行 job |
| `--docker-image docker:24-dind` | 基础镜像（Docker in Docker） |
| `--tag-list` | Runner 标签，用于匹配 job |
| `--access-level not_protected` | 允许运行非保护分支的 job |

---

## Docker-in-Docker 配置

编辑 Runner 配置 `/srv/gitlab-runner/config/config.toml`：

```toml
[[runners]]
  name = "Docker Runner for Multi-Repo Builds"
  url = "https://gitlab.yourcompany.com"
  token = "xxxxxxxxxxxxxxxx"
  executor = "docker"
  
  [runners.docker]
    tls_verify = false
    image = "docker:24-dind"
    privileged = true  # 必须开启，用于 DinD
    disable_cache = false
    volumes = [
      "/var/run/docker.sock:/var/run/docker.sock",  # 挂载宿主机 Docker
      "/cache",
      "/certs/client"
    ]
    
  [runners.cache]
    Type = "s3"  # 可选：使用 S3 缓存加速构建
    Shared = true
```

**重启 Runner：**

```bash
# Docker 部署方式
docker restart gitlab-runner

# 二进制部署方式
sudo gitlab-runner restart
# 或 systemd
sudo systemctl restart gitlab-runner
```

---

## .gitlab-ci.yml 脚本编写

### 核心概念

```yaml
# 基本结构
stages:           # 定义阶段顺序
  - build
  - test
  - deploy

variables:        # 定义变量
  KEY: "value"

job_name:         # Job 定义
  stage: build    # 所属阶段
  image: node:18  # 使用的镜像
  tags:           # Runner 标签匹配
    - docker
  before_script:  # 前置脚本
    - echo "准备环境"
  script:         # 主脚本（必须）
    - echo "执行构建"
  after_script:   # 后置脚本
    - echo "清理"
  only:           # 触发条件
    - main
  except:         # 排除条件
    - tags
  when: on_success  # 执行时机
  allow_failure: false  # 是否允许失败
```

### 完整示例：多仓库 Docker 构建

#### 示例 1：基础 Docker 构建

```yaml
# .gitlab-ci.yml
variables:
  DOCKER_REGISTRY: "harbor.yourcompany.com"
  PROJECT_NAME: "my-project"
  IMAGE_NAME: "$DOCKER_REGISTRY/$PROJECT_NAME/app"

stages:
  - build
  - docker-build
  - push

# 编译阶段（使用项目自身技术栈镜像）
build:
  stage: build
  image: golang:1.21  # 根据项目调整：node:18, maven:3.9, python:3.11
  script:
    - go build -o app ./cmd/server
  artifacts:
    paths:
      - app
    expire_in: 1 hour

# Docker 构建阶段
docker-build:
  stage: docker-build
  image: docker:24-dind
  services:
    - docker:24-dind
  tags:
    - dind
  dependencies:
    - build
  before_script:
    - docker info
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $DOCKER_REGISTRY
  script:
    # 构建镜像
    - docker build -t $IMAGE_NAME:$CI_COMMIT_SHA .
    # 额外标签
    - docker tag $IMAGE_NAME:$CI_COMMIT_SHA $IMAGE_NAME:latest
    - docker tag $IMAGE_NAME:$CI_COMMIT_SHA $IMAGE_NAME:$CI_COMMIT_REF_NAME
  only:
    - main
    - develop

# 推送阶段
push:
  stage: push
  image: docker:24-dind
  services:
    - docker:24-dind
  tags:
    - dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $DOCKER_REGISTRY
  script:
    - docker push $IMAGE_NAME:$CI_COMMIT_SHA
    - docker push $IMAGE_NAME:latest
    - docker push $IMAGE_NAME:$CI_COMMIT_REF_NAME
  only:
    - main
```

#### 示例 2：多环境部署

```yaml
variables:
  DOCKER_REGISTRY: "harbor.yourcompany.com"
  IMAGE_NAME: "$DOCKER_REGISTRY/$CI_PROJECT_NAME/app"

stages:
  - build
  - test
  - docker-build
  - deploy-dev
  - deploy-staging
  - deploy-prod

# 构建镜像
docker-build:
  stage: docker-build
  image: docker:24-dind
  services:
    - docker:24-dind
  tags:
    - dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $DOCKER_REGISTRY
  script:
    - docker build -t $IMAGE_NAME:$CI_COMMIT_SHA .
    - docker push $IMAGE_NAME:$CI_COMMIT_SHA

# 开发环境部署
deploy-dev:
  stage: deploy-dev
  image: bitnami/kubectl:latest
  tags:
    - dind
  script:
    - kubectl config use-context dev
    - kubectl set image deployment/app app=$IMAGE_NAME:$CI_COMMIT_SHA -n dev
  environment:
    name: development
    url: https://dev.yourcompany.com
  only:
    - develop

# 预发环境部署
deploy-staging:
  stage: deploy-staging
  image: bitnami/kubectl:latest
  tags:
    - dind
  script:
    - kubectl config use-context staging
    - kubectl set image deployment/app app=$IMAGE_NAME:$CI_COMMIT_SHA -n staging
  environment:
    name: staging
    url: https://staging.yourcompany.com
  only:
    - main
  when: manual  # 手动触发

# 生产环境部署
deploy-prod:
  stage: deploy-prod
  image: bitnami/kubectl:latest
  tags:
    - dind
  script:
    - kubectl config use-context prod
    - kubectl set image deployment/app app=$IMAGE_NAME:$CI_COMMIT_SHA -n prod
  environment:
    name: production
    url: https://yourcompany.com
  only:
    - main
  when: manual
  allow_failure: false
```

#### 示例 3：并行构建矩阵

```yaml
docker-build:
  stage: docker-build
  image: docker:24-dind
  services:
    - docker:24-dind
  tags:
    - dind
  parallel:
    matrix:
      - ARCH: [amd64, arm64]
        VARIANT: [slim, full]
  script:
    - docker build 
      --platform linux/$ARCH
      --build-arg VARIANT=$VARIANT
      -t $IMAGE_NAME:$ARCH-$VARIANT-$CI_COMMIT_SHA .
    - docker push $IMAGE_NAME:$ARCH-$VARIANT-$CI_COMMIT_SHA
```

#### 示例 4：使用模板复用配置

```yaml
# .gitlab-ci.yml
include:
  - project: 'devops/ci-templates'
    file: '/docker-build.yml'
    ref: main

variables:
  PROJECT_NAME: "my-service"

# 覆盖模板中的特定 job
build:
  image: golang:1.21-alpine
  script:
    - go build -o bin/server ./cmd/server
```

模板文件（`devops/ci-templates` 项目）：

```yaml
# docker-build.yml
.docker-job:
  image: docker:24-dind
  services:
    - docker:24-dind
  tags:
    - dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $DOCKER_REGISTRY

docker-build:
  extends: .docker-job
  stage: build
  script:
    - docker build -t $DOCKER_REGISTRY/$PROJECT_NAME:$CI_COMMIT_SHA .
    - docker push $DOCKER_REGISTRY/$PROJECT_NAME:$CI_COMMIT_SHA
```

---

## 权限管理

### 1. CI/CD 变量配置

在 **Group 级别**或 **Project 级别**设置：

```
Group → Settings → CI/CD → Variables
```

| 变量名 | 作用域 | 保护 | 说明 |
|--------|--------|------|------|
| `CI_REGISTRY_USER` | Group | No | Docker 仓库用户名 |
| `CI_REGISTRY_PASSWORD` | Group | Yes | Docker 仓库密码（Masked） |
| `DOCKER_REGISTRY` | Group | No | 仓库地址 |
| `KUBECONFIG_DEV` | Group | Yes | 开发环境 kubeconfig |
| `KUBECONFIG_PROD` | Group | Yes | 生产环境 kubeconfig |

### 2. Harbor 机器人账户

**Harbor 端：**
1. 进入项目 → 机器人账户 → 新建
2. 权限：推送镜像（Push）
3. 复制生成的 Token

**GitLab 端使用：**
```yaml
before_script:
  - echo "$HARBOR_ROBOT_TOKEN" | docker login harbor.yourcompany.com -u "robot$project-a" --password-stdin
```

### 3. 保护变量与分支

```yaml
# 只在保护分支上可用的变量
variables:
  PROD_DEPLOY_KEY:
    value: ""
    description: "生产部署密钥"

# Job 级别的保护
deploy-prod:
  only:
    - main
  environment:
    name: production
    deployment_tier: production  # 需要审批
```

---

## 高级技巧

### 1. 镜像层缓存

```yaml
docker-build:
  script:
    # 拉取之前的镜像用于缓存
    - docker pull $IMAGE_NAME:latest || true
    # 使用缓存构建
    - docker build 
      --cache-from $IMAGE_NAME:latest
      --build-arg BUILDKIT_INLINE_CACHE=1
      -t $IMAGE_NAME:$CI_COMMIT_SHA .
    - docker push $IMAGE_NAME:$CI_COMMIT_SHA
```

### 2. 多阶段构建优化

```yaml
# 使用 BuildKit
variables:
  DOCKER_BUILDKIT: "1"

docker-build:
  script:
    - docker build --target production -t $IMAGE_NAME:$CI_COMMIT_SHA .
```

### 3. 构建产物传递

```yaml
build:
  stage: build
  script:
    - make build
  artifacts:
    paths:
      - dist/
      - bin/
    expire_in: 1 week

docker-build:
  stage: docker
  dependencies:
    - build
  script:
    - docker build -t $IMAGE_NAME:$CI_COMMIT_SHA .
```

### 4. 条件执行

```yaml
# 只在特定文件变更时执行
docker-build:
  rules:
    - changes:
        - Dockerfile
        - src/**/*
      when: always
    - when: never

# 使用正则匹配分支
deploy:
  only:
    - /^release-\d+\.\d+$/
  except:
    - branches
```

### 5. 动态生成 Pipeline

```yaml
# 使用 parent-child pipeline
trigger-downstream:
  trigger:
    include:
      - local: path/to/child-pipeline.yml
    strategy: depend
```

---

## 故障排查

### 常用命令

```bash
# 1. 检查 Runner 状态
sudo gitlab-runner status

# 2. 查看已注册 Runner
sudo gitlab-runner list

# 3. 查看 Runner 日志
sudo gitlab-runner --debug run

# 4. 验证 GitLab 连通性
curl -I https://gitlab.yourcompany.com/api/v4/version

# 5. 测试 Docker 连接
docker run --rm docker:24-dind docker info

# 6. 验证镜像推送
docker login harbor.yourcompany.com
docker tag alpine:latest harbor.yourcompany.com/test/alpine:test
docker push harbor.yourcompany.com/test/alpine:test
```

### 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `Cannot connect to Docker daemon` | DinD 未正确配置 | 检查 privileged=true 和 services |
| `denied: requested access to the resource is denied` | 登录凭证错误 | 检查 CI_REGISTRY_USER/PASSWORD |
| `No runner matches the assigned tags` | Runner 标签不匹配 | 检查 tags 和 Runner 注册的 tag-list |
| `Job is stuck` | 没有可用 Runner | 检查 Runner 状态，确认 not locked |

---

## 架构选择建议

### 二进制部署 vs Docker 部署

| 对比项 | 二进制部署 | Docker 部署 |
|--------|-----------|-------------|
| 操作方式 | 直接在宿主机执行命令 | 需 `docker exec` 进入容器 |
| 架构支持 | 需下载对应架构二进制文件 | 需拉取对应架构镜像 |
| 服务管理 | systemd / 直接命令 | docker restart/stop |
| 资源隔离 | 无 | 有容器隔离 |
| 适合场景 | 长期使用、直接管理 | 快速测试、环境隔离 |

### 推荐选择

- **ARM64 服务器长期使用** → **二进制部署**
  - 直接在宿主机执行 `gitlab-runner` 命令
  - 使用 systemd 管理生命周期
  - 无需 `docker exec` 进入容器操作

- **快速测试或多 Runner 隔离** → **Docker 部署**
  - 注意使用 `gitlab/gitlab-runner:arm64` 镜像
  - 操作需通过 `docker exec` 进入容器

---

## 参考文档

- [GitLab CI/CD 官方文档](https://docs.gitlab.com/ee/ci/)
- [GitLab Runner 配置](https://docs.gitlab.com/runner/configuration/)
- [.gitlab-ci.yml 完整参考](https://docs.gitlab.com/ee/ci/yaml/)

---

*文档创建时间：2026-04-23*
*适用版本：GitLab 15.x+, GitLab Runner 15.x+*
