# 권한 관리 API 통합 가이드

이 문서는 krgeobuk-infra 프로젝트에서 권한 관리 시스템의 API 통합 구현 내역을 정리합니다.

## 개요

권한 관리는 authz-server(포트 8100)에서 제공하며, 개별 권한의 CRUD 작업과 검색 기능을 담당합니다. Portal-client에서는 관리자 인터페이스를 통해 권한을 관리할 수 있습니다.

## API 엔드포인트 매핑

### authz-server 권한 API

| 메서드 | 엔드포인트 | 설명 | 응답 타입 |
|--------|------------|------|-----------|
| GET | `/api/permissions` | 권한 목록 조회 (페이지네이션) | `PaginatedResult<PermissionSearchResultDto>` |
| GET | `/api/permissions/:id` | 권한 상세 조회 | `PermissionDetailDto` |
| POST | `/api/permissions` | 권한 생성 | `PermissionDetailDto` |
| PATCH | `/api/permissions/:id` | 권한 수정 | `PermissionDetailDto` |
| DELETE | `/api/permissions/:id` | 권한 삭제 | `void` |

### Portal-client 권한 관리 페이지

- **경로**: `/admin/authorization/permissions`
- **접근 권한**: 관리자 전용
- **주요 기능**: 권한 CRUD, 검색, 페이지네이션

## 구현 세부사항

### 1. 공통 라이브러리 DTO 통합

**기존 로컬 DTO → 공통패키지 DTO 대체**

```typescript
// Before: authz-server 로컬 DTO
import { CreatePermissionDto, UpdatePermissionDto, PermissionResponseDto } from './dtos/index.js';

// After: 공통패키지 DTO
import {
  CreatePermissionDto,
  UpdatePermissionDto,
  PermissionDetailDto,        // PermissionResponseDto 대체
  PermissionSearchQueryDto,
  PermissionSearchResultDto,
} from '@krgeobuk/permission/dtos';
```

### 2. 새로 생성한 공통 DTO

**CreatePermissionDto**
```typescript
// shared-lib/packages/permission/src/dtos/create.dto.ts
export class CreatePermissionDto implements CreatePermission {
  @IsValidPermissionAction()
  action!: string;

  @IsValidPermissionDescription({ isOptional: true })
  description?: string | null;

  @IsValidServiceId()
  serviceId!: string;
}
```

**UpdatePermissionDto**
```typescript
// shared-lib/packages/permission/src/dtos/update.dto.ts
export class UpdatePermissionDto extends UuidIdDto implements UpdatePermission {
  @IsValidPermissionAction({ isOptional: true })
  action?: string;

  @IsValidPermissionDescription({ isOptional: true })
  description?: string | null;

  @IsValidServiceId({ isOptional: true })
  serviceId?: string;
}
```

### 3. 새로 생성한 인터페이스

**CreatePermission Interface**
```typescript
// shared-lib/packages/permission/src/interfaces/create.interface.ts
export interface CreatePermission {
  action: string;
  description?: string | null;
  serviceId: string;
}
```

**UpdatePermission Interface**
```typescript
// shared-lib/packages/permission/src/interfaces/update.interface.ts
export interface UpdatePermission extends UuidId {
  action?: string;
  description?: string | null;
  serviceId?: string;
}
```

### 4. 컨트롤러 개선사항

**공통패키지 기반 Swagger 문서화**

```typescript
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
): Promise<PermissionDetailDto> {
  return this.permissionService.createPermission(dto);
}
```

### 5. 서비스 계층 에러 처리

**Service에서 공통패키지 예외 사용**

```typescript
// permission.service.ts
import { PermissionException } from '@krgeobuk/permission/exception';

async findByIdOrFail(id: string): Promise<PermissionEntity> {
  const permission = await this.permissionRepo.findOneById(id);
  
  if (!permission) {
    throw PermissionException.permissionNotFound();
  }
  
  return permission;
}

async createPermission(attrs: Partial<PermissionEntity>): Promise<PermissionEntity> {
  try {
    const permissionEntity = new PermissionEntity();
    Object.assign(permissionEntity, attrs);
    return await this.permissionRepo.saveEntity(permissionEntity);
  } catch (error: unknown) {
    throw PermissionException.permissionCreateError();
  }
}
```

## TypeScript 타입 개선

### 엄격한 타입 지정

```typescript
// Before: any 사용 금지
catch (error: any) { ... }

// After: unknown 타입 사용
catch (error: unknown) {
  if (error instanceof HttpException) {
    throw error;
  }
  throw PermissionException.permissionUpdateError();
}
```

### 명시적 반환 타입

```typescript
// Controller 메서드에 명시적 반환 타입 지정
async getPermissions(
  @Query() query: PermissionSearchQueryDto
): Promise<PaginatedResult<PermissionDetailDto>> {
  return this.permissionService.searchPermissions(query);
}
```

## API 응답 형식

### 성공 응답 (SerializerInterceptor)

```typescript
{
  code: "PERMISSION_200",           // PermissionCode.PERMISSION_FETCH_SUCCESS
  status_code: 200,                 // HTTP 상태 코드
  message: "권한 조회 성공",         // PermissionMessage.PERMISSION_FETCH_SUCCESS
  isLogin: true,                    // 사용자 로그인 상태
  data: {                          // snake_case로 변환된 실제 데이터
    id: "123e4567-e89b-12d3-a456-426614174000",
    action: "user:create",
    description: "사용자 생성 권한",
    service: {
      id: "service-uuid",
      name: "auth-server"
    },
    roles: [...]
  }
}
```

### 에러 응답 (HttpExceptionFilter)

```typescript
{
  statusCode: 404,
  code: "PERMISSION_100",           // PermissionCode.PERMISSION_NOT_FOUND
  message: "해당 권한을 찾을 수 없습니다."  // PermissionMessage.PERMISSION_NOT_FOUND
}
```

## 삭제된 파일 목록

다음 로컬 DTO 파일들이 공통패키지로 대체되어 삭제되었습니다:

```
authz-server/src/modules/permission/dtos/
├── create-permission.dto.ts      (삭제됨)
├── update-permission.dto.ts      (삭제됨)
├── permission-search-query.dto.ts (삭제됨)
├── permission-response.dto.ts    (삭제됨)
└── index.ts                      (공통패키지 참조로 변경)
```

## 공통패키지 파일 구조

```
shared-lib/packages/permission/src/
├── dtos/
│   ├── create.dto.ts          (새로 생성)
│   ├── update.dto.ts          (새로 생성)
│   ├── detail.dto.ts          (기존)
│   ├── search-query.dto.ts    (기존)
│   ├── search-result.dto.ts   (기존)
│   └── index.ts               (업데이트됨)
├── interfaces/
│   ├── create.interface.ts    (새로 생성)
│   ├── update.interface.ts    (새로 생성)
│   └── index.ts               (업데이트됨)
├── codes/
│   └── permission-code.constant.ts
├── messages/
│   └── permission.message.ts
├── response/
│   └── permission.response.ts
└── exception/
    ├── permission.error.ts
    └── permission.exception.ts
```

## 장점 및 효과

### 1. 코드 일관성
- 모든 서비스에서 동일한 유효성 검사 규칙 적용
- 통일된 API 스키마 및 문서화
- 표준화된 에러 처리

### 2. 유지보수성
- 중앙화된 DTO 관리로 변경사항 전파 용이
- 중복 코드 제거로 유지보수 비용 절감
- 공통 패키지 업데이트 시 모든 서비스에 자동 반영

### 3. 타입 안전성
- 공통패키지의 고급 데코레이터 활용 (`@IsValidPermissionAction`, `@ExposePermissionAction` 등)
- 컴파일 타임 타입 검증 강화
- TypeScript 엄격 모드 준수

### 4. API 문서화 향상
- Swagger 문서의 일관성 보장
- 도메인별 표준화된 스키마 제공
- 에러 응답 문서화 완성도 향상

## 향후 개선사항

1. **권한 그룹화**: 관련 권한들을 그룹으로 묶어서 관리
2. **벌크 오퍼레이션**: 여러 권한을 한 번에 생성/수정/삭제
3. **권한 템플릿**: 자주 사용되는 권한 조합을 템플릿으로 저장
4. **권한 상속**: 상위 권한에서 하위 권한 자동 파생
5. **실시간 알림**: 권한 변경 시 관련 사용자에게 알림

이 문서는 권한 관리 API 통합의 완전한 구현 가이드로, 향후 유사한 도메인 모듈 개발 시 참조 자료로 활용할 수 있습니다.