# BFF Architecture Design for krgeobuk-infra

## 📋 개요

krgeobuk-infra 프로젝트의 아키텍처 개선을 위한 단계적 접근 방법을 제시합니다. **1단계로 SRP 리팩토링을 통해 도메인 서비스를 개선**하고, **2단계에서 필요시 BFF 패턴을 도입**하여 클라이언트별 최적화를 달성합니다.

## 🎯 전략 결정: SRP 리팩토링 우선 추진

### 현재 결정사항
**Phase 1: SRP 리팩토링 먼저 진행** (현재 선택)
- 기존 아키텍처 유지하면서 점진적 개선
- 즉시 개선 효과 (테스트 용이성, 코드 가독성)
- 낮은 리스크와 학습 비용
- BFF 도입을 위한 기반 준비

**Phase 2: BFF 도입 고려** (향후 필요시)
- 모바일 앱 개발 확정 시
- 외부 API 연동 요구 증가 시
- 성능 최적화 한계 도달 시

## 🔧 Phase 1: SRP 리팩토링 상세 계획

### 현재 문제점 분석
- **RoleService**: 5개 외부 의존성, 복잡한 Aggregation 로직
- **PermissionService**: 도메인 경계 위반, 교차 서비스 의존성
- **UserRoleService**: 외부 통신과 비즈니스 로직 혼재

### krgeobuk 최적화 리팩토링 전략

#### 전략 1: krgeobuk 패키지 중심 분리 (최우선 권장)
```typescript
// krgeobuk 생태계 활용한 분리
authz-server/src/modules/role/
├── services/
│   ├── role-data.service.ts        # 순수 데이터 레이어
│   ├── role-business.service.ts    # 비즈니스 로직 레이어  
│   ├── role-tcp.service.ts         # TCP 통신 레이어
│   └── role.service.ts             # 조정 레이어 (Facade)
├── clients/                        # TCP 클라이언트 분리
│   ├── auth-client.service.ts      # @krgeobuk/clients 활용
│   ├── portal-client.service.ts
│   └── permission-client.service.ts
├── aggregators/                    # 집계 로직 분리
│   ├── role-enrichment.aggregator.ts
│   └── role-statistics.aggregator.ts
└── strategies/                     # 전략 패턴 적용
    ├── role-cache.strategy.ts      # @krgeobuk/cache 활용
    ├── role-fallback.strategy.ts   # @krgeobuk/fallback 활용
    └── role-validation.strategy.ts # @krgeobuk/validation 활용
```

#### 전략 2: CQRS 패턴 (보조)
```typescript
// 읽기/쓰기 책임 분리 (필요시 추가 적용)
├── RoleCommandService (생성, 수정, 삭제)
├── RoleQueryService (조회, 검색, 집계)
└── RoleService (위임 및 조정)
```

#### 공통 서비스 패키지 전략
```typescript
// shared-lib/packages/msa-commons/
├── strategies/
│   ├── tcp-fallback.strategy.ts    # TCP 호출 실패 시 폴백
│   ├── cache.strategy.ts           # 도메인별 캐싱 전략
│   └── validation.strategy.ts      # 비즈니스 검증 전략
├── aggregators/
│   ├── base-enrichment.aggregator.ts
│   └── batch-processor.aggregator.ts
├── clients/
│   ├── tcp-client.base.ts
│   └── microservice-client.factory.ts
└── types/
    ├── tcp-message.types.ts
    └── enrichment.types.ts
```

### 개선된 구현 로드맵

#### Week 1: 기반 서비스 구축
- [ ] `@krgeobuk/msa-commons` 패키지 생성
- [ ] TcpFallbackStrategy, CacheStrategy 구현
- [ ] RoleDataService 분리 및 테스트
- [ ] 기본 TCP 클라이언트 베이스 구현

#### Week 2: TCP 레이어 분리
- [ ] RoleTcpService 구현 (폴백 전략 포함)
- [ ] 기존 외부 호출을 TCP 서비스로 이관
- [ ] 배치 처리 최적화 적용
- [ ] TCP 통신 단위 테스트

#### Week 3: 비즈니스 레이어 구축
- [ ] RoleBusinessService 구현
- [ ] RoleEnrichmentAggregator 구현
- [ ] 비즈니스 검증 로직 분리
- [ ] 통계 및 집계 로직 최적화

#### Week 4: 통합 및 최적화
- [ ] RoleService Facade 완성
- [ ] 캐싱 전략 적용 및 최적화
- [ ] API 호환성 검증
- [ ] 성능 테스트 및 메모리 사용량 체크

#### Week 5-6: 다른 도메인 적용
- [ ] Permission, UserRole 서비스에 동일 패턴 적용
- [ ] 공통 서비스 재사용성 검증
- [ ] 중간테이블 도메인 최적화 적용
- [ ] 통합 테스트 및 문서화

### 핵심 구현 예시

#### 1. RoleDataService (순수 데이터 레이어)
```typescript
// ✅ 오직 데이터 접근만 담당
@Injectable()
export class RoleDataService {
  private readonly logger = new Logger(RoleDataService.name);

  constructor(private readonly roleRepo: RoleRepository) {}

  // ==================== BASIC CRUD ====================
  async findById(roleId: string): Promise<RoleEntity | null> {
    return this.roleRepo.findOneById(roleId);
  }

  async findByIdOrFail(roleId: string): Promise<RoleEntity> {
    const role = await this.findById(roleId);
    if (!role) throw RoleException.roleNotFound();
    return role;
  }

  async create(attrs: CreateRoleAttrs): Promise<RoleEntity> {
    const role = this.roleRepo.create(attrs);
    return this.roleRepo.save(role);
  }

  async search(query: RoleSearchQuery): Promise<PaginatedResult<RoleEntity>> {
    return this.roleRepo.searchWithPagination(query);
  }

  async existsByName(name: string, excludeId?: string): Promise<boolean> {
    return this.roleRepo.existsByName(name, excludeId);
  }
}
```

#### 2. RoleTcpService (TCP 통신 레이어)
```typescript
// ✅ TCP 통신만 담당 + @krgeobuk/clients 활용
@Injectable()
export class RoleTcpService {
  constructor(
    @Inject('AUTH_SERVICE') private readonly authClient: ClientProxy,
    @Inject('PORTAL_SERVICE') private readonly portalClient: ClientProxy,
    private readonly tcpFallbackStrategy: TcpFallbackStrategy // @krgeobuk/fallback
  ) {}

  async fetchUsers(userIds: string[]): Promise<User[]> {
    if (userIds.length === 0) return [];

    try {
      return await firstValueFrom(
        this.authClient.send<User[]>('user.findByIds', { userIds })
      );
    } catch (error) {
      return this.tcpFallbackStrategy.getUsersFallback(userIds);
    }
  }

  async fetchRoleRelatedData(roleId: string): Promise<{
    userIds: string[];
    serviceIds: string[];
    permissionIds: string[];
  }> {
    const [userIds, serviceIds, permissionIds] = await Promise.allSettled([
      this.fetchUserIdsByRole(roleId),
      this.fetchServiceIdsByRole(roleId),
      this.fetchPermissionIdsByRole(roleId)
    ]);

    return {
      userIds: this.extractSettledValue(userIds, []),
      serviceIds: this.extractSettledValue(serviceIds, []),
      permissionIds: this.extractSettledValue(permissionIds, [])
    };
  }
}
```

#### 3. RoleBusinessService (비즈니스 로직 레이어)
```typescript
// ✅ 순수 비즈니스 로직만 담당
@Injectable()
export class RoleBusinessService {
  constructor(
    private readonly roleDataService: RoleDataService,
    private readonly roleTcpService: RoleTcpService,
    private readonly roleEnrichmentAggregator: RoleEnrichmentAggregator,
    private readonly roleValidationStrategy: RoleValidationStrategy // @krgeobuk/validation
  ) {}

  async createRole(attrs: CreateRoleAttrs): Promise<RoleEntity> {
    // 1. 비즈니스 검증
    await this.roleValidationStrategy.validateCreateRole(attrs);
    
    // 2. 중복 확인
    const exists = await this.roleDataService.existsByName(attrs.name);
    if (exists) throw RoleException.roleAlreadyExists();

    // 3. 생성
    return this.roleDataService.create(attrs);
  }

  async getRoleWithDetails(roleId: string): Promise<RoleDetail> {
    const role = await this.roleDataService.findByIdOrFail(roleId);
    return this.roleEnrichmentAggregator.enrichRole(role);
  }
}
```

#### 4. 최종 RoleService (Facade 패턴)
```typescript
// ✅ 단순 위임 및 조정만 담당
@Injectable()
export class RoleService {
  constructor(
    private readonly roleDataService: RoleDataService,
    private readonly roleBusinessService: RoleBusinessService
  ) {}

  // Simple CRUD (Direct Delegation)
  async findById(roleId: string): Promise<RoleEntity | null> {
    return this.roleDataService.findById(roleId);
  }

  // Business Operations (Business Delegation)
  async createRole(attrs: CreateRoleAttrs): Promise<RoleEntity> {
    return this.roleBusinessService.createRole(attrs);
  }

  // Enriched Operations (Business Delegation)
  async getRoleById(roleId: string): Promise<RoleDetail> {
    return this.roleBusinessService.getRoleWithDetails(roleId);
  }
}
```

### 리팩토링 후 기대 효과

| 측면 | Before | After |
|------|--------|-------|
| **의존성** | RoleService: 5개 외부 클라이언트 | RoleDataService: 1개 레포지토리만 |
| **메서드 복잡도** | 50-100줄 복잡 로직 | 5-20줄 단순 로직 |
| **테스트** | 복잡 (Mock 5개) | 간단 (Mock 1개) |
| **재사용성** | 낮음 (특정 용도) | 높음 (순수 도메인) |
| **유지보수** | 어려움 | 쉬움 |
| **krgeobuk 활용도** | 낮음 | 높음 (@krgeobuk/authz-commons) |
| **TCP 최적화** | 개별 처리 | 배치 처리 + 폴백 전략 |

## 🎯 BFF 도입 배경 (Phase 2용)

### 현재 문제점
- **SRP 위반**: 도메인 서비스들이 Aggregation, 외부 통신, 데이터 변환 등 다중 책임 수행
- **도메인 경계 흐림**: RoleService가 auth, portal, permission 서비스들과 직접 통신
- **클라이언트 종속성**: 특정 클라이언트(portal-client)에 최적화된 API 구조
- **확장성 제약**: 새로운 클라이언트(모바일, 외부 API) 추가 시 기존 서비스 수정 필요

### BFF 도입 효과
- **관심사 분리**: 도메인 서비스는 순수 비즈니스 로직만, BFF는 Aggregation 담당
- **클라이언트 최적화**: 웹, 모바일, 서버 간 통신별 특화된 API 제공
- **확장성 확보**: 새 클라이언트 추가 시 도메인 서비스 변경 없이 BFF만 추가
- **성능 최적화**: 클라이언트별 캐싱 전략 및 데이터 최적화

## 🏗️ 전체 아키텍처 설계

### Multi-BFF Strategy

```yaml
# 클라이언트별 특화 BFF + 공통 BFF 조합
krgeobuk-infra/
├── auth-server/                    # 인증 도메인 (8000)
├── authz-server/                   # 권한 도메인 (8100)
├── api-gateway/                    # 🆕 API Gateway (7000)
├── bff-web/                        # 🆕 웹 전용 BFF (8200)
├── bff-mobile/                     # 🆕 모바일 전용 BFF (8210)
├── bff-server/                     # 🆕 서버 간 통신 BFF (8220)
├── portal-client/                  # 웹 클라이언트 (3000)
├── mobile-app/                     # 🆕 모바일 앱
└── shared-lib/                     # 공유 라이브러리
    └── bff-commons/                # 🆕 BFF 공통 패키지
```

### 네트워크 아키텍처

```yaml
# Client → Gateway → BFF → Services
portal-client:3000 → api-gateway:7000 → bff-web:8200 → auth/authz-servers
mobile-app → api-gateway:7000 → bff-mobile:8210 → auth/authz-servers
external-server → api-gateway:7000 → bff-server:8220 → auth/authz-servers

# Internal Communication (TCP)
bff-web ↔ auth-server:8000 (TCP)
bff-web ↔ authz-server:8100 (TCP)
```

## 📁 상세 폴더 구조

### 1. BFF-Web (웹 클라이언트 전용)

```typescript
bff-web/
├── src/
│   ├── modules/
│   │   ├── admin-portal/              # 관리자 포털 특화
│   │   │   ├── admin-dashboard.controller.ts
│   │   │   ├── admin-dashboard.service.ts
│   │   │   ├── user-management.controller.ts
│   │   │   ├── role-management.controller.ts
│   │   │   └── dtos/
│   │   │       ├── admin-dashboard.dto.ts
│   │   │       ├── user-table.dto.ts        # 웹 테이블용
│   │   │       └── role-tree.dto.ts         # 웹 트리뷰용
│   │   │
│   │   ├── user-portal/               # 일반 사용자 포털
│   │   │   ├── user-dashboard.controller.ts
│   │   │   ├── my-profile.controller.ts
│   │   │   └── dtos/
│   │   │       ├── user-dashboard.dto.ts
│   │   │       └── my-profile.dto.ts
│   │   │
│   │   ├── reports/                   # 웹 전용 리포트
│   │   │   ├── export.controller.ts   # CSV/Excel 다운로드
│   │   │   ├── charts.controller.ts   # 차트 데이터
│   │   │   └── dtos/
│   │   │       ├── chart-data.dto.ts
│   │   │       └── export-options.dto.ts
│   │   │
│   │   └── analytics/                 # 웹 전용 분석
│   │       ├── real-time.controller.ts # 실시간 웹소켓
│   │       └── dtos/
│   │
│   ├── common/
│   │   ├── guards/
│   │   │   ├── web-auth.guard.ts      # 웹 세션 기반 인증
│   │   │   └── csrf.guard.ts          # CSRF 보호
│   │   ├── interceptors/
│   │   │   ├── web-response.interceptor.ts # 웹 응답 형식
│   │   │   └── ssr-cache.interceptor.ts    # SSR 캐싱
│   │   └── strategies/
│   │       ├── pagination.strategy.ts      # 웹 페이지네이션
│   │       └── infinite-scroll.strategy.ts # 무한 스크롤
│   │
│   └── websocket/                     # 실시간 기능
│       ├── notifications.gateway.ts
│       └── real-time-updates.gateway.ts
```

### 2. BFF-Mobile (모바일 전용)

```typescript
bff-mobile/
├── src/
│   ├── modules/
│   │   ├── mobile-auth/               # 모바일 인증 플로우
│   │   │   ├── login.controller.ts
│   │   │   ├── biometric.controller.ts
│   │   │   └── dtos/
│   │   │       ├── mobile-login.dto.ts
│   │   │       └── biometric-auth.dto.ts
│   │   │
│   │   ├── mobile-dashboard/          # 모바일 대시보드
│   │   │   ├── home-screen.controller.ts
│   │   │   ├── quick-actions.controller.ts
│   │   │   └── dtos/
│   │   │       ├── home-screen.dto.ts  # 모바일 최적화
│   │   │       └── quick-actions.dto.ts
│   │   │
│   │   ├── offline/                   # 오프라인 지원
│   │   │   ├── sync.controller.ts
│   │   │   ├── cache.controller.ts
│   │   │   └── dtos/
│   │   │       ├── sync-data.dto.ts
│   │   │       └── offline-actions.dto.ts
│   │   │
│   │   └── notifications/             # 푸시 알림
│   │       ├── push.controller.ts
│   │       └── dtos/
│   │           └── push-message.dto.ts
│   │
│   ├── common/
│   │   ├── guards/
│   │   │   ├── mobile-jwt.guard.ts    # JWT + 디바이스 검증
│   │   │   └── device-binding.guard.ts
│   │   ├── interceptors/
│   │   │   ├── mobile-response.interceptor.ts # 모바일 최적 응답
│   │   │   └── offline-cache.interceptor.ts
│   │   └── strategies/
│   │       ├── mobile-pagination.strategy.ts   # 모바일 페이징
│   │       └── data-compression.strategy.ts    # 데이터 압축
│   │
│   └── push/                          # 푸시 알림 서비스
│       ├── fcm.service.ts
│       └── apns.service.ts
```

### 3. BFF-Server (서버 간 통신 전용)

```typescript
bff-server/
├── src/
│   ├── modules/
│   │   ├── service-auth/              # 서비스 간 인증
│   │   │   ├── service-token.controller.ts
│   │   │   ├── service-registry.controller.ts
│   │   │   └── dtos/
│   │   │       ├── service-token.dto.ts
│   │   │       └── service-info.dto.ts
│   │   │
│   │   ├── bulk-operations/           # 대량 처리
│   │   │   ├── bulk-users.controller.ts
│   │   │   ├── bulk-roles.controller.ts
│   │   │   └── dtos/
│   │   │       ├── bulk-request.dto.ts
│   │   │       └── bulk-result.dto.ts
│   │   │
│   │   ├── integration/               # 외부 시스템 연동
│   │   │   ├── ldap-sync.controller.ts
│   │   │   ├── sso-integration.controller.ts
│   │   │   └── dtos/
│   │   │       ├── ldap-sync.dto.ts
│   │   │       └── sso-config.dto.ts
│   │   │
│   │   └── monitoring/                # 서비스 모니터링
│   │       ├── health-check.controller.ts
│   │       ├── metrics.controller.ts
│   │       └── dtos/
│   │           ├── health-status.dto.ts
│   │           └── service-metrics.dto.ts
│   │
│   ├── common/
│   │   ├── guards/
│   │   │   ├── service-key.guard.ts   # API Key 인증
│   │   │   └── rate-limit.guard.ts    # Rate Limiting
│   │   ├── interceptors/
│   │   │   ├── server-response.interceptor.ts
│   │   │   └── audit-log.interceptor.ts # 감사 로그
│   │   └── strategies/
│   │       ├── batch-processing.strategy.ts
│   │       └── circuit-breaker.strategy.ts
│   │
│   └── jobs/                          # 배치 작업
│       ├── scheduled-sync.job.ts
│       └── cleanup.job.ts
```

### 4. 공유 BFF 패키지

```typescript
shared-lib/packages/bff-commons/
├── src/
│   ├── clients/                       # 공통 서비스 클라이언트
│   │   ├── base-service.client.ts
│   │   ├── auth-service.client.ts
│   │   ├── authz-service.client.ts
│   │   └── interfaces/
│   │       ├── service-client.interface.ts
│   │       └── tcp-message.interface.ts
│   │
│   ├── aggregation/                   # 공통 집계 로직
│   │   ├── base-aggregator.ts
│   │   ├── parallel-fetcher.ts
│   │   ├── data-enricher.ts
│   │   └── strategies/
│   │       ├── cache-first.strategy.ts
│   │       ├── cache-aside.strategy.ts
│   │       └── fallback.strategy.ts
│   │
│   ├── auth/                          # BFF 인증 공통
│   │   ├── jwt-extractor.ts
│   │   ├── permission-checker.ts
│   │   └── session-manager.ts
│   │
│   ├── cache/                         # 캐싱 전략
│   │   ├── redis-manager.ts
│   │   ├── memory-cache.ts
│   │   ├── cache-key-builder.ts
│   │   └── strategies/
│   │       ├── write-through.strategy.ts
│   │       ├── write-behind.strategy.ts
│   │       └── refresh-ahead.strategy.ts
│   │
│   ├── monitoring/                    # 공통 모니터링
│   │   ├── metrics-collector.ts
│   │   ├── health-checker.ts
│   │   ├── circuit-breaker.ts
│   │   └── request-tracer.ts
│   │
│   └── types/                         # 공통 타입
│       ├── bff-request.types.ts
│       ├── bff-response.types.ts
│       ├── aggregation.types.ts
│       └── client.types.ts
```

## 🔧 핵심 구현 예시

### 1. 도메인 서비스 최적화 (Before vs After)

#### Before: 복잡한 RoleService
```typescript
// ❌ 현재: RoleService가 모든 걸 다 하고 있음
@Injectable()
export class RoleService {
  constructor(
    private readonly roleRepo: RoleRepository,
    private readonly userRoleService: UserRoleService,           // 다른 도메인 의존
    @Inject('AUTH_SERVICE') private readonly authClient: ClientProxy,      // 외부 통신
    @Inject('PORTAL_SERVICE') private readonly portalClient: ClientProxy,  // 외부 통신  
    @Inject('ROLE_PERMISSION_SERVICE') private readonly rolePermissionClient: ClientProxy, // 외부 통신
    @Inject('PERMISSION_SERVICE') private readonly permissionClient: ClientProxy // 외부 통신
  ) {}

  // ❌ 문제: 하나의 메서드가 너무 많은 책임
  async getRoleById(roleId: string): Promise<RoleDetail> {
    // 1. 본인 데이터 조회
    const role = await this.roleRepo.findOneById(roleId);
    
    // 2. 외부 서비스들 호출 (Aggregation 책임)
    const [userIds, serviceIds, permissionIds] = await Promise.all([...]);
    
    // 3. 추가 외부 데이터 조회
    const [users, services, permissions] = await Promise.all([...]);
    
    // 4. 데이터 변환 및 조합 (Transformation 책임)
    return { /* 복잡한 조합 로직 */ };
  }
}
```

#### After: 순수한 RoleService
```typescript
// ✅ 순수한 도메인 서비스: 오직 Role 데이터만 관리
@Injectable()
export class RoleService {
  private readonly logger = new Logger(RoleService.name);

  constructor(
    private readonly roleRepo: RoleRepository  // 오직 자신의 레포지토리만
  ) {}

  // ✅ 단순하고 명확한 책임: Role 엔티티 조회만
  async findById(roleId: string): Promise<RoleEntity | null> {
    return this.roleRepo.findOneById(roleId);
  }

  async findByIdOrFail(roleId: string): Promise<RoleEntity> {
    const role = await this.roleRepo.findOneById(roleId);
    if (!role) {
      this.logger.debug('Role not found', { roleId });
      throw RoleException.roleNotFound();
    }
    return role;
  }

  // ✅ 순수한 CRUD 작업만
  async createRole(attrs: CreateRoleAttrs): Promise<RoleEntity> {
    const existingRole = await this.roleRepo.findByName(attrs.name);
    if (existingRole) {
      throw RoleException.roleAlreadyExists();
    }
    return this.roleRepo.save(this.roleRepo.create(attrs));
  }

  // ✅ 도메인 검색 로직만
  async searchRoles(query: RoleSearchQuery): Promise<PaginatedResult<RoleEntity>> {
    return this.roleRepo.searchWithPagination(query);
  }

  // ✅ 배치 조회 (BFF에서 사용)
  async findByIds(roleIds: string[]): Promise<RoleEntity[]> {
    return this.roleRepo.findByIds(roleIds);
  }
}
```

### 2. BFF에서 Aggregation 처리
```typescript
// ✅ BFF: 모든 Aggregation 책임을 담당
@Injectable()
export class RoleManagementService {
  constructor(
    private readonly authzClient: AuthzServiceClient,    // Authz 서비스 클라이언트
    private readonly authClient: AuthServiceClient,      // Auth 서비스 클라이언트
    private readonly portalClient: PortalServiceClient,  // Portal 서비스 클라이언트
    private readonly cacheManager: CacheManager,         // BFF 레벨 캐싱
    private readonly fallbackService: FallbackService    // 폴백 전략
  ) {}

  // ✅ BFF의 책임: 여러 서비스 조합
  async getRoleWithDetails(roleId: string): Promise<RoleDetailDto> {
    const cacheKey = `role-detail:${roleId}`;
    
    // 1. 캐시 확인
    const cached = await this.cacheManager.get<RoleDetailDto>(cacheKey);
    if (cached) return cached;

    try {
      // 2. 병렬로 필요한 데이터 수집
      const [role, userIds, serviceIds, permissionIds] = await Promise.allSettled([
        this.authzClient.getRole(roleId),                           
        this.authzClient.getUserIdsByRole(roleId),                  
        this.portalClient.getServiceIdsByRole(roleId),              
        this.authzClient.getPermissionIdsByRole(roleId)             
      ]);

      // 3. 기본 Role 정보 확인
      const roleData = this.extractSuccessValue(role);
      if (!roleData) throw new NotFoundException('Role not found');

      // 4. 관련 데이터 조회
      const [users, services, permissions] = await Promise.allSettled([
        userIdList.length > 0 ? this.authClient.getUsersByIds(userIdList) : Promise.resolve([]),
        serviceIdList.length > 0 ? this.portalClient.getServicesByIds(serviceIdList) : Promise.resolve([]),
        permissionIdList.length > 0 ? this.authzClient.getPermissionsByIds(permissionIdList) : Promise.resolve([])
      ]);

      // 5. 응답 DTO 구성
      const roleDetail: RoleDetailDto = {
        id: roleData.id,
        name: roleData.name,
        description: roleData.description,
        users: (this.extractSuccessValue(users) || []).map(user => ({
          id: user.id,
          username: user.username,
          email: user.email
        })),
        services: (this.extractSuccessValue(services) || []).map(service => ({
          id: service.id,
          name: service.name,
          description: service.description
        })),
        permissions: (this.extractSuccessValue(permissions) || []).map(permission => ({
          id: permission.id,
          action: permission.action,
          resource: permission.resource
        })),
        statistics: {
          userCount: (this.extractSuccessValue(users) || []).length,
          serviceCount: (this.extractSuccessValue(services) || []).length,
          permissionCount: (this.extractSuccessValue(permissions) || []).length
        }
      };

      // 6. 캐시 저장 (5분)
      await this.cacheManager.set(cacheKey, roleDetail, 300);
      return roleDetail;

    } catch (error: unknown) {
      return this.fallbackService.getRoleDetailFallback(roleId);
    }
  }
}
```

### 3. 클라이언트별 특화 최적화
```typescript
// 웹용: 풍부한 데이터
interface WebUserDashboard {
  user: DetailedUserInfo;           // 상세 정보
  roles: RoleWithPermissions[];     // 전체 권한 정보
  recentActivities: Activity[];     // 최근 활동 목록
  analytics: UserAnalytics;         // 상세 분석
  recommendations: string[];        // 추천사항
}

// 모바일용: 경량화 데이터
interface MobileUserDashboard {
  user: BasicUserInfo;              // 기본 정보만
  roles: string[];                  // 역할명만
  unreadNotifications: number;      // 숫자만
  quickActions: QuickAction[];      // 빠른 액션
}

// 서버용: 처리 최적화 데이터
interface ServerUserInfo {
  userId: string;
  permissions: string[];            // 권한 코드만
  tokenExpiry: number;              // 만료시간
  serviceScopes: string[];          // 서비스 범위
}
```

### 4. 통합 인증 서비스
```typescript
// BFF 공통 인증 서비스
@Injectable()
export class BFFAuthService {
  constructor(
    @Inject('AUTH_SERVICE') private authClient: ClientProxy,
    @Inject('AUTHZ_SERVICE') private authzClient: ClientProxy,
    private cacheManager: CacheManager
  ) {}

  // 클라이언트별 인증 전략
  async authenticateWeb(sessionToken: string): Promise<WebAuthResult> {
    // 세션 기반 + CSRF 토큰
  }

  async authenticateMobile(jwtToken: string, deviceId: string): Promise<MobileAuthResult> {
    // JWT + 디바이스 바인딩
  }

  async authenticateServer(apiKey: string, serviceId: string): Promise<ServerAuthResult> {
    // API Key + 서비스 등록 확인
  }

  // 통합 권한 체크 (캐싱 최적화)
  async checkPermissions(userId: string, resource: string, action: string): Promise<boolean> {
    const cacheKey = `perm:${userId}:${resource}:${action}`;
    
    let hasPermission = await this.cacheManager.get<boolean>(cacheKey);
    if (hasPermission === null) {
      hasPermission = await this.fetchPermissionFromServices(userId, resource, action);
      await this.cacheManager.set(cacheKey, hasPermission, 300); // 5분 캐시
    }
    
    return hasPermission;
  }
}
```

### 5. 지능형 캐싱 전략
```typescript
// 클라이언트별 캐싱 전략
@Injectable()
export class SmartCacheService {
  constructor(
    private redisManager: RedisManager,
    private memoryCache: MemoryCache
  ) {}

  // 웹: 실시간성 중요, 짧은 캐시
  async cacheWebData(key: string, data: any): Promise<void> {
    await Promise.all([
      this.memoryCache.set(key, data, 60),      // 1분 메모리 캐시
      this.redisManager.set(key, data, 300)     // 5분 Redis 캐시
    ]);
  }

  // 모바일: 네트워크 절약, 긴 캐시
  async cacheMobileData(key: string, data: any): Promise<void> {
    await Promise.all([
      this.memoryCache.set(key, data, 1800),    // 30분 메모리 캐시
      this.redisManager.set(key, data, 7200)    // 2시간 Redis 캐시
    ]);
  }

  // 서버: 성능 최우선, 매우 긴 캐시
  async cacheServerData(key: string, data: any): Promise<void> {
    await this.redisManager.set(key, data, 21600); // 6시간 캐시
  }
}
```

## 📊 성능 최적화 전략

### 클라이언트별 최적화

| 클라이언트 | 최적화 전략 | 캐시 시간 | 응답 크기 | 주요 특징 |
|------------|-------------|-----------|-----------|-----------|
| **웹** | 실시간 업데이트 | 1-5분 | 전체 데이터 | 풍부한 UI, 실시간성 |
| **모바일** | 배터리/데이터 절약 | 30분-2시간 | 압축된 데이터 | 오프라인 지원, 푸시 알림 |
| **서버** | 처리량 최대화 | 6시간+ | 필수 데이터만 | 대량 처리, 고성능 |

### Docker Compose 설정

```yaml
# docker-compose.bff.yml
version: '3.8'
services:
  api-gateway:
    build: ./api-gateway
    ports:
      - "7000:7000"
    environment:
      - BFF_WEB_URL=http://bff-web:8200
      - BFF_MOBILE_URL=http://bff-mobile:8210
      - BFF_SERVER_URL=http://bff-server:8220
    networks:
      - msa-network
    depends_on:
      - bff-web
      - bff-mobile
      - bff-server

  bff-web:
    build: ./bff-web
    ports:
      - "8200:8200"
    environment:
      - AUTH_SERVICE_URL=http://auth-server:8000
      - AUTHZ_SERVICE_URL=http://authz-server:8100
      - REDIS_URL=redis://redis:6379
    networks:
      - msa-network
    depends_on:
      - auth-server
      - authz-server
      - redis

  bff-mobile:
    build: ./bff-mobile
    ports:
      - "8210:8210"
    environment:
      - AUTH_SERVICE_URL=http://auth-server:8000
      - AUTHZ_SERVICE_URL=http://authz-server:8100
      - REDIS_URL=redis://redis:6379
      - FCM_SERVER_KEY=${FCM_SERVER_KEY}
    networks:
      - msa-network
    depends_on:
      - auth-server
      - authz-server
      - redis

  bff-server:
    build: ./bff-server
    ports:
      - "8220:8220"
    environment:
      - AUTH_SERVICE_URL=http://auth-server:8000
      - AUTHZ_SERVICE_URL=http://authz-server:8100
      - REDIS_URL=redis://redis:6379
    networks:
      - msa-network
    depends_on:
      - auth-server
      - authz-server
      - redis

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - msa-network
    volumes:
      - redis_data:/data

networks:
  msa-network:
    external: true

volumes:
  redis_data:
```

## 🚀 전체 로드맵

### 현재 진행: Phase 1 - krgeobuk 최적화 SRP 리팩토링 (6주)
**목표**: krgeobuk 생태계를 활용한 도메인 서비스 SRP 준수
- Week 1: 기반 서비스 구축 (@krgeobuk/authz-commons 패키지)
- Week 2: TCP 레이어 분리 (폴백 전략 + 배치 처리)
- Week 3: 비즈니스 레이어 구축 (집계 로직 최적화)
- Week 4: 통합 및 최적화 (캐싱 전략 + API 호환성)
- Week 5-6: 다른 도메인 적용 (중간테이블 도메인 최적화)
- **핵심 개선점**: @krgeobuk 패키지 활용, TCP 최적화, 점진적 마이그레이션

**예상 기간**: 6주 **← 현재 진행 중**

### 향후 계획: Phase 2 - BFF 도입 고려 (필요시)

#### 조건부 실행 - 다음 중 하나 이상 발생 시
- ✅ **모바일 앱 개발 확정**
- ✅ **외부 API 연동 요구 증가** (3개 이상 외부 시스템)
- ✅ **성능 최적화 한계 도달** (현재 구조로 해결 불가)
- ✅ **클라이언트 다양화** (웹 외 2개 이상 클라이언트)

#### BFF 도입 시 단계별 계획

**Phase 2A: API Gateway + BFF-Web 구축**
**목표**: 웹 클라이언트 최적화
- API Gateway 구축
- BFF-Web 개발 (SRP 리팩토링된 서비스 활용)
- portal-client API 호출 경로 변경

**예상 기간**: 4-6주

**Phase 2B: BFF-Mobile 추가 구축**  
**목표**: 모바일 앱 지원
- BFF-Mobile 개발 (mobile-auth, mobile-dashboard, offline, notifications)
- 모바일 특화 인증/캐싱 전략 구현
- 푸시 알림 서비스 구축

**예상 기간**: 3-4주

**Phase 2C: BFF-Server 구축**
**목표**: 외부 시스템 연동 및 대량 처리 지원
- BFF-Server 개발 (service-auth, bulk-operations, integration, monitoring)
- 외부 API 연동 인터페이스 구축
- 배치 작업 및 모니터링 시스템 구축

**예상 기간**: 2-3주

**Phase 3: GraphQL Federation 고려**
**목표**: 차세대 API 아키텍처 준비 (장기 계획)
- GraphQL 스키마 설계
- Apollo Federation 도입 검토
- 점진적 마이그레이션 계획

**예상 기간**: 4-6주

## 📈 기대 효과 및 KPI

### 개발 효율성
- **코드 재사용성**: 도메인 서비스 재사용률 80% 이상
- **개발 속도**: 새 클라이언트 기능 개발 시간 50% 단축
- **테스트 용이성**: 단위 테스트 커버리지 90% 이상

### 성능 개선
- **응답 시간**: 클라이언트별 최적화로 평균 30% 단축
- **네트워크 사용량**: 모바일 데이터 사용량 40% 절약
- **서버 자원**: 캐싱 최적화로 DB 부하 50% 감소

### 운영 효율성
- **장애 격리**: 클라이언트별 독립적 장애 처리
- **모니터링**: 클라이언트별 세분화된 메트릭 수집
- **확장성**: 새 클라이언트 추가 시 기존 서비스 무변경

## 🔍 모니터링 및 운영

### 메트릭 수집
```typescript
// BFF별 메트릭 수집
interface BFFMetrics {
  responseTime: number;
  cacheHitRate: number;
  errorRate: number;
  throughput: number;
  clientType: 'web' | 'mobile' | 'server';
}
```

### 헬스체크
```typescript
// 통합 헬스체크
@Get('/health')
async healthCheck() {
  const [webHealth, mobileHealth, serverHealth] = await Promise.allSettled([
    this.webBFFClient.health(),
    this.mobileBFFClient.health(),
    this.serverBFFClient.health()
  ]);

  return {
    status: 'ok',
    services: { webHealth, mobileHealth, serverHealth },
    timestamp: new Date().toISOString()
  };
}
```

### 로그 구조
```typescript
// 통합 로그 포맷
interface BFFLogEntry {
  timestamp: string;
  level: 'info' | 'warn' | 'error' | 'debug';
  service: 'bff-web' | 'bff-mobile' | 'bff-server';
  traceId: string;
  clientType: string;
  operation: string;
  duration: number;
  userId?: string;
  error?: string;
  metadata: Record<string, any>;
}
```

## 🎯 결론

### 현재 전략: krgeobuk 최적화 SRP 리팩토링 우선 추진

krgeobuk-infra 프로젝트는 **krgeobuk 생태계 특성을 활용한 단계적 접근**을 통해 아키텍처를 개선합니다:

#### Phase 1: krgeobuk 최적화 SRP 리팩토링 (현재 진행)
1. **krgeobuk 생태계 완전 활용**: @krgeobuk/authz-commons 패키지로 재사용성 극대화
2. **TCP 통신 최적화**: 전용 TCP 서비스 레이어 + 체계적 폴백 전략 + 배치 처리
3. **점진적 마이그레이션**: API 호환성 100% 유지하면서 레이어별 단계적 적용
4. **성능 중심 설계**: 지능형 캐싱 전략 + 병렬 처리 + 메모리 최적화
5. **중간테이블 도메인 최적화**: krgeobuk 특성인 중간테이블 패턴 전용 최적화

#### 핵심 개선점
- **Facade 패턴**: RoleService → 4개 레이어 분리 (Data/TCP/Business/Aggregation)
- **전략 패턴**: Cache/Fallback/Validation 전략으로 재사용성 확보
- **공통 패키지**: TCP 클라이언트, 집계 로직, 폴백 전략 공통화
- **배치 최적화**: 관련 데이터 병렬 수집 + Promise.allSettled 활용

#### Phase 2: BFF 도입 (조건부)
**다음 상황 발생 시 BFF 도입 고려:**
- 모바일 앱 개발 확정
- 외부 시스템 연동 3개 이상
- 현재 구조로 성능 최적화 한계 도달
- 웹 외 2개 이상 클라이언트 필요

#### 기대 효과
- **즉시**: SRP 준수 + krgeobuk 패키지 활용도 극대화
- **중기**: TCP 최적화를 통한 MSA 통신 성능 향상
- **장기**: BFF 도입 시 이미 최적화된 서비스 레이어 활용

이 전략을 통해 **krgeobuk 생태계 특성에 최적화된** 아키텍처를 구축하면서 **미래 확장성**을 동시에 확보할 수 있습니다.