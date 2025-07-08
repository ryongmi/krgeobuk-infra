# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

krgeobuk-infra는 Git 서브모듈을 통해 관리되는 마이크로서비스 생태계입니다. 인증, 권한, 포털 서비스와 공유 라이브러리로 구성되어 있으며, 각 서비스는 독립적으로 개발 및 배포됩니다.

## 아키텍처 구조

### 핵심 서비스 (Git 서브모듈)
- **auth-server** - 인증 서비스 (OAuth, JWT, 사용자 인증)
- **authz-server** - 권한 관리 서비스 (RBAC, 역할/권한 관리)
- **shared-lib** - 공유 라이브러리 모노레포 (pnpm 워크스페이스)
- **portal-server** - 포털 서비스 (excluded from main architecture)

### 네트워크 아키텍처
- **msa-network**: 마이크로서비스 간 통신
- **shared-network**: 공유 리소스 접근
- **auth-network**: auth-server 내부 통신
- **authz-network**: authz-server 내부 통신

### 기술 스택
- **NestJS**: 모든 서비스의 기본 프레임워크
- **TypeScript**: ES 모듈 지원과 함께 완전한 TypeScript 구현
- **MySQL 8**: 각 서비스별 독립적 데이터베이스
- **Redis**: 캐싱 및 세션 관리
- **Docker**: 모든 서비스 컨테이너화
- **Verdaccio**: 공유 라이브러리용 프라이빗 NPM 레지스트리

## 핵심 명령어

### 개발 워크플로우
각 서비스 디렉토리에서 실행:

```bash
# 개발 서버 시작
npm run start:dev          # 일반 개발 서버
npm run start:debug        # 디버그 모드 (nodemon)

# 빌드
npm run build              # TypeScript 컴파일
npm run build:watch        # 감시 모드 빌드
```

### 코드 품질 관리
모든 서비스에서 공통적으로 사용:

```bash
# 린팅
npm run lint               # ESLint 실행
npm run lint-fix           # 자동 수정과 함께 린팅
npm run format             # Prettier 포맷팅
```

### 테스트
```bash
# 단위 테스트
npm run test               # Jest 테스트 실행
npm run test:watch         # 감시 모드 테스트
npm run test:cov           # 커버리지 테스트

# E2E 테스트
npm run test:e2e           # 엔드투엔드 테스트
```

### Docker 환경 관리
auth-server와 authz-server에서 사용:

```bash
# 로컬 개발 환경
npm run docker:local:up    # 로컬 Docker 스택 시작
npm run docker:local:down  # 로컬 Docker 스택 중지

# 개발/프로덕션 환경
npm run docker:dev:up      # 개발 Docker 환경
npm run docker:prod:up     # 프로덕션 Docker 환경
```

### 공유 라이브러리 관리
shared-lib 디렉토리에서 실행:

```bash
# Verdaccio 레지스트리 관리
pnpm docker:up             # 로컬 NPM 레지스트리 시작
pnpm docker:down           # 로컬 NPM 레지스트리 중지

# 패키지 빌드
pnpm build                 # 모든 패키지 빌드
pnpm clean                 # 빌드 아티팩트 정리

# 특정 패키지 작업
pnpm --filter @krgeobuk/core build
pnpm --filter @krgeobuk/core install

# 로컬 패키지 게시
pnpm verdaccio:publish     # 패키지 디렉토리에서 실행
```

## 공유 라이브러리 패키지

### 핵심 인프라
- `@krgeobuk/core` - 기본 클래스, 데코레이터, 필터, 인터셉터
- `@krgeobuk/database-config` - TypeORM 및 Redis 설정
- `@krgeobuk/swagger` - API 문서화 설정
- `@krgeobuk/eslint-config` - ESLint 설정 (base, nest, next)
- `@krgeobuk/tsconfig` - TypeScript 설정
- `@krgeobuk/jest-config` - Jest 테스트 설정

### 도메인별 패키지
- `@krgeobuk/auth` - 인증 DTO, 인터페이스, 예외 처리
- `@krgeobuk/jwt` - JWT 토큰 관리, 가드, 데코레이터
- `@krgeobuk/oauth` - OAuth 제공자 (Google, Naver)
- `@krgeobuk/user` - 사용자 관리 기능
- `@krgeobuk/role` - 역할 기반 접근 제어
- `@krgeobuk/service` - 서비스 등록 및 관리
- `@krgeobuk/shared` - 공유 DTO 및 인터페이스

## 개발 참고사항

### 경로 별칭
모든 서비스에서 공통으로 사용하는 TypeScript 경로 별칭:
- `@modules/*` → `src/modules/*`
- `@common/*` → `src/common/*`
- `@config/*` → `src/config/*`
- `@database/*` → `src/database/*`

### 환경 설정
- 각 서비스별 `envs/` 디렉토리에 환경 파일 저장
- Docker Compose를 통해 환경 변수 로드
- 서비스별 독립적 데이터베이스 포트 설정

### 네트워크 구성
- **auth-server**: 포트 8000, MySQL 3307, Redis 6380
- **authz-server**: 포트 8100, MySQL 3308, Redis 6381
- **shared-lib**: Verdaccio 포트 4873

### 패키지 관리
- 공유 라이브러리는 `@krgeobuk` 스코프로 관리
- 로컬 개발 시 Verdaccio 레지스트리 사용
- 모든 패키지는 ESM 형식으로 구성

## 일반적인 개발 워크플로우

1. **환경 설정**: `shared-lib`에서 `pnpm docker:up` 실행하여 Verdaccio 시작
2. **공유 라이브러리 빌드**: `shared-lib`에서 `pnpm build` 실행
3. **서비스 개발**: 각 서비스에서 `npm run start:debug` 실행
4. **코드 품질 확인**: `npm run lint:fix` 및 `npm run format` 실행
5. **테스트**: `npm run test` 실행
6. **Docker 환경 테스트**: `npm run docker:local:up` 실행

각 서비스는 독립적으로 개발 및 배포되지만, 공유 라이브러리를 통해 일관된 아키텍처와 코드 품질을 유지합니다.