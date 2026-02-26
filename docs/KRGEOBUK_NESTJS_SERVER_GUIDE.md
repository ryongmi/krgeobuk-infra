# krgeobuk NestJS 서버 개발 가이드

이 문서는 krgeobuk 생태계의 모든 NestJS 서버 개발 시 준수해야 하는 공통 표준과 패턴을 정의합니다.

## 📋 목차

1. [환경 설정 및 인프라](#환경-설정-및-인프라)
2. [공유 라이브러리 통합](#공유-라이브러리-통합)
3. [서비스 아키텍처 패턴](#서비스-아키텍처-패턴)
4. [컨트롤러 패턴](#컨트롤러-패턴)
5. [Repository 패턴](#repository-패턴)
6. [DTO 및 유효성 검사](#dto-및-유효성-검사)
7. [에러 처리 및 예외 관리](#에러-처리-및-예외-관리)
8. [로깅 표준](#로깅-표준)
9. [API 응답 형식](#api-응답-형식)
10. [마이크로서비스 통신](#마이크로서비스-통신)
11. [성능 최적화](#성능-최적화)
12. [코드 품질 및 표준](#코드-품질-및-표준)
13. [테스트 가이드](#테스트-가이드)

---

## 환경 설정 및 인프라

### 1. 네트워크 아키텍처

krgeobuk 생태계는 다음과 같은 Docker 네트워크 구조를 사용합니다:

- **msa-network**: 마이크로서비스 간 통신
- **shared-network**: 공유 리소스 접근
- **auth-network**: auth-server 내부 통신
- **authz-network**: authz-server 내부 통신

### 2. 서비스별 포트 구성

```bash
# HTTP 서버 포트
- auth-server:   8000
- authz-server:  8100
- portal-client: 3100

# TCP 마이크로서비스 포트 (NestJS connectMicroservice)
- auth-server TCP:   8010
- authz-server TCP:  8110

# 데이터베이스 포트 (단일 컨테이너, 서비스별 DB 분리)
- MySQL:  3306  (auth, authz 등 DB를 하나의 컨테이너에서 분리 운영)
- Redis:  6379
```

### 3. Docker 환경 관리

#### 기본 Docker 명령어

```bash
# 로컬 개발 환경
npm run docker:local:up    # 로컬 Docker 스택 시작
npm run docker:local:down  # 로컬 Docker 스택 중지

# 개발/프로덕션 환경
npm run docker:dev:up      # 개발 Docker 환경
npm run docker:prod:up     # 프로덕션 Docker 환경
```

#### Docker Compose 설정 예시

```yaml
# docker-compose.local.yml 예시
version: "3.8"
services:
  authz-server:
    build: .
    ports:
      - "8100:8100"
      - "8110:8110"   # TCP 마이크로서비스 포트
    networks:
      - msa-network
      - shared-network
    environment:
      - NODE_ENV=local
      - MYSQL_PORT=3306
      - REDIS_PORT=6379

networks:
  msa-network:
    external: true
  shared-network:
    external: true
```

### 4. 환경 변수 관리

#### 디렉토리 구조

```
/envs
├── local.env          # 로컬 개발
├── dev.env            # 개발 서버
└── prod.env           # 프로덕션
```

#### 환경 변수 예시

```bash
# envs/local.env
NODE_ENV=local
PORT=8100
TCP_PORT=8110          # TCP 마이크로서비스 포트
APP_NAME=authz-server

# Database (단일 MySQL 컨테이너, DB 이름으로 서비스 분리)
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=krgeobuk
MYSQL_PASSWORD=password
MYSQL_DATABASE=authz   # auth-server는 auth, authz-server는 authz

# Redis (단일 Redis 컨테이너)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=your-redis-password

# JWT
JWT_ACCESS_PUBLIC_KEY_PATH=./keys/access-public.key

# Microservice Communication
PORTAL_SERVICE_HOST=localhost
PORTAL_SERVICE_PORT=8200
AUTH_SERVICE_HOST=localhost
AUTH_SERVICE_PORT=8010
```

---

## 공유 라이브러리 통합

### 1. @krgeobuk 스코프 패키지 구조

#### 핵심 인프라 패키지

```typescript
// 필수 패키지들
import { CoreModule } from "@krgeobuk/core";
import { DatabaseConfigModule } from "@krgeobuk/database-config";
import { SwaggerModule } from "@krgeobuk/swagger";
```

#### 도메인별 패키지

```typescript
// 도메인 특화 패키지들
import { AuthModule } from "@krgeobuk/auth";
import { JwtModule } from "@krgeobuk/jwt";
import { UserModule } from "@krgeobuk/user";
import { RoleModule } from "@krgeobuk/role";
```

### 2. Verdaccio 프라이빗 레지스트리

Verdaccio는 `krgeobuk-deployment/verdaccio/k8s/`에서 Kubernetes로 중앙 운영합니다.

#### 패키지 빌드 및 게시

```bash
# shared-lib 디렉토리에서 실행
pnpm build                 # 모든 패키지 빌드
pnpm verdaccio:publish     # Verdaccio 레지스트리에 게시 (패키지 디렉토리에서)
```

#### .npmrc 설정

```bash
# .npmrc (프로젝트 루트) - 환경에 맞는 URL 사용
# dev
@krgeobuk:registry=http://verdaccio.192.168.0.28.nip.io

# prod
@krgeobuk:registry=https://verdaccio.krgeobuk.com
```

### 3. ESM 모듈 설정

#### package.json 설정

```json
{
  "type": "module",
  "engines": {
    "node": ">=18.0.0"
  },
  "scripts": {
    "build": "tsc && tsc-alias",
    "postbuild": "tsc-alias -p tsconfig.json"
  }
}
```

#### tsconfig.json 경로 별칭

```json
{
  "compilerOptions": {
    "paths": {
      "@modules/*": ["src/modules/*"],
      "@common/*": ["src/common/*"],
      "@config/*": ["src/config/*"],
      "@database/*": ["src/database/*"]
    }
  }
}
```

### 4. 공유 라이브러리 사용 패턴

#### NestJS 모듈 통합

```typescript
@Module({
  imports: [
    // 핵심 인프라
    CoreModule.forRoot(),
    DatabaseConfigModule.forRoot({
      type: "mysql",
      host: process.env.DB_HOST,
      port: parseInt(process.env.DB_PORT),
      // ... 기타 설정
    }),

    // 도메인 패키지
    JwtModule.forRoot({
      secret: process.env.JWT_SECRET,
      signOptions: { expiresIn: "24h" },
    }),

    // 사용자 정의 모듈
    PermissionModule,
    RoleModule,
  ],
})
export class AppModule {}
```

---

## 서비스 아키텍처 패턴

### 1. 단일 도메인 서비스 (Single Domain Service)

하나의 엔티티를 중심으로 하는 서비스로, 해당 도메인의 비즈니스 로직과 데이터 접근을 담당합니다.

**적용 대상**: `PermissionService`, `RoleService`, `UserService` 등

#### 기본 구조 템플릿

```typescript
@Injectable()
export class PermissionService {
  private readonly logger = new Logger(PermissionService.name);

  constructor(
    private readonly permissionRepo: PermissionRepository,
    private readonly rolePermissionService: RolePermissionService, // 의존 서비스
    @Inject("PORTAL_SERVICE") private readonly portalClient: ClientProxy // 외부 서비스
  ) {}

  // ==================== PUBLIC METHODS ====================

  // 기본 조회 메서드들
  async findById(permissionId: string): Promise<Entity | null> {}
  async findByIdOrFail(permissionId: string): Promise<Entity> {}
  async findByIds(permissionIds: string[]): Promise<Entity[]> {}
  async findByServiceIds(serviceIds: string[]): Promise<Entity[]> {}
  async findByAnd(filter: Filter): Promise<Entity[]> {}
  async findByOr(filter: Filter): Promise<Entity[]> {}

  // 복합 조회 메서드들
  async searchPermissions(
    query: SearchQueryDto
  ): Promise<PaginatedResult<SearchResult>> {}
  async getPermissionById(permissionId: string): Promise<DetailResult> {}

  // 변경 메서드들
  async createPermission(
    dto: CreateDto,
    transactionManager?: EntityManager
  ): Promise<void> {}
  async updatePermission(
    permissionId: string,
    dto: UpdateDto,
    transactionManager?: EntityManager
  ): Promise<void> {}
  async deletePermission(permissionId: string): Promise<UpdateResult> {}

  // ==================== PRIVATE HELPER METHODS ====================

  private async getServiceById(serviceId: string): Promise<Service> {}
  private buildSearchResults(items: Entity[], metadata: any): SearchResult[] {}
}
```

#### 메서드 순서 표준

1. **PUBLIC METHODS**

   - 기본 조회 메서드 (`findById`, `findByIdOrFail`, `findByServiceIds`, `findByAnd`, `findByOr`)
   - 복합 조회 메서드 (`searchXXX`, `getXXXById`)
   - 변경 메서드 (`createXXX`, `updateXXX`, `deleteXXX`)

2. **PRIVATE HELPER METHODS**
   - 외부 서비스 통신 메서드
   - 데이터 변환 및 빌더 메서드

#### 에러 처리 표준

```typescript
async createPermission(dto: CreatePermissionDto, transactionManager?: EntityManager): Promise<void> {
  try {
    // 1. 비즈니스 로직 검증
    if (dto.action && dto.serviceId) {
      const existingPermission = await this.permissionRepo.findOne({
        where: { action: dto.action, serviceId: dto.serviceId },
      });

      if (existingPermission) {
        this.logger.warn('권한 생성 실패: 서비스 내 중복 액션', {
          action: dto.action,
          serviceId: dto.serviceId,
        });
        throw PermissionException.permissionAlreadyExists();
      }
    }

    // 2. 엔티티 생성 및 저장
    const entity = new PermissionEntity();
    Object.assign(entity, dto);
    await this.permissionRepo.saveEntity(entity, transactionManager);

    // 3. 성공 로깅
    this.logger.log('권한 생성 성공', {
      permissionId: entity.id,
      action: dto.action,
      serviceId: dto.serviceId,
    });
  } catch (error: unknown) {
    // 4. 에러 처리
    if (error instanceof HttpException) {
      throw error; // 이미 처리된 예외는 그대로 전파
    }

    this.logger.error('권한 생성 실패', {
      error: error instanceof Error ? error.message : 'Unknown error',
      action: dto.action,
      serviceId: dto.serviceId,
    });

    throw PermissionException.permissionCreateError();
  }
}
```

**에러 처리 원칙**:

- 도메인별 Exception 클래스 사용 (`PermissionException.permissionCreateError()`)
- HttpException 인스턴스는 그대로 전파
- 상세한 컨텍스트 정보와 함께 로깅
- 도메인명을 포함한 에러 메서드 명명

### 2. 중간 테이블 서비스 (Junction Table Service)

두 도메인 간의 관계를 관리하는 서비스입니다.

**표준 컨벤션**: `UserRoleService` 기준 (최고 성능 및 완성도)

#### 고급 엔티티 설계

```typescript
import { Entity, Index, PrimaryColumn } from "typeorm";

@Entity("user_role")
@Index("IDX_USER_ROLE_USER", ["userId"]) // 개별 FK 인덱스
@Index("IDX_USER_ROLE_ROLE", ["roleId"]) // 개별 FK 인덱스
@Index("IDX_USER_ROLE_UNIQUE", ["userId", "roleId"], { unique: true }) // 복합 유니크 인덱스
export class UserRoleEntity {
  @PrimaryColumn({ type: "uuid" })
  userId!: string;

  @PrimaryColumn({ type: "uuid" })
  roleId!: string;
}
```

**핵심 구조 요소**:

- **복합 기본 키**: 두 관련 엔티티 ID
- **개별 인덱스**: 각 FK에 대한 쿼리 최적화
- **유니크 제약조건**: 중복 관계 방지 및 성능 최적화
- **적절한 테이블 명명**: snake_case 사용

#### 기본 구조

```typescript
@Injectable()
export class UserRoleService {
  private readonly logger = new Logger(UserRoleService.name);

  constructor(private readonly userRoleRepo: UserRoleRepository) {}

  // ==================== 조회 메서드 (ID 목록 반환) ====================

  /**
   * 사용자의 역할 ID 목록 조회
   */
  async getRoleIds(userId: string): Promise<string[]> {
    try {
      return await this.userRoleRepo.findRoleIdsByUserId(userId);
    } catch (error: unknown) {
      this.logger.error("사용자별 역할 ID 조회 실패", {
        error: error instanceof Error ? error.message : "Unknown error",
        userId,
      });
      throw UserRoleException.fetchError();
    }
  }

  /**
   * 역할의 사용자 ID 목록 조회
   */
  async getUserIds(roleId: string): Promise<string[]> {
    try {
      return await this.userRoleRepo.findUserIdsByRoleId(roleId);
    } catch (error: unknown) {
      this.logger.error("역할별 사용자 ID 조회 실패", {
        error: error instanceof Error ? error.message : "Unknown error",
        roleId,
      });
      throw UserRoleException.fetchError();
    }
  }

  /**
   * 사용자-역할 관계 존재 확인
   */
  async exists(userId: string, roleId: string): Promise<boolean> {
    try {
      return await this.userRoleRepo.existsUserRole(userId, roleId);
    } catch (error: unknown) {
      this.logger.error("사용자-역할 관계 존재 확인 실패", {
        error: error instanceof Error ? error.message : "Unknown error",
        userId,
        roleId,
      });
      throw UserRoleException.fetchError();
    }
  }

  // ==================== 변경 메서드 ====================

  // 단일 관계 관리
  async assignUserRole(dto: {
    userId: string;
    roleId: string;
  }): Promise<void> {}
  async revokeUserRole(userId: string, roleId: string): Promise<void> {}

  // 배치 관계 관리
  async assignMultipleRoles(dto: {
    userId: string;
    roleIds: string[];
  }): Promise<UserRoleBatchAssignmentResult> {}
  async revokeMultipleRoles(dto: {
    userId: string;
    roleIds: string[];
  }): Promise<void> {}
  async replaceUserRoles(dto: {
    userId: string;
    roleIds: string[];
  }): Promise<void> {}

  // 성능 최적화 메서드 (필수)
  async hasUsersForRole(roleId: string): Promise<boolean> {}
}
```

#### 배치 처리 결과 반환 표준

```typescript
interface UserRoleBatchAssignmentResult {
  success: boolean;
  assigned: number;      // 신규 할당된 수
  duplicates: number;    // 중복으로 건너뛴 수
  newAssignments: string[];  // 신규 할당된 ID 목록
}
```

#### 고급 배치 처리 구현

```typescript
/**
 * 고급 배치 할당 구현 - 중복 확인 및 최적화
 */
async assignMultipleRoles(dto: {
  userId: string;
  roleIds: string[];
}): Promise<UserRoleBatchAssignmentResult> {
  try {
    // 1. 기존 관계 확인 (배치 최적화)
    const existingRoles = await this.getRoleIds(dto.userId);
    const newRoles = dto.roleIds.filter(id => !existingRoles.includes(id));
    const duplicates = dto.roleIds.filter(id => existingRoles.includes(id));

    if (newRoles.length === 0) {
      this.logger.warn('새로운 역할 할당 없음 - 모든 역할이 이미 존재', {
        userId: dto.userId,
        requestedCount: dto.roleIds.length,
        duplicateCount: duplicates.length,
      });

      return {
        success: true,
        assigned: 0,
        duplicates: duplicates.length,
        newAssignments: [],
      };
    }

    // 2. 새로운 역할만 배치 생성
    const entities = newRoles.map(roleId => {
      const entity = new UserRoleEntity();
      entity.userId = dto.userId;
      entity.roleId = roleId;
      return entity;
    });

    // 3. 배치 저장
    await this.userRoleRepo.save(entities);

    this.logger.log('사용자 다중 역할 할당 성공', {
      userId: dto.userId,
      assignedCount: newRoles.length,
      skippedCount: duplicates.length,
      totalRequested: dto.roleIds.length,
    });

    return {
      success: true,
      assigned: newRoles.length,
      duplicates: duplicates.length,
      newAssignments: newRoles,
    };
  } catch (error: unknown) {
    this.logger.error('사용자 다중 역할 할당 실패', {
      error: error instanceof Error ? error.message : 'Unknown error',
      userId: dto.userId,
      roleCount: dto.roleIds.length,
    });

    throw UserRoleException.assignMultipleError();
  }
}

/**
 * 트랜잭션 기반 완전 교체 구현
 */
async replaceUserRoles(dto: { userId: string; roleIds: string[] }): Promise<void> {
  try {
    await this.userRoleRepo.manager.transaction(async (manager) => {
      // 1. 기존 역할 모두 삭제
      await manager.delete(UserRoleEntity, { userId: dto.userId });

      // 2. 새로운 역할 배치 삽입
      if (dto.roleIds.length > 0) {
        const entities = dto.roleIds.map(roleId => {
          const entity = new UserRoleEntity();
          entity.userId = dto.userId;
          entity.roleId = roleId;
          return entity;
        });

        await manager.save(UserRoleEntity, entities);
      }
    });

    this.logger.log('사용자 역할 완전 교체 성공', {
      userId: dto.userId,
      newRoleCount: dto.roleIds.length,
    });
  } catch (error: unknown) {
    this.logger.error('사용자 역할 완전 교체 실패', {
      error: error instanceof Error ? error.message : 'Unknown error',
      userId: dto.userId,
      roleCount: dto.roleIds.length,
    });

    throw UserRoleException.replaceRolesError();
  }
}
```

#### 배치 조회 최적화 (필수)

```typescript
/**
 * 배치 조회용 Map 반환 - O(1) 접근 최적화
 */
async getRoleIdsBatch(userIds: string[]): Promise<Record<string, string[]>> {
  try {
    return await this.userRoleRepo.findRoleIdsByUserIds(userIds);
  } catch (error: unknown) {
    this.logger.error('사용자별 역할 ID 배치 조회 실패', {
      error: error instanceof Error ? error.message : 'Unknown error',
      userCount: userIds.length,
    });
    throw UserRoleException.fetchError();
  }
}

/**
 * 메모리 효율적인 카운트 쿼리 (선택적 최적화)
 */
async getUserCountsBatch(roleIds: string[]): Promise<Record<string, number>> {
  try {
    const userIdsMap = await this.userRoleRepo.findUserIdsByRoleIds(roleIds);
    const userCounts: Record<string, number> = {};

    roleIds.forEach(roleId => {
      const userIds = userIdsMap[roleId] || [];
      userCounts[roleId] = userIds.length;
    });

    return userCounts;
  } catch (error: unknown) {
    this.logger.error('역할별 사용자 수 조회 실패', {
      error: error instanceof Error ? error.message : 'Unknown error',
      roleCount: roleIds.length,
    });
    throw UserRoleException.fetchError();
  }
}
```

#### 성능 최적화 패턴 (필수 구현)

```typescript
// 🔥 최우선 최적화: 존재 확인 최적화
async hasUsersForRole(roleId: string): Promise<boolean> {
  try {
    const userIds = await this.userRoleRepo.findUserIdsByRoleId(roleId);
    return userIds.length > 0;
  } catch (error: unknown) {
    this.logger.error('역할의 사용자 존재 확인 실패', {
      error: error instanceof Error ? error.message : 'Unknown error',
      roleId,
    });
    throw UserRoleException.fetchError();
  }
}
```

---

## 컨트롤러 패턴

### 1. REST API 컨트롤러 표준

#### 단일 도메인 컨트롤러

```typescript
@Controller("permissions")
export class PermissionController {
  @Get() // 목록/검색
  async searchPermissions(
    @Query() query: PermissionSearchQueryDto
  ): Promise<PermissionPaginatedSearchResultDto> {}

  @Post() // 생성
  async createPermission(@Body() dto: CreatePermissionDto): Promise<void> {}

  @Get(":permissionId") // 상세 조회
  async getPermissionById(
    @Param() params: PermissionIdParamsDto
  ): Promise<PermissionDetailDto> {}

  @Patch(":permissionId") // 수정
  async updatePermission(
    @Param() params: PermissionIdParamsDto,
    @Body() dto: UpdatePermissionDto
  ): Promise<void> {}

  @Delete(":permissionId") // 삭제
  async deletePermission(
    @Param() params: PermissionIdParamsDto
  ): Promise<void> {}
}
```

#### 중간 테이블 도메인 컨트롤러

```typescript
@Controller()
export class RolePermissionController {
  // 양방향 관계 조회
  @Get("roles/:roleId/permissions")
  async getPermissionsByRole(
    @Param() params: RoleIdParamsDto
  ): Promise<string[]> {}

  @Get("permissions/:permissionId/roles")
  async getRolesByPermission(
    @Param() params: PermissionIdParamsDto
  ): Promise<string[]> {}

  // 관계 존재 확인
  @Get("roles/:roleId/permissions/:permissionId/exists")
  async checkRolePermissionExists(
    @Param() params: RolePermissionParamsDto
  ): Promise<boolean> {}

  // 개별 관계 관리
  @Post("roles/:roleId/permissions/:permissionId")
  async assignRolePermission(
    @Param() params: RolePermissionParamsDto
  ): Promise<void> {}

  @Delete("roles/:roleId/permissions/:permissionId")
  async revokeRolePermission(
    @Param() params: RolePermissionParamsDto
  ): Promise<void> {}

  // 배치 처리
  @Post("roles/:roleId/permissions/batch")
  async assignMultiplePermissions(
    @Param() params: RoleIdParamsDto,
    @Body() dto: PermissionIdsDto
  ): Promise<void> {}

  // 완전 교체
  @Put("roles/:roleId/permissions")
  async replaceRolePermissions(
    @Param() params: RoleIdParamsDto,
    @Body() dto: PermissionIdsDto
  ): Promise<void> {}
}
```

### 2. TCP 컨트롤러 표준

#### 단일 도메인 TCP 컨트롤러

```typescript
@Controller()
export class PermissionTcpController {
  private readonly logger = new Logger(PermissionTcpController.name);

  constructor(private readonly permissionService: PermissionService) {}

  // ==================== 조회 패턴 ====================

  @MessagePattern("permission.findById")
  async findById(
    @Payload() data: { permissionId: string }
  ): Promise<PermissionEntity | null> {
    this.logger.debug("권한 ID 조회", { permissionId: data.permissionId });
    return await this.permissionService.findById(data.permissionId);
  }

  @MessagePattern("permission.findByServiceIds")
  async findByServiceIds(
    @Payload() data: { serviceIds: string[] }
  ): Promise<PermissionEntity[]> {
    this.logger.debug("서비스별 권한 조회", {
      serviceCount: data.serviceIds.length,
    });
    return await this.permissionService.findByServiceIds(data.serviceIds);
  }

  // ==================== 검색 패턴 ====================

  @MessagePattern("permission.search")
  async search(
    @Payload() query: PermissionSearchQuery
  ): Promise<PaginatedResult<PermissionSearchResult>> {
    this.logger.debug("권한 검색", { query });
    return await this.permissionService.searchPermissions(query);
  }

  // ==================== 변경 패턴 ====================

  @MessagePattern("permission.create")
  async create(
    @Payload() data: CreatePermissionTcpDto
  ): Promise<TcpOperationResponse> {
    this.logger.log("권한 생성 요청", {
      action: data.action,
      serviceId: data.serviceId,
    });

    try {
      await this.permissionService.createPermission(data);
      return { success: true, message: "권한 생성 성공" };
    } catch (error: unknown) {
      return {
        success: false,
        message: error instanceof Error ? error.message : "권한 생성 실패",
      };
    }
  }

  // ==================== 유틸리티 패턴 ====================

  @MessagePattern("permission.exists")
  async exists(@Payload() data: { permissionId: string }): Promise<boolean> {
    this.logger.debug("권한 존재 확인", { permissionId: data.permissionId });
    const permission = await this.permissionService.findById(data.permissionId);
    return permission !== null;
  }
}
```

#### 중간 테이블 TCP 컨트롤러 (고급 패턴)

```typescript
@Controller()
export class UserRoleTcpController {
  private readonly logger = new Logger(UserRoleTcpController.name);

  constructor(private readonly userRoleService: UserRoleService) {}

  // ==================== 양방향 관계 조회 ====================

  @MessagePattern("user-role.findRolesByUser")
  async findRoleIdsByUserId(
    @Payload() data: { userId: string }
  ): Promise<string[]> {
    this.logger.debug("사용자별 역할 조회", { userId: data.userId });
    return await this.userRoleService.getRoleIds(data.userId);
  }

  @MessagePattern("user-role.findUsersByRole")
  async findUserIdsByRoleId(
    @Payload() data: { roleId: string }
  ): Promise<string[]> {
    this.logger.debug("역할별 사용자 조회", { roleId: data.roleId });
    return await this.userRoleService.getUserIds(data.roleId);
  }

  // ==================== 배치 조회 (성능 최적화) ====================

  @MessagePattern("user-role.findRolesByUsers")
  async findRoleIdsByUserIds(
    @Payload() data: { userIds: string[] }
  ): Promise<Record<string, string[]>> {
    this.logger.debug("사용자별 역할 배치 조회", {
      userCount: data.userIds.length,
    });
    return await this.userRoleService.getRoleIdsBatch(data.userIds);
  }

  // ==================== 존재 확인 ====================

  @MessagePattern("user-role.exists")
  async existsUserRole(
    @Payload() data: { userId: string; roleId: string }
  ): Promise<boolean> {
    this.logger.debug("사용자-역할 관계 존재 확인", {
      userId: data.userId,
      roleId: data.roleId,
    });
    return await this.userRoleService.exists(data.userId, data.roleId);
  }

  // ==================== 배치 처리 (할당 → 해제 → 교체 순서) ====================

  @MessagePattern("user-role.assignMultiple")
  async assignMultipleRoles(
    @Payload() data: { userId: string; roleIds: string[] }
  ): Promise<TcpOperationResponse> {
    this.logger.log("다중 역할 할당", {
      userId: data.userId,
      roleCount: data.roleIds.length,
    });

    try {
      const result = await this.userRoleService.assignMultipleRoles(data);
      return {
        success: result.success,
        message: `${result.assigned}개 역할 할당 완료`,
        data: result,
      };
    } catch (error: unknown) {
      return {
        success: false,
        message: error instanceof Error ? error.message : "역할 할당 실패",
      };
    }
  }

  @MessagePattern("user-role.revokeMultiple")
  async revokeMultipleRoles(
    @Payload() data: { userId: string; roleIds: string[] }
  ): Promise<TcpOperationResponse> {
    this.logger.log("다중 역할 해제", {
      userId: data.userId,
      roleCount: data.roleIds.length,
    });

    try {
      await this.userRoleService.revokeMultipleRoles(data);
      return { success: true, message: "역할 해제 완료" };
    } catch (error: unknown) {
      return {
        success: false,
        message: error instanceof Error ? error.message : "역할 해제 실패",
      };
    }
  }

  @MessagePattern("user-role.replace")
  async replaceUserRoles(
    @Payload() data: { userId: string; roleIds: string[] }
  ): Promise<TcpOperationResponse> {
    this.logger.log("사용자 역할 완전 교체", {
      userId: data.userId,
      newRoleCount: data.roleIds.length,
    });

    try {
      await this.userRoleService.replaceUserRoles(data);
      return { success: true, message: "역할 교체 완료" };
    } catch (error: unknown) {
      return {
        success: false,
        message: error instanceof Error ? error.message : "역할 교체 실패",
      };
    }
  }
}
```

#### TCP 응답 표준 인터페이스

```typescript
interface TcpOperationResponse {
  success: boolean;
  message: string;
  data?: any;
}

interface CreatePermissionTcpDto {
  action: string;
  serviceId: string;
  description?: string;
}
```

#### TCP 메서드 노출 기준

**포함 대상**:

- **양방향 관계 조회**: 성능상 이점이 있는 직접 통신
- **존재 확인**: 빠른 검증이 필요한 경우
- **배치 처리**: N+1 쿼리 방지를 위한 최적화

**제외 대상**:

- **단일 할당/해제**: HTTP API로 충분
- **완전 삭제**: 안전상 이유로 HTTP만 허용
- **통계 정보**: 정확성보다는 속도 우선이 아닌 경우

#### TCP 로깅 성능 최적화

```typescript
// ✅ 로그 레벨 최적화 예시
@MessagePattern('user-role.findRolesByUser')
async findRoleIdsByUserId(@Payload() data: { userId: string }): Promise<string[]> {
  // 고빈도 API는 DEBUG 레벨로 최소화
  this.logger.debug('사용자별 역할 조회', { userId: data.userId });
  return await this.userRoleService.getRoleIds(data.userId);
}

@MessagePattern('user-role.assignMultiple')
async assignMultipleRoles(@Payload() data: any): Promise<TcpOperationResponse> {
  // 중요한 변경 작업은 LOG 레벨
  this.logger.log('다중 역할 할당', {
    userId: data.userId,
    roleCount: data.roleIds.length
  });
  // 구현...
}
```

---

## Repository 패턴

### 1. 쿼리 최적화 표준

```typescript
// ✅ 올바른 패턴 - 명시적 컬럼 SELECT
async searchEntities(query: SearchQuery): Promise<PaginatedResult<Partial<Entity>>> {
  const qb = this.createQueryBuilder('entity')
    .select([
      'entity.id',
      'entity.name',
      'entity.description',
      'entity.serviceId',
      // 필요한 컬럼만
    ]);

  // 인덱스 최적화된 조건 순서
  if (query.serviceId) {
    qb.andWhere('entity.serviceId = :serviceId', { serviceId: query.serviceId });
  }

  if (query.name) {
    qb.andWhere('entity.name LIKE :name', { name: `%${query.name}%` });
  }

  // COUNT와 데이터 쿼리 분리
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
```

### 2. 중간 테이블 Repository 최적화

```typescript
@Injectable()
export class UserRoleRepository extends BaseRepository<UserRoleEntity> {
  /**
   * ID만 반환하는 최적화된 단일 조회
   */
  async findRoleIdsByUserId(userId: string): Promise<string[]> {
    const result = await this.createQueryBuilder("ur")
      .select("ur.roleId")
      .where("ur.userId = :userId", { userId })
      .getRawMany();

    return result.map((row) => row.ur_roleId);
  }

  async findUserIdsByRoleId(roleId: string): Promise<string[]> {
    const result = await this.createQueryBuilder("ur")
      .select("ur.userId")
      .where("ur.roleId = :roleId", { roleId })
      .getRawMany();

    return result.map((row) => row.ur_userId);
  }

  /**
   * 배치 처리용 Record<string, string[]> 반환 - O(1) 접근 최적화
   */
  async findRoleIdsByUserIds(
    userIds: string[]
  ): Promise<Record<string, string[]>> {
    const result = await this.createQueryBuilder("ur")
      .select(["ur.userId", "ur.roleId"])
      .where("ur.userId IN (:...userIds)", { userIds })
      .getRawMany();

    const userRoleMap: Record<string, string[]> = {};

    // 초기화 (모든 사용자에 대해 빈 배열 보장)
    userIds.forEach((userId) => {
      userRoleMap[userId] = [];
    });

    result.forEach((row) => {
      const userId = row.ur_userId;
      const roleId = row.ur_roleId;
      userRoleMap[userId].push(roleId);
    });

    return userRoleMap;
  }

  async findUserIdsByRoleIds(
    roleIds: string[]
  ): Promise<Record<string, string[]>> {
    const result = await this.createQueryBuilder("ur")
      .select(["ur.roleId", "ur.userId"])
      .where("ur.roleId IN (:...roleIds)", { roleIds })
      .getRawMany();

    const roleUserMap: Record<string, string[]> = {};

    // 초기화 (모든 역할에 대해 빈 배열 보장)
    roleIds.forEach((roleId) => {
      roleUserMap[roleId] = [];
    });

    result.forEach((row) => {
      const roleId = row.ur_roleId;
      const userId = row.ur_userId;
      roleUserMap[roleId].push(userId);
    });

    return roleUserMap;
  }

  /**
   * SELECT 1 + LIMIT 패턴으로 존재 확인 최적화
   */
  async existsUserRole(userId: string, roleId: string): Promise<boolean> {
    const result = await this.createQueryBuilder("ur")
      .select("1")
      .where("ur.userId = :userId AND ur.roleId = :roleId", { userId, roleId })
      .limit(1)
      .getRawOne();

    return result !== undefined;
  }

  /**
   * 고성능 COUNT 기반 존재 확인 (대량 데이터용)
   */
  async existsUserRoleByCount(
    userId: string,
    roleId: string
  ): Promise<boolean> {
    const count = await this.createQueryBuilder("ur")
      .where("ur.userId = :userId AND ur.roleId = :roleId", { userId, roleId })
      .getCount();

    return count > 0;
  }

  /**
   * 배치 존재 확인 - 여러 관계의 존재 여부를 한 번에 확인
   */
  async existsUserRoles(
    relations: Array<{ userId: string; roleId: string }>
  ): Promise<Record<string, boolean>> {
    if (relations.length === 0) return {};

    // 모든 관계를 OR 조건으로 한 번에 조회
    let query = this.createQueryBuilder("ur").select([
      "ur.userId",
      "ur.roleId",
    ]);

    const orConditions = relations.map(
      (_, index) =>
        `(ur.userId = :userId${index} AND ur.roleId = :roleId${index})`
    );

    query = query.where(orConditions.join(" OR "));

    // 파라미터 바인딩
    const parameters: Record<string, string> = {};
    relations.forEach((relation, index) => {
      parameters[`userId${index}`] = relation.userId;
      parameters[`roleId${index}`] = relation.roleId;
    });

    query = query.setParameters(parameters);

    const existingRelations = await query.getRawMany();
    const existsMap: Record<string, boolean> = {};

    // 요청된 모든 관계를 false로 초기화
    relations.forEach((relation) => {
      const key = `${relation.userId}:${relation.roleId}`;
      existsMap[key] = false;
    });

    // 존재하는 관계만 true로 설정
    existingRelations.forEach((row) => {
      const key = `${row.ur_userId}:${row.ur_roleId}`;
      existsMap[key] = true;
    });

    return existsMap;
  }

  /**
   * 관계 삭제 전 의존성 확인을 위한 최적화된 카운트
   */
  async countRelationsByEntity(
    entityType: "user" | "role",
    entityId: string
  ): Promise<number> {
    const qb = this.createQueryBuilder("ur");

    if (entityType === "user") {
      qb.where("ur.userId = :entityId", { entityId });
    } else {
      qb.where("ur.roleId = :entityId", { entityId });
    }

    return await qb.getCount();
  }
}
```

### Repository 최적화 체크리스트

- [ ] 필요한 컬럼만 SELECT
- [ ] 인덱스 최적화된 WHERE 조건 순서
- [ ] `Promise.all()`을 통한 COUNT와 데이터 쿼리 병렬 처리
- [ ] 명시적 타입 매핑 (`Partial<Entity>[]`)
- [ ] JOIN 조건 정확성 검증

---

## DTO 및 유효성 검사

### 1. 표준 DTO 구조

#### 검색 쿼리 DTO

```typescript
export class EntitySearchQueryDto {
  @IsOptional()
  @IsString()
  name?: string;

  @IsOptional()
  @IsUUID()
  serviceId?: string;

  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  @Min(1)
  @Max(100)
  limit?: number = 20;
}
```

#### 배치 작업 DTO

```typescript
export class RoleIdsDto {
  @IsArray()
  @IsUUID("all", { each: true })
  @ArrayNotEmpty()
  roleIds!: string[];
}
```

### 2. 배치 작업 결과 인터페이스

```typescript
interface JunctionTableOperationResult {
  success: boolean;
  assigned: number;       // 신규 할당된 수
  duplicates: number;     // 중복으로 건너뛴 수
  newAssignments: string[];  // 신규 할당된 ID 목록
}
```

---

## 에러 처리 및 예외 관리

### 1. 도메인별 예외 클래스

**예외 명명 패턴**:

- 형식: `{Domain}Exception.{domain}{Action}Error()`
- 예시: `PermissionException.permissionCreateError()`, `UserRoleException.assignMultipleError()`

### 2. 에러 처리 표준 패턴

```typescript
async createEntity(attrs: CreateAttrs): Promise<void> {
  try {
    // 1. 사전 검증 (비즈니스 규칙)
    if (attrs.name && attrs.serviceId) {
      const existing = await this.repo.findOne({
        where: { name: attrs.name, serviceId: attrs.serviceId }
      });

      if (existing) {
        this.logger.warn('엔티티 생성 실패: 중복된 이름', {
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
    this.logger.log('엔티티 생성 성공', {
      entityId: entity.id,
      name: attrs.name,
      serviceId: attrs.serviceId,
    });
  } catch (error: unknown) {
    // 5. 에러 처리 및 로깅
    if (error instanceof HttpException) {
      throw error; // 처리된 예외는 재전파
    }

    this.logger.error('엔티티 생성 실패', {
      error: error instanceof Error ? error.message : 'Unknown error',
      name: attrs.name,
      serviceId: attrs.serviceId,
      stack: error instanceof Error ? error.stack : undefined,
    });

    throw EntityException.entityCreateError();
  }
}
```

### 에러 처리 체크리스트

- [ ] 데이터베이스 오류 방지를 위한 사전 검증
- [ ] HttpException 인스턴스 확인 및 재전파
- [ ] 구조화된 에러 로깅 (error, context, stack)
- [ ] 사용자 친화적 에러 메시지 (EntityException 사용)
- [ ] 로깅에서 민감 정보 제외

---

## 로깅 표준

### 1. 로그 레벨 사용 기준

```typescript
// ERROR: 시스템 오류, 예외 상황
this.logger.error("엔티티 생성 실패", {
  error: error instanceof Error ? error.message : "Unknown error",
  entityId: id,
  operation: "create",
});

// WARN: 비정상이지만 복구 가능한 상황
this.logger.warn("외부 서비스 응답 없음, 대체 데이터 사용", {
  service: "auth-service",
  fallbackUsed: true,
  entityId: id,
});

// LOG/INFO: 중요한 비즈니스 이벤트
this.logger.log("엔티티 생성 성공", {
  entityId: result.id,
  entityType: "Role",
  serviceId: result.serviceId,
});

// DEBUG: 개발용, 고빈도 API
this.logger.debug("TCP 요청 수신", {
  operation: "findById",
  entityId: id,
  timestamp: new Date().toISOString(),
});
```

### 2. 로그 메시지 구조 표준

- **형식**: `"액션 + 결과 + 컨텍스트"`
- **메타데이터**: entityId (필수), operation, timestamp (자동), serviceId, userId, error
- **제외 대상**: 패스워드, 토큰, 개인정보

---

## API 응답 형식

### 1. 성공 응답 형식 (SerializerInterceptor)

모든 성공 API 응답은 다음 구조를 따릅니다:

```typescript
{
  code: string,           // 응답 코드 (기본: CoreCode.REQUEST_SUCCESS)
  status_code: number,    // HTTP 상태 코드 (기본: 200)
  message: string,        // 응답 메시지 (기본: CoreMessage.REQUEST_SUCCESS)
  isLogin: boolean,       // 사용자 로그인 상태
  data: object | null     // 실제 응답 데이터 (snake_case 변환됨)
}
```

### 2. 에러 응답 형식 (HttpExceptionFilter)

모든 HTTP 예외는 다음 구조로 응답합니다:

```typescript
{
  statusCode: number,     // HTTP 상태 코드
  code: string,          // 에러 코드 (기본: CoreCode.SERVER_ERROR)
  message: string        // 에러 메시지 (배열 메시지는 쉼표로 결합)
}
```

### 3. 사용 예시

#### 성공 응답 커스터마이징

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

#### 커스텀 예외 처리

```typescript
throw new BadRequestException({
  message: "잘못된 요청입니다",
  code: "AUTH_001",
});
```

---

## 마이크로서비스 통신

### 1. TCP 클라이언트 주입 패턴

#### ClientProxy 설정

```typescript
// app.module.ts
@Module({
  imports: [
    ClientsModule.register([
      {
        name: "PORTAL_SERVICE",
        transport: Transport.TCP,
        options: {
          host: process.env.PORTAL_SERVICE_HOST || "localhost",
          port: parseInt(process.env.PORTAL_SERVICE_PORT) || 8200,
        },
      },
      {
        name: "AUTH_SERVICE",
        transport: Transport.TCP,
        options: {
          host: process.env.AUTH_SERVICE_HOST || "localhost",
          port: parseInt(process.env.AUTH_SERVICE_PORT) || 8000,
        },
      },
    ]),
  ],
  // ...
})
export class AppModule {}
```

#### 서비스에서 TCP 클라이언트 사용

```typescript
@Injectable()
export class PermissionService {
  private readonly logger = new Logger(PermissionService.name);

  constructor(
    private readonly permissionRepo: PermissionRepository,
    @Inject("PORTAL_SERVICE") private readonly portalClient: ClientProxy,
    @Inject("AUTH_SERVICE") private readonly authClient: ClientProxy
  ) {}

  /**
   * 외부 서비스 조회 (폴백 전략 포함)
   */
  private async getServiceById(serviceId: string): Promise<Service> {
    try {
      const response = await firstValueFrom(
        this.portalClient.send("service.findById", { serviceId }).pipe(
          timeout(5000), // 5초 타임아웃
          retry(2) // 2회 재시도
        )
      );

      if (!response) {
        throw new Error("서비스 정보가 없습니다");
      }

      return response;
    } catch (error: unknown) {
      this.logger.warn(
        "포털 서비스에서 서비스 정보 조회 실패, 대체 데이터 사용",
        {
          error: error instanceof Error ? error.message : "Unknown error",
          serviceId,
        }
      );

      // 폴백 전략: 기본 서비스 정보 반환
      return this.buildFallbackServiceData(serviceId);
    }
  }

  /**
   * 사용자 정보 조회 (캐시 포함)
   */
  private async getUserInfo(userId: string): Promise<UserInfo> {
    try {
      const response = await firstValueFrom(
        this.authClient.send("user.findById", { userId }).pipe(
          timeout(3000),
          catchError((err) => {
            this.logger.warn("사용자 정보 조회 실패", {
              error: err.message,
              userId,
            });
            // 기본 사용자 정보 반환
            return of({ id: userId, name: "Unknown User" });
          })
        )
      );

      return response;
    } catch (error: unknown) {
      this.logger.error("사용자 정보 조회 중 예외 발생", {
        error: error instanceof Error ? error.message : "Unknown error",
        userId,
      });

      // 최소 사용자 정보 반환
      return { id: userId, name: "Unknown User" };
    }
  }

  /**
   * 폴백 데이터 생성
   */
  private buildFallbackServiceData(serviceId: string): Service {
    return {
      id: serviceId,
      name: `Service ${serviceId}`,
      description: "서비스 정보를 가져올 수 없음",
      status: "unknown",
      createdAt: new Date(),
    };
  }
}
```

### 2. 배치 통신 최적화

#### 배치 데이터 조회 패턴

```typescript
/**
 * 여러 서비스 정보를 한 번에 조회
 */
private async getServicesByIds(serviceIds: string[]): Promise<Service[]> {
  if (serviceIds.length === 0) return [];

  try {
    const response = await firstValueFrom(
      this.portalClient.send('service.findByIds', { serviceIds }).pipe(
        timeout(10000), // 배치 요청은 더 긴 타임아웃
        retry(1)
      )
    );

    return response || [];
  } catch (error: unknown) {
    this.logger.warn('서비스 배치 조회 실패, 개별 조회로 대체', {
      error: error instanceof Error ? error.message : 'Unknown error',
      serviceIds,
      serviceCount: serviceIds.length,
    });

    // 개별 조회로 폴백 (병렬 처리)
    const promises = serviceIds.map(id => this.getServiceById(id));
    return await Promise.all(promises);
  }
}

/**
 * 권한 검색 결과에 서비스 정보 포함
 */
async searchPermissions(query: SearchQueryDto): Promise<PaginatedResult<SearchResult>> {
  // 1. 권한 검색
  const result = await this.permissionRepo.searchWithOptimization(query);

  // 2. 서비스 ID 수집
  const serviceIds = [...new Set(result.items.map(item => item.serviceId))];

  // 3. 서비스 정보 배치 조회
  const services = await this.getServicesByIds(serviceIds);
  const serviceMap = new Map(services.map(s => [s.id, s]));

  // 4. 결과 조합
  const enrichedItems = result.items.map(item => ({
    ...item,
    service: serviceMap.get(item.serviceId) || this.buildFallbackServiceData(item.serviceId),
  }));

  return {
    items: enrichedItems,
    pageInfo: result.pageInfo,
  };
}
```

### 3. 에러 처리 및 복구 전략

#### 서킷 브레이커 패턴

```typescript
interface ServiceHealthCheck {
  serviceName: string;
  isHealthy: boolean;
  lastCheck: Date;
  consecutiveFailures: number;
}

@Injectable()
export class MicroserviceHealthManager {
  private readonly logger = new Logger(MicroserviceHealthManager.name);
  private readonly healthChecks = new Map<string, ServiceHealthCheck>();
  private readonly maxConsecutiveFailures = 3;
  private readonly checkInterval = 30000; // 30초

  /**
   * 서비스 상태 확인
   */
  async checkServiceHealth(
    serviceName: string,
    clientProxy: ClientProxy
  ): Promise<boolean> {
    try {
      const response = await firstValueFrom(
        clientProxy.send("health.check", {}).pipe(
          timeout(3000),
          catchError(() => of(null))
        )
      );

      this.updateHealthStatus(serviceName, response !== null);
      return response !== null;
    } catch {
      this.updateHealthStatus(serviceName, false);
      return false;
    }
  }

  /**
   * 서비스 상태 업데이트
   */
  private updateHealthStatus(serviceName: string, isHealthy: boolean): void {
    const current = this.healthChecks.get(serviceName) || {
      serviceName,
      isHealthy: true,
      lastCheck: new Date(),
      consecutiveFailures: 0,
    };

    if (isHealthy) {
      current.consecutiveFailures = 0;
      current.isHealthy = true;
    } else {
      current.consecutiveFailures++;
      current.isHealthy =
        current.consecutiveFailures < this.maxConsecutiveFailures;
    }

    current.lastCheck = new Date();
    this.healthChecks.set(serviceName, current);

    if (!current.isHealthy) {
      this.logger.warn(`서비스 ${serviceName} 비정상 상태 감지`, {
        consecutiveFailures: current.consecutiveFailures,
        maxFailures: this.maxConsecutiveFailures,
      });
    }
  }

  /**
   * 서비스 사용 가능 여부 확인
   */
  isServiceAvailable(serviceName: string): boolean {
    const health = this.healthChecks.get(serviceName);
    return health?.isHealthy ?? true; // 초기에는 사용 가능으로 간주
  }
}
```

#### 조건부 서비스 호출

```typescript
/**
 * 서비스 상태를 고려한 안전한 호출
 */
private async safeServiceCall<T>(
  serviceName: string,
  clientProxy: ClientProxy,
  message: string,
  data: any,
  fallbackFn?: () => T
): Promise<T | null> {
  // 서비스 상태 확인
  if (!this.healthManager.isServiceAvailable(serviceName)) {
    this.logger.warn(`서비스 ${serviceName}이 비정상 상태, 호출 건너뜀`, {
      message,
      data,
    });

    return fallbackFn ? fallbackFn() : null;
  }

  try {
    return await firstValueFrom(
      clientProxy.send(message, data).pipe(
        timeout(5000),
        retry(1)
      )
    );
  } catch (error: unknown) {
    this.logger.error(`서비스 ${serviceName} 호출 실패`, {
      error: error instanceof Error ? error.message : 'Unknown error',
      message,
      data,
    });

    return fallbackFn ? fallbackFn() : null;
  }
}
```

### 4. 통신 성능 최적화

#### 연결 풀링 및 재사용

```typescript
// main.ts에서 마이크로서비스 설정
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // TCP 마이크로서비스 설정 (HTTP 포트와 별도 운영)
  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.TCP,
    options: {
      host: '0.0.0.0',
      port: parseInt(process.env.TCP_PORT) || 8110,  // HTTP(8100)와 다른 포트
    },
  });

  await app.startAllMicroservices();
  await app.listen(parseInt(process.env.PORT) || 8100);
}
```

#### 비동기 패턴 및 병렬 처리

```typescript
/**
 * 병렬 서비스 호출로 성능 최적화
 */
async getPermissionDetail(permissionId: string): Promise<PermissionDetail> {
  const permission = await this.findByIdOrFail(permissionId);

  // 병렬 처리로 성능 향상
  const [service, relatedRoles, userCount] = await Promise.allSettled([
    this.getServiceById(permission.serviceId),
    this.rolePermissionService.getRoleIds(permissionId),
    this.getPermissionUserCount(permissionId),
  ]);

  return {
    ...permission,
    service: service.status === 'fulfilled' ? service.value : this.buildFallbackServiceData(permission.serviceId),
    relatedRoles: relatedRoles.status === 'fulfilled' ? relatedRoles.value : [],
    userCount: userCount.status === 'fulfilled' ? userCount.value : 0,
  };
}
```

### 5. 통신 모니터링 및 로깅

#### 통신 지표 수집

```typescript
@Injectable()
export class MicroserviceMetrics {
  private readonly logger = new Logger(MicroserviceMetrics.name);
  private readonly callCounts = new Map<string, number>();
  private readonly errorCounts = new Map<string, number>();

  recordCall(serviceName: string, method: string): void {
    const key = `${serviceName}.${method}`;
    this.callCounts.set(key, (this.callCounts.get(key) || 0) + 1);
  }

  recordError(serviceName: string, method: string): void {
    const key = `${serviceName}.${method}`;
    this.errorCounts.set(key, (this.errorCounts.get(key) || 0) + 1);
  }

  getMetrics(): Record<string, any> {
    return {
      calls: Object.fromEntries(this.callCounts),
      errors: Object.fromEntries(this.errorCounts),
      timestamp: new Date().toISOString(),
    };
  }
}
```

---

## 성능 최적화

### 1. Repository 성능 최적화

```typescript
// ✅ 최적화된 패턴
async searchWithOptimization(query: SearchQuery): Promise<PaginatedResult> {
  // 1. 필요한 컬럼만 SELECT
  const qb = this.createQueryBuilder('entity')
    .select(['entity.id', 'entity.name', 'entity.serviceId']);

  // 2. 인덱스 최적화된 WHERE 조건 순서
  if (query.serviceId) { // 인덱싱된 컬럼 우선
    qb.andWhere('entity.serviceId = :serviceId', { serviceId: query.serviceId });
  }

  if (query.name) { // LIKE 조건 후순위
    qb.andWhere('entity.name LIKE :name', { name: `%${query.name}%` });
  }

  // 3. COUNT와 데이터 쿼리 병렬 처리
  const [rows, total] = await Promise.all([
    qb.offset(skip).limit(limit).getRawMany(),
    qb.getCount()
  ]);

  return this.buildPaginatedResult(rows, total, query);
}
```

### 2. 로깅 성능 최적화

```typescript
// ✅ 로그 레벨 최적화
class OptimizedService {
  async highFrequencyOperation(id: string): Promise<Entity> {
    // 고빈도 API는 DEBUG 레벨로 최소화
    this.logger.debug("고빈도 작업 수행", { entityId: id });
    return await this.repo.findById(id);
  }

  async criticalOperation(data: CreateData): Promise<void> {
    // 중요한 작업만 LOG 레벨
    this.logger.log("중요 작업 시작", {
      operation: "create",
      entityType: data.type,
    });

    try {
      await this.performCriticalWork(data);
      this.logger.log("중요 작업 완료", { entityId: result.id });
    } catch (error: unknown) {
      this.logger.error("중요 작업 실패", {
        error: error instanceof Error ? error.message : "Unknown error",
        operation: "create",
      });
      throw error;
    }
  }
}
```

### 3. Redis 캐싱 패턴

고빈도 조회(권한 검증 등)에 Redis 캐시를 적용합니다.

#### 캐시 조회 + DB 폴백 패턴

```typescript
private async getUserPermissionsWithCache(userId: string): Promise<Permission[]> {
  const cacheKey = `user:permissions:${userId}`;

  try {
    // 1. Redis에서 조회 시도
    const cached = await this.redisService.get(cacheKey);
    if (cached) {
      this.logger.debug('Cache hit', { userId });
      return JSON.parse(cached);
    }

    // 2. 캐시 미스: DB 조회
    const permissions = await this.getUserPermissions(userId);

    // 3. Redis에 5분간 캐시 저장
    await this.redisService.setex(cacheKey, 300, JSON.stringify(permissions));

    return permissions;
  } catch (cacheError: unknown) {
    // 캐시 오류 시 DB 직접 조회로 폴백
    this.logger.warn('Cache error, fallback to DB', {
      error: cacheError instanceof Error ? cacheError.message : 'Unknown error',
      userId,
    });
    return this.getUserPermissions(userId);
  }
}
```

#### 캐시 무효화 패턴

역할·권한 변경 시 관련 사용자 캐시를 즉시 무효화합니다.

```typescript
// 단일 사용자 캐시 무효화
async invalidateUserPermissions(userId: string): Promise<void> {
  const cacheKey = `user:permissions:${userId}`;
  try {
    await this.redisService.del(cacheKey);
    this.logger.debug('User permissions cache invalidated', { userId });
  } catch (error: unknown) {
    this.logger.warn('Failed to invalidate cache', {
      error: error instanceof Error ? error.message : 'Unknown error',
      userId,
    });
  }
}

// 역할 변경 시 해당 역할을 가진 모든 사용자 캐시 무효화
async invalidateUsersInRole(roleId: string): Promise<void> {
  const userIds = await this.userRoleService.getUserIds(roleId);
  await Promise.all(userIds.map(userId => this.invalidateUserPermissions(userId)));
  this.logger.log('Role users cache invalidated', { roleId, affectedUsers: userIds.length });
}
```

#### 캐시 무효화 시점

| 이벤트 | 무효화 대상 |
|---|---|
| 사용자 역할 변경 (assign/revoke/replace) | 해당 사용자의 권한 캐시 |
| 역할-권한 변경 (assign/revoke/replace) | 해당 역할을 가진 모든 사용자의 권한 캐시 |

### 4. 중간 테이블 성능 최적화 우선순위

1. **필수 구현**: `hasUsersForRole()` - 삭제 검증용 존재 확인 최적화
2. **필수 구현**: 역할/권한 변경 시 `invalidateUserPermissions()` 호출
3. **권장 구현**: `getUserCountsBatch()` - 메모리 효율적인 카운트 쿼리
4. **선택 구현**: 트랜잭션 기반 배치 처리
5. **성능 목표**: 단일 권한 확인 50ms 이내, TCP 응답 30ms 이내, 캐시 히트율 80% 이상

---

## 코드 품질 및 표준

### 1. TypeScript 코딩 표준

#### 타입 안전성 규칙

```typescript
// ✅ 올바른 패턴 - 명시적 타입 지정
async function getUserById(id: string): Promise<UserEntity | null> {
  try {
    return await this.userRepo.findOneById(id);
  } catch (error: unknown) {
    this.logger.error("사용자 조회 실패", {
      error: error instanceof Error ? error.message : "Unknown error",
      userId: id,
    });
    throw error;
  }
}

// ❌ 잘못된 패턴 - 타입 생략
async function getUserById(id) {
  // 타입 생략
  try {
    return await this.userRepo.findOneById(id);
  } catch (error) {
    // unknown 타입 생략
    console.log(error); // 콘솔 사용 금지
    throw error;
  }
}
```

#### 핵심 규칙

- **any 타입 완전 금지**: 모든 변수와 매개변수에 명시적 타입 지정
- **함수 반환 타입 필수**: 모든 함수에 명시적 반환 타입 지정
- **catch 블록 타이핑**: `catch (error: unknown)` 패턴 사용
- **콘솔 사용 금지**: Logger 클래스만 사용

### 2. 서비스 클래스 구조 표준

#### 메서드 명명 표준

- **조회 메서드**: `findById`, `findByIdOrFail`, `findByServiceIds`, `findByAnd`, `findByOr`
- **검색 메서드**: `searchEntities`, `getEntityDetail`
- **변경 메서드**: `createEntity`, `updateEntity`, `deleteEntity`
- **Private 메서드**: `build-`, `get-`, `validate-`, `format-` 접두사

### 3. 경로 별칭

모든 서비스에서 공통으로 사용하는 TypeScript 경로 별칭:

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@modules/*": ["src/modules/*"],
      "@common/*": ["src/common/*"],
      "@config/*": ["src/config/*"],
      "@database/*": ["src/database/*"]
    }
  }
}
```

### 4. Import/Export 패턴

- ES 모듈 지원: `"type": "module"`을 package.json에 설정
- 빌드 출력에서 경로 별칭 해석: `tsc-alias` 사용
- 일관된 패키지 참조 및 타입 imports

---

## 테스트 가이드

### 1. 기본 테스트 명령어

```bash
# 단위 테스트
npm run test               # Jest 테스트 실행
npm run test:watch         # 감시 모드 테스트
npm run test:cov           # 커버리지 테스트

# E2E 테스트
npm run test:e2e           # 엔드투엔드 테스트
```

### 2. 품질 보증 표준

- Jest 테스트 실행: 모든 단위 테스트 통과
- E2E 테스트: 주요 API 엔드포인트 검증
- 코드 품질: ESLint + Prettier 통합
- 타입 검사: 완전한 TypeScript 구현

---

## 체크리스트

### 🔥 필수 구현 사항

#### 환경 설정 및 인프라

- [ ] Docker 네트워크 구성 (msa-network, authz-network 등)
- [ ] 서비스별 포트 설정 (auth-server: 8000, authz-server: 8100)
- [ ] 환경 변수 파일 구조 (`envs/` 디렉토리)
- [ ] MySQL/Redis 독립적 포트 구성

#### 공유 라이브러리 통합

- [ ] `@krgeobuk` 스코프 패키지 설정
- [ ] `.npmrc`에 Verdaccio K8s 레지스트리 URL 설정
- [ ] ESM 모듈 설정 (`"type": "module"`)
- [ ] TypeScript 경로 별칭 설정

#### 서비스 아키텍처

- [ ] 서비스별 Logger 초기화
- [ ] 메서드 순서 표준 준수 (PUBLIC → PRIVATE)
- [ ] 에러 처리 패턴 적용 (try-catch, HttpException 체크)
- [ ] 트랜잭션 지원 (선택적 EntityManager 매개변수)
- [ ] 마이크로서비스 통신 설정 (ClientProxy 주입)

#### 중간 테이블 서비스 (해당하는 경우)

- [ ] 고급 엔티티 설계 (복합 인덱스, 유니크 제약조건)
- [ ] 배치 처리 결과 인터페이스 구현 (`{ success, assigned, duplicates, newAssignments }`)
- [ ] `hasUsersForRole()` 최적화 메서드 필수 구현
- [ ] ID만 반환하는 최적화된 조회 메서드
- [ ] `Record<string, string[]>` 반환 타입 배치 메서드
- [ ] 트랜잭션 기반 `replaceUserRoles()` 구현
- [ ] 배치 존재 확인 최적화 (`existsUserRoles()`)
- [ ] 역할/권한 변경 시 Redis 캐시 무효화 (`invalidateUserPermissions()`) 호출

#### 컨트롤러 패턴

- [ ] RESTful API 엔드포인트 표준 준수
- [ ] TCP 메시지 패턴 명명 규칙 적용 (entity.action 형식)
- [ ] TCP 메서드 순서 표준 (조회 → 존재확인 → 배치처리)
- [ ] TCP 응답 표준 인터페이스 구현
- [ ] 로그 레벨 최적화 (고빈도 API는 DEBUG)

#### Repository 고급 최적화

- [ ] 필요한 컬럼만 SELECT
- [ ] 인덱스 최적화된 WHERE 조건 순서
- [ ] Promise.all()을 통한 병렬 쿼리 처리
- [ ] `SELECT 1 + LIMIT` 존재 확인 패턴
- [ ] 배치 조회용 Map 초기화 (빈 배열 보장)
- [ ] 관계 삭제 전 의존성 카운트 확인

#### 마이크로서비스 통신

- [ ] TCP 클라이언트 주입 및 설정
- [ ] 폴백 전략 구현 (서비스 장애 시)
- [ ] 서킷 브레이커 패턴 적용
- [ ] 배치 통신 최적화 (N+1 방지)
- [ ] 타임아웃 및 재시도 정책 설정
- [ ] 통신 지표 수집 및 모니터링

#### 코드 품질

- [ ] any 타입 완전 배제
- [ ] 모든 함수에 반환 타입 명시
- [ ] catch 블록에서 error: unknown 타입 지정
- [ ] Logger 클래스만 사용 (console 금지)
- [ ] ESLint + Prettier 통합
- [ ] tsc-alias를 통한 빌드 최적화

### ⚡ 고급 성능 최적화 사항

#### 로깅 최적화

- [ ] 고빈도 API DEBUG 로깅으로 전환
- [ ] 구조화된 메타데이터 포함
- [ ] 민감 정보 로깅 제외

#### 데이터베이스 최적화

- [ ] 중간 테이블 존재 확인 최적화 구현
- [ ] 배치 처리를 통한 N+1 쿼리 방지
- [ ] Raw 쿼리 매핑 최적화
- [ ] 인덱스 활용 쿼리 순서 설정
- [ ] 고빈도 조회에 Redis 캐시 적용 (TTL: 5분)
- [ ] 데이터 변경 시 관련 캐시 즉시 무효화

#### 트랜잭션 최적화

- [ ] 트랜잭션 기반 안전한 Replace 작업
- [ ] 선택적 EntityManager 매개변수 지원
- [ ] 배치 삽입 최적화

#### 마이크로서비스 통신 최적화

- [ ] 병렬 서비스 호출 구현
- [ ] 연결 풀링 설정
- [ ] 서비스 헬스체크 구현
- [ ] 배치 데이터 조회 패턴

### 🔧 인프라 및 운영 사항

#### Docker 환경

- [ ] 멀티 스테이지 빌드 구성
- [ ] 네트워크 격리 설정
- [ ] 환경별 Docker Compose 파일

#### 모니터링

- [ ] 서비스 헬스체크 엔드포인트
- [ ] 통신 지표 수집
- [ ] 에러율 모니터링

#### 보안

- [ ] JWT 토큰 검증 구현
- [ ] 서비스 간 인증 설정
- [ ] 민감 정보 환경 변수 분리

---

## 결론

이 가이드는 krgeobuk 생태계의 모든 NestJS 서버에서 **일관되고 고성능인 엔터프라이즈급 코드**를 작성하기 위한 완전한 실전 표준을 제공합니다.

### 핵심 특징

1. **완전한 인프라 통합**: Verdaccio K8s 레지스트리, 공유 라이브러리 생태계
2. **고급 중간 테이블 관리**: 배치 처리, 트랜잭션 최적화, 성능 튜닝
3. **Redis 캐싱 전략**: 고빈도 조회 캐싱(5분 TTL) + 변경 시 즉시 무효화
4. **엔터프라이즈 마이크로서비스 통신**: 폴백 전략, 서킷 브레이커, 헬스체크
5. **고성능 Repository 패턴**: Raw 쿼리 최적화, 배치 조회, N+1 방지
6. **실전 TCP 컨트롤러**: 메시지 패턴, 안전 기본값(실패 시 빈 배열/false), 로깅 최적화

### 적용 우선순위

**1단계 (핵심 기반)**: 환경 설정, 공유 라이브러리, 기본 서비스 패턴
**2단계 (고성능화)**: 중간 테이블 최적화, Repository 고도화, TCP 통신
**3단계 (엔터프라이즈)**: 마이크로서비스 통신, 모니터링, 운영 최적화

### 기대 효과

- **개발 생산성 향상**: 표준화된 패턴으로 빠른 개발
- **성능 최적화**: N+1 쿼리 방지, 배치 처리, 통신 최적화
- **운영 안정성**: 폴백 전략, 헬스체크, 서킷 브레이커
- **코드 일관성**: 팀 전체가 동일한 표준 적용
- **확장성**: 마이크로서비스 아키텍처의 안정적 확장

krgeobuk 생태계의 모든 개발자는 이 표준을 준수하여 **세계 수준의 마이크로서비스 아키텍처**를 구축하고, 지속 가능한 고품질 서비스를 제공해야 합니다.
