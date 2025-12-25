# Kubernetes 아키텍처 설계 및 구현 계획

이 문서는 krgeobuk-infra의 **폴더 기반 환경 분리 쿠버네티스 아키텍처** 설계 및 구현 계획을 설명합니다.

## 목차

- [아키텍처 개요](#아키텍처-개요)
- [핵심 설계 철학](#핵심-설계-철학)
- [리포지토리 구조](#리포지토리-구조)
- [인프라 아키텍처](#인프라-아키텍처)
- [Kubernetes 매니페스트 구조](#kubernetes-매니페스트-구조)
- [환경별 설정 전략](#환경별-설정-전략)
- [구현 계획](#구현-계획)
- [배포 워크플로우](#배포-워크플로우)
- [백업 및 재해복구](#백업-및-재해복구)

## 아키텍처 개요

### 폴더 기반 환경 분리 구조

```
┌─────────────────────────────────────────────────────────────┐
│ GitHub 리포지토리 (3개 독립 리포)                            │
├─────────────────────────────────────────────────────────────┤
│ krgeobuk-infra (메인 - 애플리케이션)                        │
│ ├── auth-server/         (서브모듈)                         │
│ ├── authz-server/        (서브모듈)                         │
│ ├── portal-client/       (서브모듈)                         │
│ └── shared-lib/          (서브모듈)                         │
│                                                              │
│ krgeobuk-k8s (독립 - Kubernetes 매니페스트)                 │
│ ├── base/                (공통 리소스)                      │
│ ├── applications/        (공통 애플리케이션 템플릿)         │
│ └── environments/        (환경별 설정)                      │
│     ├── dev/             (개발 환경)                        │
│     └── prod/            (운영 환경 - miniPC)               │
│                                                              │
│ krgeobuk-infrastructure (독립 - 인프라)                     │
│ ├── docker-compose/      (MySQL, Redis, Jenkins)            │
│ └── backup/              (백업 스크립트)                    │
│                                                              │
│ krgeobuk-deployment (독립 - CI/CD)                          │
│ ├── jenkins/             (파이프라인)                       │
│ └── scripts/             (배포 스크립트)                    │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ miniPC 서버 (단일 노드 k3s)                                  │
├─────────────────────────────────────────────────────────────┤
│ Docker 컨테이너 (클러스터 외부)                              │
│ ├── krgeobuk-mysql (단일 컨테이너)                          │
│ │   ├── auth_dev DB                                          │
│ │   ├── auth_prod DB                                         │
│ │   ├── authz_dev DB                                         │
│ │   └── authz_prod DB                                        │
│ │                                                            │
│ ├── krgeobuk-redis (단일 컨테이너)                          │
│ │   ├── DB 0 (auth-dev)                                      │
│ │   ├── DB 1 (auth-prod)                                     │
│ │   ├── DB 2 (authz-dev)                                     │
│ │   └── DB 3 (authz-prod)                                    │
│ │                                                            │
│ └── jenkins (CI/CD)                                          │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ k3s 클러스터 (애플리케이션만)                                │
├─────────────────────────────────────────────────────────────┤
│ Namespace: krgeobuk-dev (개발 환경)                          │
│ ├── auth-server (replicas: 1, resources: small)             │
│ ├── authz-server (replicas: 1, resources: small)            │
│ └── portal-client (replicas: 1)                             │
│                                                              │
│ Namespace: krgeobuk-prod (운영 환경)                         │
│ ├── auth-server (replicas: 2, resources: medium)            │
│ ├── authz-server (replicas: 2, resources: medium)           │
│ └── portal-client (replicas: 2)                             │
└─────────────────────────────────────────────────────────────┘
```

## 핵심 설계 철학

### 1. 폴더 기반 환경 분리
- **단일 브랜치**: main 브랜치만 사용
- **환경 폴더**: `environments/dev/`, `environments/prod/`로 분리
- **디스크 절약**: 중복 clone 불필요, 50% 디스크 절약
- **동시 배포**: dev, prod 동시 배포 가능

### 2. 데이터 안정성 극대화
- **DB/Redis 분리**: 쿠버네티스 외부에서 Docker로 관리
- **단일 컨테이너**: 물리적 컨테이너는 하나, 논리적으로 DB 분리
- **백업 단순화**: 전통적인 DB 백업 도구 활용

### 3. 운영 복잡도 최소화
- **Kustomize 활용**: base + overlay 패턴으로 중복 최소화
- **공통 관리**: base 한 곳만 수정 → 모든 환경 적용
- **명확한 차이**: patch 파일만 보면 환경 간 차이 파악

### 4. GitOps 친화적
- **IaC**: 모든 설정이 Git으로 관리
- **ArgoCD 호환**: 경로 기반 배포 지원
- **버전 관리**: 환경별 설정 변경 히스토리 추적

## 리포지토리 구조

### krgeobuk-k8s (Kubernetes 매니페스트)

```
krgeobuk-k8s/
├── README.md
├── .gitignore
│
├── base/                           # 공통 리소스
│   ├── namespace.yaml             # krgeobuk-dev, krgeobuk-prod
│   ├── external-mysql.yaml        # 외부 MySQL 연결
│   ├── external-redis.yaml        # 외부 Redis 연결
│   └── kustomization.yaml
│
├── applications/                   # 애플리케이션 공통 템플릿
│   ├── auth-server/
│   │   ├── deployment.yaml        # 기본 설정 (최소값)
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   ├── secret.yaml.template
│   │   └── kustomization.yaml
│   │
│   ├── authz-server/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   ├── secret.yaml.template
│   │   └── kustomization.yaml
│   │
│   └── portal-client/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── nginx-configmap.yaml
│       └── kustomization.yaml
│
├── cluster-addons/                 # 클러스터 수준 애드온
│   ├── cert-manager/              # TLS 인증서 관리
│   │   ├── namespace.yaml
│   │   ├── cluster-issuer-staging.yaml
│   │   ├── cluster-issuer-prod.yaml
│   │   └── kustomization.yaml
│   │
│   └── ingress-nginx/             # Ingress Controller
│       ├── namespace.yaml
│       └── kustomization.yaml
│
├── environments/                   # 환경별 설정 (차이만)
│   ├── dev/
│   │   ├── kustomization.yaml     # dev 환경 통합
│   │   ├── namespace-patch.yaml   # namespace: krgeobuk-dev
│   │   ├── patches/
│   │   │   ├── auth-server-dev.yaml    # replicas: 1, resources 작게
│   │   │   ├── authz-server-dev.yaml
│   │   │   └── portal-client-dev.yaml
│   │   └── configmaps/
│   │       ├── auth-server-env.yaml    # LOG_LEVEL=debug
│   │       └── authz-server-env.yaml
│   │
│   └── prod/                      # miniPC 운영 환경
│       ├── kustomization.yaml
│       ├── namespace-patch.yaml   # namespace: krgeobuk-prod
│       ├── patches/
│       │   ├── auth-server-prod.yaml   # replicas: 2, resources 크게
│       │   ├── authz-server-prod.yaml
│       │   └── portal-client-prod.yaml
│       └── configmaps/
│           ├── auth-server-env.yaml    # LOG_LEVEL=info
│           └── authz-server-env.yaml
│
├── scripts/                        # 운영 스크립트
│   ├── deploy.sh                  # 배포 스크립트
│   ├── rollback.sh                # 롤백 스크립트
│   ├── health-check.sh            # 헬스체크
│   └── logs.sh                    # 로그 수집
│
└── docs/
    ├── DEPLOYMENT.md              # 배포 가이드
    └── ARCHITECTURE.md            # 아키텍처 설명
```

### krgeobuk-infrastructure (인프라)

```
krgeobuk-infrastructure/
├── README.md
├── .gitignore
├── .env.example                   # 환경 변수 템플릿
│
├── docker-compose/
│   ├── docker-compose.yaml        # 통합 Compose 파일
│   │
│   ├── mysql/
│   │   ├── init/
│   │   │   ├── 01-create-databases.sql
│   │   │   │   # CREATE DATABASE auth_dev;
│   │   │   │   # CREATE DATABASE authz_dev;
│   │   │   │   # CREATE DATABASE auth_prod;
│   │   │   │   # CREATE DATABASE authz_prod;
│   │   │   │
│   │   │   └── 02-create-users.sql
│   │   │       # CREATE USER 'auth_user'@'%';
│   │   │       # GRANT ALL ON auth_dev.* TO 'auth_user'@'%';
│   │   │       # GRANT ALL ON auth_prod.* TO 'auth_user'@'%';
│   │   │
│   │   ├── conf/
│   │   │   └── my.cnf             # MySQL 최적화 설정
│   │   │
│   │   └── data/.gitkeep          # 데이터 볼륨 마운트 위치
│   │
│   ├── redis/
│   │   ├── redis.conf             # Redis 설정
│   │   └── data/.gitkeep
│   │
│   ├── jenkins/
│   │   ├── Dockerfile             # kubectl, docker 포함
│   │   ├── plugins.txt            # Jenkins 플러그인 목록
│   │   └── data/.gitkeep
│   │
│   └── verdaccio/
│       ├── config.yaml
│       └── storage/.gitkeep
│
├── backup/
│   ├── mysql-backup.sh            # MySQL 백업 스크립트
│   ├── redis-backup.sh            # Redis 백업 스크립트
│   ├── backup-cron                # cron 스케줄
│   └── restore.sh                 # 복구 스크립트
│
├── scripts/
│   ├── start-all.sh               # 모든 서비스 시작
│   ├── stop-all.sh                # 모든 서비스 중지
│   ├── health-check.sh            # 헬스체크
│   └── init-databases.sh          # DB 초기화
│
└── docs/
    ├── SETUP.md                   # 초기 설정 가이드
    └── BACKUP.md                  # 백업/복구 가이드
```

### krgeobuk-deployment (CI/CD)

```
krgeobuk-deployment/
├── README.md
├── .gitignore
│
├── jenkins/
│   ├── Jenkinsfile                # 통합 파이프라인
│   ├── Jenkinsfile.auth-server    # auth-server 전용
│   ├── Jenkinsfile.authz-server   # authz-server 전용
│   │
│   ├── config/
│   │   ├── dev.groovy             # dev 환경 설정
│   │   └── prod.groovy            # prod 환경 설정
│   │
│   └── shared-library/            # Jenkins Shared Library
│       ├── vars/
│       │   ├── buildImage.groovy
│       │   ├── deployToK8s.groovy
│       │   └── notifySlack.groovy
│       └── src/
│
├── argocd/                        # ArgoCD 설정 (Phase 2)
│   ├── install.yaml
│   ├── applications/
│   │   ├── auth-server-dev.yaml
│   │   ├── auth-server-prod.yaml
│   │   ├── authz-server-dev.yaml
│   │   └── authz-server-prod.yaml
│   │
│   └── projects/
│       └── krgeobuk.yaml
│
├── scripts/
│   ├── deploy-dev.sh              # dev 환경 배포
│   ├── deploy-prod.sh             # prod 환경 배포
│   ├── rollback.sh                # 롤백
│   ├── smoke-test.sh              # 배포 후 테스트
│   └── build-all-images.sh        # 모든 이미지 빌드
│
└── docs/
    ├── JENKINS_SETUP.md           # Jenkins 설정 가이드
    └── CI_CD_FLOW.md              # CI/CD 플로우 설명
```

## 인프라 아키텍처

### 네트워크 구성

```yaml
# miniPC IP: 192.168.1.100 (예시)

External Services:
├── krgeobuk-mysql:3306
│   ├── auth_dev DB
│   ├── auth_prod DB
│   ├── authz_dev DB
│   └── authz_prod DB
│
└── krgeobuk-redis:6379
    ├── DB 0 (auth-dev)
    ├── DB 1 (auth-prod)
    ├── DB 2 (authz-dev)
    └── DB 3 (authz-prod)

Kubernetes Services:
├── krgeobuk-dev namespace
│   ├── auth-server:80 → 8000
│   └── authz-server:80 → 8100
│
└── krgeobuk-prod namespace
    ├── auth-server:80 → 8000
    └── authz-server:80 → 8100

Ingress:
├── dev.krgeobuk.com → krgeobuk-dev/*
└── krgeobuk.com → krgeobuk-prod/*
```

### Docker Compose 구성

```yaml
# docker-compose/docker-compose.yaml
version: '3.8'

services:
  krgeobuk-mysql:
    image: mysql:8
    container_name: krgeobuk-mysql
    ports:
      - "3306:3306"
    volumes:
      - ./mysql/init:/docker-entrypoint-initdb.d
      - ./mysql/data:/var/lib/mysql
      - ./mysql/conf:/etc/mysql/conf.d
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    restart: unless-stopped

  krgeobuk-redis:
    image: redis:7-alpine
    container_name: krgeobuk-redis
    ports:
      - "6379:6379"
    volumes:
      - ./redis/data:/data
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    restart: unless-stopped

  jenkins:
    build:
      context: ./jenkins
      dockerfile: Dockerfile
    container_name: jenkins
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - ./jenkins/data:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      JAVA_OPTS: "-Djenkins.install.runSetupWizard=false"
    restart: unless-stopped

  verdaccio:
    image: verdaccio/verdaccio:5
    container_name: verdaccio
    ports:
      - "4873:4873"
    volumes:
      - ./verdaccio/storage:/verdaccio/storage
      - ./verdaccio/config:/verdaccio/conf
    restart: unless-stopped

networks:
  default:
    name: krgeobuk-network
```

### MySQL 초기화 스크립트

```sql
-- mysql/init/01-create-databases.sql

-- 개발 환경 데이터베이스
CREATE DATABASE IF NOT EXISTS auth_dev CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE IF NOT EXISTS authz_dev CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 운영 환경 데이터베이스
CREATE DATABASE IF NOT EXISTS auth_prod CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE IF NOT EXISTS authz_prod CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

```sql
-- mysql/init/02-create-users.sql

-- auth-server 사용자
CREATE USER IF NOT EXISTS 'auth_user'@'%' IDENTIFIED BY 'auth_password';
GRANT ALL PRIVILEGES ON auth_dev.* TO 'auth_user'@'%';
GRANT ALL PRIVILEGES ON auth_prod.* TO 'auth_user'@'%';

-- authz-server 사용자
CREATE USER IF NOT EXISTS 'authz_user'@'%' IDENTIFIED BY 'authz_password';
GRANT ALL PRIVILEGES ON authz_dev.* TO 'authz_user'@'%';
GRANT ALL PRIVILEGES ON authz_prod.* TO 'authz_user'@'%';

FLUSH PRIVILEGES;
```

## Kubernetes 매니페스트 구조

### Base 리소스

#### base/namespace.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: krgeobuk-dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: krgeobuk-prod
```

#### base/external-mysql.yaml
```yaml
# External Service로 Docker MySQL 연결
apiVersion: v1
kind: Service
metadata:
  name: krgeobuk-mysql
  namespace: krgeobuk-dev
spec:
  type: ClusterIP
  ports:
  - port: 3306
    targetPort: 3306
---
apiVersion: v1
kind: Endpoints
metadata:
  name: krgeobuk-mysql
  namespace: krgeobuk-dev
subsets:
- addresses:
  - ip: "192.168.1.100"              # miniPC IP
  ports:
  - port: 3306
---
# prod namespace도 동일하게 생성
apiVersion: v1
kind: Service
metadata:
  name: krgeobuk-mysql
  namespace: krgeobuk-prod
spec:
  type: ClusterIP
  ports:
  - port: 3306
---
apiVersion: v1
kind: Endpoints
metadata:
  name: krgeobuk-mysql
  namespace: krgeobuk-prod
subsets:
- addresses:
  - ip: "192.168.1.100"
  ports:
  - port: 3306
```

#### base/external-redis.yaml
```yaml
# Redis도 동일한 패턴
apiVersion: v1
kind: Service
metadata:
  name: krgeobuk-redis
  namespace: krgeobuk-dev
spec:
  type: ClusterIP
  ports:
  - port: 6379
---
apiVersion: v1
kind: Endpoints
metadata:
  name: krgeobuk-redis
  namespace: krgeobuk-dev
subsets:
- addresses:
  - ip: "192.168.1.100"
  ports:
  - port: 6379
---
# prod도 동일
apiVersion: v1
kind: Service
metadata:
  name: krgeobuk-redis
  namespace: krgeobuk-prod
spec:
  type: ClusterIP
  ports:
  - port: 6379
---
apiVersion: v1
kind: Endpoints
metadata:
  name: krgeobuk-redis
  namespace: krgeobuk-prod
subsets:
- addresses:
  - ip: "192.168.1.100"
  ports:
  - port: 6379
```

### Application 템플릿 (auth-server 예시)

#### applications/auth-server/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-server
spec:
  replicas: 1                      # base 기본값 (최소)
  selector:
    matchLabels:
      app: auth-server
  template:
    metadata:
      labels:
        app: auth-server
    spec:
      initContainers:
      - name: wait-for-database
        image: busybox:1.35
        command:
        - sh
        - -c
        - |
          echo "Waiting for database connections..."
          until nc -z krgeobuk-mysql 3306 && nc -z krgeobuk-redis 6379; do
            echo "Database not ready, waiting..."
            sleep 2
          done
          echo "Database connections successful!"

      containers:
      - name: auth-server
        image: krgeobuk/auth-server:latest
        ports:
        - containerPort: 8000
          name: http
        - containerPort: 8010
          name: tcp

        envFrom:
        - configMapRef:
            name: auth-server-config
        - secretRef:
            name: auth-server-secrets

        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10

        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5

        resources:
          requests:
            cpu: 100m              # base 기본값 (최소)
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

#### applications/auth-server/service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-server
spec:
  type: ClusterIP
  selector:
    app: auth-server
  ports:
  - name: http
    port: 80
    targetPort: 8000
  - name: tcp
    port: 8010
    targetPort: 8010
```

#### applications/auth-server/configmap.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-server-config
data:
  # 기본값 (환경별로 override)
  MYSQL_HOST: "krgeobuk-mysql"
  MYSQL_PORT: "3306"
  REDIS_HOST: "krgeobuk-redis"
  REDIS_PORT: "6379"
  NODE_ENV: "production"
```

## 환경별 설정 전략

### Kustomize Overlay 패턴

#### environments/dev/kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: krgeobuk-dev

resources:
- ../../base
- ../../applications/auth-server
- ../../applications/authz-server
- ../../applications/portal-client

patches:
- path: patches/auth-server-dev.yaml
- path: patches/authz-server-dev.yaml
- path: patches/portal-client-dev.yaml

configMapGenerator:
- name: auth-server-config
  behavior: merge
  literals:
  - MYSQL_DATABASE=auth_dev
  - REDIS_DB=0
  - NODE_ENV=development
  - LOG_LEVEL=debug

- name: authz-server-config
  behavior: merge
  literals:
  - MYSQL_DATABASE=authz_dev
  - REDIS_DB=2
  - NODE_ENV=development
  - LOG_LEVEL=debug
```

#### environments/dev/patches/auth-server-dev.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-server
spec:
  replicas: 1                      # dev는 1개
  template:
    spec:
      containers:
      - name: auth-server
        resources:
          requests:
            cpu: 100m              # dev는 최소 리소스
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

#### environments/prod/kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: krgeobuk-prod

resources:
- ../../base
- ../../applications/auth-server
- ../../applications/authz-server
- ../../applications/portal-client

patches:
- path: patches/auth-server-prod.yaml
- path: patches/authz-server-prod.yaml
- path: patches/portal-client-prod.yaml

configMapGenerator:
- name: auth-server-config
  behavior: merge
  literals:
  - MYSQL_DATABASE=auth_prod
  - REDIS_DB=1
  - NODE_ENV=production
  - LOG_LEVEL=info

- name: authz-server-config
  behavior: merge
  literals:
  - MYSQL_DATABASE=authz_prod
  - REDIS_DB=3
  - NODE_ENV=production
  - LOG_LEVEL=info
```

#### environments/prod/patches/auth-server-prod.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-server
spec:
  replicas: 2                      # prod는 2개
  template:
    spec:
      containers:
      - name: auth-server
        resources:
          requests:
            cpu: 500m              # prod는 더 많은 리소스
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi

      affinity:
        podAntiAffinity:           # 다른 노드에 분산 (향후)
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - auth-server
              topologyKey: kubernetes.io/hostname
```

## 구현 계획

### Phase 1: 리포지토리 생성 및 기본 구조 (1일)

#### 1.1 GitHub 리포지토리 생성
```bash
gh repo create krgeobuk-k8s --public
gh repo create krgeobuk-infrastructure --public
gh repo create krgeobuk-deployment --public
```

#### 1.2 디렉토리 구조 생성
```bash
# krgeobuk-k8s
cd krgeobuk-k8s/
mkdir -p base
mkdir -p applications/{auth-server,authz-server,portal-client}
mkdir -p environments/{dev,prod}/{patches,configmaps}
mkdir -p ingress
mkdir -p docs

# krgeobuk-infrastructure
cd ../krgeobuk-infrastructure/
mkdir -p docker-compose/{mysql/{init,conf,data},redis,jenkins,verdaccio}
mkdir -p backup
mkdir -p scripts
mkdir -p docs

# krgeobuk-deployment
cd ../krgeobuk-deployment/
mkdir -p jenkins/{config,shared-library/vars}
mkdir -p argocd/{applications,projects}
mkdir -p scripts
mkdir -p docs
```

#### 1.3 기본 파일 작성
- [ ] README.md (각 리포)
- [ ] .gitignore
- [ ] .env.example (infrastructure)

### Phase 2: 인프라 설정 (1-2일)

#### 2.1 Docker Compose 구성
- [ ] docker-compose.yaml 작성
- [ ] MySQL 초기화 스크립트 (01-create-databases.sql, 02-create-users.sql)
- [ ] Redis 설정 파일 (redis.conf)
- [ ] Jenkins Dockerfile
- [ ] Verdaccio 설정

#### 2.2 백업 스크립트
- [ ] mysql-backup.sh
- [ ] redis-backup.sh
- [ ] backup-cron
- [ ] restore.sh

#### 2.3 유틸리티 스크립트
- [ ] start-all.sh
- [ ] stop-all.sh
- [ ] health-check.sh
- [ ] init-databases.sh

### Phase 3: Kubernetes 매니페스트 작성 (2-3일)

#### 3.1 Base 리소스
- [ ] base/namespace.yaml
- [ ] base/external-mysql.yaml
- [ ] base/external-redis.yaml
- [ ] base/kustomization.yaml

#### 3.2 Applications (auth-server)
- [ ] applications/auth-server/deployment.yaml
- [ ] applications/auth-server/service.yaml
- [ ] applications/auth-server/configmap.yaml
- [ ] applications/auth-server/secret.yaml.template
- [ ] applications/auth-server/kustomization.yaml

#### 3.3 Applications (authz-server)
- [ ] applications/authz-server/deployment.yaml
- [ ] applications/authz-server/service.yaml
- [ ] applications/authz-server/configmap.yaml
- [ ] applications/authz-server/secret.yaml.template
- [ ] applications/authz-server/kustomization.yaml

#### 3.4 Applications (portal-client)
- [ ] applications/portal-client/deployment.yaml
- [ ] applications/portal-client/service.yaml
- [ ] applications/portal-client/nginx-configmap.yaml
- [ ] applications/portal-client/kustomization.yaml

#### 3.5 Environments (dev)
- [ ] environments/dev/kustomization.yaml
- [ ] environments/dev/patches/auth-server-dev.yaml
- [ ] environments/dev/patches/authz-server-dev.yaml
- [ ] environments/dev/patches/portal-client-dev.yaml

#### 3.6 Environments (prod)
- [ ] environments/prod/kustomization.yaml
- [ ] environments/prod/patches/auth-server-prod.yaml
- [ ] environments/prod/patches/authz-server-prod.yaml
- [ ] environments/prod/patches/portal-client-prod.yaml

#### 3.7 Ingress
- [ ] ingress/ingress.yaml
- [ ] ingress/kustomization.yaml

### Phase 4: CI/CD 파이프라인 구성 (2-3일)

#### 4.1 Jenkins 설정
- [ ] Jenkins 컨테이너 시작
- [ ] 초기 설정 완료
- [ ] GitHub 연동
- [ ] Docker Registry 연동
- [ ] kubectl 설정 (k3s 연결)

#### 4.2 Jenkinsfile 작성
- [ ] Jenkinsfile (통합 파이프라인)
- [ ] Jenkinsfile.auth-server
- [ ] Jenkinsfile.authz-server
- [ ] config/dev.groovy
- [ ] config/prod.groovy

#### 4.3 Shared Library
- [ ] vars/buildImage.groovy
- [ ] vars/deployToK8s.groovy
- [ ] vars/notifySlack.groovy

#### 4.4 배포 스크립트
- [ ] scripts/deploy-dev.sh
- [ ] scripts/deploy-prod.sh
- [ ] scripts/rollback.sh
- [ ] scripts/smoke-test.sh
- [ ] scripts/build-all-images.sh

### Phase 5: 테스트 및 배포 (1-2일)

#### 5.1 로컬 테스트
```bash
# Kustomize 빌드 테스트
kubectl kustomize environments/dev/
kubectl kustomize environments/prod/

# 매니페스트 검증
kubectl apply --dry-run=client -k environments/dev/
kubectl apply --dry-run=client -k environments/prod/
```

#### 5.2 Dev 환경 배포
```bash
# 인프라 시작
cd /opt/krgeobuk/infrastructure/
docker-compose -f docker-compose/docker-compose.yaml up -d

# DB 초기화 확인
docker exec krgeobuk-mysql mysql -u root -p -e "SHOW DATABASES;"

# k8s dev 배포
kubectl apply -k /opt/krgeobuk/k8s/environments/dev/

# 배포 확인
kubectl get pods -n krgeobuk-dev
kubectl logs -f deployment/auth-server -n krgeobuk-dev
```

#### 5.3 Prod 환경 배포
```bash
# dev 검증 후 동일하게 prod 배포
kubectl apply -k /opt/krgeobuk/k8s/environments/prod/
kubectl get pods -n krgeobuk-prod
```

### Phase 6: 모니터링 및 최적화 (선택, 1-2일)

#### 6.1 기본 모니터링
- [ ] kubectl top nodes
- [ ] kubectl top pods
- [ ] 로그 집계 설정

#### 6.2 백업 자동화
- [ ] cron 설정
  ```bash
  0 2 * * * /opt/krgeobuk/infrastructure/backup/mysql-backup.sh
  0 3 * * * /opt/krgeobuk/infrastructure/backup/redis-backup.sh
  ```

## 배포 워크플로우

### 개발 플로우

```bash
# 1. Feature 개발 (애플리케이션)
cd auth-server/
git checkout -b feature/new-oauth
# 코드 수정
git commit -am "Add new OAuth provider"
git push origin feature/new-oauth
# PR → dev 브랜치 merge

# 2. 이미지 빌드 (Jenkins 자동)
# Jenkins가 auth-server dev 브랜치 감지
# Docker 이미지 빌드: krgeobuk/auth-server:dev-abc123

# 3. K8s 배포 (Jenkins 자동)
# Jenkins가 자동으로 실행:
kubectl apply -k /opt/krgeobuk/k8s/environments/dev/

# 4. 검증
kubectl get pods -n krgeobuk-dev
kubectl logs -f deployment/auth-server -n krgeobuk-dev
curl http://auth-server.krgeobuk-dev.svc.cluster.local:8000/health
```

### 운영 배포 플로우

```bash
# 1. dev 검증 완료 후 승격
cd auth-server/
git checkout main
git merge dev
git push origin main

# 2. 이미지 빌드 (Jenkins 자동)
# Docker 이미지 빌드: krgeobuk/auth-server:main-xyz789
# 태그: krgeobuk/auth-server:latest

# 3. K8s 배포 (Jenkins - 수동 승인)
# Jenkins에서 승인 후:
kubectl apply -k /opt/krgeobuk/k8s/environments/prod/

# 4. 검증
kubectl get pods -n krgeobuk-prod
kubectl rollout status deployment/auth-server -n krgeobuk-prod

# 5. 롤백 (필요시)
kubectl rollout undo deployment/auth-server -n krgeobuk-prod
```

### Kustomize 빌드 및 배포

```bash
# dev 환경 빌드 확인
kubectl kustomize environments/dev/

# dev 환경 배포
kubectl apply -k environments/dev/

# prod 환경 배포
kubectl apply -k environments/prod/

# 특정 애플리케이션만 배포
kubectl apply -k applications/auth-server/ -n krgeobuk-dev
```

## 백업 및 재해복구

### MySQL 백업 시스템

```bash
#!/bin/bash
# backup/mysql-backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/opt/krgeobuk/backups/mysql"
RETENTION_DAYS=7

mkdir -p ${BACKUP_DIR}

# 모든 데이터베이스 백업
echo "Starting MySQL backup..."
docker exec krgeobuk-mysql mysqldump \
  -u root -p${MYSQL_ROOT_PASSWORD} \
  --all-databases \
  --single-transaction \
  --routines \
  --triggers \
  --set-gtid-purged=OFF \
  > ${BACKUP_DIR}/all-databases_${DATE}.sql

if [ $? -eq 0 ]; then
  echo "MySQL backup completed: all-databases_${DATE}.sql"
  gzip ${BACKUP_DIR}/all-databases_${DATE}.sql
else
  echo "MySQL backup failed!"
  exit 1
fi

# 오래된 백업 파일 삭제
find ${BACKUP_DIR} -name "*.sql.gz" -mtime +${RETENTION_DAYS} -delete

echo "Backup completed successfully: ${DATE}"
```

### Redis 백업 시스템

```bash
#!/bin/bash
# backup/redis-backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/opt/krgeobuk/backups/redis"
RETENTION_DAYS=7

mkdir -p ${BACKUP_DIR}

# Redis 백업
docker exec krgeobuk-redis redis-cli SAVE
docker cp krgeobuk-redis:/data/dump.rdb ${BACKUP_DIR}/dump_${DATE}.rdb

if [ $? -eq 0 ]; then
  echo "Redis backup completed: dump_${DATE}.rdb"
  gzip ${BACKUP_DIR}/dump_${DATE}.rdb
else
  echo "Redis backup failed!"
  exit 1
fi

# 오래된 백업 삭제
find ${BACKUP_DIR} -name "*.rdb.gz" -mtime +${RETENTION_DAYS} -delete

echo "Redis backup completed: ${DATE}"
```

### 복구 절차

```bash
#!/bin/bash
# backup/restore.sh

BACKUP_FILE=$1
TARGET_TYPE=$2  # mysql or redis

if [ -z "$BACKUP_FILE" ] || [ -z "$TARGET_TYPE" ]; then
  echo "Usage: $0 <backup_file> <mysql|redis>"
  exit 1
fi

if [ "$TARGET_TYPE" == "mysql" ]; then
  echo "Restoring MySQL from $BACKUP_FILE..."
  gunzip -c $BACKUP_FILE | docker exec -i krgeobuk-mysql mysql -u root -p${MYSQL_ROOT_PASSWORD}
  echo "MySQL restore completed!"

elif [ "$TARGET_TYPE" == "redis" ]; then
  echo "Restoring Redis from $BACKUP_FILE..."
  docker stop krgeobuk-redis
  gunzip -c $BACKUP_FILE > /tmp/dump.rdb
  docker cp /tmp/dump.rdb krgeobuk-redis:/data/dump.rdb
  docker start krgeobuk-redis
  echo "Redis restore completed!"
else
  echo "Invalid target type: $TARGET_TYPE"
  exit 1
fi
```

## 전체 타임라인

```
Week 1: 인프라 구축
├── Day 1: 리포지토리 생성, 디렉토리 구조 생성
├── Day 2-3: Docker Compose 구성, miniPC 초기 설정
└── Day 4-5: k3s 설치, 네트워크 구성

Week 2: Kubernetes 매니페스트
├── Day 1-2: Base, Applications 작성
├── Day 3-4: Environments 작성 (dev, prod)
└── Day 5: Ingress, Secret 설정

Week 3: CI/CD 및 배포
├── Day 1-2: Jenkins 설정, Jenkinsfile 작성
├── Day 3-4: Dev 환경 배포 및 테스트
└── Day 5: Prod 환경 배포 및 검증

Week 4: 최적화 및 문서화
├── Day 1-2: 모니터링 설정, 백업 자동화
└── Day 3-5: 문서 작성, 운영 매뉴얼
```

## 다음 단계

### 즉시 시작할 작업

1. **리포지토리 생성**
   ```bash
   gh repo create krgeobuk-k8s --public
   gh repo create krgeobuk-infrastructure --public
   gh repo create krgeobuk-deployment --public
   ```

2. **기본 디렉토리 구조 생성**
   ```bash
   cd krgeobuk-k8s/
   mkdir -p {base,applications/{auth-server,authz-server,portal-client},environments/{dev,prod}/{patches,configmaps},ingress,docs}
   ```

3. **첫 번째 파일 작성**
   - README.md
   - .gitignore
   - base/namespace.yaml

## 참고 자료

- [Kustomize 공식 문서](https://kustomize.io/)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [GitOps Principles](https://opengitops.dev/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)

---

**마지막 업데이트**: 2024-12-21
**버전**: 2.0 (폴더 기반 환경 분리 구조)
