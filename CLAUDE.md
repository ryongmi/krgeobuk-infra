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

#### 컨트롤러 함수명 규칙

모든 컨트롤러에서 일관된 함수명을 사용해야 합니다:

```typescript
// 표준 CRUD 엔드포인트 함수명
async search[Domain]s()     // GET /[domain]s - 목록 조회
async create[Domain]()      // POST /[domain]s - 생성
async find[Domain]ById()    // GET /[domain]s/:id - 상세 조회
async update[Domain]()      // PATCH /[domain]s/:id - 수정
async delete[Domain]()      // DELETE /[domain]s/:id - 삭제
```

**예시:**
```typescript
// Role 컨트롤러
async searchRoles()         // GET /roles
async createRole()          // POST /roles
async findRoleById()        // GET /roles/:id
async updateRole()          // PATCH /roles/:id
async deleteRole()          // DELETE /roles/:id

// Service 컨트롤러
async searchServices()      // GET /services
async createService()       // POST /services
async findServiceById()     // GET /services/:id
async updateService()       // PATCH /services/:id
async deleteService()       // DELETE /services/:id
```

#### 주요 원칙

1. **공통 패키지 우선**: 도메인별 공유 라이브러리의 Response, Error, DTO를 반드시 사용
2. **일관된 HTTP 상태 코드**: 공통 패키지에서 정의된 statusCode 사용
3. **표준화된 메시지**: 하드코딩 대신 공통 패키지의 message 사용
4. **타입 안전성**: 공통 패키지의 DTO 타입을 엄격하게 준수
5. **에러 응답 완성도**: 가능한 모든 에러 케이스에 대한 SwaggerApiErrorResponse 정의
6. **인증 가드**: 모든 엔드포인트에 AccessTokenGuard 적용
7. **직렬화 일관성**: Serialize 데코레이터에 Response 객체 전개 연산자 사용
8. **함수명 일관성**: 위에서 정의한 표준 함수명 패턴 준수
9. **Serialize DTO 타입 일관성**: `@Serialize()` 데코레이터의 `dto` 속성과 함수의 반환 타입은 동일한 DTO를 사용해야 함
10. **컨트롤러-서비스 타입 분리**: 컨트롤러는 DTO를 반환 타입으로 사용하고, 서비스는 해당 DTO에 대응하는 인터페이스를 반환 타입으로 사용해야 함

**예시:**
```typescript
// Controller - DTO 사용
@Serialize({
  dto: RoleDetailDto,  // ← 이 DTO와
  ...RoleResponse.CREATE_SUCCESS,
})
async createRole(): Promise<RoleDetailDto> {  // ← 이 반환 타입이 같아야 함
  return this.roleService.createRole(dto);
}

// Service - 인터페이스 사용
async createRole(dto: CreateRoleDto): Promise<RoleDetail> {  // ← RoleDetailDto에 대응하는 인터페이스 사용
  const role = await this.roleRepository.save(/* ... */);
  return role; // Entity가 인터페이스를 구현
}
```

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

## NestJS 서버 공통 코딩 컨벤션

krgeobuk 생태계의 모든 NestJS 백엔드 서비스 (auth-server, authz-server, portal-server 등)에서 공통으로 적용해야 하는 코딩 규칙과 베스트 프랙티스입니다.

### TypeScript 코딩 표준

#### 타입 안전성 규칙
```typescript
// ✅ 올바른 예시 - 명시적 타입 지정
async function getUserById(id: string): Promise<UserEntity | null> {
  try {
    return await this.userRepo.findOneById(id);
  } catch (error: unknown) {
    this.logger.error('User fetch failed', { 
      error: error instanceof Error ? error.message : 'Unknown error',
      userId: id 
    });
    throw error;
  }
}

// ❌ 잘못된 예시 - 타입 누락
async function getUserById(id) {  // 타입 누락
  try {
    return await this.userRepo.findOneById(id);
  } catch (error) {  // unknown 타입 누락
    console.log(error);  // console 사용 금지
    throw error;
  }
}
```

**핵심 규칙:**
- **any 타입 완전 금지**: 모든 변수와 매개변수에 명시적 타입 지정
- **함수 반환값 타입 필수**: 모든 함수에 명시적 반환 타입 지정
- **catch 블록 타입**: `catch (error: unknown)` 패턴 사용
- **console 사용 금지**: Logger 클래스만 사용

### 서비스 클래스 구조 표준

#### 메서드 순서 및 그룹화 규칙
```typescript
@Injectable()
export class ExampleService {
  private readonly logger = new Logger(ExampleService.name);

  constructor(
    private readonly repo: ExampleRepository,
    // 의존성 주입
  ) {}

  // ==================== PUBLIC METHODS ====================

  // 1. 조회 메서드들 (가장 기본적인 CRUD 순서)
  async findById(id: string): Promise<Entity | null> { }
  async findByIdOrFail(id: string): Promise<Entity> { }
  async findByServiceIds(serviceIds: string[]): Promise<Entity[]> { }
  async findByAnd(filter: Filter): Promise<Entity[]> { }
  async findByOr(filter: Filter): Promise<Entity[]> { }

  // 2. 검색 및 상세 조회 메서드들  
  async searchEntities(query: SearchQuery): Promise<PaginatedResult> { }
  async getEntityDetail(id: string): Promise<Detail> { }

  // 3. 변경 메서드들 (생성 → 수정 → 삭제 순서)
  async createEntity(attrs: CreateAttrs): Promise<void> { }
  async updateEntity(id: string, attrs: UpdateAttrs): Promise<void> { }
  async deleteEntity(id: string): Promise<UpdateResult> { }

  // ==================== PRIVATE HELPER METHODS ====================

  // TCP 통신 관련 헬퍼들
  private async getExternalData(): Promise<ExternalData> { }
  private async notifyOtherServices(): Promise<void> { }

  // 데이터 변환 관련 헬퍼들
  private buildSearchResults(): SearchResult[] { }
  private buildFallbackResults(): SearchResult[] { }
  
  // 유틸리티 헬퍼들
  private validateBusinessRules(): boolean { }
  private formatResponseData(): FormattedData { }
}
```

#### 메서드 네이밍 표준
**조회 메서드:**
- `findById(id: string)` - 단일 엔티티 조회 (null 반환 가능)
- `findByIdOrFail(id: string)` - 단일 엔티티 조회 (예외 발생)
- `findByServiceIds(serviceIds: string[])` - 서비스 ID 배열로 조회
- `findByAnd(filter: Filter)` - AND 조건 검색
- `findByOr(filter: Filter)` - OR 조건 검색

**검색 메서드:**
- `searchEntities(query: SearchQuery)` - 페이지네이션 검색
- `getEntityDetail(id: string)` - 상세 정보 조회 (외부 데이터 포함)

**변경 메서드:**
- `createEntity(attrs: CreateAttrs)` - 엔티티 생성
- `updateEntity(id: string, attrs: UpdateAttrs)` - 엔티티 수정  
- `deleteEntity(id: string)` - 엔티티 삭제

**Private 메서드:**
- `build-` 접두사: 데이터 변환 및 구축
- `get-` 접두사: 외부 데이터 조회
- `validate-` 접두사: 비즈니스 규칙 검증
- `format-` 접두사: 데이터 포맷팅

### 로깅 표준 가이드라인

#### 로그 레벨 사용 기준
```typescript
// ERROR: 시스템 오류, 예외 상황
this.logger.error('Entity creation failed', {
  error: error instanceof Error ? error.message : 'Unknown error',
  entityId: id,
  operation: 'create',
});

// WARN: 비정상적이지만 처리 가능한 상황
this.logger.warn('External service unavailable, using fallback', {
  service: 'auth-service',
  fallbackUsed: true,
  entityId: id,
});

// LOG/INFO: 중요한 비즈니스 이벤트
this.logger.log('Entity created successfully', {
  entityId: result.id,
  entityType: 'Role',
  serviceId: result.serviceId,
});

// DEBUG: 개발용, 고빈도 호출 API, 상세 디버깅
this.logger.debug('TCP request received', {
  operation: 'findById',
  entityId: id,
  timestamp: new Date().toISOString(),
});
```

#### 로그 메시지 구조 표준
**메시지 포맷**: `"Action + result + context"`

```typescript
// ✅ 올바른 로그 메시지
this.logger.log('Role created successfully', { roleId: '123', roleName: 'Admin' });
this.logger.warn('Role creation failed: duplicate name', { name: 'Admin', serviceId: '456' });
this.logger.error('Database connection failed', { error: error.message, retryCount: 3 });

// ❌ 잘못된 로그 메시지  
this.logger.log('Success');  // 컨텍스트 부족
this.logger.error('Error occurred');  // 구체적이지 않음
this.logger.log('Creating role for user admin with name Test');  // 구조화되지 않음
```

**메타데이터 표준:**
- **필수 필드**: entityId, operation, timestamp (자동)
- **선택 필드**: serviceId, userId, error, retryCount, duration
- **민감정보 제외**: 비밀번호, 토큰, 개인정보

### Repository 최적화 표준

#### 쿼리 최적화 규칙
```typescript
// ✅ 올바른 쿼리 - SELECT 컬럼 명시
async searchEntities(query: SearchQuery): Promise<PaginatedResult<Partial<Entity>>> {
  const qb = this.createQueryBuilder('entity')
    .select([
      'entity.id',
      'entity.name',
      'entity.description',
      'entity.serviceId',
      // 필요한 컬럼만 선택
    ]);
    
  // 인덱스 활용을 위한 조건 순서 최적화
  if (query.serviceId) {
    qb.andWhere('entity.serviceId = :serviceId', { serviceId: query.serviceId });
  }
  
  if (query.name) {
    qb.andWhere('entity.name LIKE :name', { name: `%${query.name}%` });
  }

  // COUNT 쿼리와 데이터 쿼리 분리
  const [rows, total] = await Promise.all([
    qb.getRawMany(),
    qb.getCount()
  ]);

  // 타입 안전한 결과 매핑
  const items: Partial<Entity>[] = rows.map((row) => ({
    id: row.entity_id,
    name: row.entity_name,
    description: row.entity_description,
    serviceId: row.entity_service_id,
  }));

  return { items, pageInfo: this.buildPageInfo(total, query) };
}

// ❌ 비효율적 쿼리
async searchEntities(query: SearchQuery) {  // 반환 타입 누락
  const qb = this.createQueryBuilder('entity'); // SELECT * (비효율적)
  // ... 조건 추가
  const items = await qb.getMany(); // any 타입
  const total = await qb.getCount(); // 별도 쿼리 (비효율적)
  return { items, total };
}
```

**Repository 최적화 체크리스트:**
- [ ] SELECT 절에 필요한 컬럼만 명시
- [ ] 인덱스 활용을 위한 WHERE 조건 순서 최적화
- [ ] `Promise.all()`을 통한 COUNT와 데이터 쿼리 병렬 처리
- [ ] 명시적 타입 매핑 (`Partial<Entity>[]`)
- [ ] JOIN 조건 정확성 검증

### 에러 처리 표준

#### 에러 검증 및 메시지 패턴
```typescript
async createEntity(attrs: CreateAttrs): Promise<void> {
  try {
    // 1. 사전 검증 (비즈니스 규칙)
    if (attrs.name && attrs.serviceId) {
      const existing = await this.repo.findOne({
        where: { name: attrs.name, serviceId: attrs.serviceId }
      });
      
      if (existing) {
        this.logger.warn('Entity creation failed: duplicate name', {
          name: attrs.name,
          serviceId: attrs.serviceId,
        });
        throw EntityException.entityAlreadyExists();
      }
    }

    // 2. 추가 비즈니스 규칙 검증
    await this.validateBusinessRules(attrs);

    // 3. 비즈니스 로직 실행
    const entity = new EntityClass();
    Object.assign(entity, attrs);
    await this.repo.save(entity);
    
    // 4. 성공 로깅
    this.logger.log('Entity created successfully', {
      entityId: entity.id,
      name: attrs.name,
      serviceId: attrs.serviceId,
    });
  } catch (error: unknown) {
    // 5. 에러 처리 및 로깅
    if (error instanceof HttpException) {
      throw error; // 이미 처리된 예외는 그대로 전파
    }

    this.logger.error('Entity creation failed', {
      error: error instanceof Error ? error.message : 'Unknown error',
      name: attrs.name,
      serviceId: attrs.serviceId,
      stack: error instanceof Error ? error.stack : undefined,
    });
    
    throw EntityException.entityCreateError();
  }
}
```

**에러 처리 체크리스트:**
- [ ] 사전 검증으로 데이터베이스 에러 방지
- [ ] HttpException 인스턴스 체크 후 재전파
- [ ] 구조화된 에러 로깅 (error, context, stack)
- [ ] 사용자 친화적 에러 메시지 (`EntityException` 사용)
- [ ] 민감정보 제외한 로깅

### TCP 컨트롤러 표준

#### 메시지 패턴 네이밍 규칙
```typescript
@Controller()
export class EntityTcpController {
  private readonly logger = new Logger(EntityTcpController.name);

  constructor(private readonly entityService: EntityService) {}

  // 조회 패턴
  @MessagePattern('entity.findById')
  async findById(@Payload() data: { entityId: string }) {
    this.logger.debug(`TCP entity detail request: ${data.entityId}`);
    // ...
  }

  @MessagePattern('entity.findByServiceIds')
  async findByServiceIds(@Payload() data: { serviceIds: string[] }) {
    this.logger.debug('TCP entities by services request', {
      serviceCount: data.serviceIds.length,
    });
    // ...
  }

  // 검색 패턴
  @MessagePattern('entity.search')
  async search(@Payload() query: EntitySearchQuery) {
    this.logger.debug('TCP entity search request', {
      hasNameFilter: !!query.name,
      serviceId: query.serviceId,
    });
    // ...
  }

  // 변경 패턴
  @MessagePattern('entity.create')
  async create(@Payload() data: CreateEntity) {
    this.logger.log('TCP entity creation requested', {
      name: data.name,
      serviceId: data.serviceId,
    });
    // ...
  }

  @MessagePattern('entity.update')
  async update(@Payload() data: { entityId: string; updateData: UpdateEntity }) {
    this.logger.log('TCP entity update requested', { entityId: data.entityId });
    // ...
  }

  @MessagePattern('entity.delete')
  async delete(@Payload() data: { entityId: string }) {
    this.logger.log('TCP entity deletion requested', { entityId: data.entityId });
    // ...
  }

  // 유틸리티 패턴
  @MessagePattern('entity.exists')
  async exists(@Payload() data: { entityId: string }) {
    this.logger.debug(`TCP entity existence check: ${data.entityId}`);
    // ...
  }
}
```

**TCP 컨트롤러 로깅 최적화:**
- **고빈도 API** (findById, exists): `DEBUG` 레벨
- **중요한 변경 작업** (create, update, delete): `LOG` 레벨
- **검색 작업**: `DEBUG` 레벨 (필요시 `LOG`)

### 성능 최적화 지침

#### Repository 성능 최적화
```typescript
// ✅ 최적화된 패턴
async searchWithOptimization(query: SearchQuery): Promise<PaginatedResult> {
  // 1. 필요한 컬럼만 SELECT
  const qb = this.createQueryBuilder('entity')
    .select(['entity.id', 'entity.name', 'entity.serviceId']);

  // 2. 인덱스 활용 순서로 WHERE 조건 구성
  if (query.serviceId) { // 인덱스가 있는 컬럼 우선
    qb.andWhere('entity.serviceId = :serviceId', { serviceId: query.serviceId });
  }
  
  if (query.name) { // LIKE 조건은 나중에
    qb.andWhere('entity.name LIKE :name', { name: `%${query.name}%` });
  }

  // 3. COUNT와 데이터를 병렬로 조회
  const [rows, total] = await Promise.all([
    qb.offset(skip).limit(limit).getRawMany(),
    qb.getCount()
  ]);

  return this.buildPaginatedResult(rows, total, query);
}
```

#### 로깅 성능 최적화
```typescript
// ✅ 로그 레벨별 최적화
class OptimizedService {
  async highFrequencyOperation(id: string): Promise<Entity> {
    // 고빈도 API는 DEBUG 레벨로 최소화
    this.logger.debug('High frequency operation', { entityId: id });
    return await this.repo.findById(id);
  }

  async criticalOperation(data: CreateData): Promise<void> {
    // 중요한 작업만 LOG 레벨
    this.logger.log('Critical operation started', {
      operation: 'create',
      entityType: data.type,
    });
    
    try {
      await this.performCriticalWork(data);
      this.logger.log('Critical operation completed', { entityId: result.id });
    } catch (error: unknown) {
      this.logger.error('Critical operation failed', {
        error: error instanceof Error ? error.message : 'Unknown error',
        operation: 'create',
      });
      throw error;
    }
  }
}
```

### 적용 체크리스트

새로운 NestJS 서비스 또는 기존 서비스 개선 시 다음 항목들을 확인:

#### TypeScript 코딩 표준
- [ ] 모든 함수에 명시적 반환 타입 지정
- [ ] any 타입 완전 제거
- [ ] catch 블록에 `error: unknown` 사용
- [ ] console 대신 Logger 사용

#### 서비스 클래스 구조
- [ ] PUBLIC METHODS와 PRIVATE HELPER METHODS 섹션 분리
- [ ] 메서드 순서: 조회 → 검색 → 변경 → Private 헬퍼
- [ ] 표준 메서드 네이밍 적용

#### 로깅 시스템
- [ ] 적절한 로그 레벨 사용 (ERROR/WARN/LOG/DEBUG)
- [ ] 구조화된 로그 메시지 형식
- [ ] 민감정보 제외
- [ ] 메타데이터 객체 포함

#### Repository 최적화
- [ ] SELECT 컬럼 명시
- [ ] 인덱스 활용 WHERE 순서
- [ ] Promise.all로 병렬 쿼리
- [ ] 명시적 타입 매핑

#### 에러 처리
- [ ] 사전 검증 구현
- [ ] HttpException 체크 및 전파
- [ ] 구조화된 에러 로깅
- [ ] 공통 Exception 클래스 사용

#### TCP 컨트롤러
- [ ] 표준 메시지 패턴 네이밍
- [ ] 적절한 로그 레벨 적용
- [ ] 구조화된 페이로드 타입

이러한 표준을 준수하면 krgeobuk 생태계의 모든 NestJS 서비스에서 일관된 코드 품질과 유지보수성을 보장할 수 있습니다.

## 표준화된 도메인 API 설계 패턴

krgeobuk 생태계의 모든 도메인 모듈에서 일관된 API 구조와 서비스 계층 설계를 위한 표준 패턴입니다.

### 기본 CRUD API 구조

모든 도메인 컨트롤러는 다음 5가지 기본 API를 필수로 구현해야 합니다:

#### 1. 표준 API 엔드포인트

```typescript
@Controller('[domain]s')
export class [Domain]Controller {
  
  // 1. 목록 조회 (검색/페이지네이션)
  @Get()
  async search[Domain]s(
    @Query() query: [Domain]SearchQueryDto,
    @CurrentJwt() jwt: JwtPayload
  ): Promise<[Domain]PaginatedSearchResultDto> {
    return this.[domain]Service.search[Domain]s(query);
  }

  // 2. 상세 조회
  @Get(':id') 
  async get[Domain]ById(
    @Param('id') id: string,
    @CurrentJwt() jwt: JwtPayload
  ): Promise<[Domain]DetailDto> {
    return this.[domain]Service.get[Domain]ById(id);
  }

  // 3. 생성
  @Post()
  async create[Domain](
    @Body() dto: Create[Domain]Dto,
    @CurrentJwt() jwt: JwtPayload
  ): Promise<void> {
    await this.[domain]Service.create[Domain](dto);
  }

  // 4. 수정
  @Patch(':id')
  async update[Domain](
    @Param('id') id: string,
    @Body() dto: Update[Domain]Dto,
    @CurrentJwt() jwt: JwtPayload
  ): Promise<void> {
    await this.[domain]Service.update[Domain](id, dto);
  }

  // 5. 삭제
  @Delete(':id')
  async delete[Domain](
    @Param('id') id: string,
    @CurrentJwt() jwt: JwtPayload
  ): Promise<void> {
    await this.[domain]Service.delete[Domain](id);
  }
}
```

#### 2. 도메인별 추가 API

기본 5개 API 외에 도메인별 특수 요구사항이 있는 경우 추가 API를 구현합니다:

```typescript
// 예시: Role 도메인의 추가 API
@Get('service/:serviceId/roles')
async getRolesByService(@Param('serviceId') serviceId: string) { }

@Post(':roleId/permissions/:permissionId')
async assignPermissionToRole(@Param('roleId') roleId: string, @Param('permissionId') permissionId: string) { }
```

### 서비스 계층 설계 패턴

#### 1. 서비스 메서드 계층 구조

```typescript
@Injectable()
export class [Domain]Service {
  private readonly logger = new Logger([Domain]Service.name);

  constructor(
    private readonly [domain]Repo: [Domain]Repository,
    // 필요한 의존성들...
  ) {}

  // ==================== PUBLIC METHODS ====================

  // Level 1: 기본 Building Blocks (재사용 가능한 기본 메서드들)
  async findById(id: string): Promise<[Domain]Entity | null> {
    return this.[domain]Repo.findOneById(id);
  }

  async findByIdOrFail(id: string): Promise<[Domain]Entity> {
    const entity = await this.[domain]Repo.findOneById(id);
    if (!entity) {
      this.logger.debug('[Domain] not found', { [domain]Id: id });
      throw [Domain]Exception.[domain]NotFound();
    }
    return entity;
  }

  async findByServiceIds(serviceIds: string[]): Promise<[Domain]Entity[]> {
    return this.[domain]Repo.find({ where: { serviceId: In(serviceIds) } });
  }

  async findByAnd(filter: [Domain]Filter = {}): Promise<[Domain]Entity[]> {
    // AND 조건 검색 로직
  }

  async findByOr(filter: [Domain]Filter = {}): Promise<[Domain]Entity[]> {
    // OR 조건 검색 로직
  }

  // Level 2: 컨트롤러 매칭 메서드 (Level 1 조합 + 비즈니스 로직)
  async search[Domain]s(query: [Domain]SearchQueryDto): Promise<PaginatedResult<[Domain]SearchResult>> {
    const entities = await this.[domain]Repo.search[Domain]s(query);
    
    if (entities.items.length === 0) {
      return { items: [], pageInfo: entities.pageInfo };
    }

    try {
      // 외부 데이터 조합 (TCP 통신 등)
      const [externalData1, externalData2] = await Promise.all([
        this.getExternalData1(),
        this.getExternalData2(),
      ]);

      const items = this.build[Domain]SearchResults(entities.items, externalData1, externalData2);
      return { items, pageInfo: entities.pageInfo };
    } catch (error: unknown) {
      // 폴백 처리
      this.logger.warn('External service communication failed, using fallback data', {
        error: error instanceof Error ? error.message : 'Unknown error',
      });
      
      const items = this.buildFallback[Domain]SearchResults(entities.items);
      return { items, pageInfo: entities.pageInfo };
    }
  }

  async get[Domain]ById(id: string): Promise<[Domain]Detail> {
    const entity = await this.findByIdOrFail(id);

    try {
      // 외부 데이터와 조합하여 상세 정보 구축
      const [service, relatedData] = await Promise.all([
        this.getServiceById(entity.serviceId),
        this.getRelatedData(id),
      ]);

      return {
        id: entity.id,
        // 도메인별 필드들...
        service,
        relatedData,
      };
    } catch (error: unknown) {
      // 폴백 처리
      this.logger.warn('Failed to enrich [domain] with external data, returning basic info', {
        error: error instanceof Error ? error.message : 'Unknown error',
        [domain]Id: id,
      });

      return {
        id: entity.id,
        // 기본 필드들...
        service: { id: '', name: 'Service unavailable' },
        relatedData: [],
      };
    }
  }

  async create[Domain](dto: Create[Domain]Dto, transactionManager?: EntityManager): Promise<void> {
    try {
      // 1. 사전 검증 (비즈니스 규칙)
      await this.validateCreate[Domain](dto);

      // 2. 엔티티 생성 및 저장
      const entity = new [Domain]Entity();
      Object.assign(entity, dto);
      await this.[domain]Repo.saveEntity(entity, transactionManager);

      // 3. 성공 로깅
      this.logger.log('[Domain] created successfully', {
        [domain]Id: entity.id,
        // 관련 컨텍스트...
      });
    } catch (error: unknown) {
      // 4. 에러 처리
      if (error instanceof HttpException) {
        throw error;
      }

      this.logger.error('[Domain] creation failed', {
        error: error instanceof Error ? error.message : 'Unknown error',
        // 컨텍스트...
      });

      throw [Domain]Exception.[domain]CreateError();
    }
  }

  async update[Domain](id: string, dto: Update[Domain]Dto, transactionManager?: EntityManager): Promise<void> {
    try {
      // 1. 존재 여부 확인
      const entity = await this.findByIdOrFail(id);

      // 2. 사전 검증 (변경 사항 검증)
      await this.validateUpdate[Domain](entity, dto);

      // 3. 업데이트 실행
      Object.assign(entity, dto);
      await this.[domain]Repo.updateEntity(entity, transactionManager);

      // 4. 성공 로깅
      this.logger.log('[Domain] updated successfully', {
        [domain]Id: id,
        updatedFields: Object.keys(dto),
      });
    } catch (error: unknown) {
      // 에러 처리
      if (error instanceof HttpException) {
        throw error;
      }

      this.logger.error('[Domain] update failed', {
        error: error instanceof Error ? error.message : 'Unknown error',
        [domain]Id: id,
      });

      throw [Domain]Exception.[domain]UpdateError();
    }
  }

  async delete[Domain](id: string): Promise<UpdateResult> {
    try {
      // 1. 존재 여부 확인
      const entity = await this.findByIdOrFail(id);

      // 2. 삭제 가능 여부 검증 (관련 데이터 확인)
      await this.validateDelete[Domain](entity);

      // 3. 소프트 삭제 실행
      const result = await this.[domain]Repo.softDelete(id);

      // 4. 성공 로깅
      this.logger.log('[Domain] deleted successfully', {
        [domain]Id: id,
      });

      return result;
    } catch (error: unknown) {
      // 에러 처리
      if (error instanceof HttpException) {
        throw error;
      }

      this.logger.error('[Domain] deletion failed', {
        error: error instanceof Error ? error.message : 'Unknown error',
        [domain]Id: id,
      });

      throw [Domain]Exception.[domain]DeleteError();
    }
  }

  // ==================== PRIVATE HELPER METHODS ====================

  // Level 3: Private Helper Methods
  private async getExternalData(): Promise<ExternalData> {
    // TCP 통신 등 외부 데이터 조회
  }

  private build[Domain]SearchResults(): [Domain]SearchResult[] {
    // 검색 결과 데이터 변환
  }

  private buildFallback[Domain]SearchResults(): [Domain]SearchResult[] {
    // 폴백 검색 결과 구축
  }

  private async validateCreate[Domain](dto: Create[Domain]Dto): Promise<void> {
    // 생성 전 비즈니스 규칙 검증
  }

  private async validateUpdate[Domain](entity: [Domain]Entity, dto: Update[Domain]Dto): Promise<void> {
    // 수정 전 비즈니스 규칙 검증
  }

  private async validateDelete[Domain](entity: [Domain]Entity): Promise<void> {
    // 삭제 전 비즈니스 규칙 검증
  }
}
```

### 컨트롤러 ↔ 서비스 매칭 규칙

#### 1. 함수명 1:1 매칭

```typescript
// Controller Method          →  Service Method
search[Domain]s()            →  search[Domain]s()
get[Domain]ById()            →  get[Domain]ById()  
create[Domain]()             →  create[Domain]()
update[Domain]()             →  update[Domain]()
delete[Domain]()             →  delete[Domain]()
```

#### 2. 반환 타입 일관성

```typescript
// Controller: DTO 타입 사용
async get[Domain]ById(): Promise<[Domain]DetailDto> {
  return this.[domain]Service.get[Domain]ById(id);
}

// Service: 인터페이스 타입 사용  
async get[Domain]ById(): Promise<[Domain]Detail> {
  // 비즈니스 로직...
  return result; // Entity가 인터페이스를 구현
}
```

### 적용 체크리스트

새로운 도메인 모듈 개발 시 다음 항목들을 확인:

#### 컨트롤러 구조
- [ ] 5가지 기본 API 엔드포인트 구현 (search, get, create, update, delete)
- [ ] 일관된 함수명 패턴 적용
- [ ] 공통 패키지의 Response, Error, DTO 사용
- [ ] 적절한 Swagger 문서화
- [ ] AccessTokenGuard 적용

#### 서비스 구조  
- [ ] PUBLIC METHODS와 PRIVATE HELPER METHODS 섹션 분리
- [ ] 기본 Building Blocks 메서드 구현 (findById, findByIdOrFail 등)
- [ ] 컨트롤러 매칭 메서드 구현 (Level 2)
- [ ] Private Helper Methods 구현 (Level 3)
- [ ] 적절한 로깅 및 에러 처리

#### 일관성 검증
- [ ] 컨트롤러-서비스 함수명 1:1 매칭
- [ ] 반환 타입 일관성 (DTO vs Interface)
- [ ] 공통 패키지 활용도
- [ ] TCP 컨트롤러 구현 (필요시)

#### 확장성 고려
- [ ] 도메인별 추가 API 설계 (필요시)
- [ ] 기본 메서드 재사용 패턴 적용
- [ ] 외부 서비스 통신 및 폴백 처리

이 패턴을 따르면 모든 도메인 모듈에서 일관된 API 설계와 서비스 구조를 유지하면서도 각 도메인의 특수 요구사항을 유연하게 수용할 수 있습니다.