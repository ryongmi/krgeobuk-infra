# Migration & Constants 리팩토링 계획

## 개요

이 문서는 두 가지 작업을 묶어서 진행하는 계획입니다.

1. **TypeORM Migration 설정** - 미니PC/RDS 환경에서 스키마 생성 및 초기 데이터 주입
2. **Authorization Constants 리팩토링** - 도메인별 패키지로 상수 이관 + superAdmin 글로벌 역할 처리 개선

---

## 작업 1: Authorization Constants 리팩토링

### 1-1. AUTHZ_PERMISSIONS 도메인 패키지로 분산

현재 `@krgeobuk/authorization/constants`에 모든 권한 상수가 집중되어 있음.
테이블(도메인) 단위로 패키지를 분리하는 기존 설계 원칙에 맞게 이관.

**이관 대상 및 위치:**

| 기존 상수 | 새 위치 | 새 파일 |
|----------|---------|---------|
| `ROLE_CREATE/READ/UPDATE/DELETE/MANAGE` | `@krgeobuk/role` | `src/constants/role.constants.ts` |
| `PERMISSION_CREATE/READ/UPDATE/DELETE/MANAGE` | `@krgeobuk/permission` | `src/constants/permission.constants.ts` |
| `USER_ROLE_ASSIGN/REVOKE/MANAGE` | `@krgeobuk/user-role` | `src/constants/user-role.constants.ts` |
| `ROLE_PERMISSION_ASSIGN/REVOKE/MANAGE` | `@krgeobuk/role-permission` | `src/constants/role-permission.constants.ts` |
| `SERVICE_VISIBLE_ROLE_MANAGE` | `@krgeobuk/service-visible-role` | `src/constants/service-visible-role.constants.ts` |
| `AUTHORIZATION_CHECK/MANAGE` | `@krgeobuk/authorization` | 기존 파일 유지 (정리) |

**이관 후 `@krgeobuk/authorization/constants` 잔류 항목:**
- 메타키 상수 (REQUIRED_ANY_PERMISSIONS_META_KEY 등)
- `AUTHORIZATION_CHECK`, `AUTHORIZATION_MANAGE`

### 1-2. AUTHZ_ROLES 도메인 패키지로 분산

| 기존 상수 | 새 위치 | 상수명 |
|----------|---------|--------|
| `AUTHZ_ROLES.ROLE_MANAGER` | `@krgeobuk/role/constants` | `RoleNames.ROLE_MANAGER` |
| `AUTHZ_ROLES.PERMISSION_MANAGER` | `@krgeobuk/permission/constants` | `PermissionNames.PERMISSION_MANAGER` |

### 1-3. @krgeobuk/core에 user.constants.ts 추가

초기 admin 유저 UUID는 auth-server DB에 있고, authz-server의 `SeedAdminUserRole` migration에서 cross-DB로 참조해야 함.
`service.constants.ts`와 동일한 방식으로 `user.constants.ts`에 미리 고정 UUID 지정.

**생성 위치:** `@krgeobuk/core/src/constants/user.constants.ts`

```typescript
export const USER_CONSTANTS = {
  SUPER_ADMIN: {
    id: '550e8400-e29b-41d4-a716-446655440010',  // 고정 UUID
    email: 'admin@krgeobuk.com',
    name: '관리자',
  }
} as const;
```

> `service.constants.ts`와 동일하게 `@krgeobuk/core/constants` index에 export 추가.

### 1-4. 신규 서비스 권한 상수 추가

기존 패키지에 없는 auth-server, portal-server 권한 상수를 해당 도메인 패키지에 추가.

| 서버 | 패키지 | 파일 | 상수명 |
|------|--------|------|--------|
| auth-server (사용자) | `@krgeobuk/user` | `src/constants/user.constants.ts` | `UserPermissions` |
| portal-server (서비스 관리) | `@krgeobuk/service` | `src/constants/service.constants.ts` | `ServicePermissions` |

**UserPermissions (auth-server):**
```
user:read, user:create, user:update, user:delete, user:manage
```

**ServicePermissions (portal-server):**
```
service:read, service:create, service:update, service:delete, service:manage
```

> mypick-server 관련 패키지는 미구현 상태로 추후 별도 진행.

### 1-5. authz-server 컨트롤러 import 수정

`@krgeobuk/authorization/constants` → 각 도메인 패키지로 import 경로 변경.

**영향 파일 (authz-server):**
- `src/modules/role/role.controller.ts`
- `src/modules/permission/permission.controller.ts`
- `src/modules/user-role/user-role.controller.ts`
- `src/modules/role-permission/role-permission.controller.ts`
- `src/modules/service-visible-role/service-visible-role.controller.ts`
- `src/modules/authorization/authorization.controller.ts`

### 1-6. package.json exports 추가

각 도메인 패키지에 `constants` 경로 exports 추가.

```json
// 예시: @krgeobuk/role/package.json
{
  "exports": {
    "./constants": "./dist/constants/index.js"
  },
  "typesVersions": {
    "*": {
      "constants": ["dist/constants/index.d.ts"]
    }
  }
}
```

---

## 작업 2: superAdmin 글로벌 역할 처리 개선

### 2-1. authorization.service.ts - checkUserRole 수정

현재 모든 역할 체크가 `service_visible_role` 필터를 우회하는 문제 수정.

**현재 동작:**
- `roleServiceId` 미지정 시 모든 역할이 서비스 필터 없이 통과
- auth-service의 `admin`이 authz-server의 `admin` 체크도 통과할 수 있음

**변경 후 동작:**
- `superAdmin`: `isGlobalRole` 판단으로 서비스 필터 완전 우회 (의도된 동작)
- 그 외 역할: `roleServiceId` 지정 시 `service_visible_role` 필터 적용

```typescript
// authorization.service.ts - checkUserRole 변경 내용
async checkUserRole(attrs: CheckRole): Promise<boolean> {
  const { userId, roleName, serviceId } = attrs;

  const userRoleIds = await this.getUserRolesWithCache(userId);

  // superAdmin은 서비스 경계를 넘는 글로벌 역할 → 필터 없이 전체 역할에서 검색
  const isGlobalRole = roleName === GLOBAL_ROLES.SUPER_ADMIN;

  let visibleRoleIds = userRoleIds;
  if (!isGlobalRole && serviceId) {
    const serviceVisibleRoleIds = await this.serviceVisibleRoleService.getRoleIds(serviceId);
    visibleRoleIds = userRoleIds.filter(id => serviceVisibleRoleIds.includes(id));
    if (visibleRoleIds.length === 0) return false;
  }

  const roles = await this.roleService.findByIds(visibleRoleIds);
  return roles.some(role => role.name.toLowerCase() === roleName.toLowerCase());
}
```

### 2-2. authz-server 컨트롤러에 roleServiceId 추가

서비스별 admin 역할이 다른 서버에서 오용되지 않도록 `roleServiceId` 지정.

```typescript
// 예시: role.controller.ts
@RequireAccess({
  permissions: [RolePermissions.ROLE_READ],
  roles: [GLOBAL_ROLES.SUPER_ADMIN, RoleNames.ROLE_MANAGER],
  combinationOperator: 'OR',
  roleServiceId: SERVICE_CONSTANTS.AUTHZ_SERVICE.id,  // 추가
})
```

---

## 작업 3: TypeORM Migration 설정

### 3-1. data-source.ts 생성 (서버별)

TypeORM CLI용 DataSource 파일을 각 서버에 생성.
NestJS DI 없이 환경변수를 직접 읽는 방식.

**생성 위치:**
- `auth-server/src/database/data-source.ts`
- `authz-server/src/database/data-source.ts`
- `portal-server/src/database/data-source.ts`

### 3-2. migration 스크립트 추가 (package.json)

```json
{
  "scripts": {
    "migration:generate": "typeorm-ts-node-esm migration:generate -d src/database/data-source.ts",
    "migration:run": "typeorm-ts-node-esm migration:run -d src/database/data-source.ts",
    "migration:revert": "typeorm-ts-node-esm migration:revert -d src/database/data-source.ts"
  }
}
```

### 3-3. Migration 파일 작성

#### portal-server (선행 필수 - serviceId 기준값 생성)

| Migration 파일 | 내용 |
|---------------|------|
| `CreateServiceTable` | service 테이블 생성 |
| `SeedInitialServices` | 4개 서비스 초기 데이터 |

**SeedInitialServices 데이터:**

| id | name | displayName | isVisible | isVisibleByRole |
|----|------|------------|-----------|----------------|
| `550e8400-...-0001` | auth-service | 인증 서비스 | false | false |
| `550e8400-...-0002` | authz-service | 권한 서비스 | false | false |
| `550e8400-...-0003` | portal-service | 포탈 서비스 | true | false |
| `550e8400-...-0004` | mypick-server | MyPick 서비스 | true | true |

#### authz-server

| Migration 파일 | 내용 |
|---------------|------|
| `CreateAuthzTables` | role, permission, role_permission, user_role, service_visible_role 테이블 생성 |
| `SeedInitialRoles` | superAdmin + 서비스별 admin 롤 |
| `SeedInitialPermissions` | 서비스별 권한 액션 |
| `SeedRolePermissions` | 역할-권한 매핑 |
| `SeedServiceVisibleRoles` | 서비스 가시성 역할 매핑 |

**SeedInitialRoles 데이터:**

| name | serviceId | priority |
|------|-----------|----------|
| `superAdmin` | portal-service(`...0003`) | 1 |
| `admin` | auth-service(`...0001`) | 2 |
| `admin` | authz-service(`...0002`) | 2 |
| `admin` | portal-service(`...0003`) | 2 |
| `admin` | mypick-service(`...0004`) | 2 |
| `roleManager` | authz-service(`...0002`) | 3 |
| `permissionManager` | authz-service(`...0002`) | 3 |

**SeedInitialPermissions 데이터:**

auth-service 권한 (`UserPermissions`):
```
user:read, user:create, user:update, user:delete, user:manage
```

authz-service 권한 (`RolePermissions`, `PermissionPermissions`, `UserRolePermissions`, `RolePermissionPermissions`, `ServiceVisibleRolePermissions`, `AuthorizationPermissions`):
```
role:read, role:create, role:update, role:delete, role:manage
permission:read, permission:create, permission:update, permission:delete, permission:manage
user-role:assign, user-role:revoke, user-role:manage
role-permission:assign, role-permission:revoke, role-permission:manage
service-visible-role:assign, service-visible-role:revoke, service-visible-role:manage
authorization:check, authorization:manage
```

portal-service 권한 (`ServicePermissions`):
```
service:read, service:create, service:update, service:delete, service:manage
```

**SeedRolePermissions - superAdmin은 모든 권한 보유:**

**SeedServiceVisibleRoles 데이터:**

| serviceId | roleId |
|-----------|--------|
| auth-service | superAdmin |
| authz-service | superAdmin |
| portal-service | superAdmin |
| mypick-service | superAdmin |
| auth-service | admin(auth) |
| authz-service | admin(authz), roleManager, permissionManager |
| portal-service | admin(portal) |
| mypick-service | admin(mypick) |

#### auth-server

| Migration 파일 | 내용 |
|---------------|------|
| `CreateAuthTables` | user, oauth_account, account_merge 테이블 생성 |
| `SeedAdminUser` | 초기 superAdmin 계정 생성 |
| `SeedAdminUserRole` | superAdmin 계정에 superAdmin 역할 할당 |

`SeedAdminUser`에서 `USER_CONSTANTS.SUPER_ADMIN.id`를 고정 UUID로 사용.
`SeedAdminUserRole`에서도 동일한 상수를 참조하여 cross-DB 참조 문제 해결.

> 비밀번호는 환경변수에서 주입 (bcrypt hash 처리).

---

## 작업 순서

```
[1단계] shared-lib 패키지 수정 (constants 이관) ✅
  ├── @krgeobuk/core         - user.constants.ts 추가 (USER_CONSTANTS)
  ├── @krgeobuk/role         - constants 추가
  ├── @krgeobuk/permission   - constants 추가
  ├── @krgeobuk/user-role    - constants 추가
  ├── @krgeobuk/role-permission - constants 추가
  ├── @krgeobuk/service-visible-role - constants 추가
  ├── @krgeobuk/user         - constants 추가 (신규, UserPermissions)
  ├── @krgeobuk/service      - constants 추가 (신규, ServicePermissions)
  └── @krgeobuk/authorization - 불필요 상수 정리

[2단계] shared-lib 빌드 & 배포 ✅
  └── pnpm build → pnpm verdaccio:publish (각 패키지)

[3단계] authz-server 수정 ✅
  ├── authorization.service.ts - superAdmin 글로벌 역할 처리 수정
  └── 컨트롤러 6개 - import 경로 수정 + serviceId 수정 (roleServiceId 오타 수정)

[4단계] Migration 설정 (서버별 data-source.ts + package.json 스크립트) ✅
  ├── portal-server - data-source.ts + migration:run/revert (--env-file=envs/.env.migration)
  ├── authz-server  - data-source.ts + migration:run/revert (--env-file=envs/.env.migration)
  └── auth-server   - data-source.ts + migration:run/revert (--env-file=envs/.env.migration)

[5단계] Migration 파일 작성 ✅
  ├── portal-server: CreateServiceTable → SeedInitialServices
  ├── authz-server: CreateAuthzTables → SeedRoles → SeedPermissions → SeedRolePermissions → SeedServiceVisibleRoles → SeedAdminUserRole
  └── auth-server: CreateAuthTables → SeedAdminUser → SeedAdminOauthAccount

[6단계] 검증 ✅
  └── 로컬 Docker 환경에서 migration:run 실행 완료 (2026-03-16)
```

---

## 미결 사항

- [x] `SeedAdminUser` 초기 관리자 계정 비밀번호 확정 → `2316@@qwer` (scrypt hash 처리 후 저장)
- [x] 로컬 Docker 환경 migration 검증 완료
- [ ] mypick-server 권한 상수 및 migration은 추후 별도 진행
