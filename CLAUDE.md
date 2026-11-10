# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

흥국생명 연수원 예약 관리 시스템의 웹 프론트엔드 (Next.js 15 App Router, React 19, TypeScript, Node 22). API 서버는 별도 레포 `academy-backend`, compose/nginx 등 운영 설정은 `academy-infra` 에 있다.

## 명령어

```bash
npm run dev     # 개발 서버 — 포트 3100 (3000 아님)
npm run lint    # ESLint (next/core-web-vitals + next/typescript)
npm run build   # 프로덕션 빌드 (standalone 출력, 타입 검사 포함)
```

- 테스트 코드는 아직 없다. CI(`.github/workflows/ci.yml`)는 PR 에서 `lint` + `build` 만 돌리므로, 변경 후 이 두 개가 통과하는지 확인한다.
- 필수 환경변수는 `NEXT_PUBLIC_API_URL` (백엔드 주소, 로컬은 보통 `http://localhost:8888`). `.env.local` 에 둔다.
- `NEXT_PUBLIC_*` 는 빌드 시점에 번들에 박힌다. 그래서 Docker 이미지는 환경(운영/맥미니)별로 따로 빌드한다.

## 브랜치 · 커밋 규칙

- `main` = 운영 배포 기준 (**직접 커밋 금지**), `develop` = 개발 기준. 작업 브랜치는 `develop` 에서 `<type>/<설명>` 으로 분기하고 PR 대상도 `develop`.
- 커밋 메시지: `<type>: <설명>` (type: `feat` / `fix` / `docs` / `chore` / `refactor` / `test`), 설명은 한국어.
- 배포(`.github/workflows/deploy.yml`): `main` push → 운영 서버(`:latest`), `develop` push → 맥미니 테스트 서버(`:develop`, Tailscale 경유). ARM 이미지를 GHCR 에 올리고 서버에서 `client` 컨테이너만 교체한다.

## 아키텍처

### 라우트 구조
- `app/(admin)/*` — 로그인 후 관리 화면 (dashboard, scheduler, reservation, document, restaurant, accommodation, survey, settings). 모든 페이지가 `'use client'` 이며 데이터는 클라이언트에서 axios 로 가져온다 (서버 컴포넌트/서버 액션으로 데이터 패칭하지 않음).
- `app/auth/login` — 로그인. 성공 시 `/dashboard` 로 이동. `/` 는 로그인으로 redirect.
- `app/survey/[token]` — 고객용 공개 설문 (인증 없음). 진입 후 `history.replaceState` 로 주소를 `/survey-link` 로 바꿔 토큰을 숨긴다 → 새로고침하면 `app/survey-link` 안내 화면이 보인다.

### 인증 흐름 (여러 파일에 걸쳐 있음)
- access 토큰은 **메모리(zustand `store/auth.ts`)에만** 저장한다. localStorage 에 넣지 말 것. refresh 토큰은 HttpOnly 쿠키라 JS 가 다루지 않는다 (`withCredentials: true`).
- `app/(admin)/layout.tsx` 가 진입 시 메모리 토큰이 없으면(새로고침/새 탭) `refreshAccessToken()` 으로 재발급받고, 준비될 때까지 **하위 화면을 렌더링하지 않는다**. 그래서 하위 페이지의 `isAdmin()` / `getCurrentUserId()` (`lib/utils/auth.ts`) 는 동기 호출해도 안전하다.
- `lib/api/instance.ts` 의 axios 인스턴스가 Bearer 토큰 첨부와 401 처리를 맡는다: 401 → reissue 1회 → 원 요청 재시도, 실패 시 `/auth/login` 으로. 동시 401 은 하나의 `refreshPromise` 를 공유하고, 여러 탭은 Web Locks(`auth-reissue`)로 직렬화한다 (refresh 토큰 rotation 때문). 인증 API(login/reissue/logout)는 재시도 대상에서 제외.
- 권한: JWT payload 의 `role === 'ROLE_ADMIN'` 이면 관리자. 관리자 전용 라우트(`document`, `settings`)는 각 `layout.tsx` 에서 `components/ui/AdminGuard` 로 감싸며 비관리자는 `/scheduler` 로 보낸다. 사이드바 메뉴 노출도 `adminOnly` 플래그로 제어. 일반 화면에서는 `isAdmin()` 으로 편집 모달 vs 상세 모달을 나눈다. JWT 디코딩은 표시용일 뿐이고 실제 권한 판단은 서버가 한다.

### API 레이어
- `lib/api/*.ts` 에 도메인별 함수가 있고 모두 `instance` 를 쓴다. 백엔드 응답은 `ApiResponse<T>` (`{ success, message, data }`) 로 감싸져 오므로 함수에서 `res.data.data` 를 반환한다.
- 관리자 API 는 `/v1/admin/...`, 공개 설문 API(`/v1/survey/...`)는 `lib/api/survey.ts` 에서 인증 없이 기본 `axios` 로 호출한다.
- 예약 생성/수정 시 `toRequestBody()` (`lib/api/reservation.ts`) 로 `Reservation` 을 요청 바디로 변환한다. 예약은 강의실(`classrooms`) · 객실(`rooms`) · 식사(`meals`) 를 날짜 단위 배열로 갖는다.

### 상태 · 데이터 갱신
- 전역 상태는 zustand 두 개뿐: `store/auth.ts`(토큰/사용자), `store/toast.ts`(토스트, `useToastStore().show(msg, 'success'|'error')`).
- 여러 사용자가 동시에 쓰므로 일정/식수/숙박 화면은 `hooks/usePolling.ts` 로 주기 갱신 + 탭 포커스 복귀 시 즉시 갱신한다 (`RefreshStatus` 컴포넌트와 함께 사용). 인라인 콜백을 그대로 넘기면 된다.
- 가격/연락처 설정은 서버 `/v1/admin/settings` 의 flat `Record<string,string>` 을 `lib/utils/priceSettings.ts` 의 `parseSettings()` 로 `AppSettings` 로 변환하며, localStorage(`appSettingsCache`)에 캐시한다.

### 도메인 상수 · 문서 출력
- 객실 목록/도면 레이아웃은 `lib/constants/rooms.ts`, 강의실 분류/가격은 `lib/constants/classrooms.ts`, 예약 상태 색상(확정/예약/문의/취소)은 `lib/constants/status.ts`.
- 견적서·예약 목록 등 파일 출력은 브라우저에서 생성한다: `lib/utils/exportQuoteToDocx.ts`(docx), `exportQuoteToExcel.ts` / `exportReservationsToExcel.ts`(xlsx), `printRoomTable.ts`(인쇄용 HTML). 엑셀 일괄 업로드는 서버로 multipart 전송(`importReservations`).
- 공휴일 판정은 `@hyunbinseo/holidays-kr` 를 사용하며, 지원 연도 밖이면 throw 하므로 try/catch 로 감싸서 쓴다.

### 스타일
- 컴포넌트별 CSS Modules (`*.module.css`) + `app/globals.css` 의 CSS 변수(`--bg`, `--surface`, `--primary` 등). 색상은 하드코딩 대신 이 변수를 쓴다.
- 다크 모드는 `<html data-theme="dark">` 로 변수를 덮어쓴다. `app/layout.tsx` 의 인라인 스크립트가 첫 페인트 전에 테마를 적용하고, `components/theme/ThemeProvider` 가 토글/저장을 맡는다.
- 경로 별칭 `@/*` → 레포 루트.
