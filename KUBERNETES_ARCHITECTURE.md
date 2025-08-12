# Kubernetes 아키텍처 설계

이 문서는 krgeobuk-infra의 **영구 분리형 하이브리드 쿠버네티스 아키텍처** 설계를 설명합니다.

## 목차

- [아키텍처 개요](#아키텍처-개요)
- [핵심 설계 철학](#핵심-설계-철학)
- [네트워크 아키텍처](#네트워크-아키텍처)
- [외부 데이터베이스 연결](#외부-데이터베이스-연결)
- [서비스별 구성](#서비스별-구성)
- [모니터링 아키텍처](#모니터링-아키텍처)
- [디렉토리 구조](#디렉토리-구조)
- [마이그레이션 계획](#마이그레이션-계획)
- [백업 및 재해복구](#백업-및-재해복구)

## 아키텍처 개요

### 영구 분리형 하이브리드 구조

```
┌─────────────────────────────────────────────────────────────────┐
│ 쿠버네티스 클러스터 (애플리케이션 + 모니터링)                  │
├─────────────────────────────────────────────────────────────────┤
│ Ingress Controller (nginx-ingress)                             │
│ ├── auth.krgeobuk.com → auth-client + auth-server             │
│ ├── portal.krgeobuk.com → portal-client                       │
│ ├── api.krgeobuk.com → API Gateway                             │
│ └── monitoring.krgeobuk.com → Grafana                          │
├─────────────────────────────────────────────────────────────────┤
│ Application Pods                                                │
│ ├── auth-server (NestJS)                                       │
│ ├── authz-server (NestJS)                                      │  
│ ├── portal-client (nginx + static)                             │
│ └── auth-client (nginx + static)                               │
├─────────────────────────────────────────────────────────────────┤
│ Monitoring Stack                                                │
│ ├── Prometheus (메트릭 수집)                                   │
│ ├── Grafana (대시보드)                                         │
│ └── AlertManager (알림)                                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓ (네트워크 연결)
┌─────────────────────────────────────────────────────────────────┐
│ 외부 데이터 서버 (영구 분리)                                   │
├─────────────────────────────────────────────────────────────────┤
│ 기존 Docker Compose 인프라 활용                                │
│ ├── auth-mysql:3307, auth-redis:6380                          │
│ ├── authz-mysql:3308, authz-redis:6381                        │
│ ├── 백업 시스템 (cron + rsync)                                │
│ └── 모니터링 에이전트 (node-exporter, mysql-exporter)         │
└─────────────────────────────────────────────────────────────────┘
```

## 핵심 설계 철학

### 1. 데이터 안정성 극대화
- DB/Redis가 쿠버네티스 라이프사이클과 독립적
- 전통적인 DB 관리 도구 및 절차 그대로 활용
- 백업/복구 절차 단순화

### 2. 운영 복잡도 최소화
- 애플리케이션 스케일링과 데이터 관리 분리
- DB 성능 튜닝 및 유지보수 독립적 수행
- 장애 영향 범위 최소화

### 3. 기존 인프라 활용
- 검증된 Docker Compose 설정 100% 재사용
- 기존 포트 체계 및 네트워크 구조 유지
- 점진적 마이그레이션으로 위험 최소화

## 네트워크 아키텍처

### 도메인 기반 라우팅

```yaml
# 서브도메인 분리 구조
auth.krgeobuk.com        → auth-client (/) + auth-server (/api)
portal.krgeobuk.com      → portal-client (/) + portal-server (/api)
monitoring.krgeobuk.com  → Grafana 대시보드
api.krgeobuk.com         → API Gateway (통합 API 엔드포인트)
```

### Ingress 설정

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: krgeobuk-gateway
  namespace: krgeobuk
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  tls:
  - hosts:
    - auth.krgeobuk.com
    - portal.krgeobuk.com
    - monitoring.krgeobuk.com
    secretName: krgeobuk-tls
  rules:
  # 인증 서비스
  - host: auth.krgeobuk.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: auth-server
            port: { number: 80 }
      - path: /
        pathType: Prefix
        backend:
          service:
            name: auth-client
            port: { number: 80 }
  
  # 포털 서비스
  - host: portal.krgeobuk.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: portal-server
            port: { number: 80 }
      - path: /
        pathType: Prefix
        backend:
          service:
            name: portal-client
            port: { number: 80 }
  
  # 모니터링
  - host: monitoring.krgeobuk.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port: { number: 80 }
```

### 네트워크 보안

```yaml
# 외부 DB 접근 제어
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-access-policy
  namespace: krgeobuk
spec:
  podSelector:
    matchLabels:
      database-access: "true"
  policyTypes:
  - Egress
  egress:
  - to: []  # 외부 DB 서버
    ports:
    - protocol: TCP
      port: 3307  # auth-mysql
    - protocol: TCP
      port: 3308  # authz-mysql
    - protocol: TCP
      port: 6380  # auth-redis
    - protocol: TCP
      port: 6381  # authz-redis
```

## 외부 데이터베이스 연결

### External Service 패턴

```yaml
# auth-server 외부 MySQL 연결
apiVersion: v1
kind: Service
metadata:
  name: auth-mysql
  namespace: krgeobuk
spec:
  type: ClusterIP
  ports:
  - port: 3306
    targetPort: 3307
    protocol: TCP
---
apiVersion: v1
kind: Endpoints
metadata:
  name: auth-mysql
  namespace: krgeobuk
subsets:
- addresses:
  - ip: "192.168.1.100"  # 외부 DB 서버 IP
  ports:
  - port: 3307
    protocol: TCP
---
# auth-server 외부 Redis 연결
apiVersion: v1
kind: Service
metadata:
  name: auth-redis
  namespace: krgeobuk
spec:
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6380
    protocol: TCP
---
apiVersion: v1
kind: Endpoints
metadata:
  name: auth-redis
  namespace: krgeobuk
subsets:
- addresses:
  - ip: "192.168.1.100"  # 외부 Redis 서버 IP
  ports:
  - port: 6380
    protocol: TCP
```

### 환경 변수 관리

```yaml
# ConfigMap + Secret 조합
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-server-config
  namespace: krgeobuk
data:
  MYSQL_HOST: "auth-mysql"
  MYSQL_PORT: "3306"
  MYSQL_DATABASE: "auth"
  REDIS_HOST: "auth-redis"
  REDIS_PORT: "6379"
  NODE_ENV: "production"
  # DB 연결 풀 최적화
  DB_CONNECTION_POOL_MIN: "5"
  DB_CONNECTION_POOL_MAX: "20"
  DB_CONNECTION_TIMEOUT: "30000"
  DB_IDLE_TIMEOUT: "600000"
  # Redis 연결 최적화
  REDIS_CONNECTION_POOL_SIZE: "10"
  REDIS_CONNECTION_TIMEOUT: "5000"
---
apiVersion: v1
kind: Secret
metadata:
  name: auth-server-secrets
  namespace: krgeobuk
type: Opaque
stringData:
  MYSQL_PASSWORD: "your-secure-password"
  REDIS_PASSWORD: "your-redis-password"
  JWT_ACCESS_PRIVATE_KEY: |
    -----BEGIN RSA PRIVATE KEY-----
    ...
    -----END RSA PRIVATE KEY-----
  JWT_REFRESH_PRIVATE_KEY: |
    -----BEGIN RSA PRIVATE KEY-----
    ...
    -----END RSA PRIVATE KEY-----
```

## 서비스별 구성

### 백엔드 서비스 (auth-server, authz-server)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-server
  namespace: krgeobuk
spec:
  replicas: 2
  selector:
    matchLabels:
      app: auth-server
  template:
    metadata:
      labels:
        app: auth-server
        database-access: "true"  # NetworkPolicy 라벨
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
    spec:
      # 외부 DB 연결 대기
      initContainers:
      - name: wait-for-database
        image: busybox:1.35
        command:
        - sh
        - -c
        - |
          echo "Waiting for database connections..."
          until nc -z auth-mysql 3306 && nc -z auth-redis 6379; do
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
        
        # 환경 변수
        envFrom:
        - configMapRef:
            name: auth-server-config
        - secretRef:
            name: auth-server-secrets
        
        # 헬스체크
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
        
        # 리소스 제한
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: auth-server
  namespace: krgeobuk
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

### 프론트엔드 서비스 (portal-client, auth-client)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portal-client
  namespace: krgeobuk
spec:
  replicas: 2
  selector:
    matchLabels:
      app: portal-client
  template:
    metadata:
      labels:
        app: portal-client
    spec:
      containers:
      - name: portal-client
        image: krgeobuk/portal-client:latest
        ports:
        - containerPort: 80
        
        # nginx 설정
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 100m
            memory: 128Mi
      
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: krgeobuk
data:
  nginx.conf: |
    events {
        worker_connections 1024;
    }
    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;
        
        server {
            listen 80;
            root /usr/share/nginx/html;
            index index.html;
            
            # SPA 라우팅 지원
            location / {
                try_files $uri $uri/ /index.html;
            }
            
            # 정적 파일 캐싱
            location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
                expires 1y;
                add_header Cache-Control "public, immutable";
            }
            
            # gzip 압축
            gzip on;
            gzip_types text/css application/javascript application/json;
        }
    }
```

## 모니터링 아키텍처

### Prometheus 설정

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    scrape_configs:
    # 쿠버네티스 애플리케이션 메트릭
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ['krgeobuk']
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
    
    # 외부 DB 서버 메트릭 (영구 분리)
    - job_name: 'external-databases'
      static_configs:
      - targets:
        - '192.168.1.100:9104'  # MySQL Exporter
        - '192.168.1.100:9121'  # Redis Exporter
        labels:
          service: 'database-infrastructure'
    
    # 외부 서버 시스템 메트릭
    - job_name: 'external-system'
      static_configs:
      - targets:
        - '192.168.1.100:9100'  # Node Exporter
        labels:
          service: 'database-server'
    
    # Kubernetes 클러스터 메트릭
    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
    
    - job_name: 'kubernetes-services'
      kubernetes_sd_configs:
      - role: service
```

### Grafana 대시보드

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
data:
  krgeobuk-overview.json: |
    {
      "dashboard": {
        "title": "krgeobuk 전체 시스템 개요",
        "panels": [
          {
            "title": "애플리케이션 상태",
            "type": "stat",
            "targets": [
              {
                "expr": "up{job=~'kubernetes-pods'}"
              }
            ]
          },
          {
            "title": "외부 데이터베이스 상태",
            "type": "stat", 
            "targets": [
              {
                "expr": "up{job='external-databases'}"
              }
            ]
          },
          {
            "title": "API 응답시간",
            "type": "graph",
            "targets": [
              {
                "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))"
              }
            ]
          }
        ]
      }
    }
```

## 디렉토리 구조

```
krgeobuk-infra/
├── k8s/                              # 쿠버네티스 매니페스트
│   ├── base/
│   │   ├── namespace.yaml
│   │   ├── networkpolicy.yaml
│   │   └── external-services.yaml    # 외부 DB 연결 정의
│   ├── applications/
│   │   ├── auth-server/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── configmap.yaml
│   │   │   └── secrets.yaml.template
│   │   ├── authz-server/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── configmap.yaml
│   │   │   └── secrets.yaml.template
│   │   ├── portal-client/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── nginx-config.yaml
│   │   └── auth-client/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── nginx-config.yaml
│   ├── monitoring/
│   │   ├── namespace.yaml
│   │   ├── prometheus/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── configmap.yaml
│   │   │   └── rbac.yaml
│   │   ├── grafana/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── configmap.yaml
│   │   │   └── dashboards.yaml
│   │   └── alertmanager/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── configmap.yaml
│   ├── ingress/
│   │   └── gateway.yaml
│   └── environments/
│       ├── development/
│       │   ├── kustomization.yaml
│       │   └── patches/
│       ├── staging/
│       │   ├── kustomization.yaml
│       │   └── patches/
│       └── production/
│           ├── kustomization.yaml
│           └── patches/
├── infrastructure/                   # 외부 DB/Redis (영구 유지)
│   ├── auth-infrastructure.yaml      # auth-server DB/Redis
│   ├── authz-infrastructure.yaml     # authz-server DB/Redis
│   ├── monitoring-agents.yaml        # 외부 서버 모니터링
│   ├── backup/
│   │   ├── mysql-backup.sh
│   │   ├── redis-backup.sh
│   │   ├── backup-schedule.cron
│   │   └── restore-procedures.md
│   ├── maintenance/
│   │   ├── db-maintenance.sh
│   │   ├── health-check.sh
│   │   └── performance-tuning.md
│   └── security/
│       ├── firewall-rules.sh
│       └── ssl-certificates.md
├── deployment/
│   ├── jenkins/
│   │   ├── Jenkinsfile.applications   # 애플리케이션 배포
│   │   ├── Jenkinsfile.monitoring     # 모니터링 배포
│   │   └── pipeline-config/
│   ├── scripts/
│   │   ├── deploy-applications.sh
│   │   ├── deploy-monitoring.sh
│   │   ├── setup-external-db.sh
│   │   ├── health-check-full.sh
│   │   └── rollback.sh
│   ├── helm/                         # Helm Charts
│   │   ├── krgeobuk-apps/
│   │   │   ├── Chart.yaml
│   │   │   ├── values.yaml
│   │   │   └── templates/
│   │   └── monitoring-stack/
│   │       ├── Chart.yaml
│   │       ├── values.yaml
│   │       └── templates/
│   └── docker/
│       ├── build-all.sh
│       └── registry-setup.sh
└── docs/
    ├── DEPLOYMENT_GUIDE.md          # 배포 가이드
    ├── EXTERNAL_DB_SETUP.md         # 외부 DB 설정 가이드
    ├── MONITORING_SETUP.md          # 모니터링 설정 가이드
    ├── BACKUP_RECOVERY.md            # 백업/복구 절차
    ├── TROUBLESHOOTING.md            # 문제 해결 가이드
    └── SECURITY_BEST_PRACTICES.md   # 보안 모범 사례
```

## 마이그레이션 계획

### Phase 1: 애플리케이션 쿠버네티스 이전 (2-3주)

#### 1주차: 인프라 준비
1. **쿠버네티스 클러스터 구축**
   - 노드 구성 및 네트워크 설정
   - Ingress Controller 설치
   - cert-manager 설치 (SSL 인증서 자동 관리)

2. **외부 DB 연결 설정**
   - External Service 및 Endpoints 구성
   - 네트워크 보안 정책 설정
   - 연결 테스트 및 검증

3. **Docker 이미지 빌드 및 Registry 설정**
   - 각 서비스별 프로덕션 이미지 빌드
   - Container Registry 구성
   - 이미지 푸시 및 검증

#### 2주차: 서비스 배포
1. **백엔드 서비스 배포**
   - auth-server, authz-server 순차 배포
   - ConfigMap, Secret 설정
   - 헬스체크 및 로드밸런싱 확인

2. **프론트엔드 서비스 배포**
   - portal-client, auth-client 배포
   - nginx 설정 최적화
   - 정적 파일 서빙 테스트

3. **Ingress 및 도메인 설정**
   - 도메인별 라우팅 구성
   - SSL 인증서 설정
   - DNS 설정 및 검증

#### 3주차: 통합 테스트 및 최적화
1. **통합 테스트**
   - 전체 시스템 기능 테스트
   - 성능 테스트 및 튜닝
   - 장애 시나리오 테스트

2. **모니터링 기본 설정**
   - 애플리케이션 메트릭 수집
   - 기본 알림 설정
   - 로그 수집 및 분석

3. **운영 절차 수립**
   - 배포 절차 문서화
   - 장애 대응 절차 수립
   - 백업 검증

### Phase 2: 모니터링 통합 (3-4개월 후)

#### 1단계: 모니터링 스택 구축
1. **Prometheus + Grafana 배포**
   - Helm Chart를 이용한 모니터링 스택 설치
   - 외부 DB 메트릭 수집 설정
   - 대시보드 구성

2. **통합 모니터링 설정**
   - 애플리케이션 + 인프라 통합 대시보드
   - 알림 규칙 설정
   - 로그 집계 시스템 구축

#### 2단계: 고도화
1. **관찰 가능성 강화**
   - 분산 추적 시스템 도입
   - 고급 메트릭 분석
   - 예측 모니터링

2. **자동화 개선**
   - 자동 스케일링 설정
   - 자동 복구 시스템
   - 지능형 알림 시스템

## 백업 및 재해복구

### 외부 DB 백업 시스템

```bash
#!/bin/bash
# infrastructure/backup/mysql-backup.sh

# 환경 변수 설정
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/mysql"
RETENTION_DAYS=7

# 백업 디렉토리 생성
mkdir -p ${BACKUP_DIR}

# auth 데이터베이스 백업
echo "Starting auth database backup..."
docker exec auth-mysql mysqldump \
  -u root -p${MYSQL_ROOT_PASSWORD} \
  --single-transaction \
  --routines \
  --triggers \
  --set-gtid-purged=OFF \
  auth > ${BACKUP_DIR}/auth_${DATE}.sql

if [ $? -eq 0 ]; then
  echo "Auth database backup completed: auth_${DATE}.sql"
else
  echo "Auth database backup failed!"
  exit 1
fi

# authz 데이터베이스 백업
echo "Starting authz database backup..."
docker exec authz-mysql mysqldump \
  -u root -p${MYSQL_ROOT_PASSWORD} \
  --single-transaction \
  --routines \
  --triggers \
  --set-gtid-purged=OFF \
  authz > ${BACKUP_DIR}/authz_${DATE}.sql

if [ $? -eq 0 ]; then
  echo "Authz database backup completed: authz_${DATE}.sql"
else
  echo "Authz database backup failed!"
  exit 1
fi

# 압축
gzip ${BACKUP_DIR}/auth_${DATE}.sql
gzip ${BACKUP_DIR}/authz_${DATE}.sql

# 오래된 백업 파일 삭제
find ${BACKUP_DIR} -name "*.sql.gz" -mtime +${RETENTION_DAYS} -delete

# 백업 상태 알림
echo "Backup completed successfully: ${DATE}"
echo "Available backups:"
ls -la ${BACKUP_DIR}/*.sql.gz | tail -5
```

### Redis 백업 시스템

```bash
#!/bin/bash
# infrastructure/backup/redis-backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/redis"
RETENTION_DAYS=7

mkdir -p ${BACKUP_DIR}

# Redis auth 백업
docker exec auth-redis redis-cli --rdb /data/dump_auth_${DATE}.rdb
docker cp auth-redis:/data/dump_auth_${DATE}.rdb ${BACKUP_DIR}/

# Redis authz 백업
docker exec authz-redis redis-cli --rdb /data/dump_authz_${DATE}.rdb
docker cp authz-redis:/data/dump_authz_${DATE}.rdb ${BACKUP_DIR}/

# 압축 및 정리
gzip ${BACKUP_DIR}/dump_auth_${DATE}.rdb
gzip ${BACKUP_DIR}/dump_authz_${DATE}.rdb

# 오래된 백업 삭제
find ${BACKUP_DIR} -name "*.rdb.gz" -mtime +${RETENTION_DAYS} -delete

echo "Redis backup completed: ${DATE}"
```

### 복구 절차

```bash
#!/bin/bash
# infrastructure/backup/restore-mysql.sh

BACKUP_FILE=$1
TARGET_DB=$2

if [ -z "$BACKUP_FILE" ] || [ -z "$TARGET_DB" ]; then
  echo "Usage: $0 <backup_file> <target_database>"
  echo "Example: $0 auth_20241201_120000.sql.gz auth"
  exit 1
fi

# 백업 파일 압축 해제
if [[ $BACKUP_FILE == *.gz ]]; then
  gunzip -c $BACKUP_FILE > temp_restore.sql
  RESTORE_FILE="temp_restore.sql"
else
  RESTORE_FILE=$BACKUP_FILE
fi

# 데이터베이스 복구
if [ "$TARGET_DB" == "auth" ]; then
  CONTAINER="auth-mysql"
elif [ "$TARGET_DB" == "authz" ]; then
  CONTAINER="authz-mysql"
else
  echo "Invalid target database: $TARGET_DB"
  exit 1
fi

echo "Restoring $TARGET_DB database from $BACKUP_FILE..."
docker exec -i $CONTAINER mysql -u root -p${MYSQL_ROOT_PASSWORD} $TARGET_DB < $RESTORE_FILE

if [ $? -eq 0 ]; then
  echo "Database restore completed successfully!"
else
  echo "Database restore failed!"
  exit 1
fi

# 임시 파일 정리
rm -f temp_restore.sql
```

## 장점 및 특징

### 1. 데이터 안정성 극대화
- **독립적 라이프사이클**: DB/Redis가 쿠버네티스와 독립적으로 관리
- **검증된 백업 시스템**: 전통적인 DB 백업 도구 활용
- **장애 격리**: 애플리케이션 장애가 데이터에 영향 없음

### 2. 운영 복잡도 최소화
- **기존 인프라 재사용**: Docker Compose 설정 100% 활용
- **분리된 관리**: 애플리케이션 스케일링과 DB 관리 독립
- **점진적 도입**: 위험 최소화된 단계별 마이그레이션

### 3. 비용 효율성
- **리소스 최적화**: DB 서버 전용 최적화 가능
- **스토리지 절약**: 쿠버네티스 스토리지 비용 절감
- **인프라 투자 보호**: 기존 투자 100% 활용

### 4. 확장성 및 유연성
- **서비스별 독립 스케일링**: 애플리케이션 레이어 유연한 확장
- **기술 스택 다양성**: 서비스별 최적 기술 선택 가능
- **미래 확장 대비**: 필요시 데이터 레이어도 k8s로 이전 가능

---

**다음 단계**: Phase 1 준비 작업부터 시작하여 단계별로 진행하며, 각 단계 완료 후 충분한 검증을 거쳐 다음 단계로 진행합니다.