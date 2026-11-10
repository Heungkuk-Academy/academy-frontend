## Project Overview

이 프로젝트는 흥국생명 연수원 예약 관리 시스템의 웹 프론트엔드입니다.
관리자용 예약 · 일정 · 식수 · 숙박 · 설문 관리 화면과 고객용 공개 설문 페이지를 제공합니다.


## Tech Stack

- Next.js 15 (App Router, `output: 'standalone'`)
- React 19, TypeScript (strict)
- Node 22
- axios (API 호출), zustand (전역 상태)
- CSS Modules + CSS 변수 (라이트/다크 테마)
- docx, xlsx (견적서 · 예약 목록 파일 생성), react-datepicker, date-fns
- lucide-react (아이콘), @hyunbinseo/holidays-kr (공휴일)



## Coding Conventions

### 디렉터리 구조

```
app/
├── (admin)/{dashboard,scheduler,reservation,document,restaurant,accommodation,survey,settings}/
│   ├── page.tsx           # 'use client' 페이지
│   └── page.module.css
├── (admin)/layout.tsx     # 토큰 준비 + Sidebar/Header/Toast
├── auth/login/            # 로그인
├── survey/[token]/        # 고객용 공개 설문 (인증 없음)
└── survey-link/           # 설문 페이지 새로고침 시 안내 화면
components/
├── {dashboard,reservation,scheduler,survey,document}/  # 도메인별 컴포넌트
├── layout/                # Header, Sidebar, ProfileDropdown, ThemeToggle
├── ui/                    # AdminGuard, Toast, MonthNavigator 등 공통
└── theme/                 # ThemeProvider
hooks/                     # usePolling, useReservationSearch, useSurveySearch
lib/
├── api/                   # 도메인별 API 함수 + instance.ts (axios 인스턴스)
├── constants/             # 객실 · 강의실 · 예약 상태 · 설문 상수
└── utils/                 # 엑셀/docx 출력, 인쇄, 인증 헬퍼 등
store/                     # zustand (auth, toast)
types/                     # 도메인별 타입 (api, auth, reservation, survey ...)
```

### 네이밍 규칙
- 컴포넌트: PascalCase 파일 + 같은 이름의 CSS Module (예: `WeekCard.tsx` + `WeekCard.module.css`)
- 훅: `useXxx.ts` (예: `usePolling.ts`)
- API · 유틸 함수: camelCase, 동사로 시작 (예: `getReservationsByRange`, `exportQuoteToExcel`)
- import 경로는 `@/` 별칭 사용 (`@/*` → 레포 루트)

### 코드 작성 규칙
- 관리 화면(`app/(admin)`)은 모두 `'use client'` 이고, 데이터는 클라이언트에서 `lib/api` 함수로 가져온다. 서버 컴포넌트 · 서버 액션으로 데이터를 가져오지 않는다.
- API 호출은 컴포넌트에서 axios 를 직접 쓰지 말고 `lib/api/*.ts` 에 함수를 추가해서 사용한다.
- 요청/응답 타입은 `types/` 의 도메인 파일에 정의한다.
- 색상은 하드코딩하지 말고 `app/globals.css` 의 CSS 변수(`--primary`, `--surface`, `--text-sub` 등)를 쓴다. 다크 모드는 `[data-theme='dark']` 에서 같은 변수를 덮어쓴다.
- 도메인 상수(객실 · 강의실 · 상태 색상)는 `lib/constants/` 에 둔다.
- 공휴일 판정(`isHoliday`)은 지원 연도 밖이면 throw 하므로 try/catch 로 감싼다.
- 다른 사용자의 변경을 반영해야 하는 화면은 `usePolling(callback, intervalMs)` 를 쓴다 (주기 갱신 + 탭 포커스 복귀 시 즉시 갱신). 콜백은 인라인 함수로 넘겨도 된다.

### API 응답 형식
- 백엔드 응답은 `ApiResponse<T>` (`{ success, message, data }`) 로 감싸져 온다. API 함수는 `res.data.data` 를 반환한다.

```ts
// lib/api/settings.ts
export async function getSettings(): Promise<Record<string, string>> {
  const res = await instance.get<ApiResponse<Record<string, string>>>('/v1/admin/settings');
  return res.data.data ?? {};
}
```

- 경로: 관리자 API 는 `/v1/admin/...`, 공개 설문 API 는 `/v1/survey/...`
- 관리자 API 는 반드시 `lib/api/instance.ts` 의 `instance` 를 쓴다 (토큰 첨부 · 401 재발급 처리). 공개 설문 API 만 기본 `axios` 를 쓴다.
- API연결시 Docs 레포에 있는 API문서를 확인하고 연결한다.

### 에러 처리
- 서버가 보낸 메시지가 있으면 우선 보여주고, 없으면 한국어 기본 메시지를 쓴다.

```ts
try {
  await deleteAccount(id);
} catch (err) {
  alert((isAxiosError(err) && err.response?.data?.message) || '계정 삭제에 실패했습니다.');
}
```

- 폴링으로 다시 불러올 때(`silent`)는 실패해도 알림을 띄우지 않는다.
- 성공 알림은 `useToastStore((s) => s.show)` 로 띄운다 (`show(message, 'success' | 'error')`).

### 인증 · 권한
- access 토큰은 **메모리(zustand `store/auth.ts`)에만** 저장한다. localStorage 에 넣지 않는다. refresh 토큰은 HttpOnly 쿠키라 JS 에서 다루지 않는다.
- `app/(admin)/layout.tsx` 가 메모리 토큰이 없으면(새로고침 · 새 탭) refresh 쿠키로 다시 받아오고, 준비될 때까지 하위 화면을 렌더링하지 않는다. 그래서 하위 화면에서 `isAdmin()` / `getCurrentUserId()` 를 동기 호출해도 된다.
- `lib/api/instance.ts`: 401 → reissue 1회 → 원래 요청 재시도, 실패 시 `/auth/login` 으로 이동. 동시에 여러 요청이 401 이어도 reissue 는 한 번만 보낸다(`refreshPromise`). 여러 탭은 Web Locks 로 차례대로 처리한다 (refresh 토큰 rotation 때문).
- 관리자 = JWT `role === 'ROLE_ADMIN'`. 관리자 전용 라우트는 해당 `layout.tsx` 를 `AdminGuard` 로 감싸고, 사이드바는 `NAV_ITEMS` 의 `adminOnly` 로 숨긴다. 최종 권한 판단은 서버가 한다.

```tsx
// app/(admin)/settings/layout.tsx
export default function SettingsLayout({ children }: { children: React.ReactNode }) {
  return <AdminGuard>{children}</AdminGuard>;
}
```


## Git 규칙
- `main`: 운영 배포 기준. 직접 커밋 금지
- `develop`: 작업 브랜치는 여기서 분기하고, PR도 `develop`으로 올린다
- `develop`, `main`은 PR로만 반영되며 CI 통과가 필요하다
- 브랜치명: `<type>/<설명>` (예: `feat/reservation-filter`)
- 커밋 메시지: `<type>: <제목>` (예: `feat: 예약 목록 검색 조건 추가`)
- type: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`
- 배포: `develop` push → 맥미니 테스트 서버(`:develop`), `main` push → 운영 서버(`:latest`). `client` 컨테이너만 교체한다.
- - 무조건 완성 후 커밋하지말고 사용자에게 물어본다.


## 응답 선호도
- 설명과 코드는 한국어로 작성
- 답변은 결론 먼저, 이유 나중 (두괄식)
- 이론보다 실제 코드 예시 우선
- 모르면 솔직히 말하고 추측하지 말 것

## 금지 사항
- 불필요한 칭찬 금지
- 너무 긴 설명보다 짧고 핵심적인 답변 선호
- 코드 없이 설명만 하는 답변 지양
