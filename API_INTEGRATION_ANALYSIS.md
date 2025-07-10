# Portal-Client API 연동 분석 결과

> 분석 일자: 2025-01-09
> 분석 대상: portal-client ↔ auth-server, authz-server API 매칭

## 📋 전체 분석 요약

### 현재 상황
- **즉시 사용 가능**: **약 80%**
- **형태 변경 필요**: **약 10%** (클라이언트에서 처리 가능)
- **신규 개발 필요**: **약 10%** (주로 서비스 관리, OAuth 클라이언트 관리)

---

## ✅ 완전히 사용 가능한 기능

### 1. 인증 관리 (auth-server:8000)
- ✅ 로그인: `POST /auth/login`
- ✅ 회원가입: `POST /auth/signup` 
- ✅ 로그아웃: `POST /auth/logout`
- ✅ 토큰 갱신: `POST /auth/refresh`
- ✅ 현재 사용자 정보: `GET /users/me`

### 2. 사용자 관리 (auth-server:8000)
- ✅ 사용자 목록/검색: `GET /users` (페이지네이션, 검색 지원)
- ✅ 사용자 상세 조회: `GET /users/:id`
- ✅ 사용자 정보 수정: `PATCH /users/me`
- ✅ 사용자 삭제: `DELETE /users/me`

### 3. 역할 관리 CRUD (authz-server:8100)
- ✅ 역할 목록: `GET /roles`
- ✅ 역할 상세: `GET /roles/:id`
- ✅ 역할 생성: `POST /roles`
- ✅ 역할 수정: `PATCH /roles/:id`
- ✅ 역할 삭제: `DELETE /roles/:id`

### 4. 권한 관리 CRUD (authz-server:8100)
- ✅ 권한 목록: `GET /permissions`
- ✅ 권한 상세: `GET /permissions/:id`
- ✅ 권한 생성: `POST /permissions`
- ✅ 권한 수정: `PATCH /permissions/:id`
- ✅ 권한 삭제: `DELETE /permissions/:id`

### 5. 역할-권한 관계 관리 (authz-server:8100)
- ✅ 역할의 권한 조회: `GET /role-permissions/roles/:roleId`
- ✅ 권한을 역할에 할당: `POST /role-permissions`
- ✅ 역할에서 권한 제거: `DELETE /role-permissions/roles/:roleId/permissions/:permissionId`

### 6. 사용자-역할 관계 관리 (authz-server:8100)
- ✅ 사용자의 역할 조회: `GET /user-roles/users/:userId`
- ✅ 사용자에게 역할 할당: `POST /user-roles`
- ✅ 사용자에서 역할 제거: `DELETE /user-roles/users/:userId/roles/:roleId`

### 7. OAuth 기본 기능 (auth-server:8000)
- ✅ Google OAuth: `GET /oauth/login-google`
- ✅ Naver OAuth: `GET /oauth/login-naver`
- ✅ OAuth 콜백 처리

---

## 🔄 기능은 있지만 형태가 다른 것

### 1. 권한 그룹화
- **Portal-client 요구**: 서비스별/리소스별 그룹화된 권한 조회
- **현재 가능**: `GET /permissions` + 클라이언트에서 그룹화 처리
- **해결방법**: 프론트엔드에서 그룹화 로직 구현

### 2. 벌크 역할 할당
- **Portal-client 요구**: 한 번에 여러 역할 할당
- **현재 가능**: `POST /user-roles` 여러 번 호출
- **해결방법**: 클라이언트에서 순차 호출 또는 백엔드에 벌크 API 추가

### 3. 벌크 권한 할당
- **Portal-client 요구**: 한 번에 여러 권한 할당
- **현재 가능**: `POST /role-permissions` 여러 번 호출
- **해결방법**: 클라이언트에서 순차 호출 또는 백엔드에 벌크 API 추가

---

## ❌ 완전히 없는 기능 (신규 개발 필요)

### 1. 서비스 관리 (우선순위: 높음)
- ❌ 서비스 CRUD 관리
- ❌ 서비스 가시성 설정
- ❌ 서비스 헬스체크
- ❌ 공개 서비스 목록

**개발 위치**: portal-server 또는 새로운 service-server 구축

### 2. OAuth 클라이언트 관리 (우선순위: 중간)
- ❌ OAuth 클라이언트 CRUD
- ❌ 클라이언트 시크릿 재생성
- ❌ 토큰 관리/취소
- ❌ OAuth 통계

**개발 위치**: auth-server 확장

### 3. 관리자 전용 사용자 관리 (우선순위: 낮음)
- ❌ 관리자가 다른 사용자 생성
- ❌ 관리자가 다른 사용자 수정/삭제

**개발 위치**: auth-server 확장

---

## 🎯 단계별 개발 계획

### Phase 1: 즉시 연동 (현재 가능)
- 인증/로그인 시스템
- 사용자 기본 관리
- 역할/권한 CRUD
- 역할-권한, 사용자-역할 관계 관리

### Phase 2: 형태 조정 (1-2주)
- 프론트엔드에서 권한 그룹화 구현
- 벌크 오퍼레이션을 순차 호출로 처리
- API 경로 매핑 조정

### Phase 3: 서비스 관리 구현 (2-4주)
- 서비스 엔티티 및 CRUD API 개발
- 서비스 가시성 로직 구현
- 헬스체크 시스템 구축

### Phase 4: OAuth 관리 확장 (4-6주)
- OAuth 클라이언트 관리 시스템
- 토큰 관리 및 통계 기능
- 고급 관리자 기능

---

## 🔧 API 매핑 가이드

### portal-client → 실제 서버 매핑

#### 인증 관련
```
/api/auth/login     → POST /auth/login
/api/auth/register  → POST /auth/signup
/api/auth/logout    → POST /auth/logout
/api/auth/refresh   → POST /auth/refresh
/api/auth/me        → GET /users/me
```

#### 사용자 관리
```
/api/admin/users           → GET /users
/api/admin/users/:id       → GET /users/:id
/api/admin/users/:id/roles → GET /user-roles/users/:id
```

#### 역할 관리
```
/api/admin/roles                    → GET /roles
/api/admin/roles/:id                → GET /roles/:id
/api/admin/roles/:id/permissions    → GET /role-permissions/roles/:id
```

#### 권한 관리
```
/api/admin/permissions         → GET /permissions
/api/admin/permissions/:id     → GET /permissions/:id
/api/admin/permissions/grouped → GET /permissions + 클라이언트 그룹화
```

---

## 📝 주의사항

1. **응답 포맷**: 현재 서버의 SerializerInterceptor 포맷과 portal-client 예상 포맷이 다를 수 있음
2. **인증 방식**: JWT 토큰 기반으로 통일되어 있음
3. **페이지네이션**: 모든 목록 API에서 지원됨
4. **권한 체크**: 모든 관리 API에 AccessTokenGuard 적용됨

---

## 🚀 다음 단계

1. **즉시 실행**: 기존 API를 활용한 기본 기능 연동
2. **단기 목표**: 서비스 관리 API 개발
3. **중기 목표**: OAuth 클라이언트 관리 시스템
4. **장기 목표**: 통합 관리 대시보드 완성

---

*마지막 업데이트: 2025-01-09*
*분석 담당: Claude Code*