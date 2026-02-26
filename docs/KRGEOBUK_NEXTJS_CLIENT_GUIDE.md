# KRGEOBUK Next.js 클라이언트 개발 가이드

이 문서는 krgeobuk 생태계에서 Next.js 15 및 React 18 기반 클라이언트 개발의 표준을 제시합니다.

## 프로젝트 개요

### 기술 스택
- **Next.js 15** - App Router 기반 React 프레임워크  
- **TypeScript** - 엄격 모드가 활성화된 완전한 타입 안전성
- **Tailwind CSS** - 유틸리티 우선 CSS 프레임워크
- **React Hook Form** - 성능 최적화된 폼 관리
- **Redux Toolkit** - 상태 관리
- **Axios** - HTTP 클라이언트

### 핵심 명령어

```bash
# 개발 서버 시작
npm run dev                # Next.js 개발 서버 (포트 3000)

# 빌드
npm run build              # 프로덕션 빌드
npm run start              # 프로덕션 서버 시작

# 타입 검사
npm run type-check         # TypeScript 타입 검사

# 코드 품질
npm run lint               # ESLint 실행
npm run lint:fix           # 자동 수정과 함께 린팅
npm run format             # Prettier 포맷팅
```

### 경로 별칭
```typescript
// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

---

# 🏗️ 아키텍처 구조

## 디렉터리 구조 표준
```
src/
├── app/                    # Next.js 15 App Router
│   ├── auth/              # 인증 관련 페이지
│   ├── admin/             # 관리자 인터페이스
│   ├── layout.tsx         # 루트 레이아웃
│   └── page.tsx           # 홈페이지
├── components/            # 재사용 가능한 컴포넌트
│   ├── common/           # 공통 UI 컴포넌트
│   ├── forms/            # 폼 컴포넌트
│   └── layout/           # 레이아웃 컴포넌트
├── hooks/                # 커스텀 훅
├── store/                # Redux 상태 관리
├── services/             # API 서비스 레이어
├── types/                # TypeScript 타입 정의
├── utils/                # 유틸리티 함수
├── context/              # React Context
└── middleware.ts         # Next.js 미들웨어
```

# 🌐 krgeobuk 생태계 통합

## 서비스 통합 아키텍처
portal-client는 krgeobuk 마이크로서비스 생태계의 통합 관리 인터페이스입니다:

### 서비스 포트 매핑
```typescript
// src/config/services.ts
export const KRGEOBUK_SERVICES = {
  AUTH_SERVER: {
    port: 8000,
    baseUrl: process.env.NEXT_PUBLIC_AUTH_SERVER_URL || 'http://localhost:8000',
    endpoints: {
      login: '/auth/login',
      signup: '/auth/signup',
      refresh: '/auth/refresh',
      logout: '/auth/logout',
      users: '/users',
      oauth: '/oauth'
    }
  },
  AUTHZ_SERVER: {
    port: 8100,
    baseUrl: process.env.NEXT_PUBLIC_AUTHZ_SERVER_URL || 'http://localhost:8100',
    endpoints: {
      roles: '/roles',
      permissions: '/permissions',
      rolePermissions: '/role-permissions',
      userRoles: '/user-roles'
    }
  },
  PORTAL_SERVER: {
    port: 8200,
    baseUrl: process.env.NEXT_PUBLIC_PORTAL_SERVER_URL || 'http://localhost:8200',
    endpoints: {
      services: '/services',
      integrations: '/integrations'
    }
  }
} as const;
```

## 공유 라이브러리 통합 패턴

### @krgeobuk 패키지 활용
```typescript
// 공유 타입 정의 활용
import type { 
  LoggedInUser,
  CreateUserDto,
  UpdateUserDto 
} from '@krgeobuk/shared/user';
import type { 
  Role,
  Permission,
  CreateRoleDto 
} from '@krgeobuk/shared/auth';
import type { ApiResponse } from '@krgeobuk/shared/api';

// 공유 유틸리티 활용
import { validateEmail, formatPhoneNumber } from '@krgeobuk/shared/utils';
import { UserStatus, RoleType } from '@krgeobuk/shared/enums';

// 폼 검증에서 공유 유틸 사용
const emailValidation = {
  required: '이메일을 입력해주세요',
  validate: (value: string) => 
    validateEmail(value) || '올바른 이메일 형식을 입력해주세요'
};
```

### 공유 컴포넌트 활용
```typescript
// src/components/shared/UserStatusBadge.tsx
import { UserStatus } from '@krgeobuk/shared/enums';
import { cn } from '@/utils/cn';

interface UserStatusBadgeProps {
  status: UserStatus;
  className?: string;
}

export function UserStatusBadge({ status, className }: UserStatusBadgeProps) {
  const statusConfig = {
    [UserStatus.ACTIVE]: {
      label: '활성',
      className: 'bg-green-100 text-green-800 border-green-200'
    },
    [UserStatus.INACTIVE]: {
      label: '비활성',
      className: 'bg-gray-100 text-gray-800 border-gray-200'
    },
    [UserStatus.SUSPENDED]: {
      label: '정지',
      className: 'bg-red-100 text-red-800 border-red-200'
    }
  };

  const config = statusConfig[status];

  return (
    <span className={cn(
      'inline-flex items-center px-2 py-1 rounded-full text-xs font-medium border',
      config.className,
      className
    )}>
      {config.label}
    </span>
  );
}
```

## 크로스 서비스 통신 패턴

### 멀티 서비스 API 클라이언트
```typescript
// src/lib/apiClients.ts
import axios, { AxiosInstance } from 'axios';
import { KRGEOBUK_SERVICES } from '@/config/services';
import { tokenManager } from './tokenManager';

class KrgeobukApiClient {
  public auth: AxiosInstance;
  public authz: AxiosInstance;
  public portal: AxiosInstance;

  constructor() {
    this.auth = this.createClient(KRGEOBUK_SERVICES.AUTH_SERVER.baseUrl);
    this.authz = this.createClient(KRGEOBUK_SERVICES.AUTHZ_SERVER.baseUrl);
    this.portal = this.createClient(KRGEOBUK_SERVICES.PORTAL_SERVER.baseUrl);
  }

  private createClient(baseURL: string): AxiosInstance {
    const client = axios.create({
      baseURL,
      timeout: 10000,
      headers: {
        'Content-Type': 'application/json',
      },
    });

    // 공통 인터셉터 설정
    client.interceptors.request.use(
      (config) => {
        const token = tokenManager.getAccessToken();
        if (token) {
          config.headers.Authorization = `Bearer ${token}`;
        }
        return config;
      },
      (error) => Promise.reject(error)
    );

    client.interceptors.response.use(
      (response) => response,
      async (error) => {
        const originalRequest = error.config;

        if (error.response?.status === 401 && !originalRequest._retry) {
          originalRequest._retry = true;

          try {
            await tokenManager.refreshToken();
            const newToken = tokenManager.getAccessToken();
            if (newToken) {
              originalRequest.headers.Authorization = `Bearer ${newToken}`;
              return client(originalRequest);
            }
          } catch (refreshError) {
            tokenManager.clearTokens();
            window.location.href = '/auth/login';
          }
        }

        return Promise.reject(error);
      }
    );

    return client;
  }
}

export const apiClients = new KrgeobukApiClient();
```

### 통합 서비스 레이어
```typescript
// src/services/krgeobukIntegrationService.ts
import { apiClients } from '@/lib/apiClients';
import type { User, Role, Permission, Service } from '@/types';
import type { ApiResponse } from '@krgeobuk/shared/api';

export class KrgeobukIntegrationService {
  // 사용자와 역할을 함께 조회
  async getUserWithRoles(userId: string): Promise<{
    user: User;
    roles: Role[];
  }> {
    const [userResponse, rolesResponse] = await Promise.all([
      apiClients.auth.get<ApiResponse<User>>(`/users/${userId}`),
      apiClients.authz.get<ApiResponse<Role[]>>(`/users/${userId}/roles`)
    ]);

    return {
      user: userResponse.data.data,
      roles: rolesResponse.data.data
    };
  }

  // 역할에 권한 할당
  async assignPermissionsToRole(roleId: string, permissionIds: string[]): Promise<void> {
    await apiClients.authz.post(`/roles/${roleId}/permissions`, {
      permissionIds
    });
  }

  // 서비스별 권한 매트릭스 조회
  async getServicePermissionMatrix(): Promise<{
    services: Service[];
    permissions: Permission[];
    matrix: Record<string, string[]>;
  }> {
    const [servicesResponse, permissionsResponse] = await Promise.all([
      apiClients.portal.get<ApiResponse<Service[]>>('/services'),
      apiClients.authz.get<ApiResponse<Permission[]>>('/permissions')
    ]);

    const services = servicesResponse.data.data;
    const permissions = permissionsResponse.data.data;
    
    // 서비스별 권한 매핑 생성
    const matrix = services.reduce((acc, service) => {
      acc[service.id] = permissions
        .filter(perm => perm.serviceId === service.id)
        .map(perm => perm.id);
      return acc;
    }, {} as Record<string, string[]>);

    return { services, permissions, matrix };
  }

  // 사용자의 특정 서비스 접근 권한 확인
  async checkUserServiceAccess(userId: string, serviceId: string): Promise<{
    hasAccess: boolean;
    permissions: string[];
  }> {
    const response = await apiClients.authz.get<ApiResponse<{
      hasAccess: boolean;
      permissions: string[];
    }>>(`/users/${userId}/services/${serviceId}/access`);

    return response.data.data;
  }
}

export const krgeobukIntegration = new KrgeobukIntegrationService();
```

## 통합 상태 관리 패턴

### 크로스 서비스 Redux 슬라이스
```typescript
// src/store/slices/integrationSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { krgeobukIntegration } from '@/services/krgeobukIntegrationService';
import type { User, Role, Permission, Service } from '@/types';

interface IntegrationState {
  permissionMatrix: {
    services: Service[];
    permissions: Permission[];
    matrix: Record<string, string[]>;
  } | null;
  userServiceAccess: Record<string, {
    hasAccess: boolean;
    permissions: string[];
  }>;
  loading: boolean;
  error: string | null;
}

// 권한 매트릭스 로드
export const loadPermissionMatrix = createAsyncThunk(
  'integration/loadPermissionMatrix',
  async (_, { rejectWithValue }) => {
    try {
      return await krgeobukIntegration.getServicePermissionMatrix();
    } catch (error: any) {
      return rejectWithValue(error.message);
    }
  }
);

// 사용자 서비스 접근 권한 확인
export const checkServiceAccess = createAsyncThunk(
  'integration/checkServiceAccess',
  async ({ userId, serviceId }: { userId: string; serviceId: string }, { rejectWithValue }) => {
    try {
      const result = await krgeobukIntegration.checkUserServiceAccess(userId, serviceId);
      return { serviceId, ...result };
    } catch (error: any) {
      return rejectWithValue(error.message);
    }
  }
);

const integrationSlice = createSlice({
  name: 'integration',
  initialState: {
    permissionMatrix: null,
    userServiceAccess: {},
    loading: false,
    error: null,
  } as IntegrationState,
  reducers: {
    clearError: (state) => {
      state.error = null;
    },
    resetUserAccess: (state) => {
      state.userServiceAccess = {};
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(loadPermissionMatrix.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(loadPermissionMatrix.fulfilled, (state, action) => {
        state.loading = false;
        state.permissionMatrix = action.payload;
      })
      .addCase(loadPermissionMatrix.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload as string;
      })
      .addCase(checkServiceAccess.fulfilled, (state, action) => {
        const { serviceId, hasAccess, permissions } = action.payload;
        state.userServiceAccess[serviceId] = { hasAccess, permissions };
      });
  },
});

export const { clearError, resetUserAccess } = integrationSlice.actions;
export default integrationSlice.reducer;
```

### 관리자 인터페이스 구조
`/admin` 라우트 하위의 중첩 페이지 구조:
- **사용자 관리** (`/admin/auth/users`) - auth-server:8000 연동
- **OAuth 클라이언트 관리** (`/admin/auth/oauth`) - auth-server:8000 연동
- **역할 관리** (`/admin/authorization/roles`) - authz-server:8100 연동
- **권한 관리** (`/admin/authorization/permissions`) - authz-server:8100 연동
- **서비스 관리** (`/admin/portal/services`) - portal-server:8200 연동

# 🚀 Next.js 15 고급 패턴

## 1. App Router 최적화 전략

### Server Components vs Client Components
```typescript
// app/users/page.tsx (Server Component)
import { UserList } from '@/components/users/UserList';
import { userService } from '@/services/userService';

export default async function UsersPage() {
  // 서버에서 데이터 페칭
  const users = await userService.getUsers();
  
  return (
    <div className="container mx-auto p-6">
      <h1 className="text-2xl font-bold mb-6">사용자 관리</h1>
      <UserList initialUsers={users} />
    </div>
  );
}

// components/users/UserList.tsx (Client Component)
'use client';
import { useState, useEffect } from 'react';
import { User } from '@/types';

interface UserListProps {
  initialUsers: User[];
}

export function UserList({ initialUsers }: UserListProps) {
  const [users, setUsers] = useState<User[]>(initialUsers);
  
  // 클라이언트 상태 관리
  const handleUserUpdate = (updatedUser: User) => {
    setUsers(prev => prev.map(user => 
      user.id === updatedUser.id ? updatedUser : user
    ));
  };
  
  return (
    <div className="space-y-4">
      {users.map(user => (
        <UserCard key={user.id} user={user} onUpdate={handleUserUpdate} />
      ))}
    </div>
  );
}
```

### 동적 라우팅 고급 패턴
```typescript
// app/admin/users/[id]/page.tsx
interface UserDetailPageProps {
  params: { id: string };
  searchParams: { tab?: string };
}

export default function UserDetailPage({ params, searchParams }: UserDetailPageProps) {
  const userId = params.id;
  const activeTab = searchParams.tab || 'profile';
  
  return (
    <div className="space-y-6">
      <UserDetailTabs userId={userId} activeTab={activeTab} />
    </div>
  );
}

// 동적 세그먼트와 병렬 라우팅
// app/admin/users/[id]/@sidebar/page.tsx
export default function UserSidebar({ params }: { params: { id: string } }) {
  return <UserActionSidebar userId={params.id} />;
}
```

### Next.js Image 최적화 패턴
```typescript
// components/common/OptimizedImage.tsx
import Image from 'next/image';
import { useState } from 'react';

interface OptimizedImageProps {
  src: string;
  alt: string;
  width?: number;
  height?: number;
  className?: string;
  priority?: boolean;
}

export function OptimizedImage({ 
  src, 
  alt, 
  width = 400, 
  height = 300, 
  className,
  priority = false 
}: OptimizedImageProps) {
  const [isLoading, setIsLoading] = useState(true);
  const [hasError, setHasError] = useState(false);

  return (
    <div className={`relative overflow-hidden ${className}`}>
      {isLoading && (
        <div className="absolute inset-0 bg-gray-200 animate-pulse rounded-lg" />
      )}
      
      <Image
        src={src}
        alt={alt}
        width={width}
        height={height}
        priority={priority}
        className={`transition-opacity duration-300 ${
          isLoading ? 'opacity-0' : 'opacity-100'
        }`}
        onLoad={() => setIsLoading(false)}
        onError={() => {
          setIsLoading(false);
          setHasError(true);
        }}
        placeholder="blur"
        blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQABAAD/2wBDAAYEBQYFBAYGBQYHBwYIChAKCgkJChQODwwQFxQYGBcUFhYaHSUfGhsjHBYWICwgIyYnKSopGR8tMC0oMCUoKSj/2wBDAQcHBwoIChMKChMoGhYaKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCj/wAARCAABAAEDASIAAhEBAxEB/8QAFQABAQAAAAAAAAAAAAAAAAAAAAv/xAAUEAEAAAAAAAAAAAAAAAAAAAAA/8QAFQEBAQAAAAAAAAAAAAAAAAAAAAX/xAAUEQEAAAAAAAAAAAAAAAAAAAAA/9oADAMBAAIRAxEAPwCdABmX/9k="
      />
      
      {hasError && (
        <div className="absolute inset-0 bg-gray-100 flex items-center justify-center">
          <span className="text-gray-500">이미지를 불러올 수 없습니다</span>
        </div>
      )}
    </div>
  );
}
```

## 2. 고급 데이터 페칭 패턴

### Streaming과 Suspense
```typescript
// app/dashboard/loading.tsx
export default function DashboardLoading() {
  return (
    <div className="space-y-6">
      <div className="h-8 bg-gray-200 animate-pulse rounded-lg w-48" />
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {[...Array(3)].map((_, i) => (
          <div key={i} className="h-32 bg-gray-200 animate-pulse rounded-lg" />
        ))}
      </div>
    </div>
  );
}

// app/dashboard/page.tsx
import { Suspense } from 'react';
import { DashboardStats } from '@/components/dashboard/DashboardStats';
import { RecentActivity } from '@/components/dashboard/RecentActivity';

export default function DashboardPage() {
  return (
    <div className="space-y-6">
      <h1 className="text-2xl font-bold">대시보드</h1>
      
      <Suspense fallback={<StatsSkeleton />}>
        <DashboardStats />
      </Suspense>
      
      <Suspense fallback={<ActivitySkeleton />}>
        <RecentActivity />
      </Suspense>
    </div>
  );
}
```

### 병렬 데이터 로딩
```typescript
// lib/data-fetching.ts
export async function getParallelData() {
  const [users, roles, permissions] = await Promise.all([
    userService.getUsers(),
    roleService.getRoles(),
    permissionService.getPermissions()
  ]);
  
  return { users, roles, permissions };
}

// app/admin/page.tsx
export default async function AdminPage() {
  const { users, roles, permissions } = await getParallelData();
  
  return (
    <AdminDashboard 
      users={users} 
      roles={roles} 
      permissions={permissions} 
    />
  );
}
```

## 3. 라우트 핸들러 고급 패턴

### API 라우트 최적화
```typescript
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { headers } from 'next/headers';
import { userService } from '@/services/userService';

export async function GET(request: NextRequest) {
  try {
    const headersList = headers();
    const authorization = headersList.get('authorization');
    
    if (!authorization) {
      return NextResponse.json(
        { error: 'Unauthorized' }, 
        { status: 401 }
      );
    }

    const { searchParams } = new URL(request.url);
    const page = Number(searchParams.get('page')) || 1;
    const limit = Number(searchParams.get('limit')) || 10;
    
    const users = await userService.getUsers({ page, limit });
    
    return NextResponse.json(users, {
      headers: {
        'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=300',
      },
    });
  } catch (error) {
    console.error('API Error:', error);
    return NextResponse.json(
      { error: 'Internal Server Error' }, 
      { status: 500 }
    );
  }
}
```

### 미들웨어 고급 활용
```typescript
// middleware.ts 확장
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

// A/B 테스트 구현
function getABTestVariant(request: NextRequest): 'A' | 'B' {
  const cookie = request.cookies.get('ab-test');
  if (cookie) return cookie.value as 'A' | 'B';
  
  const variant = Math.random() > 0.5 ? 'A' : 'B';
  return variant;
}

export function middleware(request: NextRequest) {
  const response = NextResponse.next();
  
  // A/B 테스트 헤더 추가
  if (request.nextUrl.pathname.startsWith('/dashboard')) {
    const variant = getABTestVariant(request);
    response.cookies.set('ab-test', variant);
    response.headers.set('x-ab-test', variant);
  }
  
  // 지역화 처리
  const pathname = request.nextUrl.pathname;
  const pathnameIsMissingLocale = ['/ko', '/en'].every(
    (locale) => !pathname.startsWith(`/${locale}/`) && pathname !== `/${locale}`
  );

  if (pathnameIsMissingLocale) {
    const locale = request.headers.get('accept-language')?.includes('ko') ? 'ko' : 'en';
    return NextResponse.redirect(
      new URL(`/${locale}${pathname}`, request.url)
    );
  }

  return response;
}
```

---

# 🧩 컴포넌트 아키텍처 패턴

## 1. 컴포넌트 분류 및 네이밍

### Common 컴포넌트 (재사용 가능한 UI)
```typescript
// src/components/common/Button.tsx
interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger' | 'outline';
  size?: 'sm' | 'md' | 'lg';
  children: ReactNode;
  loading?: boolean;
  'aria-label'?: string;
  'aria-describedby'?: string;
}

export default function Button({
  variant = 'primary',
  size = 'md',
  children,
  loading = false,
  disabled,
  className = '',
  ...props
}: ButtonProps): JSX.Element {
  const baseClasses = 'inline-flex items-center justify-center font-medium rounded-md transition-colors focus:outline-none focus:ring-2 focus:ring-offset-2';
  
  const variantClasses = {
    primary: 'bg-blue-500 dark:bg-blue-600 text-white hover:bg-blue-600 dark:hover:bg-blue-700 focus:ring-blue-400',
    secondary: 'bg-gray-500 dark:bg-gray-600 text-white hover:bg-gray-600 dark:hover:bg-gray-700 focus:ring-gray-400',
    danger: 'bg-red-500 dark:bg-red-600 text-white hover:bg-red-600 dark:hover:bg-red-700 focus:ring-red-400',
    outline: 'border border-gray-300 dark:border-gray-500 bg-white dark:bg-gray-800 text-gray-800 dark:text-gray-100 hover:bg-gray-100 dark:hover:bg-gray-700 focus:ring-blue-400'
  };
  
  const sizeClasses = {
    sm: 'px-3 py-1.5 text-sm',
    md: 'px-4 py-2 text-sm',
    lg: 'px-6 py-3 text-base'
  };
  
  const isDisabled = disabled || loading;
  
  return (
    <button
      className={`${baseClasses} ${variantClasses[variant]} ${sizeClasses[size]} ${
        isDisabled ? 'opacity-50 cursor-not-allowed' : ''
      } ${className}`}
      disabled={isDisabled}
      {...props}
    >
      {loading && (
        <svg 
          className="animate-spin -ml-1 mr-2 h-4 w-4" 
          fill="none" 
          viewBox="0 0 24 24"
          aria-hidden="true"
        >
          <circle cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4" className="opacity-25" />
          <path className="opacity-75" fill="currentColor" d="m4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z" />
        </svg>
      )}
      {children}
    </button>
  );
}
```

### Layout 컴포넌트 (구조적 레이아웃)
```typescript
// src/components/layout/Layout.tsx
interface LayoutProps {
  children: React.ReactNode;
  showSidebar?: boolean;
}

export const Layout: React.FC<LayoutProps> = ({ children, showSidebar = false }) => {
  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 via-indigo-50 to-purple-50">
      <Header />
      <div className="flex">
        {showSidebar && <Sidebar />}
        <main className="flex-1 p-6">
          {children}
        </main>
      </div>
    </div>
  );
};
```

### Form 컴포넌트 (특화된 폼)
```typescript
// src/components/forms/RoleForm.tsx
interface RoleFormProps {
  role?: Role;
  onSubmit: (data: CreateRoleDto | UpdateRoleDto) => void;
  onCancel: () => void;
}

export const RoleForm: React.FC<RoleFormProps> = ({ role, onSubmit, onCancel }) => {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<RoleFormData>();
  
  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
      {/* 폼 필드들 */}
    </form>
  );
};
```

# 🎭 고급 React 패턴

## 1. Error Boundary 완전 구현

### 전역 Error Boundary
```typescript
// src/components/error/GlobalErrorBoundary.tsx
'use client';
import { Component, ReactNode, ErrorInfo } from 'react';
import { Button } from '@/components/common/Button';

interface ErrorBoundaryState {
  hasError: boolean;
  error?: Error;
  errorInfo?: ErrorInfo;
}

interface ErrorBoundaryProps {
  children: ReactNode;
  fallback?: ReactNode;
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
}

export class GlobalErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return {
      hasError: true,
      error,
    };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    // 에러 로깅
    console.error('Error Boundary caught an error:', error, errorInfo);
    
    // 에러 추적 서비스로 전송
    if (this.props.onError) {
      this.props.onError(error, errorInfo);
    }

    this.setState({
      error,
      errorInfo,
    });
  }

  handleReset = () => {
    this.setState({ hasError: false, error: undefined, errorInfo: undefined });
  };

  render() {
    if (this.state.hasError) {
      if (this.props.fallback) {
        return this.props.fallback;
      }

      return (
        <div className="min-h-screen flex items-center justify-center bg-gray-50">
          <div className="max-w-md w-full bg-white rounded-lg shadow-lg p-8">
            <div className="text-center">
              <div className="text-red-500 text-6xl mb-4">⚠️</div>
              <h1 className="text-2xl font-bold text-gray-900 mb-2">
                앗! 문제가 발생했습니다
              </h1>
              <p className="text-gray-600 mb-6">
                예상치 못한 오류가 발생했습니다. 잠시 후 다시 시도해주세요.
              </p>
              
              {process.env.NODE_ENV === 'development' && (
                <details className="text-left bg-gray-100 p-4 rounded-lg mb-6">
                  <summary className="cursor-pointer font-medium">개발자 정보</summary>
                  <pre className="mt-2 text-sm text-red-600 overflow-x-auto">
                    {this.state.error?.stack}
                  </pre>
                </details>
              )}
              
              <div className="space-x-3">
                <Button onClick={this.handleReset}>
                  다시 시도
                </Button>
                <Button 
                  variant="outline" 
                  onClick={() => window.location.reload()}
                >
                  페이지 새로고침
                </Button>
              </div>
            </div>
          </div>
        </div>
      );
    }

    return this.props.children;
  }
}
```

### 특화된 Error Boundary
```typescript
// src/components/error/AsyncErrorBoundary.tsx
'use client';
import { useEffect } from 'react';
import { GlobalErrorBoundary } from './GlobalErrorBoundary';

interface AsyncErrorBoundaryProps {
  children: React.ReactNode;
  onError?: (error: Error) => void;
}

export function AsyncErrorBoundary({ children, onError }: AsyncErrorBoundaryProps) {
  useEffect(() => {
    const handleUnhandledRejection = (event: PromiseRejectionEvent) => {
      const error = new Error(event.reason);
      if (onError) {
        onError(error);
      }
    };

    window.addEventListener('unhandledrejection', handleUnhandledRejection);
    
    return () => {
      window.removeEventListener('unhandledrejection', handleUnhandledRejection);
    };
  }, [onError]);

  return (
    <GlobalErrorBoundary onError={onError}>
      {children}
    </GlobalErrorBoundary>
  );
}
```

## 2. Suspense와 지연 로딩 고급 패턴

### 스마트 Suspense 래퍼
```typescript
// src/components/loading/SmartSuspense.tsx
import { Suspense, ReactNode } from 'react';

interface SmartSuspenseProps {
  children: ReactNode;
  fallback?: ReactNode;
  minDelay?: number; // 최소 로딩 시간 (너무 빠른 깜빡임 방지)
}

export function SmartSuspense({ 
  children, 
  fallback, 
  minDelay = 200 
}: SmartSuspenseProps) {
  return (
    <Suspense 
      fallback={
        <DelayedFallback delay={minDelay}>
          {fallback || <DefaultLoadingSpinner />}
        </DelayedFallback>
      }
    >
      {children}
    </Suspense>
  );
}

// 지연된 폴백 컴포넌트
function DelayedFallback({ children, delay }: { children: ReactNode; delay: number }) {
  const [show, setShow] = useState(false);

  useEffect(() => {
    const timer = setTimeout(() => setShow(true), delay);
    return () => clearTimeout(timer);
  }, [delay]);

  return show ? <>{children}</> : null;
}
```

### 리소스 기반 지연 로딩
```typescript
// src/hooks/useResourceLoader.ts
import { useState, useEffect } from 'react';

interface LoaderState<T> {
  data?: T;
  loading: boolean;
  error?: Error;
}

export function useResourceLoader<T>(
  loader: () => Promise<T>,
  deps: any[] = []
): LoaderState<T> {
  const [state, setState] = useState<LoaderState<T>>({
    loading: true,
  });

  useEffect(() => {
    let cancelled = false;

    const loadResource = async () => {
      setState({ loading: true });
      
      try {
        const data = await loader();
        
        if (!cancelled) {
          setState({ data, loading: false });
        }
      } catch (error) {
        if (!cancelled) {
          setState({ 
            error: error instanceof Error ? error : new Error(String(error)), 
            loading: false 
          });
        }
      }
    };

    loadResource();

    return () => {
      cancelled = true;
    };
  }, deps);

  return state;
}

// 사용 예제
function UserDetail({ userId }: { userId: string }) {
  const { data: user, loading, error } = useResourceLoader(
    () => userService.getUserById(userId),
    [userId]
  );

  if (loading) return <LoadingSpinner />;
  if (error) throw error;
  if (!user) return <div>사용자를 찾을 수 없습니다</div>;

  return <UserCard user={user} />;
}
```

## 3. Context API 고급 패턴

### 분할 컨텍스트 패턴
```typescript
// src/context/ThemeContext.tsx
'use client';
import { createContext, useContext, useReducer, ReactNode } from 'react';

type Theme = 'light' | 'dark' | 'system';
type Language = 'ko' | 'en';

interface ThemeState {
  theme: Theme;
  language: Language;
  isLoading: boolean;
}

type ThemeAction = 
  | { type: 'SET_THEME'; payload: Theme }
  | { type: 'SET_LANGUAGE'; payload: Language }
  | { type: 'SET_LOADING'; payload: boolean };

const ThemeStateContext = createContext<ThemeState | undefined>(undefined);
const ThemeDispatchContext = createContext<React.Dispatch<ThemeAction> | undefined>(undefined);

function themeReducer(state: ThemeState, action: ThemeAction): ThemeState {
  switch (action.type) {
    case 'SET_THEME':
      return { ...state, theme: action.payload };
    case 'SET_LANGUAGE':
      return { ...state, language: action.payload };
    case 'SET_LOADING':
      return { ...state, isLoading: action.payload };
    default:
      return state;
  }
}

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(themeReducer, {
    theme: 'system',
    language: 'ko',
    isLoading: false,
  });

  // 시스템 테마 감지
  useEffect(() => {
    if (state.theme === 'system') {
      const mediaQuery = window.matchMedia('(prefers-color-scheme: dark)');
      const handleChange = () => {
        document.documentElement.classList.toggle('dark', mediaQuery.matches);
      };
      
      handleChange(); // 초기 설정
      mediaQuery.addEventListener('change', handleChange);
      
      return () => mediaQuery.removeEventListener('change', handleChange);
    } else {
      document.documentElement.classList.toggle('dark', state.theme === 'dark');
    }
  }, [state.theme]);

  return (
    <ThemeStateContext.Provider value={state}>
      <ThemeDispatchContext.Provider value={dispatch}>
        {children}
      </ThemeDispatchContext.Provider>
    </ThemeStateContext.Provider>
  );
}

// 커스텀 훅들
export function useThemeState() {
  const context = useContext(ThemeStateContext);
  if (context === undefined) {
    throw new Error('useThemeState must be used within a ThemeProvider');
  }
  return context;
}

export function useThemeDispatch() {
  const context = useContext(ThemeDispatchContext);
  if (context === undefined) {
    throw new Error('useThemeDispatch must be used within a ThemeProvider');
  }
  return context;
}

// 편의 훅
export function useTheme() {
  const state = useThemeState();
  const dispatch = useThemeDispatch();

  const setTheme = (theme: Theme) => {
    dispatch({ type: 'SET_THEME', payload: theme });
    localStorage.setItem('theme', theme);
  };

  const setLanguage = (language: Language) => {
    dispatch({ type: 'SET_LANGUAGE', payload: language });
    localStorage.setItem('language', language);
  };

  return {
    ...state,
    setTheme,
    setLanguage,
  };
}
```

## 4. 복합 컴포넌트 (Compound Component) 패턴

### Modal 복합 컴포넌트
```typescript
// src/components/common/Modal/index.tsx
'use client';
import { 
  createContext, 
  useContext, 
  useState, 
  ReactNode, 
  useEffect, 
  useRef 
} from 'react';
import { createPortal } from 'react-dom';

interface ModalContextType {
  isOpen: boolean;
  onClose: () => void;
}

const ModalContext = createContext<ModalContextType | null>(null);

// 메인 Modal 컴포넌트
interface ModalProps {
  children: ReactNode;
  isOpen: boolean;
  onClose: () => void;
  closeOnOverlayClick?: boolean;
  closeOnEscape?: boolean;
}

function ModalRoot({ 
  children, 
  isOpen, 
  onClose, 
  closeOnOverlayClick = true,
  closeOnEscape = true 
}: ModalProps) {
  const [mounted, setMounted] = useState(false);
  
  useEffect(() => {
    setMounted(true);
  }, []);

  useEffect(() => {
    if (!closeOnEscape) return;

    const handleEscape = (event: KeyboardEvent) => {
      if (event.key === 'Escape' && isOpen) {
        onClose();
      }
    };

    document.addEventListener('keydown', handleEscape);
    return () => document.removeEventListener('keydown', handleEscape);
  }, [isOpen, onClose, closeOnEscape]);

  useEffect(() => {
    if (isOpen) {
      document.body.style.overflow = 'hidden';
    } else {
      document.body.style.overflow = 'unset';
    }

    return () => {
      document.body.style.overflow = 'unset';
    };
  }, [isOpen]);

  if (!mounted || !isOpen) return null;

  const contextValue = { isOpen, onClose };

  return createPortal(
    <ModalContext.Provider value={contextValue}>
      <div 
        className="fixed inset-0 z-50 flex items-center justify-center"
        onClick={closeOnOverlayClick ? onClose : undefined}
      >
        {children}
      </div>
    </ModalContext.Provider>,
    document.body
  );
}

// Overlay 컴포넌트
function ModalOverlay({ className = '' }: { className?: string }) {
  return (
    <div className={`absolute inset-0 bg-black/50 backdrop-blur-sm ${className}`} />
  );
}

// Content 컴포넌트
interface ModalContentProps {
  children: ReactNode;
  className?: string;
}

function ModalContent({ children, className = '' }: ModalContentProps) {
  const contentRef = useRef<HTMLDivElement>(null);

  return (
    <div 
      ref={contentRef}
      className={`relative bg-white rounded-lg shadow-xl max-w-lg w-full mx-4 ${className}`}
      onClick={(e) => e.stopPropagation()}
    >
      {children}
    </div>
  );
}

// Header 컴포넌트
function ModalHeader({ children, className = '' }: { children: ReactNode; className?: string }) {
  const { onClose } = useContext(ModalContext) || {};

  return (
    <div className={`flex items-center justify-between p-6 border-b ${className}`}>
      <div className="text-lg font-semibold">{children}</div>
      <button
        onClick={onClose}
        className="text-gray-400 hover:text-gray-600 text-2xl leading-none"
        aria-label="닫기"
      >
        ×
      </button>
    </div>
  );
}

// Body 컴포넌트
function ModalBody({ children, className = '' }: { children: ReactNode; className?: string }) {
  return (
    <div className={`p-6 ${className}`}>
      {children}
    </div>
  );
}

// Footer 컴포넌트
function ModalFooter({ children, className = '' }: { children: ReactNode; className?: string }) {
  return (
    <div className={`flex justify-end space-x-2 p-6 border-t bg-gray-50 rounded-b-lg ${className}`}>
      {children}
    </div>
  );
}

// 복합 컴포넌트 export
export const Modal = {
  Root: ModalRoot,
  Overlay: ModalOverlay,
  Content: ModalContent,
  Header: ModalHeader,
  Body: ModalBody,
  Footer: ModalFooter,
};

// 사용 예제
function UserDetailModal({ user, isOpen, onClose }: {
  user: User;
  isOpen: boolean;
  onClose: () => void;
}) {
  return (
    <Modal.Root isOpen={isOpen} onClose={onClose}>
      <Modal.Overlay />
      <Modal.Content className="max-w-2xl">
        <Modal.Header>
          사용자 상세 정보
        </Modal.Header>
        <Modal.Body>
          <UserDetails user={user} />
        </Modal.Body>
        <Modal.Footer>
          <Button variant="outline" onClick={onClose}>
            닫기
          </Button>
          <Button onClick={() => console.log('편집')}>
            편집
          </Button>
        </Modal.Footer>
      </Modal.Content>
    </Modal.Root>
  );
}
```

## 5. 고급 커스텀 훅 패턴

### useLocalStorage with SSR 지원
```typescript
// src/hooks/useLocalStorage.ts
import { useState, useEffect } from 'react';

function useLocalStorage<T>(
  key: string,
  initialValue: T
): [T, (value: T | ((prev: T) => T)) => void] {
  // SSR 안전성을 위한 초기 상태
  const [storedValue, setStoredValue] = useState<T>(initialValue);
  const [isInitialized, setIsInitialized] = useState(false);

  useEffect(() => {
    try {
      const item = window.localStorage.getItem(key);
      if (item) {
        setStoredValue(JSON.parse(item));
      }
    } catch (error) {
      console.error(`Error reading localStorage key "${key}":`, error);
    } finally {
      setIsInitialized(true);
    }
  }, [key]);

  const setValue = (value: T | ((prev: T) => T)) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      
      if (typeof window !== 'undefined') {
        window.localStorage.setItem(key, JSON.stringify(valueToStore));
      }
    } catch (error) {
      console.error(`Error setting localStorage key "${key}":`, error);
    }
  };

  return [isInitialized ? storedValue : initialValue, setValue];
}

export default useLocalStorage;
```

---

# 📋 React Hook Form 표준 패턴

## 1. 폼 훅 설정 표준
```typescript
// 기본 폼 설정
const {
  register,
  handleSubmit,
  formState: { errors, isSubmitting },
  reset,
  watch,
  setValue
} = useForm<FormData>({
  defaultValues: {
    // 기본값 설정
  },
  mode: 'onChange', // 실시간 검증
});
```

## 2. 폼 검증 패턴
```typescript
// 검증 규칙 정의
const validationRules = {
  email: {
    required: '이메일을 입력해주세요',
    pattern: {
      value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
      message: '올바른 이메일 형식을 입력해주세요'
    }
  },
  password: {
    required: '비밀번호를 입력해주세요',
    minLength: {
      value: 8,
      message: '비밀번호는 최소 8자 이상이어야 합니다'
    }
  }
};

// 입력 필드 구현
<input
  {...register('email', validationRules.email)}
  className={cn(
    'w-full px-3 py-2 border rounded-md',
    errors.email ? 'border-red-500' : 'border-gray-300'
  )}
/>
{errors.email && (
  <p className="text-red-500 text-sm mt-1">{errors.email.message}</p>
)}
```

## 3. 폼 제출 패턴
```typescript
const onSubmit = async (data: FormData) => {
  try {
    setIsLoading(true);
    await apiService.create(data);
    reset(); // 폼 초기화
    onSuccess?.();
  } catch (error) {
    setError(error.message);
  } finally {
    setIsLoading(false);
  }
};
```

---

# 📝 고급 폼 관리

## 1. 다중 단계 폼 구현

### 스마트 다단계 폼 컨테이너
```typescript
// src/components/forms/MultiStepForm.tsx
import { useState, ReactNode } from 'react';
import { useForm, FormProvider } from 'react-hook-form';

interface Step {
  id: string;
  title: string;
  component: ReactNode;
  validation?: any;
}

interface MultiStepFormProps<T> {
  steps: Step[];
  onComplete: (data: T) => void;
  onCancel: () => void;
  defaultValues?: Partial<T>;
}

export function MultiStepForm<T extends Record<string, any>>({
  steps,
  onComplete,
  onCancel,
  defaultValues = {}
}: MultiStepFormProps<T>) {
  const [currentStep, setCurrentStep] = useState(0);
  const [completedSteps, setCompletedSteps] = useState<Set<number>>(new Set());
  
  const methods = useForm<T>({
    defaultValues: defaultValues as T,
    mode: 'onChange',
  });

  const { trigger, getValues, handleSubmit } = methods;

  const nextStep = async () => {
    const currentStepData = steps[currentStep];
    const isStepValid = await trigger(currentStepData.validation);
    
    if (isStepValid) {
      setCompletedSteps(prev => new Set([...prev, currentStep]));
      
      if (currentStep < steps.length - 1) {
        setCurrentStep(currentStep + 1);
      } else {
        // 마지막 단계 - 완료
        handleSubmit(onComplete)();
      }
    }
  };

  const prevStep = () => {
    if (currentStep > 0) {
      setCurrentStep(currentStep - 1);
    }
  };

  const goToStep = async (stepIndex: number) => {
    // 이전 단계들이 모두 완료되었는지 확인
    const canGoToStep = stepIndex <= Math.max(...completedSteps) + 1;
    
    if (canGoToStep) {
      setCurrentStep(stepIndex);
    }
  };

  const progress = ((currentStep + 1) / steps.length) * 100;

  return (
    <FormProvider {...methods}>
      <div className="max-w-2xl mx-auto p-6">
        {/* 진행률 표시 */}
        <div className="mb-8">
          <div className="flex justify-between items-center mb-2">
            <span className="text-sm font-medium text-gray-700">
              단계 {currentStep + 1} / {steps.length}
            </span>
            <span className="text-sm font-medium text-blue-600">
              {Math.round(progress)}%
            </span>
          </div>
          <div className="w-full bg-gray-200 rounded-full h-2">
            <div 
              className="bg-blue-600 h-2 rounded-full transition-all duration-300"
              style={{ width: `${progress}%` }}
            />
          </div>
        </div>

        {/* 단계 네비게이션 */}
        <div className="mb-8">
          <div className="flex justify-between">
            {steps.map((step, index) => (
              <button
                key={step.id}
                type="button"
                onClick={() => goToStep(index)}
                disabled={index > Math.max(...completedSteps) + 1}
                className={`flex-1 py-2 px-4 text-sm font-medium rounded-lg mx-1 transition-colors ${
                  index === currentStep
                    ? 'bg-blue-100 text-blue-700 border-2 border-blue-300'
                    : completedSteps.has(index)
                    ? 'bg-green-100 text-green-700 border-2 border-green-300'
                    : 'bg-gray-100 text-gray-500 border-2 border-gray-200'
                } ${
                  index > Math.max(...completedSteps) + 1
                    ? 'cursor-not-allowed opacity-50'
                    : 'cursor-pointer hover:bg-opacity-80'
                }`}
              >
                {completedSteps.has(index) && '✓ '}
                {step.title}
              </button>
            ))}
          </div>
        </div>

        {/* 현재 단계 컨텐츠 */}
        <div className="mb-8 min-h-[400px]">
          <div className="bg-white rounded-lg border border-gray-200 p-6">
            <h2 className="text-xl font-semibold mb-4">
              {steps[currentStep].title}
            </h2>
            {steps[currentStep].component}
          </div>
        </div>

        {/* 네비게이션 버튼 */}
        <div className="flex justify-between">
          <div>
            {currentStep > 0 && (
              <button
                type="button"
                onClick={prevStep}
                className="px-4 py-2 text-gray-600 bg-gray-100 rounded-lg hover:bg-gray-200 transition-colors"
              >
                ← 이전
              </button>
            )}
          </div>
          
          <div className="flex space-x-3">
            <button
              type="button"
              onClick={onCancel}
              className="px-4 py-2 text-gray-600 bg-gray-100 rounded-lg hover:bg-gray-200 transition-colors"
            >
              취소
            </button>
            <button
              type="button"
              onClick={nextStep}
              className="px-6 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors"
            >
              {currentStep === steps.length - 1 ? '완료' : '다음 →'}
            </button>
          </div>
        </div>
      </div>
    </FormProvider>
  );
}
```

### 단계별 폼 컴포넌트 예제
```typescript
// 사용자 등록 다단계 폼 예제
interface UserRegistrationData {
  // 1단계: 기본 정보
  email: string;
  password: string;
  confirmPassword: string;
  
  // 2단계: 개인 정보
  name: string;
  phoneNumber: string;
  birthDate: string;
  
  // 3단계: 선호 설정
  preferredLanguage: string;
  emailNotifications: boolean;
  smsNotifications: boolean;
  
  // 4단계: 약관 동의
  agreeToTerms: boolean;
  agreeToPrivacy: boolean;
  agreeToMarketing: boolean;
}

// 1단계: 계정 정보
function AccountStep() {
  const { register, formState: { errors }, watch } = useFormContext<UserRegistrationData>();
  const password = watch('password');

  return (
    <div className="space-y-4">
      <div>
        <label className="block text-sm font-medium mb-1">이메일</label>
        <input
          {...register('email', {
            required: '이메일을 입력해주세요',
            pattern: {
              value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
              message: '올바른 이메일 형식을 입력해주세요'
            }
          })}
          type="email"
          className="w-full px-3 py-2 border rounded-md"
        />
        {errors.email && (
          <p className="text-red-500 text-sm mt-1">{errors.email.message}</p>
        )}
      </div>

      <div>
        <label className="block text-sm font-medium mb-1">비밀번호</label>
        <input
          {...register('password', {
            required: '비밀번호를 입력해주세요',
            minLength: {
              value: 8,
              message: '비밀번호는 최소 8자 이상이어야 합니다'
            },
            pattern: {
              value: /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]/,
              message: '영문 대/소문자, 숫자, 특수문자를 포함해야 합니다'
            }
          })}
          type="password"
          className="w-full px-3 py-2 border rounded-md"
        />
        {errors.password && (
          <p className="text-red-500 text-sm mt-1">{errors.password.message}</p>
        )}
      </div>

      <div>
        <label className="block text-sm font-medium mb-1">비밀번호 확인</label>
        <input
          {...register('confirmPassword', {
            required: '비밀번호를 다시 입력해주세요',
            validate: value => value === password || '비밀번호가 일치하지 않습니다'
          })}
          type="password"
          className="w-full px-3 py-2 border rounded-md"
        />
        {errors.confirmPassword && (
          <p className="text-red-500 text-sm mt-1">{errors.confirmPassword.message}</p>
        )}
      </div>
    </div>
  );
}

// 사용 예제
function UserRegistrationForm() {
  const steps = [
    {
      id: 'account',
      title: '계정 정보',
      component: <AccountStep />,
      validation: ['email', 'password', 'confirmPassword']
    },
    {
      id: 'personal',
      title: '개인 정보',
      component: <PersonalStep />,
      validation: ['name', 'phoneNumber', 'birthDate']
    },
    {
      id: 'preferences',
      title: '환경 설정',
      component: <PreferencesStep />,
      validation: []
    },
    {
      id: 'terms',
      title: '약관 동의',
      component: <TermsStep />,
      validation: ['agreeToTerms', 'agreeToPrivacy']
    }
  ];

  const handleComplete = async (data: UserRegistrationData) => {
    try {
      await userService.register(data);
      router.push('/auth/login?message=registration-success');
    } catch (error) {
      console.error('Registration failed:', error);
    }
  };

  return (
    <MultiStepForm
      steps={steps}
      onComplete={handleComplete}
      onCancel={() => router.push('/auth/login')}
    />
  );
}
```

## 2. 조건부 검증 로직

### 동적 필드 검증
```typescript
// src/components/forms/ConditionalValidationForm.tsx
interface ConditionalFormData {
  userType: 'individual' | 'business';
  email: string;
  
  // 개인 사용자 필드
  firstName?: string;
  lastName?: string;
  birthDate?: string;
  
  // 사업자 필드
  companyName?: string;
  businessNumber?: string;
  businessType?: string;
}

function ConditionalValidationForm() {
  const { register, watch, formState: { errors } } = useForm<ConditionalFormData>({
    mode: 'onChange'
  });

  const userType = watch('userType');

  // 조건부 검증 규칙
  const getValidationRules = (fieldName: keyof ConditionalFormData) => {
    const baseRules: Record<string, any> = {
      email: {
        required: '이메일을 입력해주세요',
        pattern: {
          value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
          message: '올바른 이메일 형식을 입력해주세요'
        }
      }
    };

    // 개인 사용자 조건부 검증
    if (userType === 'individual') {
      baseRules.firstName = { required: '이름을 입력해주세요' };
      baseRules.lastName = { required: '성을 입력해주세요' };
      baseRules.birthDate = { required: '생년월일을 입력해주세요' };
    }

    // 사업자 조건부 검증
    if (userType === 'business') {
      baseRules.companyName = { required: '회사명을 입력해주세요' };
      baseRules.businessNumber = {
        required: '사업자등록번호를 입력해주세요',
        pattern: {
          value: /^\d{3}-\d{2}-\d{5}$/,
          message: '올바른 사업자등록번호 형식을 입력해주세요 (예: 123-45-67890)'
        }
      };
      baseRules.businessType = { required: '사업자 유형을 선택해주세요' };
    }

    return baseRules[fieldName];
  };

  return (
    <form className="space-y-6">
      {/* 사용자 타입 선택 */}
      <div>
        <label className="block text-sm font-medium mb-2">사용자 타입</label>
        <div className="flex space-x-4">
          <label className="flex items-center">
            <input
              {...register('userType', { required: true })}
              type="radio"
              value="individual"
              className="mr-2"
            />
            개인
          </label>
          <label className="flex items-center">
            <input
              {...register('userType', { required: true })}
              type="radio"
              value="business"
              className="mr-2"
            />
            사업자
          </label>
        </div>
      </div>

      {/* 공통 필드 */}
      <div>
        <label className="block text-sm font-medium mb-1">이메일</label>
        <input
          {...register('email', getValidationRules('email'))}
          type="email"
          className="w-full px-3 py-2 border rounded-md"
        />
        {errors.email && (
          <p className="text-red-500 text-sm mt-1">{errors.email.message}</p>
        )}
      </div>

      {/* 개인 사용자 필드 */}
      {userType === 'individual' && (
        <>
          <div className="grid grid-cols-2 gap-4">
            <div>
              <label className="block text-sm font-medium mb-1">이름</label>
              <input
                {...register('firstName', getValidationRules('firstName'))}
                className="w-full px-3 py-2 border rounded-md"
              />
              {errors.firstName && (
                <p className="text-red-500 text-sm mt-1">{errors.firstName.message}</p>
              )}
            </div>
            <div>
              <label className="block text-sm font-medium mb-1">성</label>
              <input
                {...register('lastName', getValidationRules('lastName'))}
                className="w-full px-3 py-2 border rounded-md"
              />
              {errors.lastName && (
                <p className="text-red-500 text-sm mt-1">{errors.lastName.message}</p>
              )}
            </div>
          </div>
          
          <div>
            <label className="block text-sm font-medium mb-1">생년월일</label>
            <input
              {...register('birthDate', getValidationRules('birthDate'))}
              type="date"
              className="w-full px-3 py-2 border rounded-md"
            />
            {errors.birthDate && (
              <p className="text-red-500 text-sm mt-1">{errors.birthDate.message}</p>
            )}
          </div>
        </>
      )}

      {/* 사업자 필드 */}
      {userType === 'business' && (
        <>
          <div>
            <label className="block text-sm font-medium mb-1">회사명</label>
            <input
              {...register('companyName', getValidationRules('companyName'))}
              className="w-full px-3 py-2 border rounded-md"
            />
            {errors.companyName && (
              <p className="text-red-500 text-sm mt-1">{errors.companyName.message}</p>
            )}
          </div>
          
          <div>
            <label className="block text-sm font-medium mb-1">사업자등록번호</label>
            <input
              {...register('businessNumber', getValidationRules('businessNumber'))}
              placeholder="123-45-67890"
              className="w-full px-3 py-2 border rounded-md"
            />
            {errors.businessNumber && (
              <p className="text-red-500 text-sm mt-1">{errors.businessNumber.message}</p>
            )}
          </div>
          
          <div>
            <label className="block text-sm font-medium mb-1">사업자 유형</label>
            <select
              {...register('businessType', getValidationRules('businessType'))}
              className="w-full px-3 py-2 border rounded-md"
            >
              <option value="">선택해주세요</option>
              <option value="sole_proprietor">개인사업자</option>
              <option value="corporation">법인사업자</option>
              <option value="partnership">합명회사</option>
            </select>
            {errors.businessType && (
              <p className="text-red-500 text-sm mt-1">{errors.businessType.message}</p>
            )}
          </div>
        </>
      )}
    </form>
  );
}
```

## 3. 동적 필드 추가/제거

### 동적 배열 필드 관리
```typescript
// src/components/forms/DynamicFieldForm.tsx
import { useFieldArray } from 'react-hook-form';

interface Contact {
  type: 'email' | 'phone' | 'address';
  value: string;
  isPrimary: boolean;
}

interface DynamicFormData {
  name: string;
  contacts: Contact[];
}

function DynamicFieldForm() {
  const { control, register, formState: { errors }, watch } = useForm<DynamicFormData>({
    defaultValues: {
      name: '',
      contacts: [{ type: 'email', value: '', isPrimary: true }]
    }
  });

  const { fields, append, remove, move } = useFieldArray({
    control,
    name: 'contacts'
  });

  const contacts = watch('contacts');

  const addContact = () => {
    append({ 
      type: 'email', 
      value: '', 
      isPrimary: contacts.length === 0 
    });
  };

  const removeContact = (index: number) => {
    if (fields.length > 1) {
      // 주 연락처를 삭제하는 경우 다음 연락처를 주 연락처로 설정
      if (contacts[index]?.isPrimary && fields.length > 1) {
        const nextIndex = index === 0 ? 1 : 0;
        // setValue를 사용하여 다음 연락처를 주 연락처로 설정
      }
      remove(index);
    }
  };

  const setPrimary = (index: number) => {
    // 모든 연락처의 isPrimary를 false로 설정하고 선택된 것만 true로
    contacts.forEach((_, i) => {
      if (i !== index) {
        // setValue(`contacts.${i}.isPrimary`, false);
      }
    });
    // setValue(`contacts.${index}.isPrimary`, true);
  };

  return (
    <form className="space-y-6">
      <div>
        <label className="block text-sm font-medium mb-1">이름</label>
        <input
          {...register('name', { required: '이름을 입력해주세요' })}
          className="w-full px-3 py-2 border rounded-md"
        />
        {errors.name && (
          <p className="text-red-500 text-sm mt-1">{errors.name.message}</p>
        )}
      </div>

      <div>
        <div className="flex justify-between items-center mb-4">
          <label className="block text-sm font-medium">연락처</label>
          <button
            type="button"
            onClick={addContact}
            className="px-3 py-1 bg-blue-600 text-white rounded text-sm hover:bg-blue-700"
          >
            + 연락처 추가
          </button>
        </div>

        <div className="space-y-3">
          {fields.map((field, index) => (
            <div key={field.id} className="border rounded-lg p-4 bg-gray-50">
              <div className="flex items-center justify-between mb-3">
                <div className="flex items-center space-x-3">
                  <select
                    {...register(`contacts.${index}.type` as const)}
                    className="px-2 py-1 border rounded text-sm"
                  >
                    <option value="email">이메일</option>
                    <option value="phone">전화번호</option>
                    <option value="address">주소</option>
                  </select>
                  
                  <label className="flex items-center text-sm">
                    <input
                      {...register(`contacts.${index}.isPrimary` as const)}
                      type="checkbox"
                      onChange={() => setPrimary(index)}
                      className="mr-1"
                    />
                    주 연락처
                  </label>
                </div>

                <div className="flex items-center space-x-2">
                  {index > 0 && (
                    <button
                      type="button"
                      onClick={() => move(index, index - 1)}
                      className="text-gray-500 hover:text-gray-700"
                      title="위로 이동"
                    >
                      ↑
                    </button>
                  )}
                  {index < fields.length - 1 && (
                    <button
                      type="button"
                      onClick={() => move(index, index + 1)}
                      className="text-gray-500 hover:text-gray-700"
                      title="아래로 이동"
                    >
                      ↓
                    </button>
                  )}
                  {fields.length > 1 && (
                    <button
                      type="button"
                      onClick={() => removeContact(index)}
                      className="text-red-500 hover:text-red-700"
                      title="삭제"
                    >
                      ✕
                    </button>
                  )}
                </div>
              </div>

              <input
                {...register(`contacts.${index}.value` as const, {
                  required: '값을 입력해주세요',
                  validate: (value) => {
                    const contactType = contacts[index]?.type;
                    if (contactType === 'email') {
                      return /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i.test(value) || 
                             '올바른 이메일 형식을 입력해주세요';
                    }
                    if (contactType === 'phone') {
                      return /^[\d-+() ]+$/.test(value) || 
                             '올바른 전화번호 형식을 입력해주세요';
                    }
                    return true;
                  }
                })}
                placeholder={
                  contacts[index]?.type === 'email' ? 'example@email.com' :
                  contacts[index]?.type === 'phone' ? '010-1234-5678' :
                  '주소를 입력해주세요'
                }
                className="w-full px-3 py-2 border rounded-md"
              />
              {errors.contacts?.[index]?.value && (
                <p className="text-red-500 text-sm mt-1">
                  {errors.contacts[index]?.value?.message}
                </p>
              )}
            </div>
          ))}
        </div>
      </div>
    </form>
  );
}
```

## 4. 폼 상태 영속화 패턴

### 자동 저장 기능
```typescript
// src/hooks/useAutoSave.ts
import { useEffect, useRef } from 'react';
import { UseFormWatch } from 'react-hook-form';
import { debounce } from 'lodash-es';

interface UseAutoSaveOptions<T> {
  watch: UseFormWatch<T>;
  onSave: (data: T) => Promise<void>;
  delay?: number;
  key: string;
}

export function useAutoSave<T>({ 
  watch, 
  onSave, 
  delay = 2000, 
  key 
}: UseAutoSaveOptions<T>) {
  const [isSaving, setIsSaving] = useState(false);
  const [lastSaved, setLastSaved] = useState<Date | null>(null);
  
  const debouncedSave = useRef(
    debounce(async (data: T) => {
      try {
        setIsSaving(true);
        await onSave(data);
        setLastSaved(new Date());
        
        // 로컬 스토리지에도 저장
        localStorage.setItem(`autosave_${key}`, JSON.stringify(data));
      } catch (error) {
        console.error('Auto save failed:', error);
      } finally {
        setIsSaving(false);
      }
    }, delay)
  );

  useEffect(() => {
    const subscription = watch((data) => {
      debouncedSave.current(data as T);
    });

    return () => {
      subscription.unsubscribe();
      debouncedSave.current.cancel();
    };
  }, [watch]);

  // 로컬 스토리지에서 복원
  const restoreFromStorage = (): T | null => {
    try {
      const saved = localStorage.getItem(`autosave_${key}`);
      return saved ? JSON.parse(saved) : null;
    } catch {
      return null;
    }
  };

  const clearStorage = () => {
    localStorage.removeItem(`autosave_${key}`);
  };

  return {
    isSaving,
    lastSaved,
    restoreFromStorage,
    clearStorage,
  };
}

// 사용 예제
function AutoSaveForm() {
  const { register, watch, reset, handleSubmit } = useForm<FormData>();
  
  const { isSaving, lastSaved, restoreFromStorage, clearStorage } = useAutoSave({
    watch,
    onSave: async (data) => {
      // 서버에 임시 저장
      await draftService.save(data);
    },
    key: 'user-form'
  });

  useEffect(() => {
    // 페이지 로드 시 복원
    const restored = restoreFromStorage();
    if (restored) {
      reset(restored);
    }
  }, []);

  const onSubmit = async (data: FormData) => {
    await apiService.submit(data);
    clearStorage(); // 제출 완료 후 임시 저장 데이터 삭제
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* 자동 저장 상태 표시 */}
      <div className="flex justify-between items-center mb-4">
        <div className="flex items-center space-x-2 text-sm text-gray-600">
          {isSaving && (
            <>
              <div className="animate-spin w-3 h-3 border border-blue-500 border-t-transparent rounded-full" />
              <span>저장 중...</span>
            </>
          )}
          {lastSaved && !isSaving && (
            <span>마지막 저장: {lastSaved.toLocaleTimeString()}</span>
          )}
        </div>
      </div>
      
      {/* 폼 필드들 */}
    </form>
  );
}
```

---

# 🎣 커스텀 훅 표준

## 1. API 연동 훅 패턴
```typescript
// src/hooks/useUsers.ts
export const useUsers = () => {
  const [users, setUsers] = useState<User[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const fetchUsers = useCallback(async (params?: SearchParams) => {
    try {
      setLoading(true);
      setError(null);
      const response = await userService.getUsers(params);
      setUsers(response.data);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }, []);

  useEffect(() => {
    fetchUsers();
  }, [fetchUsers]);

  return {
    users,
    loading,
    error,
    refetch: fetchUsers
  };
};
```

## 2. 페이지네이션 훅 패턴
```typescript
// src/hooks/usePagination.ts
export const usePagination = <T>(
  fetchFunction: (params: SearchParams) => Promise<PaginatedResponse<T>>,
  initialParams: SearchParams = {}
) => {
  const [data, setData] = useState<T[]>([]);
  const [pagination, setPagination] = useState<PageInfo>({
    currentPage: 1,
    totalPages: 1,
    totalItems: 0,
    itemsPerPage: 10
  });
  const [filters, setFilters] = useState(initialParams);
  const [loading, setLoading] = useState(false);

  const loadData = useCallback(async () => {
    try {
      setLoading(true);
      const response = await fetchFunction({
        ...filters,
        page: pagination.currentPage,
        limit: pagination.itemsPerPage
      });
      setData(response.data);
      setPagination(response.pageInfo);
    } catch (error) {
      console.error('데이터 로딩 실패:', error);
    } finally {
      setLoading(false);
    }
  }, [fetchFunction, filters, pagination.currentPage, pagination.itemsPerPage]);

  return {
    data,
    pagination,
    filters,
    loading,
    setFilters,
    setCurrentPage: (page: number) => 
      setPagination(prev => ({ ...prev, currentPage: page })),
    refetch: loadData
  };
};
```

---

# 🎨 Tailwind CSS 디자인 시스템

## 1. 색상 팔레트 표준
```typescript
// 주요 색상 시스템
const colors = {
  // 기본 그라데이션
  background: 'bg-gradient-to-br from-blue-50 via-indigo-50 to-purple-50',
  
  // 카드 및 컨테이너
  card: 'bg-white/90 backdrop-blur-sm',
  cardHover: 'hover:bg-white/95',
  
  // 버튼 스타일
  primary: 'bg-blue-600 hover:bg-blue-700 text-white',
  secondary: 'bg-gray-600 hover:bg-gray-700 text-white',
  danger: 'bg-red-600 hover:bg-red-700 text-white',
  outline: 'border-2 border-blue-600 text-blue-600 hover:bg-blue-50',
  
  // 텍스트
  textPrimary: 'text-gray-900',
  textSecondary: 'text-gray-600',
  textMuted: 'text-gray-400'
};
```

## 2. 스페이싱 및 레이아웃 표준
```typescript
// 표준 스페이싱
const spacing = {
  section: 'space-y-6',         // 섹션 간 간격
  card: 'p-6',                  // 카드 내부 패딩
  form: 'space-y-4',            // 폼 요소 간격
  button: 'px-4 py-2',          // 버튼 패딩
  modal: 'p-6 max-w-2xl',       // 모달 스타일
};

// 반응형 그리드
const grid = {
  responsive: 'grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6',
  table: 'grid grid-cols-1 gap-4 sm:gap-6',
  form: 'grid grid-cols-1 md:grid-cols-2 gap-4'
};
```

## 3. 애니메이션 및 트랜지션
```typescript
// 표준 트랜지션
const transitions = {
  default: 'transition-all duration-200 ease-in-out',
  hover: 'transform hover:scale-105 transition-transform duration-200',
  fade: 'transition-opacity duration-300',
  slide: 'transition-transform duration-300 ease-in-out'
};
```

---

# 🔒 TypeScript 타입 안전성 표준

## 1. API 응답 타입 정의
```typescript
// src/types/api.ts
export interface ApiResponse<T> {
  code: string;
  status_code: number;
  message: string;
  isLogin: boolean;
  data: T;
}

export interface PaginatedResponse<T> {
  items: T[];
  pageInfo: PageInfo;
}

export interface PageInfo {
  currentPage: number;
  totalPages: number;
  totalItems: number;
  itemsPerPage: number;
}
```

## 2. 도메인 모델 타입
```typescript
// src/types/index.ts
export interface User {
  id: string;
  email: string;
  name: string;
  phoneNumber?: string;
  isEmailVerified: boolean;
  createdAt: string;
  updatedAt: string;
}

export interface Role {
  id: string;
  name: string;
  description: string;
  priority: number;
  serviceId: string;
  createdAt: string;
  updatedAt: string;
}

export interface Permission {
  id: string;
  action: string;
  description: string;
  serviceId: string;
  createdAt: string;
  updatedAt: string;
}
```

## 3. 폼 데이터 타입
```typescript
// src/types/forms.ts
export interface LoginFormData {
  email: string;
  password: string;
}

export interface RegisterFormData {
  email: string;
  password: string;
  confirmPassword: string;
  name: string;
  phoneNumber?: string;
  agreeToTerms: boolean;
}

export interface RoleFormData {
  name: string;
  description: string;
  priority: number;
  serviceId: string;
}
```

---

# 🔌 서비스 레이어 패턴

## 1. API 서비스 구조
```typescript
// src/services/userService.ts
import { httpClient } from '@/lib/httpClient';
import type { User, CreateUserDto, UpdateUserDto, SearchParams } from '@/types';

export const userService = {
  async getUsers(params?: SearchParams): Promise<PaginatedResponse<User>> {
    const response = await apiClient.get('/users', { params });
    return response.data;
  },

  async getUserById(id: string): Promise<User> {
    const response = await apiClient.get(`/users/${id}`);
    return response.data;
  },

  async createUser(userData: CreateUserDto): Promise<void> {
    await apiClient.post('/users', userData);
  },

  async updateUser(id: string, userData: UpdateUserDto): Promise<void> {
    await apiClient.patch(`/users/${id}`, userData);
  },

  async deleteUser(id: string): Promise<void> {
    await apiClient.delete(`/users/${id}`);
  }
};
```

## 2. Axios 설정 표준
```typescript
// src/lib/axios.ts
import axios from 'axios';

export const apiClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000/api',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// 요청 인터셉터
apiClient.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('accessToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// 응답 인터셉터  
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // 토큰 만료 처리
      localStorage.removeItem('accessToken');
      window.location.href = '/auth/login';
    }
    return Promise.reject(error);
  }
);
```

---

# 📦 상태 관리 패턴 (Redux Toolkit)

## 1. 스토어 설정
```typescript
// src/store/index.ts
import { configureStore } from '@reduxjs/toolkit';
import authSlice from './slices/authSlice';
import userSlice from './slices/userSlice';
import roleSlice from './slices/roleSlice';
import permissionSlice from './slices/permissionSlice';
import serviceSlice from './slices/serviceSlice';

export const store = configureStore({
  reducer: {
    auth: authSlice,
    user: userSlice,
    role: roleSlice,
    permission: permissionSlice,
    service: serviceSlice,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: ['persist/PERSIST'],
      },
    }),
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

## 2. 비동기 액션 패턴
```typescript
// src/store/slices/authSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

// 비동기 액션
export const loginUser = createAsyncThunk(
  'auth/login',
  async (credentials: LoginRequest, { rejectWithValue }) => {
    try {
      const response = await authApi.post<ApiResponse<LoginResponse>>('/auth/login', credentials);
      const { accessToken, user } = response.data.data;
      
      tokenManager.setAccessToken(accessToken);
      return { accessToken, user };
    } catch (error: unknown) {
      const axiosError = error as { response?: { data?: { message?: string } } };
      return rejectWithValue(axiosError.response?.data?.message || '로그인에 실패했습니다.');
    }
  }
);

// 슬라이스 정의
const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    clearError: (state) => {
      state.error = null;
    },
    setUser: (state, action: PayloadAction<User>) => {
      state.user = action.payload;
      state.isAuthenticated = true;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(loginUser.pending, (state) => {
        state.isLoading = true;
        state.error = null;
      })
      .addCase(loginUser.fulfilled, (state, action) => {
        state.isLoading = false;
        state.user = convertLoggedInUserToUser(action.payload.user);
        state.accessToken = action.payload.accessToken;
        state.isAuthenticated = true;
        state.error = null;
      })
      .addCase(loginUser.rejected, (state, action) => {
        state.isLoading = false;
        state.error = action.payload as string;
      });
  },
});
```

## 3. 로컬 상태 관리
```typescript
// 컴포넌트 로컬 상태
const [users, setUsers] = useState<User[]>([]);
const [loading, setLoading] = useState(false);
const [error, setError] = useState<string | null>(null);
const [selectedUser, setSelectedUser] = useState<User | null>(null);

// UI 상태 관리
const [showModal, setShowModal] = useState(false);
const [activeTab, setActiveTab] = useState<'basic' | 'permissions'>('basic');
const [sidebarCollapsed, setSidebarCollapsed] = useState(false);
```

---

# 🚀 성능 최적화 표준

## 1. 메모이제이션 패턴
```typescript
// React.memo로 불필요한 리렌더링 방지
export const UserCard = React.memo<UserCardProps>(({ user, onEdit, onDelete }) => {
  return (
    <div className="bg-white/90 backdrop-blur-sm rounded-lg p-6">
      {/* 사용자 카드 내용 */}
    </div>
  );
});

// useMemo로 계산 비용이 큰 값 캐싱
const filteredUsers = useMemo(() => {
  return users.filter(user => 
    user.name.toLowerCase().includes(searchTerm.toLowerCase())
  );
}, [users, searchTerm]);

// useCallback으로 함수 참조 안정화
const handleUserUpdate = useCallback((userData: UpdateUserDto) => {
  return userService.updateUser(selectedUser.id, userData);
}, [selectedUser.id]);
```

## 2. 지연 로딩 패턴
```typescript
// 동적 import를 통한 컴포넌트 지연 로딩
const UserDetailModal = lazy(() => import('@/components/modals/UserDetailModal'));
const RolePermissionModal = lazy(() => import('@/components/modals/RolePermissionModal'));

// 조건부 렌더링으로 성능 최적화
{showModal && (
  <Suspense fallback={<LoadingSpinner />}>
    <UserDetailModal user={selectedUser} onClose={() => setShowModal(false)} />
  </Suspense>
)}
```

---

# 🛡️ 보안 패턴 (Next.js 미들웨어)

## 1. 보안 헤더 설정
```typescript
// src/middleware.ts
const securityHeaders = {
  // Content Security Policy
  'Content-Security-Policy': [
    "default-src 'self'",
    "script-src 'self' 'unsafe-inline' 'unsafe-eval'",
    "style-src 'self' 'unsafe-inline'",
    "img-src 'self' data: https:",
    "font-src 'self' data:",
    "connect-src 'self' https://localhost:8000 https://localhost:8100 https://*.krgeobuk.com",
    "frame-ancestors 'none'",
  ].join('; '),

  // XSS 방지
  'X-XSS-Protection': '1; mode=block',
  
  // MIME 타입 스니핑 방지
  'X-Content-Type-Options': 'nosniff',
  
  // 클릭재킹 방지
  'X-Frame-Options': 'DENY',
};
```

## 2. Rate Limiting 구현
```typescript
// Rate Limiting을 위한 간단한 메모리 저장소
const rateLimitMap = new Map<string, { count: number; lastReset: number }>();

function rateLimit(ip: string, maxRequests: number = 100, windowMs: number = 60000): boolean {
  const now = Date.now();
  const windowStart = now - windowMs;

  const current = rateLimitMap.get(ip) || { count: 0, lastReset: now };

  // 윈도우 시간이 지나면 리셋
  if (current.lastReset < windowStart) {
    current.count = 0;
    current.lastReset = now;
  }

  current.count++;
  rateLimitMap.set(ip, current);

  return current.count <= maxRequests;
}
```

## 3. CSRF 보호
```typescript
// CSRF 토큰 검증 (POST, PUT, DELETE 요청)
if (['POST', 'PUT', 'DELETE', 'PATCH'].includes(request.method)) {
  const origin = request.headers.get('origin');
  const referer = request.headers.get('referer');

  const allowedOrigins = [
    'http://localhost:3000',
    'https://localhost:3000',
    process.env.NEXT_PUBLIC_APP_URL,
  ].filter(Boolean);

  const isValidOrigin = allowedOrigins.some(
    (allowed) => origin === allowed || referer?.startsWith(allowed + '/')
  );

  if (!isValidOrigin) {
    return new NextResponse('Forbidden', { status: 403 });
  }
}
```

---

# 🎯 서비스 가시성 관리

## 서비스 가시성 플래그
서비스는 두 가지 플래그를 통해 포탈에서의 가시성을 제어합니다:

- **`isVisible`**: 포탈에서 표시 여부
- **`isVisibleByRole`**: 권한 기반 표시 여부

## 가시성 조건
- **비공개** (`isVisible = false`): 관리자만 볼 수 있음, 포탈 사용자는 접근 불가
- **공개** (`isVisible = true && isVisibleByRole = false`): 모든 포탈 사용자가 접근 가능
- **권한 기반** (`isVisible = true && isVisibleByRole = true`): 특정 역할을 가진 사용자만 접근 가능

---

# ✅ 개발 체크리스트

## 컴포넌트 개발
- [ ] TypeScript 타입 완전성 (모든 props, state 타입 정의)
- [ ] React.memo 적용 (불필요한 리렌더링 방지)
- [ ] 접근성 고려 (ARIA 라벨, 키보드 네비게이션)
- [ ] 반응형 디자인 (모바일 우선 접근법)
- [ ] 에러 바운더리 적용
- [ ] 로딩 상태 시각적 피드백
- [ ] 다크 모드 지원 (dark: 클래스 활용)

## 폼 개발
- [ ] React Hook Form 사용
- [ ] 실시간 검증 구현 (`mode: 'onChange'`)
- [ ] 로딩 상태 관리
- [ ] 에러 처리 및 사용자 피드백
- [ ] 접근성 (레이블, 에러 메시지)
- [ ] 폼 초기화 (reset 함수 활용)
- [ ] 검증 규칙 분리 및 재사용

## Redux 상태 관리
- [ ] createAsyncThunk를 통한 비동기 액션 구현
- [ ] 에러 상태 적절한 처리
- [ ] 로딩 상태 관리
- [ ] 타입 안전성 확보 (RootState, AppDispatch 활용)
- [ ] 불필요한 상태 중복 방지
- [ ] 액션 크리에이터 네이밍 일관성

## 성능 최적화
- [ ] 불필요한 리렌더링 방지 (memo, useMemo, useCallback)
- [ ] 지연 로딩 구현 (lazy loading)
- [ ] 번들 크기 최적화 (트리 셰이킹)
- [ ] 이미지 최적화 (Next.js Image 컴포넌트)
- [ ] 동적 import 활용 (큰 컴포넌트, 라이브러리)
- [ ] 메모이제이션 적절한 의존성 배열

## 보안
- [ ] Next.js 미들웨어를 통한 보안 헤더 설정
- [ ] Rate Limiting 구현
- [ ] CSRF 보호
- [ ] XSS 방지 (CSP 헤더)
- [ ] 인증 토큰 안전한 저장 및 관리
- [ ] 민감한 정보 클라이언트 노출 방지

## 코드 품질
- [ ] ESLint 규칙 준수
- [ ] TypeScript 엄격 모드 통과
- [ ] 컴포넌트 및 함수 네이밍 일관성
- [ ] 적절한 코드 분리 (관심사의 분리)
- [ ] 재사용 가능한 커스텀 훅 작성
- [ ] 에러 경계 구현

## 사용자 경험
- [ ] 로딩 상태 시각적 피드백
- [ ] 에러 상태 사용자 친화적 메시지
- [ ] 호버 및 포커스 상태 구현
- [ ] 키보드 네비게이션 지원
- [ ] 반응형 디자인 구현
- [ ] 다국어 지원 고려 (i18n)

## API 통신
- [ ] Axios 인터셉터를 통한 토큰 자동 첨부
- [ ] 응답 에러 처리
- [ ] API 응답 타입 정의
- [ ] 타임아웃 설정
- [ ] 재시도 로직 (필요한 경우)
- [ ] 캐싱 전략 구현

## 테스트
- [ ] 단위 테스트 작성
- [ ] 통합 테스트 작성 (주요 플로우)
- [ ] 접근성 테스트
- [ ] 성능 테스트 (Lighthouse)
- [ ] 크로스 브라우저 테스트
- [ ] 모바일 기기 테스트

---

# ⚡ 성능 최적화 심화

## 1. Bundle Analyzer 활용법

### Next.js Bundle 분석 설정
```typescript
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

/** @type {import('next').NextConfig} */
const nextConfig = {
  // 프로덕션 빌드 최적화
  experimental: {
    optimizeCss: true,
    swcMinify: true,
  },
  
  // 이미지 최적화 설정
  images: {
    domains: ['localhost', 'krgeobuk.com'],
    formats: ['image/webp', 'image/avif'],
  },
  
  // webpack 설정 최적화
  webpack: (config, { dev, isServer }) => {
    if (!dev && !isServer) {
      // 프로덕션 클라이언트 빌드 최적화
      config.optimization.splitChunks = {
        chunks: 'all',
        cacheGroups: {
          vendor: {
            test: /[\\/]node_modules[\\/]/,
            name: 'vendors',
            chunks: 'all',
          },
          common: {
            name: 'common',
            minChunks: 2,
            chunks: 'all',
            enforce: true,
          },
        },
      };
    }
    return config;
  },
  
  // 컴파일러 최적화
  compiler: {
    removeConsole: process.env.NODE_ENV === 'production',
  },
};

module.exports = withBundleAnalyzer(nextConfig);
```

### 번들 분석 스크립트
```json
// package.json
{
  "scripts": {
    "analyze": "ANALYZE=true npm run build",
    "analyze:server": "BUNDLE_ANALYZE=server npm run build",
    "analyze:browser": "BUNDLE_ANALYZE=browser npm run build"
  }
}
```

## 2. Core Web Vitals 최적화

### Web Vitals 측정 및 최적화
```typescript
// src/lib/webVitals.ts
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

interface WebVitalMetric {
  name: string;
  value: number;
  rating: 'good' | 'needs-improvement' | 'poor';
  delta: number;
  id: string;
}

function sendToAnalytics(metric: WebVitalMetric) {
  // 분석 도구로 전송 (Google Analytics, etc.)
  if (typeof window !== 'undefined' && window.gtag) {
    window.gtag('event', metric.name, {
      event_category: 'Web Vitals',
      value: Math.round(metric.name === 'CLS' ? metric.value * 1000 : metric.value),
      event_label: metric.id,
      non_interaction: true,
    });
  }
}

export function reportWebVitals() {
  getCLS(sendToAnalytics); // Cumulative Layout Shift
  getFID(sendToAnalytics); // First Input Delay  
  getFCP(sendToAnalytics); // First Contentful Paint
  getLCP(sendToAnalytics); // Largest Contentful Paint
  getTTFB(sendToAnalytics); // Time to First Byte
}

// app/layout.tsx에서 사용
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  useEffect(() => {
    reportWebVitals();
  }, []);

  return (
    <html lang="ko">
      <body>{children}</body>
    </html>
  );
}
```

### LCP 최적화 패턴
```typescript
// src/components/optimization/OptimizedHero.tsx
import { useState, useEffect } from 'react';
import Image from 'next/image';

interface OptimizedHeroProps {
  title: string;
  subtitle: string;
  imageUrl: string;
  priority?: boolean;
}

export function OptimizedHero({ title, subtitle, imageUrl, priority = true }: OptimizedHeroProps) {
  const [imageLoaded, setImageLoaded] = useState(false);

  return (
    <section className="relative h-screen flex items-center justify-center">
      {/* 배경 이미지 - LCP 최적화 */}
      <div className="absolute inset-0 z-0">
        <Image
          src={imageUrl}
          alt="Hero background"
          fill
          priority={priority} // LCP 요소는 priority 설정
          quality={85} // 적절한 품질로 파일 크기 최적화
          placeholder="blur"
          blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQABAAD/2wBDAAYEBQYFBAYGBQYHBwYIChAKCgkJChQODwwQFxQYGBcUFhYaHSUfGhsjHBYWICwgIyYnKSopGR8tMC0oMCUoKSj/2wBDAQcHBwoIChMKChMoGhYaKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCj/wAARCAABAAEDASIAAhEBAxEB/8QAFQABAQAAAAAAAAAAAAAAAAAAAAv/xAAUEAEAAAAAAAAAAAAAAAAAAAAA/8QAFQEBAQAAAAAAAAAAAAAAAAAAAAX/xAAUEQEAAAAAAAAAAAAAAAAAAAAA/9oADAMBAAIRAxEAPwCdABmX/9k="
          sizes="100vw"
          style={{ objectFit: 'cover' }}
          onLoad={() => setImageLoaded(true)}
        />
        
        {/* 로딩 오버레이 */}
        {!imageLoaded && (
          <div className="absolute inset-0 bg-gray-200 animate-pulse" />
        )}
      </div>

      {/* 콘텐츠 */}
      <div className="relative z-10 text-center text-white">
        <h1 className="text-5xl font-bold mb-4 drop-shadow-lg">
          {title}
        </h1>
        <p className="text-xl drop-shadow-md">
          {subtitle}
        </p>
      </div>
    </section>
  );
}
```

## 3. 메모리 누수 방지 패턴

### 이벤트 리스너 정리
```typescript
// src/hooks/useEventListener.ts
import { useEffect, useRef } from 'react';

export function useEventListener<T extends keyof WindowEventMap>(
  eventName: T,
  handler: (event: WindowEventMap[T]) => void,
  element?: Element | Window | null,
  options?: boolean | AddEventListenerOptions
) {
  const savedHandler = useRef<(event: WindowEventMap[T]) => void>();

  useEffect(() => {
    savedHandler.current = handler;
  }, [handler]);

  useEffect(() => {
    const targetElement = element ?? window;
    
    if (!targetElement?.addEventListener) {
      return;
    }

    const eventListener = (event: WindowEventMap[T]) => {
      savedHandler.current?.(event);
    };

    targetElement.addEventListener(eventName, eventListener as EventListener, options);

    return () => {
      targetElement.removeEventListener(eventName, eventListener as EventListener, options);
    };
  }, [eventName, element, options]);
}

// 사용 예제
function ScrollTracker() {
  const [scrollY, setScrollY] = useState(0);

  useEventListener('scroll', (e) => {
    setScrollY(window.scrollY);
  });

  return <div>Scroll position: {scrollY}</div>;
}
```

### 메모리 누수 감지 및 디버깅
```typescript
// src/utils/memoryTracker.ts
class MemoryTracker {
  private subscribers: Set<string> = new Set();
  private timers: Set<NodeJS.Timeout> = new Set();
  private abortControllers: Set<AbortController> = new Set();

  // 구독 추적
  addSubscriber(id: string) {
    this.subscribers.add(id);
    if (process.env.NODE_ENV === 'development') {
      console.log(`Subscriber added: ${id}. Total: ${this.subscribers.size}`);
    }
  }

  removeSubscriber(id: string) {
    this.subscribers.delete(id);
    if (process.env.NODE_ENV === 'development') {
      console.log(`Subscriber removed: ${id}. Total: ${this.subscribers.size}`);
    }
  }

  // 타이머 추적
  addTimer(timer: NodeJS.Timeout) {
    this.timers.add(timer);
  }

  clearTimer(timer: NodeJS.Timeout) {
    clearTimeout(timer);
    this.timers.delete(timer);
  }

  // AbortController 추적
  addAbortController(controller: AbortController) {
    this.abortControllers.add(controller);
  }

  removeAbortController(controller: AbortController) {
    this.abortControllers.delete(controller);
  }

  // 모든 리소스 정리
  cleanup() {
    // 모든 타이머 정리
    this.timers.forEach(timer => clearTimeout(timer));
    this.timers.clear();

    // 모든 요청 취소
    this.abortControllers.forEach(controller => controller.abort());
    this.abortControllers.clear();

    // 구독자 목록 정리
    this.subscribers.clear();
  }

  // 메모리 사용량 리포트
  getMemoryReport() {
    if (typeof window !== 'undefined' && 'performance' in window) {
      const memory = (performance as any).memory;
      return {
        subscribers: this.subscribers.size,
        timers: this.timers.size,
        abortControllers: this.abortControllers.size,
        memoryUsage: memory ? {
          used: Math.round(memory.usedJSHeapSize / 1024 / 1024),
          total: Math.round(memory.totalJSHeapSize / 1024 / 1024),
          limit: Math.round(memory.jsHeapSizeLimit / 1024 / 1024),
        } : null
      };
    }
    return null;
  }
}

export const memoryTracker = new MemoryTracker();

// React 컴포넌트에서 사용
export function useMemoryTracker(componentName: string) {
  useEffect(() => {
    memoryTracker.addSubscriber(componentName);

    return () => {
      memoryTracker.removeSubscriber(componentName);
    };
  }, [componentName]);
}
```

## 4. 런타임 성능 모니터링

### 성능 메트릭 수집기
```typescript
// src/lib/performanceMonitor.ts
interface PerformanceMetrics {
  renderTime: number;
  apiResponseTime: number;
  memoryUsage: number;
  componentRenderCount: number;
}

class PerformanceMonitor {
  private metrics: Map<string, PerformanceMetrics> = new Map();
  private renderCounts: Map<string, number> = new Map();

  // 렌더링 시간 측정
  measureRenderTime(componentName: string, renderFn: () => void) {
    const startTime = performance.now();
    renderFn();
    const endTime = performance.now();
    
    const renderTime = endTime - startTime;
    this.updateMetric(componentName, 'renderTime', renderTime);
    
    // 렌더 카운트 증가
    const currentCount = this.renderCounts.get(componentName) || 0;
    this.renderCounts.set(componentName, currentCount + 1);
    
    if (renderTime > 16) { // 16ms (60fps) 초과시 경고
      console.warn(`Slow render detected: ${componentName} took ${renderTime.toFixed(2)}ms`);
    }
  }

  // API 응답 시간 측정
  async measureApiCall<T>(apiCall: () => Promise<T>): Promise<T> {
    const startTime = performance.now();
    try {
      const result = await apiCall();
      const endTime = performance.now();
      const responseTime = endTime - startTime;
      
      this.updateMetric('api', 'apiResponseTime', responseTime);
      
      if (responseTime > 1000) { // 1초 초과시 경고
        console.warn(`Slow API call detected: took ${responseTime.toFixed(2)}ms`);
      }
      
      return result;
    } catch (error) {
      const endTime = performance.now();
      const responseTime = endTime - startTime;
      this.updateMetric('api', 'apiResponseTime', responseTime);
      throw error;
    }
  }

  // 메모리 사용량 모니터링
  trackMemoryUsage() {
    if (typeof window !== 'undefined' && 'performance' in window) {
      const memory = (performance as any).memory;
      if (memory) {
        const usedMB = memory.usedJSHeapSize / 1024 / 1024;
        this.updateMetric('memory', 'memoryUsage', usedMB);
        
        if (usedMB > 100) { // 100MB 초과시 경고
          console.warn(`High memory usage detected: ${usedMB.toFixed(2)}MB`);
        }
      }
    }
  }

  private updateMetric(component: string, metric: keyof PerformanceMetrics, value: number) {
    const existing = this.metrics.get(component) || {
      renderTime: 0,
      apiResponseTime: 0,
      memoryUsage: 0,
      componentRenderCount: 0
    };
    
    existing[metric] = value;
    if (metric === 'renderTime') {
      existing.componentRenderCount = this.renderCounts.get(component) || 0;
    }
    
    this.metrics.set(component, existing);
  }

  // 성능 리포트 생성
  generateReport() {
    const report = {
      timestamp: new Date().toISOString(),
      metrics: Object.fromEntries(this.metrics),
      topSlowComponents: this.getTopSlowComponents(),
      memoryPressure: this.getMemoryPressure(),
    };
    
    return report;
  }

  private getTopSlowComponents() {
    return Array.from(this.metrics.entries())
      .sort(([, a], [, b]) => b.renderTime - a.renderTime)
      .slice(0, 5)
      .map(([name, metrics]) => ({ name, renderTime: metrics.renderTime }));
  }

  private getMemoryPressure() {
    if (typeof window !== 'undefined' && 'performance' in window) {
      const memory = (performance as any).memory;
      if (memory) {
        const usage = memory.usedJSHeapSize / memory.jsHeapSizeLimit;
        return {
          level: usage > 0.9 ? 'high' : usage > 0.7 ? 'medium' : 'low',
          percentage: Math.round(usage * 100),
          usedMB: Math.round(memory.usedJSHeapSize / 1024 / 1024),
          limitMB: Math.round(memory.jsHeapSizeLimit / 1024 / 1024),
        };
      }
    }
    return null;
  }
}

export const performanceMonitor = new PerformanceMonitor();

// React 훅으로 사용
export function usePerformanceMonitor(componentName: string) {
  const renderCountRef = useRef(0);
  
  useEffect(() => {
    renderCountRef.current++;
    
    if (renderCountRef.current > 10) { // 10회 이상 렌더링시 경고
      console.warn(`Component ${componentName} has rendered ${renderCountRef.current} times`);
    }
  });

  const measureRender = useCallback((renderFn: () => void) => {
    performanceMonitor.measureRenderTime(componentName, renderFn);
  }, [componentName]);

  return { measureRender };
}
```

---

# 🛠️ 개발 환경 및 도구

## 1. 환경 변수 관리 전략

### 환경별 설정 파일
```typescript
// src/config/environment.ts
interface EnvironmentConfig {
  apiUrl: string;
  authServerUrl: string;
  authzServerUrl: string;
  portalServerUrl: string;
  googleClientId: string;
  naverClientId: string;
  enableDebug: boolean;
  enableAnalytics: boolean;
}

const environments: Record<string, EnvironmentConfig> = {
  development: {
    apiUrl: 'http://localhost:3000/api',
    authServerUrl: 'http://localhost:8000',
    authzServerUrl: 'http://localhost:8100',
    portalServerUrl: 'http://localhost:8200',
    googleClientId: process.env.NEXT_PUBLIC_GOOGLE_CLIENT_ID_DEV!,
    naverClientId: process.env.NEXT_PUBLIC_NAVER_CLIENT_ID_DEV!,
    enableDebug: true,
    enableAnalytics: false,
  },
  staging: {
    apiUrl: 'https://staging-api.krgeobuk.com',
    authServerUrl: 'https://staging-auth.krgeobuk.com',
    authzServerUrl: 'https://staging-authz.krgeobuk.com',
    portalServerUrl: 'https://staging-portal.krgeobuk.com',
    googleClientId: process.env.NEXT_PUBLIC_GOOGLE_CLIENT_ID_STAGING!,
    naverClientId: process.env.NEXT_PUBLIC_NAVER_CLIENT_ID_STAGING!,
    enableDebug: true,
    enableAnalytics: true,
  },
  production: {
    apiUrl: 'https://api.krgeobuk.com',
    authServerUrl: 'https://auth.krgeobuk.com',
    authzServerUrl: 'https://authz.krgeobuk.com',
    portalServerUrl: 'https://portal.krgeobuk.com',
    googleClientId: process.env.NEXT_PUBLIC_GOOGLE_CLIENT_ID_PROD!,
    naverClientId: process.env.NEXT_PUBLIC_NAVER_CLIENT_ID_PROD!,
    enableDebug: false,
    enableAnalytics: true,
  },
};

export function getEnvironmentConfig(): EnvironmentConfig {
  const env = process.env.NODE_ENV || 'development';
  const stage = process.env.NEXT_PUBLIC_STAGE || env;
  
  const config = environments[stage];
  if (!config) {
    throw new Error(`Unknown environment: ${stage}`);
  }
  
  return config;
}

export const config = getEnvironmentConfig();
```

### 환경 변수 검증
```typescript
// src/lib/envValidation.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'staging', 'production']),
  NEXT_PUBLIC_APP_URL: z.string().url(),
  NEXT_PUBLIC_STAGE: z.enum(['development', 'staging', 'production']).optional(),
  
  // API URLs
  NEXT_PUBLIC_AUTH_SERVER_URL: z.string().url(),
  NEXT_PUBLIC_AUTHZ_SERVER_URL: z.string().url(),
  NEXT_PUBLIC_PORTAL_SERVER_URL: z.string().url(),
  
  // OAuth 설정
  NEXT_PUBLIC_GOOGLE_CLIENT_ID_DEV: z.string().optional(),
  NEXT_PUBLIC_GOOGLE_CLIENT_ID_STAGING: z.string().optional(),
  NEXT_PUBLIC_GOOGLE_CLIENT_ID_PROD: z.string().optional(),
  
  NEXT_PUBLIC_NAVER_CLIENT_ID_DEV: z.string().optional(),
  NEXT_PUBLIC_NAVER_CLIENT_ID_STAGING: z.string().optional(),
  NEXT_PUBLIC_NAVER_CLIENT_ID_PROD: z.string().optional(),
  
  // 분석 도구
  NEXT_PUBLIC_GA_MEASUREMENT_ID: z.string().optional(),
  NEXT_PUBLIC_SENTRY_DSN: z.string().optional(),
});

export function validateEnvironment() {
  try {
    return envSchema.parse(process.env);
  } catch (error) {
    console.error('Environment validation failed:', error);
    throw new Error('Invalid environment configuration');
  }
}

// 앱 시작시 검증
if (typeof window === 'undefined') {
  validateEnvironment();
}
```

## 2. krgeobuk ESLint 설정 활용

### ESLint 확장 설정
```json
// .eslintrc.json
{
  "extends": [
    "@krgeobuk/eslint-config/next",
    "@krgeobuk/eslint-config/typescript"
  ],
  "rules": {
    // krgeobuk 특화 규칙 오버라이드
    "@krgeobuk/prefer-named-exports": "error",
    "@krgeobuk/no-unused-props": "error",
    "@krgeobuk/component-naming-convention": "error",
    
    // Next.js 특화 규칙
    "@next/next/no-img-element": "error",
    "@next/next/no-html-link-for-pages": "error",
    
    // React Hook 규칙 강화
    "react-hooks/exhaustive-deps": "error",
    "react-hooks/rules-of-hooks": "error",
    
    // 성능 관련 규칙
    "react/jsx-no-leaked-render": "error",
    "react/no-unstable-nested-components": "error"
  },
  "overrides": [
    {
      "files": ["*.tsx", "*.jsx"],
      "rules": {
        "@krgeobuk/component-props-interface": "error",
        "@krgeobuk/prefer-function-component": "error"
      }
    },
    {
      "files": ["src/pages/**/*.tsx", "src/app/**/*.tsx"],
      "rules": {
        "@krgeobuk/page-component-naming": "error"
      }
    }
  ]
}
```

### Prettier 통합 설정
```json
// .prettierrc.json
{
  "extends": "@krgeobuk/prettier-config",
  "overrides": [
    {
      "files": "*.tsx",
      "options": {
        "jsxSingleQuote": false,
        "jsxBracketSameLine": false
      }
    }
  ]
}
```

## 3. 디버깅 도구 통합

### Redux DevTools 향상
```typescript
// src/store/index.ts
import { configureStore } from '@reduxjs/toolkit';
import { enableMapSet } from 'immer';

// Immer Map/Set 지원 활성화
enableMapSet();

export const store = configureStore({
  reducer: {
    // reducers...
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: ['persist/PERSIST', 'persist/REHYDRATE'],
        ignoredPaths: ['register'],
      },
    }).concat(
      // 개발 환경에서만 로깅 미들웨어 추가
      process.env.NODE_ENV === 'development' ? [logger] : []
    ),
  devTools: process.env.NODE_ENV !== 'production' && {
    trace: true,
    traceLimit: 25,
    actionsBlacklist: ['@@redux-form/CHANGE'],
  },
});

// 개발 환경 디버깅 헬퍼
if (process.env.NODE_ENV === 'development' && typeof window !== 'undefined') {
  (window as any).__KRGEOBUK_STORE__ = store;
}
```

### 컴포넌트 디버깅 도구
```typescript
// src/components/debug/ComponentDebugger.tsx
import { useState, useEffect } from 'react';

interface ComponentDebuggerProps {
  componentName: string;
  props?: Record<string, any>;
  enabled?: boolean;
}

export function ComponentDebugger({ 
  componentName, 
  props = {}, 
  enabled = process.env.NODE_ENV === 'development' 
}: ComponentDebuggerProps) {
  const [renderCount, setRenderCount] = useState(0);
  const [propsHistory, setPropsHistory] = useState<Record<string, any>[]>([]);

  useEffect(() => {
    setRenderCount(prev => prev + 1);
    setPropsHistory(prev => [...prev.slice(-4), props]);
  });

  if (!enabled) return null;

  return (
    <div className="fixed bottom-4 right-4 bg-black/80 text-white p-3 rounded-lg text-xs font-mono max-w-sm">
      <div className="font-bold text-green-400">{componentName}</div>
      <div>Renders: {renderCount}</div>
      <details className="mt-2">
        <summary className="cursor-pointer">Props History</summary>
        <div className="mt-1 max-h-32 overflow-y-auto">
          {propsHistory.map((propSnapshot, index) => (
            <div key={index} className="border-t border-gray-600 pt-1 mt-1">
              <div className="text-yellow-400">Render #{index + 1}:</div>
              <pre className="text-xs">{JSON.stringify(propSnapshot, null, 1)}</pre>
            </div>
          ))}
        </div>
      </details>
    </div>
  );
}

// 사용 예제
function MyComponent({ userId, settings }: MyComponentProps) {
  return (
    <>
      <ComponentDebugger 
        componentName="MyComponent"
        props={{ userId, settings }}
      />
      {/* 컴포넌트 내용 */}
    </>
  );
}
```

## 4. 개발 워크플로우 최적화

### 개발 서버 최적화
```typescript
// next.config.js 개발 환경 최적화
/** @type {import('next').NextConfig} */
const nextConfig = {
  // 개발 환경에서 빠른 새로고침
  fastRefresh: true,
  
  // 타입 체킹 최적화
  typescript: {
    // 프로덕션 빌드에서만 타입 검사 수행
    ignoreBuildErrors: process.env.NODE_ENV === 'development',
  },
  
  // ESLint 최적화
  eslint: {
    // 개발 중에는 ESLint 무시, CI/CD에서 별도 체크
    ignoreDuringBuilds: process.env.NODE_ENV === 'development',
  },
  
  // webpack 개발 서버 최적화
  webpack: (config, { dev }) => {
    if (dev) {
      // 개발 환경에서 더 빠른 빌드를 위한 설정
      config.watchOptions = {
        poll: 1000,
        aggregateTimeout: 300,
      };
    }
    
    return config;
  },
};

module.exports = nextConfig;
```

### Git 훅 설정
```json
// package.json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "commit-msg": "commitlint -E HUSKY_GIT_PARAMS",
      "pre-push": "npm run type-check && npm run test:ci"
    }
  },
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write",
      "git add"
    ],
    "*.{css,scss}": [
      "prettier --write",
      "git add"
    ]
  },
  "commitlint": {
    "extends": ["@commitlint/config-conventional"]
  }
}
```

---

# 🚀 배포 및 모니터링

## 1. 빌드 최적화 전략

### 프로덕션 빌드 설정
```dockerfile
# Dockerfile
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine AS runner
WORKDIR /app
ENV NODE_ENV production

RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT 3000

CMD ["node", "server.js"]
```

### CI/CD 파이프라인
```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm run test:ci
      
      - name: Type check
        run: npm run type-check
      
      - name: Lint
        run: npm run lint
      
      - name: Build
        run: npm run build

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to staging
        run: |
          # 스테이징 배포 스크립트
          
      - name: Run E2E tests
        run: npm run test:e2e
        
      - name: Deploy to production
        if: success()
        run: |
          # 프로덕션 배포 스크립트
```

## 2. 운영 모니터링 설정

### 에러 추적 설정 (Sentry)
```typescript
// src/lib/monitoring.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  
  // 성능 모니터링
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  
  // 세션 리플레이
  replaysSessionSampleRate: 0.01,
  replaysOnErrorSampleRate: 1.0,
  
  beforeSend(event) {
    // 개발 환경에서는 콘솔에도 출력
    if (process.env.NODE_ENV === 'development') {
      console.error('Sentry Event:', event);
    }
    return event;
  },
  
  integrations: [
    new Sentry.Replay({
      maskAllText: false,
      blockAllMedia: false,
    }),
  ],
});
```

### 사용자 분석 설정 (Google Analytics)
```typescript
// src/lib/analytics.ts
declare global {
  interface Window {
    gtag: (...args: any[]) => void;
  }
}

export const GA_MEASUREMENT_ID = process.env.NEXT_PUBLIC_GA_MEASUREMENT_ID;

// 페이지뷰 추적
export const pageview = (url: string) => {
  if (typeof window !== 'undefined' && GA_MEASUREMENT_ID) {
    window.gtag('config', GA_MEASUREMENT_ID, {
      page_path: url,
    });
  }
};

// 이벤트 추적
export const event = ({ action, category, label, value }: {
  action: string;
  category: string;
  label?: string;
  value?: number;
}) => {
  if (typeof window !== 'undefined' && GA_MEASUREMENT_ID) {
    window.gtag('event', action, {
      event_category: category,
      event_label: label,
      value: value,
    });
  }
};

// 사용자 행동 추적
export const trackUserAction = (action: string, details?: Record<string, any>) => {
  event({
    action,
    category: 'User Interaction',
    label: JSON.stringify(details),
  });
};

// 앱 사용 예제
export function useAnalytics() {
  const router = useRouter();
  
  useEffect(() => {
    const handleRouteChange = (url: string) => {
      pageview(url);
    };
    
    router.events.on('routeChangeComplete', handleRouteChange);
    return () => {
      router.events.off('routeChangeComplete', handleRouteChange);
    };
  }, [router.events]);
  
  return {
    trackUserAction,
    trackEvent: event,
  };
}
```

## 3. 헬스체크 및 상태 모니터링

### API 상태 모니터링
```typescript
// src/lib/healthCheck.ts
interface ServiceStatus {
  name: string;
  status: 'healthy' | 'unhealthy' | 'degraded';
  responseTime: number;
  lastChecked: Date;
  error?: string;
}

class HealthChecker {
  private services: Map<string, ServiceStatus> = new Map();
  private checkInterval: NodeJS.Timeout | null = null;

  constructor(private intervalMs = 60000) { // 1분마다 체크
    this.startMonitoring();
  }

  async checkService(name: string, url: string): Promise<ServiceStatus> {
    const startTime = performance.now();
    
    try {
      const response = await fetch(`${url}/health`, {
        method: 'GET',
        timeout: 5000, // 5초 타임아웃
      });
      
      const endTime = performance.now();
      const responseTime = endTime - startTime;
      
      const status: ServiceStatus = {
        name,
        status: response.ok ? 'healthy' : 'unhealthy',
        responseTime,
        lastChecked: new Date(),
      };
      
      if (!response.ok) {
        status.error = `HTTP ${response.status}: ${response.statusText}`;
      }
      
      this.services.set(name, status);
      return status;
      
    } catch (error) {
      const endTime = performance.now();
      const responseTime = endTime - startTime;
      
      const status: ServiceStatus = {
        name,
        status: 'unhealthy',
        responseTime,
        lastChecked: new Date(),
        error: error instanceof Error ? error.message : 'Unknown error',
      };
      
      this.services.set(name, status);
      return status;
    }
  }

  async checkAllServices(): Promise<ServiceStatus[]> {
    const { config } = await import('@/config/environment');
    
    const checks = await Promise.allSettled([
      this.checkService('auth-server', config.authServerUrl),
      this.checkService('authz-server', config.authzServerUrl),
      this.checkService('portal-server', config.portalServerUrl),
    ]);
    
    return checks
      .filter((result): result is PromiseFulfilledResult<ServiceStatus> => 
        result.status === 'fulfilled'
      )
      .map(result => result.value);
  }

  getServiceStatus(name: string): ServiceStatus | undefined {
    return this.services.get(name);
  }

  getAllStatuses(): ServiceStatus[] {
    return Array.from(this.services.values());
  }

  getOverallHealth(): 'healthy' | 'degraded' | 'unhealthy' {
    const statuses = this.getAllStatuses();
    if (statuses.length === 0) return 'unhealthy';
    
    const unhealthy = statuses.filter(s => s.status === 'unhealthy').length;
    const total = statuses.length;
    
    if (unhealthy === 0) return 'healthy';
    if (unhealthy < total) return 'degraded';
    return 'unhealthy';
  }

  private startMonitoring() {
    this.checkInterval = setInterval(() => {
      this.checkAllServices();
    }, this.intervalMs);
  }

  destroy() {
    if (this.checkInterval) {
      clearInterval(this.checkInterval);
      this.checkInterval = null;
    }
  }
}

export const healthChecker = new HealthChecker();

// React 훅으로 사용
export function useServiceHealth() {
  const [statuses, setStatuses] = useState<ServiceStatus[]>([]);
  
  useEffect(() => {
    const updateStatuses = () => {
      setStatuses(healthChecker.getAllStatuses());
    };
    
    // 초기 상태 설정
    updateStatuses();
    
    // 주기적 업데이트
    const interval = setInterval(updateStatuses, 10000); // 10초마다
    
    return () => clearInterval(interval);
  }, []);
  
  return {
    statuses,
    overallHealth: healthChecker.getOverallHealth(),
    refresh: () => healthChecker.checkAllServices(),
  };
}
```

---

# 🔗 API 응답 포맷 통합

portal-client는 krgeobuk 백엔드 서비스들과의 일관된 통신을 위해 표준화된 API 응답 포맷을 사용합니다.

상세한 API 응답 포맷 표준은 **KRGEOBUK_NESTJS_SERVER_GUIDE.md**의 **"API 응답 포맷 표준"** 섹션을 참조하세요.

## 클라이언트 사이드 구현
```typescript
// API 응답 타입 활용
interface ApiResponse<T> {
  code: string;
  status_code: number;
  message: string;
  isLogin: boolean;
  data: T;
}

// 서비스에서 응답 처리
const response = await apiClient.get<ApiResponse<User[]>>('/users');
const users = response.data.data; // 실제 데이터 추출
```

---

# 🔐 완전한 인증 시스템

## 1. OAuth 통합 완전 플로우

### Google OAuth 구현
```typescript
// src/lib/oauth/googleOAuth.ts
import { KRGEOBUK_SERVICES } from '@/config/services';

export class GoogleOAuthService {
  private clientId: string;
  private redirectUri: string;

  constructor() {
    this.clientId = process.env.NEXT_PUBLIC_GOOGLE_CLIENT_ID!;
    this.redirectUri = `${window.location.origin}/auth/oauth/google/callback`;
  }

  // OAuth 인증 URL 생성
  getAuthUrl(): string {
    const params = new URLSearchParams({
      client_id: this.clientId,
      redirect_uri: this.redirectUri,
      response_type: 'code',
      scope: 'openid email profile',
      state: this.generateState(),
      prompt: 'consent'
    });

    return `https://accounts.google.com/oauth/authorize?${params.toString()}`;
  }

  // 인증 페이지로 리다이렉트
  initiateLogin(): void {
    const authUrl = this.getAuthUrl();
    window.location.href = authUrl;
  }

  // 인증 콜백 처리
  async handleCallback(code: string, state: string): Promise<{
    accessToken: string;
    user: any;
  }> {
    // 상태 검증
    if (!this.validateState(state)) {
      throw new Error('Invalid state parameter');
    }

    // auth-server에 인증 코드 전송
    const response = await fetch(`${KRGEOBUK_SERVICES.AUTH_SERVER.baseUrl}/auth/oauth/google`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        code,
        redirectUri: this.redirectUri,
      }),
    });

    if (!response.ok) {
      throw new Error('OAuth authentication failed');
    }

    return response.json();
  }

  private generateState(): string {
    const state = Math.random().toString(36).substring(2, 15);
    sessionStorage.setItem('oauth_state', state);
    return state;
  }

  private validateState(state: string): boolean {
    const storedState = sessionStorage.getItem('oauth_state');
    sessionStorage.removeItem('oauth_state');
    return storedState === state;
  }
}

export const googleOAuth = new GoogleOAuthService();
```

### Naver OAuth 구현
```typescript
// src/lib/oauth/naverOAuth.ts
export class NaverOAuthService {
  private clientId: string;
  private redirectUri: string;

  constructor() {
    this.clientId = process.env.NEXT_PUBLIC_NAVER_CLIENT_ID!;
    this.redirectUri = `${window.location.origin}/auth/oauth/naver/callback`;
  }

  getAuthUrl(): string {
    const params = new URLSearchParams({
      response_type: 'code',
      client_id: this.clientId,
      redirect_uri: this.redirectUri,
      state: this.generateState(),
    });

    return `https://nid.naver.com/oauth2.0/authorize?${params.toString()}`;
  }

  initiateLogin(): void {
    window.location.href = this.getAuthUrl();
  }

  async handleCallback(code: string, state: string): Promise<{
    accessToken: string;
    user: any;
  }> {
    if (!this.validateState(state)) {
      throw new Error('Invalid state parameter');
    }

    const response = await fetch(`${KRGEOBUK_SERVICES.AUTH_SERVER.baseUrl}/auth/oauth/naver`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        code,
        state,
        redirectUri: this.redirectUri,
      }),
    });

    if (!response.ok) {
      throw new Error('OAuth authentication failed');
    }

    return response.json();
  }

  private generateState(): string {
    const state = Math.random().toString(36).substring(2, 15);
    sessionStorage.setItem('naver_oauth_state', state);
    return state;
  }

  private validateState(state: string): boolean {
    const storedState = sessionStorage.getItem('naver_oauth_state');
    sessionStorage.removeItem('naver_oauth_state');
    return storedState === state;
  }
}

export const naverOAuth = new NaverOAuthService();
```

## 2. 토큰 갱신 자동화 패턴

### 고급 토큰 관리자
```typescript
// src/lib/tokenManager.ts
type TokenEventType = 'tokenSet' | 'tokenRefreshed' | 'tokenCleared';
type TokenEventCallback = (token?: string) => void;

export class AdvancedTokenManager {
  private accessToken: string | null = null;
  private refreshTimer: NodeJS.Timeout | null = null;
  private eventListeners: Map<TokenEventType, Set<TokenEventCallback>> = new Map();
  private isRefreshing = false;
  private refreshPromise: Promise<string> | null = null;

  constructor() {
    this.initEventListeners();
  }

  private initEventListeners() {
    this.eventListeners.set('tokenSet', new Set());
    this.eventListeners.set('tokenRefreshed', new Set());
    this.eventListeners.set('tokenCleared', new Set());
  }

  // 이벤트 리스너 등록
  on(event: TokenEventType, callback: TokenEventCallback): () => void {
    const listeners = this.eventListeners.get(event);
    if (listeners) {
      listeners.add(callback);
    }

    // 구독 해제 함수 반환
    return () => {
      const listeners = this.eventListeners.get(event);
      if (listeners) {
        listeners.delete(callback);
      }
    };
  }

  private emit(event: TokenEventType, token?: string) {
    const listeners = this.eventListeners.get(event);
    if (listeners) {
      listeners.forEach(callback => callback(token));
    }
  }

  // JWT 토큰 디코딩
  private decodeJWT(token: string): any {
    try {
      const base64Url = token.split('.')[1];
      const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/');
      const jsonPayload = decodeURIComponent(
        atob(base64)
          .split('')
          .map(c => '%' + ('00' + c.charCodeAt(0).toString(16)).slice(-2))
          .join('')
      );
      return JSON.parse(jsonPayload);
    } catch {
      return null;
    }
  }

  // 토큰 만료 시간 확인
  private getTokenExpiry(token: string): number | null {
    const decoded = this.decodeJWT(token);
    return decoded?.exp ? decoded.exp * 1000 : null;
  }

  // 자동 갱신 타이머 설정
  private scheduleRefresh(token: string) {
    this.clearRefreshTimer();

    const expiry = this.getTokenExpiry(token);
    if (!expiry) return;

    // 만료 5분 전에 갱신 시도
    const refreshTime = expiry - Date.now() - 5 * 60 * 1000;
    
    if (refreshTime > 0) {
      this.refreshTimer = setTimeout(() => {
        this.refreshToken();
      }, refreshTime);
    }
  }

  private clearRefreshTimer() {
    if (this.refreshTimer) {
      clearTimeout(this.refreshTimer);
      this.refreshTimer = null;
    }
  }

  // 토큰 설정
  setAccessToken(token: string): void {
    this.accessToken = token;
    this.scheduleRefresh(token);
    this.emit('tokenSet', token);
  }

  // 이벤트 없이 토큰 설정 (순환 참조 방지)
  setAccessTokenSilent(token: string): void {
    this.accessToken = token;
    this.scheduleRefresh(token);
  }

  // 토큰 가져오기
  getAccessToken(): string | null {
    return this.accessToken;
  }

  // 토큰 갱신
  async refreshToken(): Promise<string> {
    // 이미 갱신 중인 경우 기존 Promise 반환
    if (this.isRefreshing && this.refreshPromise) {
      return this.refreshPromise;
    }

    this.isRefreshing = true;
    
    this.refreshPromise = (async () => {
      try {
        const response = await fetch(`${KRGEOBUK_SERVICES.AUTH_SERVER.baseUrl}/auth/refresh`, {
          method: 'POST',
          credentials: 'include', // 쿠키에 있는 refresh token 포함
          headers: {
            'Content-Type': 'application/json',
          },
        });

        if (!response.ok) {
          throw new Error('Token refresh failed');
        }

        const data = await response.json();
        const newToken = data.data.accessToken;
        
        this.setAccessTokenSilent(newToken);
        this.emit('tokenRefreshed', newToken);
        
        return newToken;
      } catch (error) {
        this.clearAccessToken();
        throw error;
      } finally {
        this.isRefreshing = false;
        this.refreshPromise = null;
      }
    })();

    return this.refreshPromise;
  }

  // 토큰 검증
  isTokenValid(): boolean {
    if (!this.accessToken) return false;

    const expiry = this.getTokenExpiry(this.accessToken);
    if (!expiry) return false;

    // 1분의 여유를 둠
    return Date.now() < expiry - 60 * 1000;
  }

  // 토큰 정리
  clearAccessToken(): void {
    this.accessToken = null;
    this.clearRefreshTimer();
    this.emit('tokenCleared');
  }

  // 완전 정리
  dispose(): void {
    this.clearAccessToken();
    this.eventListeners.clear();
  }
}

export const tokenManager = new AdvancedTokenManager();
```

## 3. 역할 기반 렌더링 구현

### 권한 기반 컴포넌트
```typescript
// src/components/auth/ProtectedComponent.tsx
import { ReactNode } from 'react';
import { useSelector } from 'react-redux';
import { RootState } from '@/store';
import { checkServiceAccess } from '@/store/slices/integrationSlice';
import { useAppDispatch } from '@/hooks/redux';

interface ProtectedComponentProps {
  children: ReactNode;
  requiredPermissions?: string[];
  requiredRoles?: string[];
  serviceId?: string;
  fallback?: ReactNode;
  mode?: 'any' | 'all'; // 권한 확인 모드
}

export function ProtectedComponent({
  children,
  requiredPermissions = [],
  requiredRoles = [],
  serviceId,
  fallback = null,
  mode = 'any'
}: ProtectedComponentProps) {
  const dispatch = useAppDispatch();
  const { user, isAuthenticated } = useSelector((state: RootState) => state.auth);
  const { userServiceAccess } = useSelector((state: RootState) => state.integration);

  // 인증되지 않은 경우
  if (!isAuthenticated || !user) {
    return <>{fallback}</>;
  }

  // 서비스 접근 권한 확인
  if (serviceId) {
    const serviceAccess = userServiceAccess[serviceId];
    
    if (!serviceAccess) {
      // 권한 정보가 없으면 확인 요청
      dispatch(checkServiceAccess({ userId: user.id, serviceId }));
      return <div className="animate-pulse">권한 확인 중...</div>;
    }

    if (!serviceAccess.hasAccess) {
      return <>{fallback}</>;
    }

    // 특정 권한 확인
    if (requiredPermissions.length > 0) {
      const hasRequiredPermissions = mode === 'all'
        ? requiredPermissions.every(permission => 
            serviceAccess.permissions.includes(permission)
          )
        : requiredPermissions.some(permission => 
            serviceAccess.permissions.includes(permission)
          );

      if (!hasRequiredPermissions) {
        return <>{fallback}</>;
      }
    }
  }

  // 역할 확인 (사용자의 역할 정보가 있는 경우)
  if (requiredRoles.length > 0) {
    // 실제 구현에서는 사용자의 역할 정보를 Redux에서 가져와야 함
    const userRoles = user.roles || [];
    const hasRequiredRoles = mode === 'all'
      ? requiredRoles.every(role => userRoles.includes(role))
      : requiredRoles.some(role => userRoles.includes(role));

    if (!hasRequiredRoles) {
      return <>{fallback}</>;
    }
  }

  return <>{children}</>;
}

// 사용 예제
function AdminPanel() {
  return (
    <ProtectedComponent
      requiredRoles={['admin', 'super_admin']}
      fallback={
        <div className="text-center p-8">
          <p className="text-gray-600">관리자 권한이 필요합니다.</p>
        </div>
      }
    >
      <div className="admin-panel">
        {/* 관리자 전용 컨텐츠 */}
      </div>
    </ProtectedComponent>
  );
}
```

### 권한 기반 라우트 가드
```typescript
// src/components/auth/RouteGuard.tsx
'use client';
import { useEffect } from 'react';
import { useRouter } from 'next/navigation';
import { useSelector } from 'react-redux';
import { RootState } from '@/store';
import { LoadingSpinner } from '@/components/common/LoadingSpinner';

interface RouteGuardProps {
  children: React.ReactNode;
  requireAuth?: boolean;
  requiredRoles?: string[];
  redirectTo?: string;
}

export function RouteGuard({
  children,
  requireAuth = true,
  requiredRoles = [],
  redirectTo = '/auth/login'
}: RouteGuardProps) {
  const router = useRouter();
  const { isAuthenticated, user, isInitialized } = useSelector((state: RootState) => state.auth);

  useEffect(() => {
    if (!isInitialized) return; // 초기화 대기

    // 인증이 필요한데 인증되지 않은 경우
    if (requireAuth && !isAuthenticated) {
      router.replace(redirectTo);
      return;
    }

    // 특정 역할이 필요한 경우
    if (isAuthenticated && user && requiredRoles.length > 0) {
      const userRoles = user.roles || [];
      const hasRequiredRole = requiredRoles.some(role => userRoles.includes(role));
      
      if (!hasRequiredRole) {
        router.replace('/unauthorized');
        return;
      }
    }
  }, [isAuthenticated, user, isInitialized, requireAuth, requiredRoles, router, redirectTo]);

  // 초기화 중인 경우 로딩 표시
  if (!isInitialized) {
    return (
      <div className="min-h-screen flex items-center justify-center">
        <LoadingSpinner />
      </div>
    );
  }

  // 인증이 필요한데 인증되지 않은 경우
  if (requireAuth && !isAuthenticated) {
    return null; // 리다이렉트 중
  }

  // 역할 요구사항 확인
  if (isAuthenticated && user && requiredRoles.length > 0) {
    const userRoles = user.roles || [];
    const hasRequiredRole = requiredRoles.some(role => userRoles.includes(role));
    
    if (!hasRequiredRole) {
      return null; // 리다이렉트 중
    }
  }

  return <>{children}</>;
}
```

## 4. 보안 토큰 관리 Best Practices

### 보안 토큰 저장소
```typescript
// src/lib/secureStorage.ts
export class SecureTokenStorage {
  private static readonly ACCESS_TOKEN_KEY = 'krgeobuk_access_token';
  private static readonly TOKEN_TIMESTAMP_KEY = 'krgeobuk_token_timestamp';

  // 토큰 암호화 (실제 구현에서는 더 강력한 암호화 사용)
  private static encrypt(data: string): string {
    // 간단한 Base64 인코딩 (실제로는 AES 등 사용)
    return btoa(data);
  }

  private static decrypt(encryptedData: string): string {
    try {
      return atob(encryptedData);
    } catch {
      throw new Error('Invalid token data');
    }
  }

  // 토큰 저장
  static setToken(token: string): void {
    try {
      const encryptedToken = this.encrypt(token);
      const timestamp = Date.now().toString();
      
      sessionStorage.setItem(this.ACCESS_TOKEN_KEY, encryptedToken);
      sessionStorage.setItem(this.TOKEN_TIMESTAMP_KEY, timestamp);
    } catch (error) {
      console.error('Failed to store token:', error);
    }
  }

  // 토큰 가져오기
  static getToken(): string | null {
    try {
      const encryptedToken = sessionStorage.getItem(this.ACCESS_TOKEN_KEY);
      const timestamp = sessionStorage.getItem(this.TOKEN_TIMESTAMP_KEY);
      
      if (!encryptedToken || !timestamp) {
        return null;
      }

      // 토큰 만료 확인 (예: 24시간)
      const tokenAge = Date.now() - parseInt(timestamp);
      const maxAge = 24 * 60 * 60 * 1000; // 24시간
      
      if (tokenAge > maxAge) {
        this.clearToken();
        return null;
      }

      return this.decrypt(encryptedToken);
    } catch (error) {
      console.error('Failed to retrieve token:', error);
      this.clearToken();
      return null;
    }
  }

  // 토큰 제거
  static clearToken(): void {
    sessionStorage.removeItem(this.ACCESS_TOKEN_KEY);
    sessionStorage.removeItem(this.TOKEN_TIMESTAMP_KEY);
  }

  // 토큰 존재 여부 확인
  static hasToken(): boolean {
    return this.getToken() !== null;
  }
}
```

### OAuth 콜백 페이지 구현
```typescript
// app/auth/oauth/google/callback/page.tsx
'use client';
import { useEffect, useState } from 'react';
import { useRouter, useSearchParams } from 'next/navigation';
import { useAppDispatch } from '@/hooks/redux';
import { setOAuthToken } from '@/store/slices/authSlice';
import { googleOAuth } from '@/lib/oauth/googleOAuth';

export default function GoogleCallbackPage() {
  const router = useRouter();
  const searchParams = useSearchParams();
  const dispatch = useAppDispatch();
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const handleCallback = async () => {
      try {
        const code = searchParams.get('code');
        const state = searchParams.get('state');
        const errorParam = searchParams.get('error');

        if (errorParam) {
          throw new Error(`OAuth Error: ${errorParam}`);
        }

        if (!code || !state) {
          throw new Error('Missing required parameters');
        }

        // OAuth 콜백 처리
        const { accessToken } = await googleOAuth.handleCallback(code, state);
        
        // Redux에 토큰 설정 및 사용자 정보 로드
        await dispatch(setOAuthToken(accessToken)).unwrap();
        
        // 성공 후 대시보드로 리다이렉트
        router.replace('/dashboard');
      } catch (error) {
        console.error('OAuth callback error:', error);
        setError(error instanceof Error ? error.message : 'Authentication failed');
        
        // 3초 후 로그인 페이지로 리다이렉트
        setTimeout(() => {
          router.replace('/auth/login');
        }, 3000);
      }
    };

    handleCallback();
  }, [searchParams, dispatch, router]);

  if (error) {
    return (
      <div className="min-h-screen flex items-center justify-center">
        <div className="text-center">
          <div className="text-red-500 text-6xl mb-4">❌</div>
          <h1 className="text-2xl font-bold text-gray-900 mb-2">인증 실패</h1>
          <p className="text-gray-600 mb-4">{error}</p>
          <p className="text-sm text-gray-500">잠시 후 로그인 페이지로 이동합니다...</p>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen flex items-center justify-center">
      <div className="text-center">
        <div className="animate-spin text-blue-500 text-6xl mb-4">⏳</div>
        <h1 className="text-2xl font-bold text-gray-900 mb-2">인증 처리 중</h1>
        <p className="text-gray-600">Google 인증을 완료하고 있습니다...</p>
      </div>
    </div>
  );
}
```

---

# 🌟 마무리

이 가이드는 krgeobuk 생태계에서 Next.js 15 및 React 18 기반 클라이언트 개발의 완전한 표준을 제시합니다. 모든 개발자는 이 패턴들을 준수하여 일관되고 유지보수 가능한 코드를 작성해야 합니다.

**핵심 원칙:**
1. **타입 안전성 우선** - 모든 컴포넌트와 함수에 완전한 TypeScript 타입 정의
2. **성능 최적화** - 메모이제이션과 지연 로딩을 통한 최적화
3. **보안 강화** - Next.js 미들웨어를 통한 포괄적인 보안 조치
4. **사용자 경험** - 접근성과 반응형 디자인을 고려한 UI/UX 구현
5. **일관성 유지** - 공유 라이브러리와 디자인 시스템 활용

이 표준을 따르면 krgeobuk 생태계의 모든 클라이언트 애플리케이션이 일관된 품질과 성능을 보장할 수 있습니다.