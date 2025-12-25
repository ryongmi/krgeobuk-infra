# 빠른 시작 가이드

이 가이드는 krgeobuk-infra 프로젝트를 처음 시작하는 개발자를 위한 단계별 설명서입니다.

## 목차

1. [사전 요구사항](#사전-요구사항)
2. [초기 설정](#초기-설정)
3. [공유 라이브러리 설정](#공유-라이브러리-설정)
4. [백엔드 서비스 시작](#백엔드-서비스-시작)
5. [프론트엔드 시작](#프론트엔드-시작)
6. [환경 변수 설정](#환경-변수-설정)
7. [서비스 확인](#서비스-확인)
8. [트러블슈팅](#트러블슈팅)
9. [다음 단계](#다음-단계)

## 사전 요구사항

시작하기 전에 다음 도구들이 설치되어 있는지 확인하세요:

### 필수 도구

| 도구 | 최소 버전 | 확인 명령어 | 설치 가이드 |
|------|-----------|-------------|-------------|
| Node.js | 18.0.0+ | `node --version` | [nodejs.org](https://nodejs.org) |
| pnpm | 8.0.0+ | `pnpm --version` | `npm install -g pnpm` |
| Docker | 20.10.0+ | `docker --version` | [docker.com](https://docs.docker.com/get-docker/) |
| Docker Compose | 2.0.0+ | `docker compose version` | Docker Desktop 포함 |
| Git | 2.30.0+ | `git --version` | [git-scm.com](https://git-scm.com) |

### 권장 도구

- **VS Code**: TypeScript 개발에 최적화
- **NVM**: Node.js 버전 관리
- **Postman**: API 테스트

### 시스템 요구사항

- **메모리**: 최소 8GB RAM (권장 16GB)
- **디스크**: 최소 10GB 여유 공간
- **OS**: Windows 10+, macOS 10.15+, Ubuntu 20.04+

## 초기 설정

### 1. 저장소 클론

Git 서브모듈을 포함하여 프로젝트를 클론합니다:

```bash
# 서브모듈 포함 클론
git clone --recursive https://github.com/your-org/krgeobuk-infra.git
cd krgeobuk-infra

# 또는 일반 클론 후 서브모듈 초기화
git clone https://github.com/your-org/krgeobuk-infra.git
cd krgeobuk-infra
git submodule update --init --recursive
```

### 2. 프로젝트 구조 확인

다음 디렉토리가 존재하는지 확인:

```bash
ls -la

# 예상 출력:
# auth-server/
# authz-server/
# portal-client/
# shared-lib/
# docs/
# scripts/
```

서브모듈이 비어있다면 다시 업데이트:

```bash
git submodule update --init --recursive
```

## 공유 라이브러리 설정

공유 라이브러리는 모든 서비스에서 사용되므로 **가장 먼저** 설정해야 합니다.

### 1. Verdaccio 시작

Verdaccio는 프라이빗 NPM 레지스트리로, `@krgeobuk` 스코프 패키지를 관리합니다.

```bash
cd shared-lib

# Verdaccio Docker 컨테이너 시작
pnpm docker:up

# 컨테이너 상태 확인
docker ps | grep verdaccio
```

**확인**: http://localhost:4873 접속 시 Verdaccio UI가 표시되어야 합니다.

### 2. 의존성 설치

```bash
# shared-lib 디렉토리에서
pnpm install
```

### 3. 패키지 빌드

모든 공유 라이브러리 패키지를 빌드합니다:

```bash
# 전체 빌드
pnpm build

# 빌드 성공 확인
ls -la packages/core/dist
ls -la packages/auth/dist
```

### 4. Verdaccio에 패키지 게시 (선택사항)

로컬 개발 시 Verdaccio에 패키지를 게시할 수 있습니다:

```bash
# 각 패키지 디렉토리에서
cd packages/core
pnpm verdaccio:publish

cd ../auth
pnpm verdaccio:publish

# 또는 루트에서 모든 패키지 게시
pnpm -r verdaccio:publish
```

## 백엔드 서비스 시작

### auth-server 설정

인증 서비스를 설정하고 시작합니다.

```bash
cd ../auth-server  # shared-lib에서 이동

# 1. 의존성 설치
npm install

# 2. 환경 변수 설정 (필요 시)
cp envs/.env.local.example envs/.env.local
# envs/.env.local 파일 편집

# 3. Docker 인프라 시작 (MySQL, Redis)
npm run docker:local:up

# 4. 데이터베이스 마이그레이션 (필요 시)
npm run migration:run

# 5. 개발 서버 시작
npm run start:debug
```

**확인**:
- 서버: http://localhost:8000
- Swagger UI: http://localhost:8000/api-docs
- Health Check: http://localhost:8000/health

### authz-server 설정

권한 관리 서비스를 설정합니다.

```bash
cd ../authz-server

# 1. 의존성 설치
npm install

# 2. 환경 변수 설정 (필요 시)
cp envs/.env.local.example envs/.env.local

# 3. Docker 인프라 시작
npm run docker:local:up

# 4. 데이터베이스 마이그레이션 (필요 시)
npm run migration:run

# 5. 개발 서버 시작
npm run start:debug
```

**확인**:
- 서버: http://localhost:8100
- Swagger UI: http://localhost:8100/api-docs
- Health Check: http://localhost:8100/health

## 프론트엔드 시작

### portal-client 설정

Next.js 기반 포털 클라이언트를 시작합니다.

```bash
cd ../portal-client

# 1. 의존성 설치
npm install

# 2. 환경 변수 설정
cp .env.local.example .env.local
# .env.local 파일 편집 (API 엔드포인트 등)

# 3. 개발 서버 시작
npm run dev
```

**확인**:
- 포털: http://localhost:3000

## 환경 변수 설정

각 서비스의 환경 변수 설정이 필요합니다.

### auth-server (.env.local)

```bash
# Database
DB_HOST=localhost
DB_PORT=3307
DB_USERNAME=root
DB_PASSWORD=your_password
DB_DATABASE=auth_db

# Redis
REDIS_HOST=localhost
REDIS_PORT=6380
REDIS_PASSWORD=your_redis_password

# JWT
JWT_SECRET=your_jwt_secret
JWT_EXPIRATION=3600

# OAuth (선택사항)
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
NAVER_CLIENT_ID=your_naver_client_id
NAVER_CLIENT_SECRET=your_naver_client_secret
```

### authz-server (.env.local)

```bash
# Database
DB_HOST=localhost
DB_PORT=3308
DB_USERNAME=root
DB_PASSWORD=your_password
DB_DATABASE=authz_db

# Redis
REDIS_HOST=localhost
REDIS_PORT=6381
REDIS_PASSWORD=your_redis_password

# JWT (auth-server와 동일한 시크릿 사용)
JWT_SECRET=your_jwt_secret
```

### portal-client (.env.local)

```bash
# API Endpoints
NEXT_PUBLIC_AUTH_API_URL=http://localhost:8000
NEXT_PUBLIC_AUTHZ_API_URL=http://localhost:8100

# Environment
NEXT_PUBLIC_ENV=local
```

## 서비스 확인

모든 서비스가 정상 작동하는지 확인합니다.

### 1. Docker 컨테이너 상태

```bash
# 실행 중인 모든 컨테이너 확인
docker ps

# 예상 출력:
# - verdaccio (포트 4873)
# - auth-mysql (포트 3307)
# - auth-redis (포트 6380)
# - authz-mysql (포트 3308)
# - authz-redis (포트 6381)
```

### 2. 서비스 Health Check

```bash
# auth-server
curl http://localhost:8000/health

# authz-server
curl http://localhost:8100/health

# portal-client (브라우저에서)
# http://localhost:3000
```

### 3. API 테스트

Swagger UI를 통해 API를 테스트할 수 있습니다:

- **auth-server**: http://localhost:8000/api-docs
- **authz-server**: http://localhost:8100/api-docs

## 트러블슈팅

### 서브모듈이 비어있음

```bash
# 서브모듈 강제 업데이트
git submodule update --init --recursive --force
```

### 포트 충돌

다른 애플리케이션이 필요한 포트를 사용 중인 경우:

```bash
# 포트 사용 확인 (Linux/Mac)
lsof -i :8000
lsof -i :3307

# 포트 사용 확인 (Windows)
netstat -ano | findstr :8000

# 해결: 해당 프로세스 종료 또는 .env 파일에서 포트 변경
```

### Docker 컨테이너 시작 실패

```bash
# 기존 컨테이너 정리
docker compose down -v

# 다시 시작
npm run docker:local:up

# 로그 확인
docker compose logs -f
```

### 의존성 설치 오류

```bash
# npm 캐시 정리
npm cache clean --force

# node_modules 삭제 후 재설치
rm -rf node_modules package-lock.json
npm install

# pnpm의 경우
pnpm store prune
rm -rf node_modules pnpm-lock.yaml
pnpm install
```

### TypeScript 컴파일 오류

```bash
# TypeScript 버전 확인
npx tsc --version

# tsconfig.json 검증
npx tsc --noEmit

# shared-lib 재빌드
cd shared-lib
pnpm clean
pnpm build
```

### Verdaccio 연결 실패

```bash
# Verdaccio 상태 확인
docker ps | grep verdaccio

# 재시작
cd shared-lib
pnpm docker:down
pnpm docker:up

# 레지스트리 설정 확인
npm config get registry
# http://localhost:4873 또는 기본 레지스트리여야 함
```

## 다음 단계

프로젝트 설정이 완료되었습니다! 다음 문서를 참조하세요:

### 개발 가이드
- **[CLAUDE.md](./CLAUDE.md)** - 전체 프로젝트 구조 및 개발 표준
- **[authz-server/CLAUDE.md](./authz-server/CLAUDE.md)** - NestJS 공통 개발 가이드
- **[auth-server/CLAUDE.md](./auth-server/CLAUDE.md)** - 인증 서비스 특화 가이드
- **[portal-client/CLAUDE.md](./portal-client/CLAUDE.md)** - Next.js 개발 가이드
- **[shared-lib/CLAUDE.md](./shared-lib/CLAUDE.md)** - 공유 라이브러리 개발 가이드

### 운영 가이드
- **[krgeobuk-k8s/docs/DEPLOYMENT.md](./krgeobuk-k8s/docs/DEPLOYMENT.md)** - 배포 전략 및 환경 설정
- **[docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md)** - 상세 아키텍처 문서

### 개발 워크플로우

1. **새 기능 개발**
   ```bash
   # Feature 브랜치 생성
   git checkout -b feature/new-feature

   # 공유 라이브러리가 필요한 경우
   cd shared-lib
   # 패키지 수정 및 빌드
   pnpm build

   # 서비스 개발
   cd ../auth-server
   npm run start:debug
   ```

2. **코드 품질 검사**
   ```bash
   # 린팅
   npm run lint-fix

   # 포맷팅
   npm run format

   # 타입 검사 (portal-client)
   npm run type-check

   # 테스트
   npm run test
   ```

3. **Pull Request 생성**
   ```bash
   git add .
   git commit -m "feat: add new feature"
   git push origin feature/new-feature
   # GitHub에서 PR 생성
   ```

## 도움말

문제가 발생하거나 질문이 있으시면:

- **Issues**: [GitHub Issues](https://github.com/your-org/krgeobuk-infra/issues)
- **문서**: [docs/](./docs/) 디렉토리 참조
- **서비스별 가이드**: 각 서비스의 `CLAUDE.md` 참조

Happy Coding! 🚀
