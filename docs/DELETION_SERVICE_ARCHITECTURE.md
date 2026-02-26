# Deletion Service Architecture

krgeobuk 마이크로서비스 생태계에서 전용 삭제 처리 서버를 통한 데이터 정리 아키텍처 설계 문서입니다.

## 목차
- [개요](#개요)
- [현재 문제점](#현재-문제점)
- [아키텍처 구조](#아키텍처-구조)
- [구현 가이드](#구현-가이드)
- [장단점 분석](#장단점-분석)
- [구현 단계](#구현-단계)
- [운영 고려사항](#운영-고려사항)

## 개요

### 배경
MSA 환경에서 서비스 간 데이터 의존성으로 인한 삭제 처리 복잡도 증가와 데이터 일관성 문제를 해결하기 위한 전용 삭제 처리 서버 아키텍처입니다.

### 목표
- 서비스 간 결합도 감소
- 삭제 작업의 안정성 보장
- 확장 가능한 삭제 처리 메커니즘 제공
- 모니터링 및 복구 가능성 향상

## 현재 문제점

### 데이터베이스 의존성 문제
```
portal-server DB: service 테이블
authz-server DB: role, permission, service_visible_role 테이블 (service 외래키 참조)
```

### MSA 원칙 위반
- 데이터베이스 간 강한 결합
- 참조 무결성 보장 어려움
- 삭제 시 일관성 문제 발생

### 현재 해결 방식의 한계
```typescript
// 현재: 동기 TCP 호출로 의존성 확인
async deleteService(id: string): Promise<void> {
  const hasRoles = await this.hasRolesForService(id);
  const hasPermissions = await this.hasPermissionsForService(id);
  const hasVisibleRoles = await this.hasVisibleRolesForService(id);
  
  if (hasRoles || hasPermissions || hasVisibleRoles) {
    throw ServiceException.serviceDeleteError(); // 삭제 차단
  }
  
  await this.serviceRepo.softDelete(id);
}
```

## 아키텍처 구조

### 전체 구조도
```
┌─────────────────┐    TCP     ┌─────────────────┐    스케줄링    ┌─────────────────┐
│   portal-server │ ────────→  │ deletion-server │ ────────────→  │   authz-server  │
│                 │            │                 │                │                 │
│ service 삭제    │            │ deletion_queue  │                │ 관련 데이터 삭제│
│ 요청 발생       │            │ 테이블 관리     │                │ 실행           │
└─────────────────┘            └─────────────────┘                └─────────────────┘
```

### 컴포넌트별 책임

#### Portal-Server
- 서비스 메타데이터 관리
- 삭제 요청 상태 관리 (`ACTIVE` → `DELETING` → `DELETED`)
- deletion-server에 삭제 요청 전송

#### Deletion-Server
- 삭제 작업 오케스트레이션
- 삭제 큐 관리 및 스케줄링
- 재시도 메커니즘 및 상태 추적
- 각 서비스별 삭제 로직 조율

#### Authz-Server
- 관련 데이터 삭제 실행
- 트랜잭션 보장된 의존성 정리
- 삭제 결과 보고

## 구현 가이드

### 1. Deletion-Server 구성

#### 1.1 삭제 큐 테이블 구조
```typescript
@Entity('deletion_queue')
export class DeletionQueueEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  resourceType: string; // 'service', 'role', 'permission'

  @Column()
  resourceId: string;

  @Column()
  requestedBy: string; // 요청한 서버/사용자

  @Column({ type: 'json' })
  metadata: any; // 삭제에 필요한 추가 정보

  @Column({ default: 'PENDING' })
  status: DeletionStatus; // PENDING, PROCESSING, COMPLETED, FAILED

  @Column({ type: 'int', default: 0 })
  retryCount: number;

  @Column({ type: 'text', nullable: true })
  errorMessage: string;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @Column({ nullable: true })
  scheduledAt: Date; // 지연 삭제 지원
}
```

#### 1.2 삭제 상태 관리
```typescript
export enum DeletionStatus {
  PENDING = 'PENDING',
  PROCESSING = 'PROCESSING',
  COMPLETED = 'COMPLETED',
  FAILED = 'FAILED',
  CANCELLED = 'CANCELLED'
}
```

#### 1.3 삭제 요청 인터페이스
```typescript
export interface DeletionRequest {
  resourceType: string;
  resourceId: string;
  requestedBy: string;
  metadata?: any;
  scheduledAt?: Date;
  priority?: 'LOW' | 'NORMAL' | 'HIGH';
}
```

### 2. Portal-Server 구현

#### 2.1 ServiceManager 수정
```typescript
@Injectable()
export class ServiceManager {
  constructor(
    private readonly serviceRepo: ServiceRepository,
    @Inject('DELETION_SERVICE') private readonly deletionClient: ClientProxy
  ) {}

  async deleteService(serviceId: string): Promise<void> {
    const service = await this.findByIdOrFail(serviceId);
    
    try {
      // 1. 서비스 상태를 DELETING으로 변경
      await this.serviceRepo.update(serviceId, { 
        status: ServiceStatus.DELETING,
        deletedAt: new Date()
      });

      // 2. deletion-server에 삭제 요청 전송
      await firstValueFrom(
        this.deletionClient.send('deletion.schedule', {
          resourceType: 'service',
          resourceId: serviceId,
          requestedBy: 'portal-server',
          metadata: {
            serviceName: service.name,
            hasRoles: true,
            hasPermissions: true,
            hasVisibleRoles: true
          }
        })
      );

      this.logger.log('서비스 삭제 요청 전송 완료', { serviceId });
    } catch (error) {
      // 요청 실패 시 상태 복구
      await this.serviceRepo.update(serviceId, { 
        status: ServiceStatus.ACTIVE,
        deletedAt: null
      });
      throw error;
    }
  }

  // deletion-server에서 호출하는 최종 삭제 메서드
  @MessagePattern('service.finalizeDelete')
  async finalizeDelete(@Payload() data: { serviceId: string }): Promise<void> {
    await this.serviceRepo.hardDelete(data.serviceId);
    this.logger.log('서비스 최종 삭제 완료', { serviceId: data.serviceId });
  }
}
```

#### 2.2 Module 설정
```typescript
@Module({
  imports: [
    TypeOrmModule.forFeature([ServiceEntity]),
    ClientsModule.register([
      {
        name: 'DELETION_SERVICE',
        transport: Transport.TCP,
        options: {
          host: process.env.DELETION_HOST || 'localhost',
          port: parseInt(process.env.DELETION_PORT || '8200', 10),
        },
      },
    ]),
  ],
  controllers: [ServiceController, ServiceTcpController],
  providers: [ServiceManager, ServiceRepository],
  exports: [ServiceManager],
})
export class ServiceModule {}
```

### 3. Deletion-Server 구현

#### 3.1 삭제 요청 컨트롤러
```typescript
@Controller()
export class DeletionController {
  constructor(private readonly deletionService: DeletionService) {}

  @MessagePattern('deletion.schedule')
  async scheduleDeletion(@Payload() request: DeletionRequest): Promise<void> {
    await this.deletionService.scheduleImmediateDeletion(request);
  }

  @MessagePattern('deletion.scheduleDelayed')
  async scheduleDelayedDeletion(@Payload() request: DeletionRequest & { delayMinutes: number }): Promise<void> {
    await this.deletionService.scheduleDelayedDeletion(request, request.delayMinutes);
  }

  @MessagePattern('deletion.cancel')
  async cancelDeletion(@Payload() data: { id: string }): Promise<void> {
    await this.deletionService.cancelDeletion(data.id);
  }

  @MessagePattern('deletion.getStatus')
  async getDeletionStatus(@Payload() data: { id: string }): Promise<DeletionQueueEntity> {
    return await this.deletionService.getDeletionStatus(data.id);
  }
}
```

#### 3.2 삭제 서비스
```typescript
@Injectable()
export class DeletionService {
  constructor(
    private readonly deletionQueueRepo: DeletionQueueRepository,
    @Inject('AUTHZ_SERVICE') private readonly authzClient: ClientProxy,
    @Inject('PORTAL_SERVICE') private readonly portalClient: ClientProxy
  ) {}

  async scheduleImmediateDeletion(request: DeletionRequest): Promise<DeletionQueueEntity> {
    const queueItem = new DeletionQueueEntity();
    queueItem.resourceType = request.resourceType;
    queueItem.resourceId = request.resourceId;
    queueItem.requestedBy = request.requestedBy;
    queueItem.metadata = request.metadata;
    queueItem.status = DeletionStatus.PENDING;
    queueItem.scheduledAt = new Date();
    
    return await this.deletionQueueRepo.save(queueItem);
  }

  async scheduleDelayedDeletion(request: DeletionRequest, delayMinutes: number): Promise<DeletionQueueEntity> {
    const queueItem = new DeletionQueueEntity();
    queueItem.resourceType = request.resourceType;
    queueItem.resourceId = request.resourceId;
    queueItem.requestedBy = request.requestedBy;
    queueItem.metadata = request.metadata;
    queueItem.status = DeletionStatus.PENDING;
    queueItem.scheduledAt = new Date(Date.now() + delayMinutes * 60 * 1000);
    
    return await this.deletionQueueRepo.save(queueItem);
  }

  async cancelDeletion(id: string): Promise<void> {
    await this.deletionQueueRepo.update(id, { 
      status: DeletionStatus.CANCELLED,
      updatedAt: new Date()
    });
  }

  async getDeletionStatus(id: string): Promise<DeletionQueueEntity> {
    return await this.deletionQueueRepo.findOneByOrFail({ id });
  }

  // 스케줄링 프로세서 (5분마다 실행)
  @Cron('0 */5 * * * *')
  async processDeletionQueue(): Promise<void> {
    const pendingItems = await this.deletionQueueRepo.find({
      where: { 
        status: DeletionStatus.PENDING,
        scheduledAt: LessThanOrEqual(new Date())
      },
      order: { createdAt: 'ASC' },
      take: 100
    });

    for (const item of pendingItems) {
      await this.processDeletionItem(item);
    }
  }

  private async processDeletionItem(item: DeletionQueueEntity): Promise<void> {
    try {
      // 상태를 PROCESSING으로 변경
      await this.deletionQueueRepo.update(item.id, { 
        status: DeletionStatus.PROCESSING 
      });

      // 리소스 타입에 따른 삭제 처리
      switch (item.resourceType) {
        case 'service':
          await this.processServiceDeletion(item);
          break;
        case 'role':
          await this.processRoleDeletion(item);
          break;
        case 'permission':
          await this.processPermissionDeletion(item);
          break;
        default:
          throw new Error(`Unknown resource type: ${item.resourceType}`);
      }

      // 성공 시 상태 업데이트
      await this.deletionQueueRepo.update(item.id, { 
        status: DeletionStatus.COMPLETED,
        updatedAt: new Date()
      });

      this.logger.log('삭제 처리 완료', { 
        id: item.id,
        resourceType: item.resourceType,
        resourceId: item.resourceId
      });

    } catch (error) {
      await this.handleDeletionError(item, error);
    }
  }

  private async processServiceDeletion(item: DeletionQueueEntity): Promise<void> {
    const { resourceId: serviceId, metadata } = item;

    // 1. authz-server에 관련 데이터 삭제 요청 (순서 중요)
    if (metadata.hasVisibleRoles) {
      await firstValueFrom(
        this.authzClient.send('deletion.deleteServiceVisibleRoles', { serviceId })
      );
    }

    if (metadata.hasPermissions) {
      await firstValueFrom(
        this.authzClient.send('deletion.deleteServicePermissions', { serviceId })
      );
    }

    if (metadata.hasRoles) {
      await firstValueFrom(
        this.authzClient.send('deletion.deleteServiceRoles', { serviceId })
      );
    }

    // 2. portal-server에 실제 서비스 삭제 요청
    await firstValueFrom(
      this.portalClient.send('service.finalizeDelete', { serviceId })
    );
  }

  private async processRoleDeletion(item: DeletionQueueEntity): Promise<void> {
    const { resourceId: roleId } = item;

    // 역할 삭제 로직
    await firstValueFrom(
      this.authzClient.send('deletion.deleteRole', { roleId })
    );
  }

  private async processPermissionDeletion(item: DeletionQueueEntity): Promise<void> {
    const { resourceId: permissionId } = item;

    // 권한 삭제 로직
    await firstValueFrom(
      this.authzClient.send('deletion.deletePermission', { permissionId })
    );
  }

  private async handleDeletionError(item: DeletionQueueEntity, error: any): Promise<void> {
    const newRetryCount = item.retryCount + 1;
    const maxRetries = 3;

    if (newRetryCount <= maxRetries) {
      // 지수적 백오프 (5분, 10분, 15분)
      const delayMinutes = newRetryCount * 5;
      await this.deletionQueueRepo.update(item.id, { 
        status: DeletionStatus.PENDING,
        retryCount: newRetryCount,
        errorMessage: error.message,
        scheduledAt: new Date(Date.now() + delayMinutes * 60 * 1000)
      });
    } else {
      await this.deletionQueueRepo.update(item.id, { 
        status: DeletionStatus.FAILED,
        errorMessage: error.message
      });
    }

    this.logger.error('삭제 처리 실패', { 
      id: item.id,
      error: error.message,
      retryCount: newRetryCount
    });
  }
}
```

### 4. Authz-Server 구현

#### 4.1 삭제 전용 TCP 컨트롤러
```typescript
@Controller()
export class DeletionTcpController {
  constructor(
    private readonly roleService: RoleService,
    private readonly permissionService: PermissionService,
    private readonly serviceVisibleRoleService: ServiceVisibleRoleService,
    private readonly userRoleService: UserRoleService,
    private readonly rolePermissionService: RolePermissionService
  ) {}

  @MessagePattern('deletion.deleteServiceVisibleRoles')
  async deleteServiceVisibleRoles(@Payload() data: { serviceId: string }): Promise<void> {
    await this.serviceVisibleRoleService.deleteByServiceId(data.serviceId);
  }

  @MessagePattern('deletion.deleteServicePermissions')
  async deleteServicePermissions(@Payload() data: { serviceId: string }): Promise<void> {
    // 트랜잭션 내에서 권한과 관련된 모든 데이터 삭제
    await this.dataSource.transaction(async (manager) => {
      // 1. 권한 ID 목록 조회
      const permissions = await this.permissionService.findByServiceId(data.serviceId);
      const permissionIds = permissions.map(p => p.id);

      // 2. role-permission 관계 삭제
      for (const permissionId of permissionIds) {
        await this.rolePermissionService.deleteByPermissionId(permissionId);
      }

      // 3. 권한 삭제
      await this.permissionService.deleteByServiceId(data.serviceId);
    });
  }

  @MessagePattern('deletion.deleteServiceRoles')
  async deleteServiceRoles(@Payload() data: { serviceId: string }): Promise<void> {
    // 트랜잭션 내에서 역할과 관련된 모든 데이터 삭제
    await this.dataSource.transaction(async (manager) => {
      // 1. 역할 ID 목록 조회
      const roles = await this.roleService.findByServiceId(data.serviceId);
      const roleIds = roles.map(r => r.id);

      // 2. user-role 관계 삭제
      for (const roleId of roleIds) {
        await this.userRoleService.deleteByRoleId(roleId);
      }

      // 3. role-permission 관계 삭제 (이미 위에서 처리됨)
      // 4. 역할 삭제
      await this.roleService.deleteByServiceId(data.serviceId);
    });
  }

  @MessagePattern('deletion.deleteRole')
  async deleteRole(@Payload() data: { roleId: string }): Promise<void> {
    await this.dataSource.transaction(async (manager) => {
      // 1. user-role 관계 삭제
      await this.userRoleService.deleteByRoleId(data.roleId);

      // 2. role-permission 관계 삭제
      await this.rolePermissionService.deleteByRoleId(data.roleId);

      // 3. service-visible-role 관계 삭제
      await this.serviceVisibleRoleService.deleteByRoleId(data.roleId);

      // 4. 역할 삭제
      await this.roleService.deleteById(data.roleId);
    });
  }

  @MessagePattern('deletion.deletePermission')
  async deletePermission(@Payload() data: { permissionId: string }): Promise<void> {
    await this.dataSource.transaction(async (manager) => {
      // 1. role-permission 관계 삭제
      await this.rolePermissionService.deleteByPermissionId(data.permissionId);

      // 2. 권한 삭제
      await this.permissionService.deleteById(data.permissionId);
    });
  }
}
```

#### 4.2 서비스별 삭제 메서드 추가
```typescript
// 각 서비스에 추가할 메서드들
export class RoleService {
  async deleteByServiceId(serviceId: string): Promise<void> {
    await this.roleRepo.delete({ serviceId });
  }

  async deleteById(roleId: string): Promise<void> {
    await this.roleRepo.delete({ id: roleId });
  }
}

export class PermissionService {
  async deleteByServiceId(serviceId: string): Promise<void> {
    await this.permissionRepo.delete({ serviceId });
  }

  async deleteById(permissionId: string): Promise<void> {
    await this.permissionRepo.delete({ id: permissionId });
  }
}

export class ServiceVisibleRoleService {
  async deleteByServiceId(serviceId: string): Promise<void> {
    await this.svrRepo.delete({ serviceId });
  }

  async deleteByRoleId(roleId: string): Promise<void> {
    await this.svrRepo.delete({ roleId });
  }
}

export class UserRoleService {
  async deleteByRoleId(roleId: string): Promise<void> {
    await this.userRoleRepo.delete({ roleId });
  }
}

export class RolePermissionService {
  async deleteByRoleId(roleId: string): Promise<void> {
    await this.rolePermissionRepo.delete({ roleId });
  }

  async deleteByPermissionId(permissionId: string): Promise<void> {
    await this.rolePermissionRepo.delete({ permissionId });
  }
}
```

## 장단점 분석

### 장점 ✅

#### 1. 관심사 분리 (Separation of Concerns)
- 각 서버의 명확한 책임 분리
- Portal-server: 서비스 메타데이터 관리
- Deletion-server: 삭제 작업 오케스트레이션  
- Authz-server: 권한 데이터 관리

#### 2. 안정성 및 복구 가능성
- 재시도 메커니즘 (지수적 백오프)
- 상태 추적 및 모니터링
- 실패한 작업 재처리 가능
- 트랜잭션 보장

#### 3. 확장성 및 유연성
- 새로운 리소스 타입 쉽게 추가
- 즉시/지연/배치 삭제 지원
- 우선순위 기반 처리 가능
- 다양한 삭제 패턴 지원

#### 4. 모니터링 및 운영
- 삭제 작업 상태 추적 용이
- 실시간 모니터링 가능
- 성능 메트릭 수집
- 장애 격리 효과

#### 5. MSA 원칙 준수
- 서비스 간 느슨한 결합
- 데이터 일관성 보장
- 독립적 배포 가능
- 장애 격리

### 단점 ❌

#### 1. 복잡도 증가
- 새로운 서버 및 인프라 필요
- 추가 개발 및 유지보수 비용
- 더 많은 장애 지점

#### 2. 처리 지연
- 스케줄링으로 인한 지연 시간
- 비동기 처리 특성
- 실시간 응답 불가

#### 3. 운영 부담
- 추가 서버 모니터링 필요
- 더 복잡한 배포 프로세스
- 전문적인 운영 지식 요구

#### 4. 일관성 문제
- 중간 상태에서 불일치 가능
- 최종 일관성 모델 적용
- 복잡한 상태 관리

#### 5. 리소스 사용
- 추가 서버 자원 필요
- 데이터베이스 부하 증가
- 네트워크 트래픽 증가

## 구현 단계

### Phase 1: 기본 Deletion-Server 구축 (2-3주)

#### 목표
- 기본 deletion-server 구축
- service 리소스 타입 삭제 지원
- 단순한 스케줄링 처리

#### 작업 목록
1. **Deletion-Server 프로젝트 생성**
   - NestJS 프로젝트 초기화
   - 기본 모듈 구성
   - Docker 설정

2. **데이터베이스 설정**
   - DeletionQueue 엔티티 생성
   - TypeORM 설정
   - 마이그레이션 스크립트

3. **기본 API 구현**
   - 삭제 요청 수신 API
   - 상태 조회 API
   - 기본 스케줄링 처리

4. **Portal-Server 연동**
   - TCP 클라이언트 설정
   - ServiceManager 수정
   - 기본 삭제 플로우 구현

5. **Authz-Server 연동**
   - 삭제 전용 TCP API 구현
   - 트랜잭션 처리
   - 기본 의존성 정리

#### 검증 기준
- [ ] Service 삭제 요청 정상 처리
- [ ] 의존성 데이터 정상 삭제
- [ ] 실패 시 상태 복구
- [ ] 기본 모니터링 동작

### Phase 2: 고급 기능 추가 (2-3주)

#### 목표
- 재시도 메커니즘 구현
- 지연 삭제 지원
- 모니터링 대시보드

#### 작업 목록
1. **재시도 메커니즘**
   - 지수적 백오프 구현
   - 최대 재시도 횟수 제한
   - 실패 알림 시스템

2. **지연 삭제 기능**
   - 스케줄링 시간 지원
   - 취소 기능 구현
   - 대기 중인 작업 관리

3. **모니터링 시스템**
   - 삭제 작업 대시보드
   - 성능 메트릭 수집
   - 알람 시스템

4. **배치 처리**
   - 다중 삭제 요청 지원
   - 성능 최적화
   - 우선순위 처리

#### 검증 기준
- [ ] 재시도 메커니즘 정상 동작
- [ ] 지연 삭제 정상 처리
- [ ] 모니터링 데이터 정확성
- [ ] 배치 처리 성능 향상

### Phase 3: 확장 및 최적화 (3-4주)

#### 목표
- 다양한 리소스 타입 지원
- 성능 최적화
- 고가용성 보장

#### 작업 목록
1. **리소스 타입 확장**
   - Role 삭제 지원
   - Permission 삭제 지원
   - 커스텀 리소스 타입 지원

2. **성능 최적화**
   - 병렬 처리 구현
   - 커넥션 풀 최적화
   - 쿼리 성능 개선

3. **고가용성**
   - 클러스터링 지원
   - 로드 밸런싱
   - 장애 복구 자동화

4. **통합 테스트**
   - E2E 테스트 작성
   - 성능 테스트
   - 부하 테스트

#### 검증 기준
- [ ] 모든 리소스 타입 지원
- [ ] 성능 요구사항 만족
- [ ] 고가용성 확보
- [ ] 전체 시스템 안정성

## 운영 고려사항

### 1. 모니터링 및 알람

#### 핵심 메트릭
```typescript
// 삭제 작업 처리량
deletion_queue_processed_total{status="completed|failed|cancelled"}

// 처리 시간
deletion_processing_duration_seconds{resource_type="service|role|permission"}

// 큐 크기
deletion_queue_size{status="pending|processing"}

// 재시도 횟수
deletion_retry_count{resource_type="service|role|permission"}
```

#### 알람 조건
- 처리 실패율 > 10%
- 큐 크기 > 1000개
- 평균 처리 시간 > 5분
- 재시도 횟수 > 평균의 2배

### 2. 성능 최적화

#### 데이터베이스 최적화
```sql
-- 인덱스 생성
CREATE INDEX idx_deletion_queue_status_scheduled ON deletion_queue (status, scheduled_at);
CREATE INDEX idx_deletion_queue_resource ON deletion_queue (resource_type, resource_id);
CREATE INDEX idx_deletion_queue_created ON deletion_queue (created_at);

-- 파티셔닝 (월별)
CREATE TABLE deletion_queue_2024_01 PARTITION OF deletion_queue
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

#### 애플리케이션 최적화
```typescript
// 병렬 처리 설정
@Injectable()
export class DeletionService {
  private readonly concurrencyLimit = 5;
  private readonly processingQueue = new PQueue({ concurrency: this.concurrencyLimit });

  async processDeletionQueue(): Promise<void> {
    const pendingItems = await this.deletionQueueRepo.find({
      where: { status: DeletionStatus.PENDING },
      order: { createdAt: 'ASC' },
      take: this.concurrencyLimit * 2
    });

    const promises = pendingItems.map(item => 
      this.processingQueue.add(() => this.processDeletionItem(item))
    );

    await Promise.allSettled(promises);
  }
}
```

### 3. 장애 복구

#### 데이터 정합성 확인
```typescript
// 정합성 검사 스크립트
@Injectable()
export class DeletionConsistencyChecker {
  @Cron('0 0 2 * * *') // 매일 오전 2시
  async checkConsistency(): Promise<void> {
    // 1. PROCESSING 상태가 1시간 이상인 작업 확인
    const stuckItems = await this.deletionQueueRepo.find({
      where: {
        status: DeletionStatus.PROCESSING,
        updatedAt: LessThan(new Date(Date.now() - 60 * 60 * 1000))
      }
    });

    // 2. 상태를 PENDING으로 복구
    for (const item of stuckItems) {
      await this.deletionQueueRepo.update(item.id, {
        status: DeletionStatus.PENDING,
        scheduledAt: new Date()
      });
    }

    // 3. 데이터 정합성 검사
    await this.checkDataConsistency();
  }

  private async checkDataConsistency(): Promise<void> {
    // DELETING 상태인 서비스 중 관련 데이터가 남아있는지 확인
    const deletingServices = await this.serviceRepo.find({
      where: { status: ServiceStatus.DELETING }
    });

    for (const service of deletingServices) {
      const hasRelatedData = await this.checkRelatedData(service.id);
      if (!hasRelatedData) {
        // 관련 데이터가 모두 삭제됐으면 최종 삭제
        await this.serviceRepo.hardDelete(service.id);
      }
    }
  }
}
```

### 4. 보안 고려사항

#### 접근 제어
```typescript
// 삭제 권한 확인
@Injectable()
export class DeletionAuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToRpc().getData();
    
    // 요청자 확인
    const allowedServices = ['portal-server', 'admin-service'];
    return allowedServices.includes(request.requestedBy);
  }
}
```

#### 감사 로그
```typescript
// 삭제 작업 감사 로그
@Injectable()
export class DeletionAuditLogger {
  async logDeletion(item: DeletionQueueEntity, result: 'success' | 'failure'): Promise<void> {
    const auditLog = {
      timestamp: new Date(),
      action: 'deletion',
      resourceType: item.resourceType,
      resourceId: item.resourceId,
      requestedBy: item.requestedBy,
      result,
      metadata: item.metadata
    };

    await this.auditLogRepo.save(auditLog);
  }
}
```

### 5. 백업 및 복구

#### 백업 전략
```typescript
// 삭제 전 백업
@Injectable()
export class DeletionBackupService {
  async createBackup(resourceType: string, resourceId: string): Promise<void> {
    switch (resourceType) {
      case 'service':
        await this.backupServiceData(resourceId);
        break;
      case 'role':
        await this.backupRoleData(resourceId);
        break;
      case 'permission':
        await this.backupPermissionData(resourceId);
        break;
    }
  }

  private async backupServiceData(serviceId: string): Promise<void> {
    // 서비스 및 관련 데이터 백업
    const backupData = await this.gatherServiceRelatedData(serviceId);
    await this.saveBackup('service', serviceId, backupData);
  }
}
```

## 마무리

이 Deletion Service Architecture는 krgeobuk 마이크로서비스 생태계에서 안전하고 확장 가능한 삭제 처리를 위한 종합적인 솔루션입니다. 

### 다음 단계
1. 팀과 아키텍처 검토 및 피드백
2. Phase 1 구현 계획 수립
3. 개발 리소스 할당 및 일정 조정
4. 프로토타입 개발 시작

### 문의사항
이 문서에 대한 질문이나 개선사항이 있으시면 언제든지 논의해주세요.