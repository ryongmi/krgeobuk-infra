# krgeobuk-infra

Git 서브모듈로 관리하는 마이크로서비스 생태계입니다.
인증·권한·포털·My Pick 서비스와 공유 라이브러리, 인프라 리포지토리로 구성되어 있으며 각 서비스는 독립적으로 개발 및 배포됩니다.

---

## 리포지토리 구조

```
krgeobuk-infra/
├── auth-server/              # 인증 서비스 (OAuth, JWT)
├── authz-server/             # 권한 관리 서비스 (RBAC)
├── auth-client/              # 인증 UI 클라이언트
├── portal-server/            # 포털 백엔드 서비스
├── portal-client/            # 포털 사용자 클라이언트
├── portal-admin-client/      # 포털 관리자 클라이언트
├── my-pick-server/           # My Pick 백엔드 서비스
├── my-pick-client/           # My Pick 사용자 클라이언트
├── my-pick-admin-client/     # My Pick 관리자 클라이언트
├── shared-lib/               # 공유 라이브러리 모노레포 (23개 @krgeobuk/* 패키지)
├── krgeobuk-deployment/      # CI/CD 파이프라인, Jenkins·Verdaccio K8s 배포
├── krgeobuk-infrastructure/  # Docker Compose 로컬 인프라 (MySQL, Redis)
├── krgeobuk-k8s/             # Kubernetes 매니페스트 (Kustomize)
├── docs/                     # 아키텍처·가이드 문서
├── CLAUDE.md
└── README.md
```

---

## 서브모듈 목록

### 애플리케이션 서비스

| 서브모듈 | 설명 | 기술 스택 | 포트 |
|---|---|---|---|
| `auth-server` | 인증 서비스 (OAuth, JWT, 사용자 인증) | NestJS, MySQL 8, Redis | 8000 |
| `authz-server` | 권한 관리 서비스 (RBAC, 역할·권한) | NestJS, MySQL 8, Redis | 8100 |
| `auth-client` | 인증 UI 클라이언트 | Next.js 15, Tailwind CSS | 3001 |
| `portal-server` | 포털 백엔드 서비스 | NestJS | - |
| `portal-client` | 포털 사용자 클라이언트 | Next.js 15, Tailwind CSS | 3000 |
| `portal-admin-client` | 포털 관리자 클라이언트 | Next.js 15, Tailwind CSS | 3002 |
| `my-pick-server` | My Pick 백엔드 서비스 | NestJS | - |
| `my-pick-client` | My Pick 사용자 클라이언트 | Next.js 15, Tailwind CSS | - |
| `my-pick-admin-client` | My Pick 관리자 클라이언트 | Next.js 15, Tailwind CSS | - |

### 공유·인프라

| 서브모듈 | 설명 |
|---|---|
| `shared-lib` | 공유 라이브러리 모노레포 (pnpm 워크스페이스, 23개 `@krgeobuk/*` 패키지) |
| `krgeobuk-deployment` | Jenkins CI/CD 파이프라인, Jenkins·Verdaccio K8s 배포 매니페스트 |
| `krgeobuk-infrastructure` | Docker Compose 로컬 인프라 (MySQL, Redis, K8s Dashboard) |
| `krgeobuk-k8s` | 서비스별 Kubernetes 매니페스트 (Kustomize dev/prod 오버레이) |

---

## 기술 스택

| 분류 | 기술 |
|---|---|
| 백엔드 프레임워크 | NestJS 10 |
| 프론트엔드 프레임워크 | Next.js 15 |
| 언어 | TypeScript (ESM) |
| 데이터베이스 | MySQL 8 (서비스별 독립) |
| 캐시·세션 | Redis |
| 컨테이너 | Docker, Kubernetes (k3s) |
| 패키지 관리 | pnpm (공유 라이브러리), npm (각 서비스) |
| NPM 레지스트리 | Verdaccio (K8s 운영, `krgeobuk-deployment`에서 관리) |
| CI/CD | Jenkins (K8s 운영) |

---

## 초기 설정

### 1. 저장소 클론 (서브모듈 포함)

```bash
git clone --recursive https://github.com/ryongmi/krgeobuk-infra.git
cd krgeobuk-infra
```

이미 클론한 경우 서브모듈 초기화:

```bash
git submodule update --init --recursive
```

### 2. 로컬 인프라 시작

MySQL, Redis 등 로컬 개발용 인프라를 시작합니다.

```bash
cd krgeobuk-infrastructure
npm run docker:local:up
```

### 3. 공유 라이브러리 빌드

```bash
cd shared-lib
pnpm install
pnpm build
```

### 4. 백엔드 서비스 시작

```bash
cd auth-server
npm install
npm run start:debug

cd ../authz-server
npm install
npm run start:debug
```

### 5. 프론트엔드 시작

```bash
cd portal-client
npm install
npm run dev
```

---

## 개발 워크플로우

### 백엔드 서비스 (NestJS)

```bash
npm run start:dev        # 개발 서버
npm run start:debug      # 디버그 모드

npm run build            # TypeScript 빌드
npm run lint:fix         # 린팅 + 자동 수정
npm run format           # Prettier 포맷팅

npm run test             # 단위 테스트
npm run test:cov         # 커버리지
npm run test:e2e         # E2E 테스트
```

### 프론트엔드 (Next.js)

```bash
npm run dev              # 개발 서버
npm run build            # 프로덕션 빌드
npm run type-check       # TypeScript 타입 검사
npm run lint:fix         # 린팅 + 자동 수정
```

### 공유 라이브러리 (shared-lib)

```bash
pnpm build               # 전체 빌드
pnpm clean               # 빌드 아티팩트 정리
pnpm lint:fix            # 린팅
pnpm format              # 포맷팅

# 특정 패키지만 빌드
pnpm --filter @krgeobuk/core build

# 패키지 게시 (패키지 디렉토리에서)
pnpm verdaccio:publish
```

---

## 포트 구성

| 서비스 | 앱 포트 | MySQL | Redis |
|---|---|---|---|
| auth-server | 8000 | 3307 | 6380 |
| authz-server | 8100 | 3308 | 6381 |
| portal-client | 3000 | - | - |
| auth-client | 3001 | - | - |
| portal-admin-client | 3002 | - | - |
| Verdaccio | - | - | - | ← K8s 운영 |

---

## 서브모듈 업데이트

```bash
# 전체 서브모듈 최신화
git submodule update --remote --merge

# 특정 서브모듈만
git submodule update --remote auth-server
```

---

## 문서

| 문서 | 설명 |
|---|---|
| [CLAUDE.md](./CLAUDE.md) | Claude Code 개발 가이드 (AI 어시스턴트용) |
| [docs/QUICKSTART.md](./docs/QUICKSTART.md) | 상세 초기 설정 가이드 |
| [docs/KUBERNETES_ARCHITECTURE.md](./docs/KUBERNETES_ARCHITECTURE.md) | K8s 아키텍처 |
| [docs/DOMAIN_ARCHITECTURE_PLAN.md](./docs/DOMAIN_ARCHITECTURE_PLAN.md) | 도메인 아키텍처 |
| [docs/BFF_ARCHITECTURE_DESIGN.md](./docs/BFF_ARCHITECTURE_DESIGN.md) | BFF 아키텍처 설계 |

### 서비스별 개발 가이드

| 가이드 | 설명 |
|---|---|
| [authz-server/CLAUDE.md](./authz-server/CLAUDE.md) | NestJS 공통 개발 표준 (모든 백엔드 서버 적용) |
| [auth-server/CLAUDE.md](./auth-server/CLAUDE.md) | OAuth·JWT 특화 패턴 |
| [portal-client/CLAUDE.md](./portal-client/CLAUDE.md) | Next.js 15 개발 가이드 |
| [shared-lib/CLAUDE.md](./shared-lib/CLAUDE.md) | 공유 라이브러리 패키지 개발 표준 |
| [krgeobuk-deployment/README.md](./krgeobuk-deployment/README.md) | CI/CD·Jenkins·Verdaccio 배포 가이드 |
| [krgeobuk-infrastructure/README.md](./krgeobuk-infrastructure/README.md) | 로컬 인프라 운영 가이드 |
| [krgeobuk-k8s/README.md](./krgeobuk-k8s/README.md) | Kubernetes 배포 가이드 |
