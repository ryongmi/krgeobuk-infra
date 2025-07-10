# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

krgeobuk-infra는 Git 서브모듈을 통해 관리되는 마이크로서비스 생태계입니다. 인증, 권한, 포털 서비스와 공유 라이브러리로 구성되어 있으며, 각 서비스는 독립적으로 개발 및 배포됩니다.

## 아키텍처 구조

### 핵심 서비스 (Git 서브모듈)
- **auth-server** - 인증 서비스 (OAuth, JWT, 사용자 인증)
- **authz-server** - 권한 관리 서비스 (RBAC, 역할/권한 관리)
- **portal-client** - 통합 포털 클라이언트 (Next.js 15, 관리자 인터페이스, 사용자 대시보드)
- **shared-lib** - 공유 라이브러리 모노레포 (pnpm 워크스페이스)
- **portal-server** - 포털 서비스 (excluded from main architecture)

### 네트워크 아키텍처
- **msa-network**: 마이크로서비스 간 통신
- **shared-network**: 공유 리소스 접근
- **auth-network**: auth-server 내부 통신
- **authz-network**: authz-server 내부 통신

### 기술 스택
- **NestJS**: 백엔드 서비스의 기본 프레임워크 (auth-server, authz-server)
- **Next.js 15**: 프론트엔드 프레임워크 (portal-client)
- **TypeScript**: ES 모듈 지원과 함께 완전한 TypeScript 구현
- **MySQL 8**: 각 서비스별 독립적 데이터베이스
- **Redis**: 캐싱 및 세션 관리
- **Docker**: 모든 서비스 컨테이너화
- **Verdaccio**: 공유 라이브러리용 프라이빗 NPM 레지스트리
- **Tailwind CSS**: 프론트엔드 UI 스타일링 (portal-client)

## 핵심 명령어

### 개발 워크플로우
각 서비스 디렉토리에서 실행:

**백엔드 서비스 (auth-server, authz-server):**
```bash
# 개발 서버 시작
npm run start:dev          # 일반 개발 서버
npm run start:debug        # 디버그 모드 (nodemon)

# 빌드
npm run build              # TypeScript 컴파일
npm run build:watch        # 감시 모드 빌드
```

**프론트엔드 (portal-client):**
```bash
# 개발 서버 시작
npm run dev                # Next.js 개발 서버

# 빌드
npm run build              # 프로덕션 빌드
npm run start              # 프로덕션 서버 시작

# 타입 검사
npm run type-check         # TypeScript 타입 검사
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
- **portal-client**: 포트 3000 (Next.js 개발 서버)
- **shared-lib**: Verdaccio 포트 4873

### 패키지 관리
- 공유 라이브러리는 `@krgeobuk` 스코프로 관리
- 로컬 개발 시 Verdaccio 레지스트리 사용
- 모든 패키지는 ESM 형식으로 구성

## portal-client 아키텍처 특징

portal-client는 KRGeobuk 서비스 생태계를 위한 통합 포털 클라이언트로, Next.js 15로 구축된 현대적인 웹 애플리케이션입니다.

### 주요 구성 요소
- **완전한 사용자 인증 시스템**: 로그인/회원가입, OAuth 지원
- **고급 관리자 인터페이스**: 사용자, 역할, 권한, 서비스 관리
- **반응형 사용자 경험**: 모바일 우선 디자인, 글래스모피즘 효과
- **관리자 모듈**: `/admin` 라우트 하위의 중첩 페이지 구조
  - 사용자 관리 (`/admin/auth/users`)
  - OAuth 클라이언트 관리 (`/admin/auth/oauth`)
  - 역할 관리 (`/admin/authorization/roles`)
  - 권한 관리 (`/admin/authorization/permissions`)
  - 서비스 관리 (`/admin/portal/services`)

### 기술적 특징
- **Next.js 15 App Router**: 최신 React 패턴 활용
- **TypeScript 엄격 모드**: 완전한 타입 안전성
- **Tailwind CSS**: 유틸리티 우선 CSS, 그라데이션 테마
- **React Hook Form**: 성능 최적화된 폼 관리
- **경로 별칭**: `@/*` → `./src/*`

### 서비스 가시성 관리
서비스는 두 가지 플래그를 통해 포탈에서의 가시성을 제어합니다:
- `isVisible`: 포탈에서 표시 여부
- `isVisibleByRole`: 권한 기반 표시 여부

## 일반적인 개발 워크플로우

1. **환경 설정**: `shared-lib`에서 `pnpm docker:up` 실행하여 Verdaccio 시작
2. **공유 라이브러리 빌드**: `shared-lib`에서 `pnpm build` 실행
3. **백엔드 서비스 개발**: auth-server, authz-server에서 `npm run start:debug` 실행
4. **프론트엔드 개발**: portal-client에서 `npm run dev` 실행
5. **코드 품질 확인**: `npm run lint:fix` 및 `npm run format` 실행
6. **타입 검사**: portal-client에서 `npm run type-check` 실행
7. **테스트**: `npm run test` 실행
8. **Docker 환경 테스트**: `npm run docker:local:up` 실행

각 서비스는 독립적으로 개발 및 배포되지만, 공유 라이브러리를 통해 일관된 아키텍처와 코드 품질을 유지합니다.

## API 응답 포맷 표준

krgeobuk 생태계는 `@krgeobuk/core` 패키지의 SerializerInterceptor와 HttpExceptionFilter를 통해 일관된 API 응답 포맷을 제공합니다.

### 성공 응답 포맷 (SerializerInterceptor)

모든 성공적인 API 응답은 다음 구조를 따릅니다:

```typescript
{
  code: string,           // 응답 코드 (기본: CoreCode.REQUEST_SUCCESS)
  status_code: number,    // HTTP 상태 코드 (기본: 200)
  message: string,        // 응답 메시지 (기본: CoreMessage.REQUEST_SUCCESS)
  isLogin: boolean,       // 사용자 로그인 상태
  data: object | null     // 실제 응답 데이터 (snake_case로 변환됨)
}
```

**주요 특징:**
- 모든 응답 데이터는 `toSnakeCase()` 함수를 통해 snake_case로 변환
- `@Serialize()` 데코레이터를 통해 커스텀 code, message, DTO 지정 가능
- DTO가 지정된 경우 `class-transformer`의 `plainToInstance()`로 변환
- `isLogin` 필드로 사용자 인증 상태 확인 가능

### 에러 응답 포맷 (HttpExceptionFilter)

모든 HTTP 예외는 다음 구조로 응답됩니다:

```typescript
{
  statusCode: number,     // HTTP 상태 코드
  code: string,          // 에러 코드 (기본: CoreCode.SERVER_ERROR)
  message: string        // 에러 메시지 (배열인 경우 join으로 결합)
}
```

**주요 특징:**
- 배열 형태의 메시지는 쉼표로 결합하여 단일 문자열로 변환
- 커스텀 에러 코드 지원 (exception response에 code 필드 포함 시)
- Chrome DevTools 요청은 자동으로 필터링
- 상세한 로깅: 요청 정보, 사용자 정보, 파라미터, 에러 상세 등

### 사용 예시

**성공 응답 커스터마이징:**
```typescript
@Get()
@Serialize({ 
  dto: UserResponseDto, 
  code: 'USER_001', 
  message: '사용자 조회 성공' 
})
getUser() {
  return { id: 1, name: 'John', email: 'john@example.com' };
}
```

**커스텀 예외 처리:**
```typescript
throw new BadRequestException({
  message: '잘못된 요청입니다',
  code: 'AUTH_001'
});
```

이러한 표준화된 응답 포맷을 통해 프론트엔드와 백엔드 간의 일관된 데이터 교환이 보장됩니다.

## NestJS 컨트롤러 개발 가이드

### 공유 패키지를 활용한 표준화된 컨트롤러 작성

krgeobuk 생태계에서는 일관된 API 설계를 위해 공유 라이브러리의 Response, Error, DTO를 활용해야 합니다.

#### 기본 구조 및 Import

모든 컨트롤러는 다음 패턴을 따라야 합니다:

```typescript
import {
  Body,
  Controller,
  Delete,
  Get,
  HttpCode,
  Param,
  Patch,
  Post,
  Query,
  UseGuards,
} from '@nestjs/common';

import { Serialize } from '@krgeobuk/core/decorators';
import {
  SwaggerApiTags,
  SwaggerApiOperation,
  SwaggerApiBearerAuth,
  SwaggerApiBody,
  SwaggerApiParam,
  SwaggerApiOkResponse,
  SwaggerApiPaginatedResponse,
  SwaggerApiErrorResponse,
} from '@krgeobuk/swagger/decorators';
import { JwtPayload } from '@krgeobuk/jwt/interfaces';
import { CurrentJwt } from '@krgeobuk/jwt/decorators';
import { AccessTokenGuard } from '@krgeobuk/jwt/guards';

// 도메인별 공유 패키지 import
import { [Domain]Response } from '@krgeobuk/[domain]/response';
import { [Domain]Error } from '@krgeobuk/[domain]/exception';
import {
  [Domain]SearchQueryDto,
  [Domain]DetailDto,
  [Domain]SearchResultDto,
} from '@krgeobuk/[domain]/dtos';
```

#### 컨트롤러 클래스 데코레이터

```typescript
@SwaggerApiTags({ tags: ['[domain]s'] })
@SwaggerApiBearerAuth()
@Controller('[domain]s')
export class [Domain]Controller {
  constructor(private readonly [domain]Service: [Domain]Service) {}
}
```

#### 표준화된 엔드포인트 패턴

**1. 목록 조회 (GET /[domain]s)**

```typescript
@Get()
@HttpCode([Domain]Response.FETCH_SUCCESS.statusCode)
@SwaggerApiOperation({
  summary: '[도메인] 목록 조회',
  description: '[도메인] 목록을 검색 조건에 따라 조회합니다.',
})
@SwaggerApiPaginatedResponse({
  status: [Domain]Response.FETCH_SUCCESS.statusCode,
  description: [Domain]Response.FETCH_SUCCESS.message,
  dto: [Domain]SearchResultDto,
})
@SwaggerApiErrorResponse({
  status: [Domain]Error.[DOMAIN]_FETCH_ERROR.statusCode,
  description: [Domain]Error.[DOMAIN]_FETCH_ERROR.message,
})
@UseGuards(AccessTokenGuard)
@Serialize({
  dto: [Domain]SearchResultDto,
  ...[Domain]Response.FETCH_SUCCESS,
})
async get[Domain]s(
  @Query() query: [Domain]SearchQueryDto,
  @CurrentJwt() jwt: JwtPayload
): Promise<PaginatedResult<[Domain]Entity>> {
  return this.[domain]Service.search[Domain]s(query);
}
```

**2. 상세 조회 (GET /[domain]s/:id)**

```typescript
@Get(':id')
@HttpCode([Domain]Response.FETCH_SUCCESS.statusCode)
@SwaggerApiOperation({ 
  summary: '[도메인] 상세 조회', 
  description: 'ID로 특정 [도메인]을 조회합니다.' 
})
@SwaggerApiParam({
  name: 'id',
  type: String,
  description: '[도메인] ID',
  example: '123e4567-e89b-12d3-a456-426614174000',
})
@SwaggerApiOkResponse({
  status: [Domain]Response.FETCH_SUCCESS.statusCode,
  description: [Domain]Response.FETCH_SUCCESS.message,
  dto: [Domain]DetailDto,
})
@SwaggerApiErrorResponse({
  status: [Domain]Error.[DOMAIN]_NOT_FOUND.statusCode,
  description: [Domain]Error.[DOMAIN]_NOT_FOUND.message,
})
@SwaggerApiErrorResponse({
  status: [Domain]Error.[DOMAIN]_FETCH_ERROR.statusCode,
  description: [Domain]Error.[DOMAIN]_FETCH_ERROR.message,
})
@UseGuards(AccessTokenGuard)
@Serialize({
  dto: [Domain]DetailDto,
  ...[Domain]Response.FETCH_SUCCESS,
})
async get[Domain](
  @Param('id') id: string,
  @CurrentJwt() jwt: JwtPayload
): Promise<[Domain]Entity> {
  return this.[domain]Service.findByIdOrFail(id);
}
```

**3. 생성 (POST /[domain]s)**

```typescript
@Post()
@HttpCode([Domain]Response.CREATE_SUCCESS.statusCode)
@SwaggerApiOperation({ 
  summary: '[도메인] 생성', 
  description: '새로운 [도메인]을 생성합니다.' 
})
@SwaggerApiBody({ 
  dto: Create[Domain]Dto, 
  description: '[도메인] 생성 데이터' 
})
@SwaggerApiOkResponse({
  status: [Domain]Response.CREATE_SUCCESS.statusCode,
  description: [Domain]Response.CREATE_SUCCESS.message,
  dto: [Domain]DetailDto,
})
@SwaggerApiErrorResponse({
  status: [Domain]Error.[DOMAIN]_CREATE_ERROR.statusCode,
  description: [Domain]Error.[DOMAIN]_CREATE_ERROR.message,
})
@SwaggerApiErrorResponse({
  status: [Domain]Error.[DOMAIN]_ALREADY_EXISTS.statusCode,
  description: [Domain]Error.[DOMAIN]_ALREADY_EXISTS.message,
})
@UseGuards(AccessTokenGuard)
@Serialize({
  dto: [Domain]DetailDto,
  ...[Domain]Response.CREATE_SUCCESS,
})
async create[Domain](
  @Body() dto: Create[Domain]Dto,
  @CurrentJwt() jwt: JwtPayload
): Promise<[Domain]Entity> {
  return this.[domain]Service.create[Domain](dto);
}
```

**4. 수정 (PATCH /[domain]s/:id)**

```typescript
@Patch(':id')
@HttpCode([Domain]Response.UPDATE_SUCCESS.statusCode)
@SwaggerApiOperation({ 
  summary: '[도메인] 수정', 
  description: '기존 [도메인]을 수정합니다.' 
})
@SwaggerApiParam({
  name: 'id',
  type: String,
  description: '[도메인] ID',
  example: '123e4567-e89b-12d3-a456-426614174000',
})
@SwaggerApiBody({ 
  dto: Update[Domain]Dto, 
  description: '[도메인] 수정 데이터' 
})
@SwaggerApiOkResponse({
  status: [Domain]Response.UPDATE_SUCCESS.statusCode,
  description: [Domain]Response.UPDATE_SUCCESS.message,
  dto: [Domain]DetailDto,
})
@SwaggerApiErrorResponse({
  status: [Domain]Error.[DOMAIN]_NOT_FOUND.statusCode,
  description: [Domain]Error.[DOMAIN]_NOT_FOUND.message,
})
@SwaggerApiErrorResponse({
  status: [Domain]Error.[DOMAIN]_UPDATE_ERROR.statusCode,
  description: [Domain]Error.[DOMAIN]_UPDATE_ERROR.message,
})
@UseGuards(AccessTokenGuard)
@Serialize({
  dto: [Domain]DetailDto,
  ...[Domain]Response.UPDATE_SUCCESS,
})
async update[Domain](
  @Param('id') id: string,
  @Body() dto: Update[Domain]Dto,
  @CurrentJwt() jwt: JwtPayload
): Promise<[Domain]Entity> {
  return this.[domain]Service.update[Domain](id, dto);
}
```

**5. 삭제 (DELETE /[domain]s/:id)**

```typescript
@Delete(':id')
@HttpCode([Domain]Response.DELETE_SUCCESS.statusCode)
@SwaggerApiOperation({ 
  summary: '[도메인] 삭제', 
  description: '[도메인]을 소프트 삭제합니다.' 
})
@SwaggerApiParam({
  name: 'id',
  type: String,
  description: '[도메인] ID',
  example: '123e4567-e89b-12d3-a456-426614174000',
})
@SwaggerApiOkResponse({
  status: [Domain]Response.DELETE_SUCCESS.statusCode,
  description: [Domain]Response.DELETE_SUCCESS.message,
})
@SwaggerApiErrorResponse({
  status: [Domain]Error.[DOMAIN]_NOT_FOUND.statusCode,
  description: [Domain]Error.[DOMAIN]_NOT_FOUND.message,
})
@SwaggerApiErrorResponse({
  status: [Domain]Error.[DOMAIN]_DELETE_ERROR.statusCode,
  description: [Domain]Error.[DOMAIN]_DELETE_ERROR.message,
})
@UseGuards(AccessTokenGuard)
@Serialize({
  ...[Domain]Response.DELETE_SUCCESS,
})
async delete[Domain](
  @Param('id') id: string, 
  @CurrentJwt() jwt: JwtPayload
): Promise<void> {
  await this.[domain]Service.delete[Domain](id);
}
```

#### 주요 원칙

1. **공통 패키지 우선**: 도메인별 공유 라이브러리의 Response, Error, DTO를 반드시 사용
2. **일관된 HTTP 상태 코드**: 공통 패키지에서 정의된 statusCode 사용
3. **표준화된 메시지**: 하드코딩 대신 공통 패키지의 message 사용
4. **타입 안전성**: 공통 패키지의 DTO 타입을 엄격하게 준수
5. **에러 응답 완성도**: 가능한 모든 에러 케이스에 대한 SwaggerApiErrorResponse 정의
6. **인증 가드**: 모든 엔드포인트에 AccessTokenGuard 적용
7. **직렬화 일관성**: Serialize 데코레이터에 Response 객체 전개 연산자 사용

#### 실제 적용 예시 (Permission Controller)

```typescript
// 권한 생성 예시
@Post()
@HttpCode(PermissionResponse.CREATE_SUCCESS.statusCode)
@SwaggerApiOperation({ summary: '권한 생성', description: '새로운 권한을 생성합니다.' })
@SwaggerApiBody({ dto: CreatePermissionDto, description: '권한 생성 데이터' })
@SwaggerApiOkResponse({
  status: PermissionResponse.CREATE_SUCCESS.statusCode,
  description: PermissionResponse.CREATE_SUCCESS.message,
  dto: PermissionDetailDto,
})
@SwaggerApiErrorResponse({
  status: PermissionError.PERMISSION_CREATE_ERROR.statusCode,
  description: PermissionError.PERMISSION_CREATE_ERROR.message,
})
@UseGuards(AccessTokenGuard)
@Serialize({
  dto: PermissionDetailDto,
  ...PermissionResponse.CREATE_SUCCESS,
})
async createPermission(
  @Body() dto: CreatePermissionDto,
  @CurrentJwt() jwt: JwtPayload
): Promise<PermissionEntity> {
  return this.permissionService.createPermission(dto);
}
```

이 가이드를 따르면 krgeobuk 생태계 전반에서 일관된 API 설계와 문서화를 보장할 수 있습니다.

## 공통패키지 DTO 대체 가이드

각 마이크로서비스의 도메인 모듈에서 로컬 DTO를 공통 라이브러리의 DTO로 대체하여 코드 일관성과 재사용성을 높이는 가이드입니다.

### 대체 대상 DTO 식별

**1. 공통패키지에서 대체 가능한 DTO 확인**
```bash
# 공통패키지의 DTO 확인
ls /shared-lib/packages/[domain]/src/dtos/

# 예: role 도메인
ls /shared-lib/packages/role/src/dtos/
# ├── create.dto.ts          -> CreateRoleDto
# ├── update.dto.ts          -> UpdateRoleDto  
# ├── detail.dto.ts          -> RoleDetailDto
# ├── search-query.dto.ts    -> RoleSearchQueryDto
# └── search-result.dto.ts   -> RoleSearchResultDto
```

**2. 마이크로서비스의 로컬 DTO 확인**
```bash
# 마이크로서비스의 로컬 DTO 확인
ls /[service]/src/modules/[domain]/dtos/

# 예: authz-server의 role 모듈
ls /authz-server/src/modules/role/dtos/
# ├── create-role.dto.ts           -> 대체 대상
# ├── update-role.dto.ts           -> 대체 대상
# ├── role-response.dto.ts         -> 대체 대상 (DetailDto로)
# └── role-search-query.dto.ts     -> 대체 대상
```

### 대체 작업 프로세스

**1. 누락된 공통 DTO 생성 (필요시)**

만약 공통패키지에 필요한 DTO가 없다면 먼저 생성합니다:

```typescript
// shared-lib/packages/[domain]/src/dtos/create.dto.ts
import { IsValid[Field] } from '@krgeobuk/shared/[domain]';

export class Create[Domain]Dto implements Create[Domain] {
  @IsValid[Field]()
  field!: string;
  
  @IsValid[Field]({ isOptional: true })
  optionalField?: string | null;
}

// shared-lib/packages/[domain]/src/dtos/update.dto.ts  
import { ExposeUuidIdDto } from '@krgeobuk/core/dtos';

export class Update[Domain]Dto extends ExposeUuidIdDto implements Update[Domain] {
  @IsValid[Field]({ isOptional: true })
  field?: string;
}
```

필요한 인터페이스도 함께 생성:
```typescript
// shared-lib/packages/[domain]/src/interfaces/create.interface.ts
export interface Create[Domain] {
  field: string;
  optionalField?: string | null;
}

// shared-lib/packages/[domain]/src/interfaces/update.interface.ts
import type { UuidId } from '@krgeobuk/core/interfaces';

export interface Update[Domain] extends UuidId {
  field?: string;
}
```

**2. 컨트롤러 임포트 변경**

```typescript
// Before: 로컬 DTO 임포트
import { CreateRoleDto, UpdateRoleDto, RoleResponseDto } from './dtos/index.js';

// After: 공통패키지 DTO 임포트
import {
  CreateRoleDto,
  UpdateRoleDto, 
  RoleDetailDto,        // RoleResponseDto -> RoleDetailDto로 대체
  RoleSearchQueryDto,
  RoleSearchResultDto,
} from '@krgeobuk/role/dtos';
```

**3. 서비스 임포트 변경**

```typescript
// Before: 로컬 DTO 임포트
import { RoleSearchQueryDto } from './dtos/role-search-query.dto.js';

// After: 공통패키지 DTO 임포트
import { RoleSearchQueryDto } from '@krgeobuk/role/dtos';
```

**4. 타입 시그니처 업데이트**

```typescript
// Before: 로컬 ResponseDto 사용
async createRole(): Promise<RoleResponseDto> {
  // ...
}

// After: 공통패키지 DetailDto 사용  
async createRole(): Promise<RoleDetailDto> {
  // ...
}
```

**5. 로컬 DTO 파일 정리**

```typescript
// dtos/index.ts 업데이트
// Before:
export * from './create-role.dto.js';
export * from './update-role.dto.js';
export * from './role-response.dto.js';

// After:
// DTO들은 @krgeobuk/role/dtos 공통패키지에서 가져옵니다.
```

로컬 DTO 파일들 삭제:
```bash
rm create-role.dto.ts update-role.dto.ts role-response.dto.ts role-search-query.dto.ts
```

### 장점 및 효과

**1. 코드 일관성**
- 모든 서비스에서 동일한 유효성 검사 규칙 적용
- 통일된 API 스키마 및 문서화

**2. 유지보수성**
- 중앙화된 DTO 관리로 변경사항 전파 용이
- 중복 코드 제거로 유지보수 비용 절감

**3. 타입 안전성**
- 공통패키지의 고급 데코레이터 활용 (`@IsValidRoleName`, `@ExposeRoleName` 등)
- 컴파일 타임 타입 검증 강화

**4. API 문서화 향상**
- Swagger 문서의 일관성 보장
- 도메인별 표준화된 스키마 제공

### 체크리스트

대체 작업 완료 시 다음 사항들을 확인:

- [ ] 공통패키지에 필요한 모든 DTO가 존재하는가?
- [ ] 공통패키지에 필요한 인터페이스가 모두 정의되어 있는가?
- [ ] 컨트롤러에서 공통패키지 DTO를 올바르게 임포트하고 있는가?
- [ ] 서비스에서 공통패키지 DTO를 올바르게 임포트하고 있는가?
- [ ] ResponseDto가 DetailDto로 올바르게 대체되었는가?
- [ ] 로컬 DTO 파일들이 모두 삭제되었는가?
- [ ] dtos/index.ts에서 로컬 export가 제거되었는가?
- [ ] 빌드가 성공적으로 완료되는가?
- [ ] 타입 검사가 통과하는가?

### 실제 적용 사례

**Role 모듈 대체 예시:**
- `CreateRoleDto` → `@krgeobuk/role/dtos`
- `UpdateRoleDto` → `@krgeobuk/role/dtos`  
- `RoleSearchQueryDto` → `@krgeobuk/role/dtos`
- `RoleResponseDto` → `RoleDetailDto` (`@krgeobuk/role/dtos`)

**Permission 모듈 대체 예시:**
- `CreatePermissionDto` → `@krgeobuk/permission/dtos` (새로 생성)
- `UpdatePermissionDto` → `@krgeobuk/permission/dtos` (새로 생성)
- `PermissionSearchQueryDto` → `@krgeobuk/permission/dtos`
- `PermissionResponseDto` → `PermissionDetailDto` (`@krgeobuk/permission/dtos`)

이 가이드를 따르면 마이크로서비스 간 DTO 일관성을 보장하고 공통 라이브러리의 이점을 최대한 활용할 수 있습니다.