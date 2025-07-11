# 공통 패키지 하이브리드 트리 쉐이킹 마이그레이션 계획

## 개요

krgeobuk 생태계의 모든 공통 패키지를 하이브리드 트리 쉐이킹 방식으로 최적화하여 번들 크기를 최소화하고 개발 경험을 향상시키는 계획입니다.

## 트리 쉐이킹 전략

### 1. 기능 레벨 트리 쉐이킹
**대상**: 복잡한 패키지 (5개 이상 모듈)
**특징**: 각 기능별로 세분화된 import path 제공

### 2. 도메인 레벨 트리 쉐이킹  
**대상**: 단순한 패키지 (5개 이하 모듈)
**특징**: 도메인별 통합 import path 제공

## 패키지별 마이그레이션 계획

### 🔴 우선순위 1: 기능 레벨 트리 쉐이킹

#### 1. @krgeobuk/user
**현재 상태**: 기능 레벨 적용 중, types 필드 누락
**모듈 수**: 6개 (codes, decorators, dtos, exception, interfaces, messages, response)
**작업 내용**:
- [ ] package.json exports에 types 필드 추가
- [ ] typesVersions 정리 (불필요한 매핑 제거)
- [ ] 테스트 및 검증

```json
{
  "exports": {
    ".": "./dist/index.js",
    "./decorators": {
      "import": "./dist/decorators/index.js",
      "types": "./dist/decorators/index.d.ts"
    },
    "./dtos": {
      "import": "./dist/dtos/index.js",
      "types": "./dist/dtos/index.d.ts"
    },
    "./exception": {
      "import": "./dist/exception/index.js",
      "types": "./dist/exception/index.d.ts"
    },
    "./interfaces": {
      "import": "./dist/interfaces/index.js",
      "types": "./dist/interfaces/index.d.ts"
    },
    "./response": {
      "import": "./dist/response/index.js",
      "types": "./dist/response/index.d.ts"
    }
  }
}
```

#### 2. @krgeobuk/core
**현재 상태**: 단순한 string exports
**모듈 수**: 10개+ (codes, decorators, dtos, entities, enum, exception, filters, guards, interceptors, interfaces, logger, messages, repositories, response, utils)
**작업 내용**:
- [ ] 현재 구조 분석 및 주요 모듈 식별
- [ ] 기능 레벨 exports 설계
- [ ] package.json 업데이트
- [ ] 의존성 패키지 호환성 검증

```json
{
  "exports": {
    ".": "./dist/index.js",
    "./decorators": {
      "import": "./dist/decorators/index.js",
      "types": "./dist/decorators/index.d.ts"
    },
    "./dtos": {
      "import": "./dist/dtos/index.js",
      "types": "./dist/dtos/index.d.ts"
    },
    "./interceptors": {
      "import": "./dist/interceptors/index.js",
      "types": "./dist/interceptors/index.d.ts"
    },
    "./logger": {
      "import": "./dist/logger/index.js",
      "types": "./dist/logger/index.d.ts"
    }
  }
}
```

#### 3. @krgeobuk/role
**현재 상태**: 단순한 string exports
**모듈 수**: 6개 (codes, decorators, dtos, exception, interfaces, messages, response)
**작업 내용**:
- [ ] user 패키지 패턴 적용
- [ ] package.json 업데이트
- [ ] 테스트 및 검증

#### 4. @krgeobuk/permission
**현재 상태**: 단순한 string exports  
**모듈 수**: 6개 (codes, decorators, dtos, exception, interfaces, messages, response)
**작업 내용**:
- [ ] user 패키지 패턴 적용
- [ ] package.json 업데이트
- [ ] 테스트 및 검증

#### 5. @krgeobuk/service
**현재 상태**: 단순한 string exports
**모듈 수**: 6개 (codes, decorators, dtos, exception, interfaces, messages, response)
**작업 내용**:
- [ ] user 패키지 패턴 적용
- [ ] package.json 업데이트
- [ ] 테스트 및 검증

#### 6. @krgeobuk/auth
**현재 상태**: 단순한 string exports
**모듈 수**: 6개 (codes, decorators, dtos, exception, interfaces, messages, response)
**작업 내용**:
- [ ] user 패키지 패턴 적용
- [ ] package.json 업데이트
- [ ] 테스트 및 검증

#### 7. @krgeobuk/oauth
**현재 상태**: 단순한 string exports
**모듈 수**: 6개 (codes, decorators, dtos, enum, exception, interfaces, messages, response)
**작업 내용**:
- [ ] user 패키지 패턴 적용
- [ ] package.json 업데이트
- [ ] 테스트 및 검증

#### 8. @krgeobuk/swagger
**현재 상태**: 단순한 string exports
**모듈 수**: 4개 (config, constants, decorators, interface)
**작업 내용**:
- [ ] 도메인 레벨 vs 기능 레벨 재검토
- [ ] package.json 업데이트
- [ ] 테스트 및 검증

### 🟡 우선순위 2: 도메인 레벨 트리 쉐이킹

#### 1. @krgeobuk/shared
**현재 상태**: 도메인 레벨 적용 완료
**모듈 수**: 5개 도메인 (oauth, permission, role, service, user)
**작업 내용**:
- [x] 이미 최적화 완료
- [ ] 문서화 및 베스트 프랙티스 추출

#### 2. @krgeobuk/tsconfig
**현재 상태**: 도메인 레벨 적용 완료
**모듈 수**: 3개 (base, nest, next)
**작업 내용**:
- [x] 이미 최적화 완료
- [ ] 문서화

#### 3. @krgeobuk/jest-config
**현재 상태**: 도메인 레벨 적용 완료
**모듈 수**: 3개 (base, nest, next)
**작업 내용**:
- [x] 이미 최적화 완료
- [ ] 문서화

#### 4. @krgeobuk/eslint-config
**현재 상태**: 도메인 레벨 적용 완료
**모듈 수**: 3개 (base, nest, next)
**작업 내용**:
- [x] 이미 최적화 완료
- [ ] 문서화

#### 5. @krgeobuk/database
**현재 상태**: 단순한 구조
**모듈 수**: 3개 (constants, redis, typeorm)
**작업 내용**:
- [ ] 현재 구조 분석
- [ ] 도메인 레벨 적용 검토
- [ ] package.json 업데이트

### 🟢 우선순위 3: 새로운 패키지

#### 1. @krgeobuk/authz-relations
**현재 상태**: 기능 레벨 적용 완료
**모듈 수**: 5개 (role-permission 하위)
**작업 내용**:
- [x] 이미 최적화 완료
- [ ] 문서화 및 베스트 프랙티스 추출

## 마이그레이션 단계

### Phase 1: 기획 및 분석 (1주)
- [ ] 모든 패키지 현재 상태 상세 분석
- [ ] 의존성 관계 매핑
- [ ] 호환성 이슈 사전 식별
- [ ] 테스트 계획 수립

### Phase 2: 핵심 패키지 마이그레이션 (2주)
- [ ] @krgeobuk/user 완성
- [ ] @krgeobuk/core 마이그레이션
- [ ] @krgeobuk/role 마이그레이션
- [ ] @krgeobuk/permission 마이그레이션
- [ ] 각 단계별 테스트 및 검증

### Phase 3: 나머지 패키지 마이그레이션 (2주)
- [ ] @krgeobuk/service 마이그레이션
- [ ] @krgeobuk/auth 마이그레이션
- [ ] @krgeobuk/oauth 마이그레이션
- [ ] @krgeobuk/swagger 마이그레이션
- [ ] @krgeobuk/database 마이그레이션

### Phase 4: 통합 테스트 및 문서화 (1주)
- [ ] 전체 생태계 호환성 테스트
- [ ] 성능 벤치마크 측정
- [ ] 마이그레이션 가이드 문서 작성
- [ ] 베스트 프랙티스 정리

## 성공 기준

### 기술적 목표
- [ ] 모든 패키지 번들 크기 20% 이상 감소
- [ ] TypeScript 지원 강화 (types 필드 추가)
- [ ] 트리 쉐이킹 효율성 95% 이상
- [ ] 빌드 시간 단축

### 개발 경험 목표
- [ ] 일관된 import 패턴 구축
- [ ] IDE 자동완성 개선
- [ ] 문서화 완성도 향상
- [ ] 개발자 가이드 정립

## 리스크 관리

### 주요 리스크
1. **호환성 이슈**: 기존 프로젝트와의 breaking change
2. **의존성 문제**: 순환 의존성 및 버전 충돌
3. **성능 저하**: 잘못된 설정으로 인한 번들 크기 증가
4. **개발 지연**: 예상보다 복잡한 마이그레이션 과정

### 대응 방안
- [ ] 점진적 마이그레이션 (패키지별 단계적 적용)
- [ ] 철저한 테스트 (단위, 통합, E2E)
- [ ] 롤백 계획 수립
- [ ] 지속적인 모니터링

## 체크리스트 템플릿

각 패키지 마이그레이션 시 사용할 체크리스트:

### 마이그레이션 전 체크리스트
- [ ] 현재 package.json exports 구조 분석
- [ ] 의존성 패키지 호환성 확인
- [ ] 사용 패턴 분석 (어떤 모듈이 주로 사용되는지)
- [ ] 테스트 환경 구성

### 마이그레이션 중 체크리스트
- [ ] package.json exports 업데이트
- [ ] typesVersions 정리
- [ ] types 필드 추가
- [ ] sideEffects: false 확인
- [ ] 빌드 테스트
- [ ] 타입 검사 통과

### 마이그레이션 후 체크리스트
- [ ] 번들 크기 측정 및 비교
- [ ] 의존성 패키지 테스트
- [ ] E2E 테스트 통과
- [ ] 문서 업데이트
- [ ] 커밋 및 태그

## 문서화 계획

### 업데이트 대상 문서
- [ ] 루트 CLAUDE.md - 하이브리드 트리 쉐이킹 가이드 보완
- [ ] 각 패키지 README.md - 새로운 import 패턴 안내
- [ ] 마이그레이션 가이드 - 다른 프로젝트 적용 시 참고용
- [ ] 베스트 프랙티스 문서 - 향후 패키지 개발 시 참고용

---

## 실행 명령어

이 계획을 실행할 때 사용할 주요 명령어:

```bash
# 1. 패키지 분석
pnpm --filter @krgeobuk/user ls

# 2. 빌드 테스트
pnpm --filter @krgeobuk/user build

# 3. 의존성 테스트
pnpm --filter @krgeobuk/user test

# 4. 번들 크기 측정
pnpm --filter @krgeobuk/user bundle-analyzer

# 5. 전체 생태계 테스트
pnpm build && pnpm test
```

이 계획서를 기반으로 단계별로 마이그레이션을 진행하여 krgeobuk 생태계의 번들 최적화를 완성하겠습니다.