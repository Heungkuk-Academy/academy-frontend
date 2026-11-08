# academy-frontend

흥국생명 연수원 예약 관리 시스템의 웹 프론트엔드.

| 항목 | 내용 |
|---|---|
| 프레임워크 | Next.js, TypeScript |
| Node | 22 (Dockerfile, CI 기준) |

## 관련 레포

| 레포 | 역할 |
|---|---|
| [academy-backend](https://github.com/Heungkuk-Academy/academy-backend) | API 서버 |
| academy-frontend | 웹 프론트엔드 |
| academy-infra | 운영 설정 |
| academy-docs | 문서 |


## 환경변수

| 변수 | 설명 |
|---|---|
| `NEXT_PUBLIC_API_URL` | 백엔드 API 주소 |


## 빌드

```bash
npm run lint
npm run build
```


## 브랜치 전략

- `main`: 운영 배포 기준. 머지되면 배포된다. **직접 커밋 금지**
- `develop`: 개발 기준 브랜치. 작업 브랜치는 여기서 분기하고 PR 대상도 `develop`
- 작업 브랜치: `<type>/<설명>` (예: `feat/reservation-filter`, `fix/login-redirect`, `chore/deploy-ci`)

작업 흐름: 이슈 생성 → `develop`에서 브랜치 분기 → PR → 리뷰 후 머지 → 릴리스 시 `develop` → `main` PR

## 커밋 메시지


| type | 용도 |
|---|---|
| `feat` | 기능 추가 |
| `fix` | 버그 수정 |
| `docs` | 문서 |
| `chore` | 설정, 빌드, 기타 |
| `refactor` | 구조 개선 |
| `test` | 테스트 |
