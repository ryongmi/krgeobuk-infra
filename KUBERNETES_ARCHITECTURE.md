# Kubernetes 아키텍처 설계

이 문서는 krgeobuk-infra의 Docker Compose에서 Kubernetes로의 마이그레이션 계획을 설명합니다.

## 목차

- [현재 상황 vs 목표 아키텍처](#현재-상황-vs-목표-아키텍처)
- [Kubernetes 네트워크 아키텍처](#kubernetes-네트워크-아키텍처)
- [서비스별 구성 변화](#서비스별-구성-변화)
- [Docker 이미지 빌드 전략](#docker-이미지-빌드-전략)
- [파일 구조 변화](#파일-구조-변화)
- [모니터링 아키텍처](#모니터링-아키텍처)
- [마이그레이션 계획](#마이그레이션-계획)

## 현재 상황 vs 목표 아키텍처

### 현재 Docker Compose 구조
```
Gateway Nginx (최상단)
├── auth-server nginx → auth-server:8000
├── authz-server nginx → authz-server:8100  
├── portal-client nginx → portal-client:3000
└── portal-server nginx → portal-server:8200

각 서비스별 독립적인 MySQL, Redis 인스턴스
```

### 목표 Kubernetes 구조
```
Internet
    ↓ (80, 443만 외부 노출)
Ingress Controller (nginx-ingress)
    ├── /api/auth → auth-server Service → auth-server Pods
    ├── /api/authz → authz-server Service → authz-server Pods
    ├── /api/portal → portal-server Service → portal-server Pods
    ├── / → portal-client Service → portal-client Pods
    ├── /grafana → grafana Service → grafana Pods
    └── /prometheus → prometheus Service → prometheus Pods (선택적)

Cluster 내부 (외부 접근 불가):
├── MySQL Services & Pods (각 서비스별)
├── Redis Services & Pods (각 서비스별)
└── Monitoring Stack
```

## Kubernetes 네트워크 아키텍처

### 네트워크 보안 모델
- **외부 노출**: Ingress Controller만 80, 443 포트 노출
- **내부 통신**: 모든 서비스는 ClusterIP로 클러스터 내부에서만 접근
- **Zero Trust**: Pod 간 통신은 네트워크 정책으로 제어

### 도메인 및 라우팅
```yaml
# krgeobuk.com 기반 단일 도메인 구조
krgeobuk.com/api/auth/*     → auth-server
krgeobuk.com/api/authz/*    → authz-server  
krgeobuk.com/api/portal/*   → portal-server
krgeobuk.com/*              → portal-client (SPA)
krgeobuk.com/grafana/*      → grafana (모니터링)
```

### Ingress 설정 예시
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: krgeobuk-gateway
spec:
  rules:
  - host: krgeobuk.com
    http:
      paths:
      # API 엔드포인트들 (높은 우선순위)
      - path: /api/auth
        pathType: Prefix
        backend:
          service:
            name: auth-server
            port:
              number: 80
      - path: /api/authz
        pathType: Prefix
        backend:
          service:
            name: authz-server
            port:
              number: 80
      - path: /api/portal
        pathType: Prefix
        backend:
          service:
            name: portal-server
            port:
              number: 80
      # 프론트엔드 (낮은 우선순위, 모든 나머지 경로)
      - path: /
        pathType: Prefix
        backend:
          service:
            name: portal-client
            port:
              number: 80
```

## 서비스별 구성 변화

### 백엔드 서비스 (auth-server, authz-server, portal-server)

**변화 내용:**
- ✅ **Dockerfile**: 유지 (로컬 개발 + CI/CD용)
- ❌ **개별 nginx**: 제거 (Kubernetes Service로 대체)
- ❌ **docker-compose.prod.yaml**: 제거
- ✅ **docker-compose.yaml**: 로컬 개발용으로 유지

**Dockerfile 멀티스테이지 구조:**
```dockerfile
FROM node:23-alpine AS deps
FROM node:23-alpine AS build  
FROM node:23-alpine AS local      # 로컬 개발용
FROM node:23-alpine AS development # CI/CD 테스트용
FROM node:23-alpine AS production  # Kubernetes 운영용
```

**Service 설정:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-server
spec:
  type: ClusterIP  # 클러스터 내부에서만 접근
  selector:
    app: auth-server
  ports:
  - port: 80
    targetPort: 8000
```

### 프론트엔드 서비스 (portal-client)

**nginx 역할 변화:**
- **기존**: 복잡한 프록시 설정 + 정적 파일 서빙
- **변경**: 단순한 정적 파일 서빙만

**간소화된 nginx.conf:**
```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;
    
    # SPA 라우팅 지원 (핵심 기능)
    location / {
        try_files $uri $uri/ /index.html;
    }
    
    # 정적 파일 캐싱
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
    
    gzip on;
    gzip_types text/css application/javascript application/json;
}
```

### 데이터베이스 및 인프라 서비스

**MySQL, Redis 구성:**
- Dockerfile 불필요 (공식 이미지 사용)
- 각 서비스별 독립적인 인스턴스 유지
- PersistentVolume으로 데이터 영속성 보장

```yaml
# MySQL 예시
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-mysql
spec:
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_DATABASE
          value: auth
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
```

## Docker 이미지 빌드 전략

### 현재 방식 (Docker Compose)
```bash
# 매번 로컬에서 빌드
docker-compose -f docker-compose.prod.yaml up --build
```

### 새로운 방식 (CI/CD + Kubernetes)
```bash
# CI/CD 파이프라인에서 한 번 빌드
docker build --target production -t krgeobuk/auth-server:v1.0.0 .
docker push registry/krgeobuk/auth-server:v1.0.0

# Kubernetes에서 빌드된 이미지 사용
kubectl apply -f k8s/auth-server-deployment.yml
```

### 장점
1. **성능**: 배포 시 빌드 과정 없음
2. **일관성**: 동일한 이미지를 모든 환경에서 사용
3. **롤백**: 이전 버전으로 즉시 롤백 가능
4. **확장성**: 멀티 인스턴스 배포 시 이미지 재사용

## 파일 구조 변화

### 현재 구조
```
krgeobuk-infra/
├── auth-server/
│   ├── docker-compose.yaml
│   ├── docker-compose.prod.yaml  # 제거 예정
│   └── nginx/                    # 제거 예정
├── authz-server/
│   ├── docker-compose.yaml  
│   ├── docker-compose.prod.yaml  # 제거 예정
│   └── nginx/                    # 제거 예정
└── portal-client/
    ├── docker-compose.yaml
    ├── docker-compose.prod.yaml  # 제거 예정
    └── nginx.conf                # 간소화
```

### 목표 구조
```
krgeobuk-infra/
├── k8s/                          # 새로 생성
│   ├── auth-server/
│   │   ├── deployment.yml
│   │   ├── service.yml
│   │   ├── mysql-deployment.yml
│   │   └── redis-deployment.yml
│   ├── authz-server/
│   ├── portal-client/
│   ├── portal-server/
│   ├── ingress.yml
│   └── monitoring/
│       ├── prometheus.yml
│       └── grafana.yml
├── auth-server/
│   ├── Dockerfile                # 유지
│   ├── docker-compose.yaml       # 로컬용으로 유지
│   └── envs/
├── authz-server/
├── portal-client/
│   ├── Dockerfile                # 유지  
│   ├── docker-compose.yaml       # 로컬용으로 유지
│   └── nginx.conf                # 간소화
└── shared-lib/
```

## 모니터링 아키텍처

### Prometheus + Grafana 스택

**구성 요소:**
- **Prometheus**: 메트릭 수집 및 저장
- **Grafana**: 시각화 대시보드
- **AlertManager**: 알림 관리
- **Node Exporter**: 노드 메트릭 수집

**배포 방법:**
```bash
# Helm Chart 사용 (권장)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack
```

**모니터링 대상:**
1. **인프라 메트릭**: CPU, 메모리, 디스크, 네트워크
2. **Kubernetes 메트릭**: Pod, Service, Ingress 상태
3. **애플리케이션 메트릭**: API 응답시간, 에러율, 처리량
4. **데이터베이스 메트릭**: 연결 수, 쿼리 성능

**접근 경로:**
```
krgeobuk.com/grafana     → Grafana 대시보드
krgeobuk.com/prometheus  → Prometheus (선택적, 디버그용)
```

### 애플리케이션 메트릭 설정

**NestJS 서버에 메트릭 엔드포인트 추가:**
```typescript
// 공통: @krgeobuk/core 패키지에 추가 예정
@Controller('metrics')
export class MetricsController {
  @Get()
  async getMetrics(): Promise<string> {
    return register.metrics();
  }
}
```

## 마이그레이션 계획

### Phase 1: 준비 단계 (2주)
1. **Kubernetes 매니페스트 작성**
   - 각 서비스별 Deployment, Service 파일
   - Ingress 설정
   - ConfigMap, Secret 설정

2. **CI/CD 파이프라인 구축**  
   - Docker 이미지 자동 빌드
   - Container Registry 설정
   - Kubernetes 배포 파이프라인

3. **로컬 테스트 환경 구축**
   - minikube 또는 kind를 이용한 로컬 클러스터
   - 전체 스택 테스트

### Phase 2: 개발환경 적용 (1주)
1. **개발 클러스터 구축**
2. **서비스별 순차 배포**
   - shared-lib → auth-server → authz-server → portal-client
3. **통합 테스트 및 디버깅**

### Phase 3: 모니터링 구축 (1주)  
1. **Prometheus + Grafana 배포**
2. **애플리케이션 메트릭 설정**
3. **알림 설정 및 대시보드 구성**

### Phase 4: 운영환경 적용 (1주)
1. **프로덕션 클러스터 구축**
2. **데이터 마이그레이션**
3. **DNS 및 도메인 설정**
4. **무중단 배포 테스트**

### Phase 5: 최적화 (지속적)
1. **성능 모니터링 및 튜닝**
2. **보안 강화 (네트워크 정책, RBAC)**
3. **백업 및 재해복구 계획**

## 기대 효과

### 리소스 효율성
- **nginx 컨테이너 3개 제거**: 백엔드 서비스별 nginx 불필요
- **중앙집중식 라우팅**: Ingress Controller 1개로 모든 트래픽 처리
- **자동 스케일링**: HPA(Horizontal Pod Autoscaler) 적용 가능

### 운영 효율성
- **무중단 배포**: Rolling Update 지원
- **자동 복구**: Pod 장애 시 자동 재시작
- **일관된 환경**: 개발/스테이징/운영 환경 일치

### 보안 강화
- **네트워크 격리**: Pod 간 통신 제어
- **최소 권한**: RBAC을 통한 세밀한 권한 관리
- **외부 노출 최소화**: 80, 443 포트만 외부 노출

### 확장성
- **마이크로서비스 추가**: 새 서비스 배포 용이
- **글로벌 배포**: 멀티 리전 클러스터 구성 가능
- **서비스 메시**: Istio 등 추가 도구 적용 가능

---

**다음 단계**: Phase 1부터 순차적으로 진행하며, 각 단계별 완료 후 다음 단계로 이동하는 것을 권장합니다.