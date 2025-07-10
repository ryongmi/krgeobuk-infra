# Portal-Client 인증 API 연동 가이드

> 백엔드 표준 기준으로 작성됨
> auth-server(port:8000) 연동 가이드

## 🔧 Axios 인스턴스 설정

### 1. 케이스 변환 유틸리티 함수

```typescript
// src/utils/caseConverter.ts

/**
 * snake_case를 camelCase로 변환 (전체 응답 객체 처리)
 */
export function snakeToCamel(obj: any): any {
  if (obj === null || obj === undefined) return obj;

  if (Array.isArray(obj)) {
    return obj.map(snakeToCamel);
  }

  if (typeof obj === "object") {
    const converted: any = {};
    for (const key in obj) {
      if (obj.hasOwnProperty(key)) {
        const camelKey = key.replace(/_([a-z])/g, (_, letter) =>
          letter.toUpperCase()
        );
        converted[camelKey] = snakeToCamel(obj[key]);
      }
    }
    return converted;
  }

  return obj;
}

/**
 * camelCase를 snake_case로 변환 (전체 요청 객체 처리)
 */
export function camelToSnake(obj: any): any {
  if (obj === null || obj === undefined) return obj;

  if (Array.isArray(obj)) {
    return obj.map(camelToSnake);
  }

  if (typeof obj === "object") {
    const converted: any = {};
    for (const key in obj) {
      if (obj.hasOwnProperty(key)) {
        const snakeKey = key.replace(
          /[A-Z]/g,
          (letter) => `_${letter.toLowerCase()}`
        );
        converted[snakeKey] = camelToSnake(obj[key]);
      }
    }
    return converted;
  }

  return obj;
}
```

### 2. Axios 인스턴스 설정

```typescript
// src/lib/axios.ts
import axios, { AxiosResponse, InternalAxiosRequestConfig } from "axios";
import { snakeToCamel, camelToSnake } from "@/utils/caseConverter";

// 공유 라이브러리 인터페이스 활용
import type { 
  AuthLoginRequest, 
  AuthLoginResponse, 
  AuthSignupRequest, 
  AuthRefreshResponse 
} from "@krgeobuk/auth";
import type { UserDetail } from "@krgeobuk/user";
import type { JwtPayload } from "@krgeobuk/jwt";

// 백엔드 응답 타입 정의 (snake_case)
interface BackendResponse<T = any> {
  code: string;
  status_code: number;
  message: string;
  isLogin: boolean;
  data: T;
}

// 프론트엔드에서 사용할 응답 타입 (camelCase)
interface ApiResponse<T = any> {
  code: string;
  statusCode: number;
  message: string;
  isLogin: boolean;
  data: T;
}

// Auth Server 인스턴스
export const authApi = axios.create({
  baseURL:
    process.env.NEXT_PUBLIC_AUTH_SERVER_URL || "http://localhost:8000/api",
  timeout: 10000,
  withCredentials: true, // 쿠키 포함 (refresh token용)
  headers: {
    "Content-Type": "application/json",
  },
});

// Authz Server 인스턴스
export const authzApi = axios.create({
  baseURL: process.env.NEXT_PUBLIC_AUTHZ_SERVER_URL || "http://localhost:8100",
  timeout: 10000,
  withCredentials: true,
  headers: {
    "Content-Type": "application/json",
  },
});

// 토큰 관리
class TokenManager {
  private static instance: TokenManager;
  private accessToken: string | null = null;

  static getInstance(): TokenManager {
    if (!TokenManager.instance) {
      TokenManager.instance = new TokenManager();
    }
    return TokenManager.instance;
  }

  setAccessToken(token: string) {
    this.accessToken = token;
    if (typeof window !== "undefined") {
      localStorage.setItem("accessToken", token);
    }
  }

  getAccessToken(): string | null {
    if (!this.accessToken && typeof window !== "undefined") {
      this.accessToken = localStorage.getItem("accessToken");
    }
    return this.accessToken;
  }

  clearAccessToken() {
    this.accessToken = null;
    if (typeof window !== "undefined") {
      localStorage.removeItem("accessToken");
    }
  }
}

const tokenManager = TokenManager.getInstance();

// 요청 인터셉터 (camelCase -> snake_case)
const requestInterceptor = (config: InternalAxiosRequestConfig) => {
  // Authorization 헤더 추가
  const token = tokenManager.getAccessToken();
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }

  // 요청 데이터를 snake_case로 변환
  if (config.data) {
    config.data = camelToSnake(config.data);
  }

  return config;
};

// 응답 인터셉터 (snake_case -> camelCase)
const responseInterceptor = (response: AxiosResponse<BackendResponse>) => {
  // 전체 응답 객체를 camelCase로 변환
  const transformedResponse = snakeToCamel(response.data);

  return {
    ...response,
    data: transformedResponse,
  };
};

// 에러 인터셉터
const errorInterceptor = async (error: any) => {
  const originalRequest = error.config;

  // 401 에러 시 토큰 갱신 시도
  if (error.response?.status === 401 && !originalRequest._retry) {
    originalRequest._retry = true;

    try {
      // refresh 토큰으로 새 access token 요청
      const refreshResponse = await authApi.post("/auth/refresh");
      const newToken = refreshResponse.data.data.accessToken;

      tokenManager.setAccessToken(newToken);

      // 실패했던 요청 재시도
      originalRequest.headers.Authorization = `Bearer ${newToken}`;
      return authApi(originalRequest);
    } catch (refreshError) {
      // Refresh 실패 시 로그아웃 처리
      tokenManager.clearAccessToken();
      if (typeof window !== "undefined") {
        window.location.href = "/auth/login";
      }
      return Promise.reject(refreshError);
    }
  }

  return Promise.reject(error);
};

// 인터셉터 적용
[authApi, authzApi].forEach((api) => {
  api.interceptors.request.use(requestInterceptor);
  api.interceptors.response.use(responseInterceptor, errorInterceptor);
});

export { tokenManager };
export type { ApiResponse, BackendResponse };
```

## 🔐 인증 API 서비스

### 3. 인증 서비스 구현

```typescript
// src/services/authService.ts
import { authApi, tokenManager } from "@/lib/axios";
import type { ApiResponse } from "@/lib/axios";

// 공유 라이브러리 인터페이스 활용
import type { 
  AuthLoginRequest, 
  AuthLoginResponse, 
  AuthSignupRequest, 
  AuthRefreshResponse 
} from "@krgeobuk/auth";
import type { UserDetail } from "@krgeobuk/user";
import type { LoggedInUser } from "@krgeobuk/shared";

// 타입 별칭 정의 (명확성을 위해)
type LoginRequest = AuthLoginRequest;
type SignupRequest = AuthSignupRequest;
type LoginResponse = AuthLoginResponse;
type UserMeResponse = UserDetail;
type RefreshResponse = AuthRefreshResponse;

export class AuthService {
  /**
   * 로그인
   */
  static async login(
    credentials: LoginRequest
  ): Promise<ApiResponse<LoginResponse>> {
    const response = await authApi.post<ApiResponse<LoginResponse>>(
      "/auth/login",
      credentials
    );

    // 토큰 저장
    if (response.data.data?.accessToken) {
      tokenManager.setAccessToken(response.data.data.accessToken);
    }

    return response.data;
  }

  /**
   * 회원가입
   */
  static async signup(
    userData: SignupRequest
  ): Promise<ApiResponse<LoginResponse>> {
    const response = await authApi.post<ApiResponse<LoginResponse>>(
      "/auth/signup",
      userData
    );

    // 회원가입 성공 시 자동 로그인 처리
    if (response.data.data?.accessToken) {
      tokenManager.setAccessToken(response.data.data.accessToken);
    }

    return response.data;
  }

  /**
   * 로그아웃
   */
  static async logout(): Promise<ApiResponse<null>> {
    const response = await authApi.post<ApiResponse<null>>("/auth/logout");

    // 토큰 제거
    tokenManager.clearAccessToken();

    return response.data;
  }

  /**
   * 토큰 갱신
   */
  static async refresh(): Promise<ApiResponse<RefreshResponse>> {
    const response = await authApi.post<ApiResponse<RefreshResponse>>(
      "/auth/refresh"
    );

    // 새 토큰 저장
    if (response.data.data?.accessToken) {
      tokenManager.setAccessToken(response.data.data.accessToken);
    }

    return response.data;
  }

  /**
   * 현재 사용자 정보 조회
   */
  static async getMe(): Promise<ApiResponse<UserMeResponse>> {
    const response = await authApi.get<ApiResponse<UserMeResponse>>(
      "/users/me"
    );
    return response.data;
  }

  /**
   * 현재 로그인 상태 확인
   */
  static isLoggedIn(): boolean {
    return !!tokenManager.getAccessToken();
  }

  /**
   * OAuth 로그인 URL 생성
   */
  static getOAuthUrl(provider: "google" | "naver"): string {
    const baseUrl =
      process.env.NEXT_PUBLIC_AUTH_SERVER_URL || "http://localhost:8000/api";
    return `${baseUrl}/oauth/login-${provider}`;
  }
}
```

## 🎯 사용 예시

### 4. React 컴포넌트에서 사용

```typescript
// src/components/auth/LoginForm.tsx
"use client";

import { useState } from "react";
import { AuthService } from "@/services/authService";
import { useRouter } from "next/navigation";

export default function LoginForm() {
  const [formData, setFormData] = useState({
    email: "",
    password: "",
  });
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState("");
  const router = useRouter();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError("");

    try {
      const response = await AuthService.login(formData);

      if (response.statusCode === 200) {
        // 로그인 성공
        console.log("로그인 성공:", response.data.user);
        router.push("/dashboard");
      }
    } catch (error: any) {
      // 에러 처리 (이미 camelCase로 변환됨)
      setError(error.response?.data?.message || "로그인에 실패했습니다.");
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <div>
        <input
          type="email"
          placeholder="이메일"
          value={formData.email}
          onChange={(e) => setFormData({ ...formData, email: e.target.value })}
          required
        />
      </div>
      <div>
        <input
          type="password"
          placeholder="비밀번호"
          value={formData.password}
          onChange={(e) =>
            setFormData({ ...formData, password: e.target.value })
          }
          required
        />
      </div>
      {error && <p className="text-red-500">{error}</p>}
      <button type="submit" disabled={loading}>
        {loading ? "로그인 중..." : "로그인"}
      </button>
    </form>
  );
}
```

### 5. 사용자 정보 조회 예시

```typescript
// src/hooks/useAuth.ts
import { useState, useEffect } from "react";
import { AuthService } from "@/services/authService";
import type { UserDetail } from "@krgeobuk/user";

// 공유 라이브러리 인터페이스 활용
type User = UserDetail;

export function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const checkAuth = async () => {
      if (AuthService.isLoggedIn()) {
        try {
          const response = await AuthService.getMe();
          setUser(response.data); // 이미 camelCase로 변환됨
        } catch (error) {
          console.error("사용자 정보 조회 실패:", error);
        }
      }
      setLoading(false);
    };

    checkAuth();
  }, []);

  const logout = async () => {
    try {
      await AuthService.logout();
      setUser(null);
    } catch (error) {
      console.error("로그아웃 실패:", error);
    }
  };

  return {
    user,
    loading,
    isLoggedIn: !!user,
    logout,
  };
}
```

## 🔧 환경 변수 설정

```bash
# .env.local
NEXT_PUBLIC_AUTH_SERVER_URL=http://localhost:8000/api
NEXT_PUBLIC_AUTHZ_SERVER_URL=http://localhost:8100
```

## ✅ 주요 특징

1. **전체 응답 케이스 변환**: 응답 객체의 모든 속성을 snake_case → camelCase로 변환 (data 내부뿐만 아니라 전체)
2. **공유 라이브러리 활용**: `@krgeobuk/auth`, `@krgeobuk/user` 등 기존 인터페이스 재사용으로 타입 일관성 보장
3. **자동 토큰 관리**: Authorization 헤더 자동 설정, 401 에러 시 자동 갱신
4. **쿠키 지원**: refresh token을 위한 withCredentials 설정
5. **타입 안전성**: 공유 라이브러리 인터페이스 활용으로 완전한 타입 안전성
6. **에러 핸들링**: 통합 에러 처리 및 자동 로그아웃

## 📦 필요한 패키지 설치

```bash
# portal-client에서 실행
npm install @krgeobuk/auth @krgeobuk/user @krgeobuk/shared @krgeobuk/jwt
```

이제 포탈 클라이언트에서 이 설정을 사용하면 백엔드 API와 완벽하게 연동되며, 공유 라이브러리의 타입 정의를 통해 일관성 있는 개발이 가능합니다!
