# krgeobuk-infra 아키텍처 문서

이 문서는 krgeobuk-infra 마이크로서비스 생태계의 전체 아키텍처, 설계 결정, 기술적 세부사항을 설명합니다.

## 목차

1. [시스템 개요](#시스템-개요)
2. [아키텍처 설계](#아키텍처-설계)
3. [서비스 상세](#서비스-상세)
4. [데이터 모델](#데이터-모델)
5. [통신 패턴](#통신-패턴)
6. [보안 아키텍처](#보안-아키텍처)
7. [확장성 및 성능](#확장성-및-성능)
8. [개발 표준](#개발-표준)

## 시스템 개요

### 비즈니스 컨텍스트

krgeobuk-infra는 엔터프라이즈급 웹 애플리케이션을 위한 인증, 권한 관리, 포털 서비스를 제공하는 마이크로서비스 플랫폼입니다.

### 핵심 목표

- **확장성**: 수평 확장이 가능한 아키텍처
- **보안**: 엔터프라이즈급 인증 및 권한 관리
- **유지보수성**: 명확한 책임 분리와 모듈화
- **재사용성**: 공유 라이브러리를 통한 코드 재사용
- **개발 생산성**: 일관된 개발 경험과 도구

### 기술적 제약사항

- Node.js 18 이상 (ES 모듈 지원)
- TypeScript 5.x (완전한 타입 안정성)
- MySQL 8 (JSON 컬럼, CTE 지원)
- Redis 6 (스트림, ACL 지원)

## 아키텍처 설계

### 전체 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Portal Client│  │ Admin Client │  │ Mobile App   │          │
│  │ (Next.js 15) │  │              │  │              │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
└─────────┼──────────────────┼──────────────────┼─────────────────┘
          │                  │                  │
          └──────────────────┴──────────────────┘
                             │
┌─────────────────────────────┼─────────────────────────────────┐
│                   API Gateway / Load Balancer                  │
└─────────────────────────────┼─────────────────────────────────┘
                             │
          ┌──────────────────┴──────────────────┐
          │                                      │
┌─────────▼─────────┐              ┌────────────▼────────────┐
│   auth-server     │              │   authz-server          │
│   (Port 8000)     │◄────────────►│   (Port 8100)           │
│                   │   Service    │                         │
│  - OAuth Login    │   Discovery  │  - RBAC                 │
│  - JWT Tokens     │              │  - Permissions          │
│  - User Auth      │              │  - Role Management      │
└─────────┬─────────┘              └────────────┬────────────┘
          │                                      │
          │                                      │
    ┌─────▼──────┐                        ┌─────▼──────┐
    │ auth-mysql │                        │authz-mysql │
    │ (Port 3307)│                        │(Port 3308) │
    └────────────┘                        └────────────┘
    ┌────────────┐                        ┌────────────┐
    │auth-redis  │                        │authz-redis │
    │(Port 6380) │                        │(Port 6381) │
    └────────────┘                        └────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                   Shared Libraries                           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ @krgeobuk│ │ @krgeobuk│ │ @krgeobuk│ │ @krgeobuk│       │
│  │  /core   │ │  /auth   │ │  /jwt    │ │  /oauth  │  ...  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### 네트워크 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                     Internet / CDN                           │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                    Public Subnet                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │         Load Balancer / API Gateway                  │   │
│  └──────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                   Private Subnet (Application)               │
│                                                               │
│  ┌────────────────┐              ┌────────────────┐         │
│  │  auth-server   │◄────────────►│ authz-server   │         │
│  │                │ msa-network  │                │         │
│  └────────────────┘              └────────────────┘         │
│           │                               │                  │
│           └───────────────┬───────────────┘                  │
│                           │ shared-network                   │
└───────────────────────────┼─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                  Private Subnet (Data)                       │
│                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   MySQL     │  │   MySQL     │  │   Redis     │         │
│  │ (auth-db)   │  │ (authz-db)  │  │  Cluster    │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

### 마이크로서비스 패턴

#### 1. Service Discovery
- **패턴**: DNS 기반 서비스 디스커버리
- **구현**: Docker 네트워크 / Kubernetes Service
- **장점**: 자동 로드 밸런싱, 장애 격리

#### 2. API Gateway
- **패턴**: 단일 진입점
- **구현**: Nginx / Kong / AWS API Gateway
- **역할**: 라우팅, 인증, Rate Limiting

#### 3. Database per Service
- **패턴**: 서비스별 독립 데이터베이스
- **이유**: 느슨한 결합, 독립적 확장
- **트레이드오프**: 분산 트랜잭션 복잡도

#### 4. Shared Library
- **패턴**: 모노레포 기반 공유 라이브러리
- **구현**: pnpm workspace + Verdaccio
- **이점**: 코드 재사용, 일관된 표준

## 서비스 상세

### auth-server (인증 서비스)

#### 책임
- 사용자 인증 (로그인/로그아웃)
- OAuth 2.0 통합 (Google, Naver)
- JWT 토큰 발급 및 검증
- 사용자 세션 관리
- 비밀번호 재설정

#### 기술 스택
- **프레임워크**: NestJS
- **데이터베이스**: MySQL 8 (사용자 정보)
- **캐시**: Redis (세션, 토큰 블랙리스트)
- **인증**: Passport.js, JWT

#### 주요 엔드포인트

| 엔드포인트 | 메서드 | 설명 |
|-----------|--------|------|
| `/auth/login` | POST | 이메일/비밀번호 로그인 |
| `/auth/logout` | POST | 로그아웃 |
| `/auth/refresh` | POST | 토큰 갱신 |
| `/auth/google` | GET | Google OAuth 시작 |
| `/auth/google/callback` | GET | Google OAuth 콜백 |
| `/auth/naver` | GET | Naver OAuth 시작 |
| `/auth/naver/callback` | GET | Naver OAuth 콜백 |
| `/auth/me` | GET | 현재 사용자 정보 |

#### 데이터베이스 스키마

```sql
-- users 테이블
CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255) UNIQUE NOT NULL,
  password VARCHAR(255),  -- OAuth 사용자는 NULL
  name VARCHAR(100) NOT NULL,
  profile_image VARCHAR(500),
  provider VARCHAR(50) DEFAULT 'local',  -- local, google, naver
  provider_id VARCHAR(255),
  is_active BOOLEAN DEFAULT true,
  last_login_at DATETIME,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_email (email),
  INDEX idx_provider (provider, provider_id)
);

-- refresh_tokens 테이블
CREATE TABLE refresh_tokens (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  token VARCHAR(500) NOT NULL,
  expires_at DATETIME NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  INDEX idx_token (token(255)),
  INDEX idx_user_id (user_id)
);
```

#### Redis 데이터 구조

```
# 세션 (해시)
session:{userId} -> {
  "accessToken": "...",
  "refreshToken": "...",
  "expiresAt": "2024-01-01T12:00:00Z"
}
TTL: 7 days

# 토큰 블랙리스트 (문자열)
blacklist:{tokenId} -> "revoked"
TTL: token expiration time

# Rate Limiting (문자열)
ratelimit:{ip}:{endpoint} -> counter
TTL: 60 seconds
```

### authz-server (권한 관리 서비스)

#### 책임
- 역할 기반 접근 제어 (RBAC)
- 권한 검증
- 역할 및 권한 관리
- 서비스 등록 및 관리
- 리소스 접근 제어

#### 기술 스택
- **프레임워크**: NestJS
- **데이터베이스**: MySQL 8 (역할, 권한)
- **캐시**: Redis (권한 캐시)
- **통신**: REST API, 내부는 gRPC/TCP 고려

#### 주요 엔드포인트

| 엔드포인트 | 메서드 | 설명 |
|-----------|--------|------|
| `/authz/check` | POST | 권한 검증 |
| `/authz/roles` | GET | 역할 목록 조회 |
| `/authz/roles` | POST | 역할 생성 |
| `/authz/roles/:id/permissions` | GET | 역할의 권한 조회 |
| `/authz/roles/:id/permissions` | POST | 역할에 권한 추가 |
| `/authz/users/:id/roles` | GET | 사용자 역할 조회 |
| `/authz/users/:id/roles` | POST | 사용자에 역할 할당 |

#### 데이터베이스 스키마

```sql
-- services 테이블
CREATE TABLE services (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100) UNIQUE NOT NULL,
  description TEXT,
  is_active BOOLEAN DEFAULT true,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- roles 테이블
CREATE TABLE roles (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  service_id BIGINT NOT NULL,
  name VARCHAR(100) NOT NULL,
  description TEXT,
  is_system BOOLEAN DEFAULT false,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (service_id) REFERENCES services(id) ON DELETE CASCADE,
  UNIQUE KEY uk_service_role (service_id, name)
);

-- permissions 테이블
CREATE TABLE permissions (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  service_id BIGINT NOT NULL,
  resource VARCHAR(100) NOT NULL,
  action VARCHAR(50) NOT NULL,
  description TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (service_id) REFERENCES services(id) ON DELETE CASCADE,
  UNIQUE KEY uk_permission (service_id, resource, action)
);

-- role_permissions 테이블
CREATE TABLE role_permissions (
  role_id BIGINT NOT NULL,
  permission_id BIGINT NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (role_id, permission_id),
  FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE,
  FOREIGN KEY (permission_id) REFERENCES permissions(id) ON DELETE CASCADE
);

-- user_roles 테이블
CREATE TABLE user_roles (
  user_id BIGINT NOT NULL,
  role_id BIGINT NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (user_id, role_id),
  FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE,
  INDEX idx_user_id (user_id)
);
```

#### Redis 캐시 전략

```
# 사용자 권한 캐시 (Set)
user:{userId}:permissions -> Set<permissionId>
TTL: 1 hour

# 역할 권한 캐시 (Set)
role:{roleId}:permissions -> Set<permissionId>
TTL: 24 hours

# 권한 검증 결과 캐시 (문자열)
authz:{userId}:{resource}:{action} -> "allowed" | "denied"
TTL: 5 minutes
```

### portal-client (포털 클라이언트)

#### 책임
- 사용자 인터페이스
- 관리자 대시보드
- API 통합
- 클라이언트 사이드 라우팅
- SSR/SSG

#### 기술 스택
- **프레임워크**: Next.js 15 (App Router)
- **UI 라이브러리**: React 18
- **스타일링**: Tailwind CSS
- **상태 관리**: React Context / Zustand
- **API 클라이언트**: Axios / SWR

#### 주요 페이지

| 경로 | 설명 | 인증 필요 |
|------|------|-----------|
| `/` | 홈 페이지 | X |
| `/login` | 로그인 | X |
| `/dashboard` | 사용자 대시보드 | O |
| `/admin` | 관리자 페이지 | O (Admin) |
| `/admin/users` | 사용자 관리 | O (Admin) |
| `/admin/roles` | 역할 관리 | O (Admin) |
| `/profile` | 프로필 설정 | O |

## 데이터 모델

### 도메인 모델

```
┌──────────────┐
│     User     │
└──────┬───────┘
       │ 1
       │
       │ *
┌──────▼───────┐      *  ┌──────────────┐
│  UserRole    │─────────│     Role     │
└──────────────┘         └──────┬───────┘
                                │ 1
                                │
                                │ *
                         ┌──────▼─────────────┐
                         │ RolePermission     │
                         └──────┬─────────────┘
                                │ *
                                │
                                │ 1
                         ┌──────▼───────┐
                         │  Permission  │
                         └──────────────┘
```

### 데이터 흐름

#### 인증 플로우

```
1. 사용자 로그인 요청
   Client -> auth-server: POST /auth/login

2. 자격증명 검증
   auth-server -> MySQL: SELECT * FROM users WHERE email = ?
   auth-server -> bcrypt: compare password

3. JWT 토큰 생성
   auth-server: generate access_token (15min)
   auth-server: generate refresh_token (7days)
   auth-server -> Redis: store session

4. 토큰 반환
   auth-server -> Client: { access_token, refresh_token }
```

#### 권한 검증 플로우

```
1. API 요청 (인증 토큰 포함)
   Client -> API: GET /resource (Authorization: Bearer token)

2. 토큰 검증
   API -> auth-server: verify JWT token
   auth-server: decode & validate token

3. 권한 확인
   API -> authz-server: POST /authz/check
   {
     userId: 123,
     resource: "users",
     action: "read"
   }

4. 권한 검증
   authz-server -> Redis: GET user:123:permissions
   (캐시 미스 시)
   authz-server -> MySQL:
     SELECT p.* FROM permissions p
     JOIN role_permissions rp ON p.id = rp.permission_id
     JOIN user_roles ur ON rp.role_id = ur.role_id
     WHERE ur.user_id = 123

5. 결과 반환
   authz-server -> API: { allowed: true }
   API -> Client: 200 OK + resource data
```

## 통신 패턴

### 동기 통신 (REST API)

```typescript
// auth-server에서 authz-server 호출
@Injectable()
export class AuthzService {
  constructor(private httpService: HttpService) {}

  async checkPermission(
    userId: number,
    resource: string,
    action: string,
  ): Promise<boolean> {
    const response = await this.httpService
      .post(`${AUTHZ_SERVICE_URL}/authz/check`, {
        userId,
        resource,
        action,
      })
      .toPromise();

    return response.data.allowed;
  }
}
```

### 비동기 통신 (이벤트 기반)

```typescript
// 사용자 생성 이벤트 발행
@Injectable()
export class UserService {
  constructor(private eventEmitter: EventEmitter2) {}

  async createUser(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepository.save(dto);

    // 이벤트 발행
    this.eventEmitter.emit('user.created', {
      userId: user.id,
      email: user.email,
      timestamp: new Date(),
    });

    return user;
  }
}

// 다른 서비스에서 이벤트 구독
@Injectable()
export class NotificationService {
  @OnEvent('user.created')
  async handleUserCreated(payload: UserCreatedEvent) {
    // 환영 이메일 발송
    await this.sendWelcomeEmail(payload.email);
  }
}
```

## 보안 아키텍처

### 인증 메커니즘

#### JWT 토큰 구조

```json
{
  "header": {
    "alg": "HS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "user_id",
    "email": "user@example.com",
    "iat": 1640000000,
    "exp": 1640003600,
    "jti": "token_unique_id"
  }
}
```

#### OAuth 2.0 플로우

```
1. Authorization Request
   Client -> auth-server: GET /auth/google

2. Redirect to Provider
   auth-server -> Client: 302 Redirect
   Location: https://accounts.google.com/oauth/authorize?...

3. User Authorization
   Client -> Google: Login & Consent

4. Authorization Code
   Google -> auth-server: GET /auth/google/callback?code=...

5. Token Exchange
   auth-server -> Google: POST /oauth/token
   { code, client_id, client_secret }

6. User Profile
   auth-server -> Google: GET /oauth/userinfo
   Authorization: Bearer access_token

7. JWT Generation
   auth-server: Create/Update User
   auth-server: Generate JWT tokens

8. Response
   auth-server -> Client: 302 Redirect
   Location: /dashboard?token=...
```

### 권한 모델 (RBAC)

#### 권한 계층

```
System Admin
  └─ Service Admin
      └─ Manager
          └─ User
              └─ Guest
```

#### 권한 정의

```typescript
// 권한 형식: service:resource:action
const permissions = [
  'auth:users:create',
  'auth:users:read',
  'auth:users:update',
  'auth:users:delete',
  'authz:roles:create',
  'authz:roles:read',
  'authz:roles:update',
  'authz:roles:delete',
  'portal:dashboard:view',
];
```

### 보안 모범 사례

#### 1. 비밀번호 보안
- **해싱**: bcrypt (cost factor 10)
- **정책**: 최소 8자, 대소문자, 숫자, 특수문자
- **재설정**: 이메일 인증 링크 (1시간 유효)

#### 2. 토큰 보안
- **Access Token**: 15분 유효기간
- **Refresh Token**: 7일 유효기간
- **Rotation**: Refresh Token은 사용 후 재발급
- **Blacklist**: 로그아웃 시 토큰 블랙리스트 추가

#### 3. API 보안
- **HTTPS**: 모든 통신 암호화
- **CORS**: 허용된 도메인만 접근
- **Rate Limiting**: IP당 분당 100 요청
- **Input Validation**: DTO + class-validator

#### 4. 데이터베이스 보안
- **암호화**: 민감 정보 AES-256 암호화
- **최소 권한**: 서비스별 DB 계정 분리
- **감사 로그**: 중요 작업 기록

## 확장성 및 성능

### 수평 확장

#### Stateless 서비스
- 모든 상태는 Redis/데이터베이스에 저장
- 인스턴스 간 세션 공유

#### 로드 밸런싱
- **알고리즘**: Round Robin / Least Connections
- **Health Check**: `/health` 엔드포인트
- **Sticky Session**: 불필요 (Stateless)

### 캐싱 전략

#### 다층 캐싱

```
Client (Browser Cache)
  └─ CDN Cache
      └─ Application Cache (Redis)
          └─ Database Query Cache
              └─ Database
```

#### Redis 캐싱 패턴

```typescript
// Cache-Aside 패턴
async getUserPermissions(userId: number): Promise<Permission[]> {
  // 1. 캐시 확인
  const cached = await this.redis.get(`user:${userId}:permissions`);
  if (cached) {
    return JSON.parse(cached);
  }

  // 2. 데이터베이스 조회
  const permissions = await this.permissionRepository.find({
    where: { userId },
  });

  // 3. 캐시 저장 (1시간)
  await this.redis.setex(
    `user:${userId}:permissions`,
    3600,
    JSON.stringify(permissions),
  );

  return permissions;
}
```

### 데이터베이스 최적화

#### 인덱싱 전략
- **Primary Key**: AUTO_INCREMENT BIGINT
- **Foreign Key**: 모든 외래키에 인덱스
- **복합 인덱스**: WHERE 절에 자주 사용되는 컬럼 조합
- **커버링 인덱스**: SELECT 절 컬럼 포함

#### 쿼리 최적화
- **N+1 문제**: JOIN 또는 DataLoader 사용
- **Pagination**: LIMIT/OFFSET 또는 Cursor 기반
- **Batch Operations**: INSERT/UPDATE 배치 처리

#### Connection Pooling

```typescript
// TypeORM 설정
{
  type: 'mysql',
  poolSize: 20,
  extra: {
    connectionLimit: 20,
    queueLimit: 0,
    waitForConnections: true,
  },
}
```

### 성능 목표

| 메트릭 | 목표 |
|--------|------|
| API 응답 시간 (p50) | < 100ms |
| API 응답 시간 (p95) | < 300ms |
| API 응답 시간 (p99) | < 1s |
| 처리량 | > 1000 RPS per instance |
| 가용성 | 99.9% (월 43분 다운타임) |
| 에러율 | < 0.1% |

## 개발 표준

### 코딩 컨벤션

#### TypeScript

```typescript
// ✅ Good
interface UserDto {
  id: number;
  email: string;
  name: string;
}

class UserService {
  async findById(id: number): Promise<User> {
    // ...
  }
}

// ❌ Bad
interface user_dto {
  Id: number;
  Email: string;
  Name: string;
}

class userService {
  async FindById(Id: number): Promise<any> {
    // ...
  }
}
```

#### 파일 구조

```
src/
├── modules/
│   ├── users/
│   │   ├── dto/
│   │   │   ├── create-user.dto.ts
│   │   │   └── update-user.dto.ts
│   │   ├── entities/
│   │   │   └── user.entity.ts
│   │   ├── users.controller.ts
│   │   ├── users.service.ts
│   │   ├── users.repository.ts
│   │   └── users.module.ts
├── common/
│   ├── decorators/
│   ├── filters/
│   ├── guards/
│   ├── interceptors/
│   └── pipes/
├── config/
│   ├── database.config.ts
│   └── redis.config.ts
└── main.ts
```

### API 응답 포맷

#### 성공 응답

```typescript
{
  "success": true,
  "data": {
    "id": 1,
    "email": "user@example.com",
    "name": "John Doe"
  },
  "message": "사용자 조회 성공"
}
```

#### 에러 응답

```typescript
{
  "success": false,
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "사용자를 찾을 수 없습니다",
    "details": {
      "userId": 999
    }
  },
  "timestamp": "2024-01-01T12:00:00.000Z",
  "path": "/api/users/999"
}
```

### 테스트 전략

#### 단위 테스트 (Unit Test)
- **대상**: Service, Util 함수
- **도구**: Jest
- **커버리지**: > 80%

#### 통합 테스트 (Integration Test)
- **대상**: Controller, Repository
- **도구**: Jest + Supertest
- **범위**: API 엔드포인트 전체

#### E2E 테스트
- **대상**: 주요 사용자 플로우
- **도구**: Jest + Test Database
- **시나리오**: 로그인 → 권한 검증 → API 호출

## 참고 문서

- **[README.md](../README.md)** - 프로젝트 개요
- **[QUICKSTART.md](../QUICKSTART.md)** - 빠른 시작 가이드
- **[CLAUDE.md](../CLAUDE.md)** - AI 어시스턴트용 개발 가이드
- **[scripts/DEPLOYMENT.md](../scripts/DEPLOYMENT.md)** - 배포 가이드

### 외부 참고 자료

- [NestJS Documentation](https://docs.nestjs.com)
- [Next.js Documentation](https://nextjs.org/docs)
- [TypeORM Documentation](https://typeorm.io)
- [Redis Documentation](https://redis.io/docs)
- [Docker Documentation](https://docs.docker.com)

## 용어집

| 용어 | 설명 |
|------|------|
| **RBAC** | Role-Based Access Control (역할 기반 접근 제어) |
| **JWT** | JSON Web Token |
| **OAuth** | Open Authorization |
| **MSA** | Microservice Architecture |
| **DTO** | Data Transfer Object |
| **ORM** | Object-Relational Mapping |
| **SSR** | Server-Side Rendering |
| **SSG** | Static Site Generation |
| **HPA** | Horizontal Pod Autoscaler |
| **CDN** | Content Delivery Network |

## 변경 이력

| 버전 | 날짜 | 변경 내용 |
|------|------|-----------|
| 1.0.0 | 2024-01-01 | 초기 아키텍처 문서 작성 |
