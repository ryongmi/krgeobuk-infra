# krgeobuk-infra

> Git 서브모듈 기반 마이크로서비스 생태계

krgeobuk-infra는 인증, 권한 관리, 포털 서비스로 구성된 엔터프라이즈급 마이크로서비스 아키텍처입니다. 각 서비스는 독립적으로 개발 및 배포되며, 공유 라이브러리를 통해 일관된 코드 품질과 아키텍처를 유지합니다.

## 주요 특징

- **마이크로서비스 아키텍처**: 독립적인 서비스 개발 및 배포
- **Git 서브모듈 관리**: 각 서비스별 독립 리포지토리
- **공유 라이브러리**: 모노레포 기반 코드 재사용
- **프라이빗 NPM 레지스트리**: Verdaccio를 통한 패키지 관리
- **컨테이너화**: Docker 기반 개발 및 배포 환경
- **타입 안정성**: 완전한 TypeScript 구현

## 아키텍처 구조

### 핵심 서비스

| 서비스 | 설명 | 포트 | 기술 스택 |
|--------|------|------|-----------|
| **auth-server** | 인증 서비스 (OAuth, JWT) | 8000 | NestJS, MySQL 8, Redis |
| **authz-server** | 권한 관리 서비스 (RBAC) | 8100 | NestJS, MySQL 8, Redis |
| **portal-client** | 통합 포털 클라이언트 | 3000 | Next.js 15, Tailwind CSS |
| **shared-lib** | 공유 라이브러리 모노레포 | - | pnpm workspace |

### 네트워크 구성

- **msa-network**: 마이크로서비스 간 통신
- **shared-network**: 공유 리소스 접근
- **auth-network**: auth-server 내부 통신
- **authz-network**: authz-server 내부 통신

## 기술 스택

### 백엔드
- **NestJS** - 엔터프라이즈급 Node.js 프레임워크
- **TypeScript** - ES 모듈 완전 지원
- **MySQL 8** - 서비스별 독립 데이터베이스
- **Redis** - 캐싱 및 세션 관리

### 프론트엔드
- **Next.js 15** - React 프레임워크
- **Tailwind CSS** - 유틸리티 기반 CSS

### 인프라
- **Docker** - 컨테이너화
- **Verdaccio** - 프라이빗 NPM 레지스트리
- **pnpm** - 고성능 패키지 매니저

## 빠른 시작

### 사전 요구사항

- Node.js 18+
- pnpm 8+
- Docker & Docker Compose
- Git

### 초기 설정

```bash
# 1. 저장소 클론 (서브모듈 포함)
git clone --recursive https://github.com/your-org/krgeobuk-infra.git
cd krgeobuk-infra

# 2. 서브모듈 업데이트
git submodule update --init --recursive

# 3. Verdaccio 시작 (공유 라이브러리 레지스트리)
cd shared-lib
pnpm docker:up
pnpm build

# 4. 백엔드 서비스 시작
cd ../auth-server
npm install
npm run docker:local:up
npm run start:debug

cd ../authz-server
npm install
npm run docker:local:up
npm run start:debug

# 5. 프론트엔드 시작
cd ../portal-client
npm install
npm run dev
```

자세한 가이드는 [QUICKSTART.md](./QUICKSTART.md)를 참조하세요.

## 프로젝트 구조

```
krgeobuk-infra/
├── auth-server/          # 인증 서비스
├── authz-server/         # 권한 관리 서비스
├── portal-client/        # 포털 클라이언트
├── shared-lib/           # 공유 라이브러리 모노레포
│   ├── packages/
│   │   ├── core/         # 핵심 클래스, 데코레이터
│   │   ├── auth/         # 인증 관련 기능
│   │   ├── jwt/          # JWT 토큰 관리
│   │   ├── oauth/        # OAuth 제공자
│   │   └── ...
├── docs/                 # 문서
├── scripts/              # 자동화 스크립트
├── CLAUDE.md            # AI 어시스턴트 가이드
└── README.md            # 이 파일
```

## 공유 라이브러리 패키지

### 핵심 인프라
- `@krgeobuk/core` - 기본 클래스, 데코레이터, 필터, 인터셉터
- `@krgeobuk/database-config` - TypeORM 및 Redis 설정
- `@krgeobuk/swagger` - API 문서화
- `@krgeobuk/eslint-config` - 코드 품질 설정
- `@krgeobuk/tsconfig` - TypeScript 설정
- `@krgeobuk/jest-config` - 테스트 설정

### 도메인 패키지
- `@krgeobuk/auth` - 인증 DTO, 인터페이스
- `@krgeobuk/jwt` - JWT 토큰 관리, 가드
- `@krgeobuk/oauth` - OAuth (Google, Naver)
- `@krgeobuk/user` - 사용자 관리
- `@krgeobuk/role` - 역할 기반 접근 제어
- `@krgeobuk/service` - 서비스 등록 관리
- `@krgeobuk/shared` - 공유 DTO 및 인터페이스

## 개발 워크플로우

### 일반 개발

```bash
# 개발 서버 시작 (NestJS 서비스)
npm run start:dev          # 일반 모드
npm run start:debug        # 디버그 모드

# 개발 서버 시작 (Next.js)
npm run dev

# 빌드
npm run build
npm run build:watch        # 감시 모드

# 코드 품질
npm run lint               # 린팅
npm run lint-fix           # 자동 수정
npm run format             # Prettier 포맷팅

# 테스트
npm run test               # 단위 테스트
npm run test:watch         # 감시 모드
npm run test:cov           # 커버리지
npm run test:e2e           # E2E 테스트
```

### Docker 환경

```bash
# 로컬 개발
npm run docker:local:up
npm run docker:local:down

# 개발/프로덕션
npm run docker:dev:up
npm run docker:prod:up
```

### 공유 라이브러리 관리

```bash
# Verdaccio 관리
pnpm docker:up
pnpm docker:down

# 패키지 빌드
pnpm build
pnpm clean

# 특정 패키지 작업
pnpm --filter @krgeobuk/core build

# 로컬 게시
pnpm verdaccio:publish
```

## 문서

- [빠른 시작 가이드](./QUICKSTART.md) - 상세한 설치 및 실행 가이드
- [배포 가이드](./krgeobuk-k8s/docs/DEPLOYMENT.md) - 환경별 배포 전략
- [아키텍처 문서](./docs/ARCHITECTURE.md) - 시스템 아키텍처 상세 설명
- [Claude 개발 가이드](./CLAUDE.md) - AI 어시스턴트용 개발 표준

### 서비스별 가이드
- [auth-server/CLAUDE.md](./auth-server/CLAUDE.md) - 인증 서비스 개발 가이드
- [authz-server/CLAUDE.md](./authz-server/CLAUDE.md) - NestJS 공통 개발 표준
- [portal-client/CLAUDE.md](./portal-client/CLAUDE.md) - Next.js 개발 가이드
- [shared-lib/CLAUDE.md](./shared-lib/CLAUDE.md) - 공유 라이브러리 개발 표준

## 개발 참고사항

### TypeScript 경로 별칭
모든 서비스에서 공통 사용:
```typescript
@modules/*  → src/modules/*
@common/*   → src/common/*
@config/*   → src/config/*
@database/* → src/database/*
```

### 포트 구성
- **auth-server**: 8000 (MySQL: 3307, Redis: 6380)
- **authz-server**: 8100 (MySQL: 3308, Redis: 6381)
- **portal-client**: 3000
- **Verdaccio**: 4873

### 환경 설정
- 각 서비스별 `envs/` 디렉토리에 환경 파일 저장
- Docker Compose를 통해 환경 변수 로드
- 서비스별 독립적 설정 관리

## 기여하기

1. Feature 브랜치 생성 (`git checkout -b feature/AmazingFeature`)
2. 변경사항 커밋 (`git commit -m 'Add some AmazingFeature'`)
3. 브랜치에 Push (`git push origin feature/AmazingFeature`)
4. Pull Request 생성

## 라이선스

이 프로젝트는 MIT 라이선스 하에 배포됩니다.

## 지원

문의사항이나 이슈가 있으시면 [GitHub Issues](https://github.com/your-org/krgeobuk-infra/issues)에 등록해주세요.
