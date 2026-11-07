# academy-frontend

흥국생명 연수원 예약 관리 시스템의 웹 프론트엔드.

| 항목 | 내용 |
|---|---|
| 프레임워크 | Next.js (App Router, `output: 'standalone'`), TypeScript |
| Node | 20 (Dockerfile, CI 기준) |
| 개발 포트 | 3100 (`npm run dev`) |
| 운영 포트 | 3000 (컨테이너) |

## 관련 레포

| 레포 | 역할 |
|---|---|
| [academy-backend](https://github.com/Heungkuk-Academy/academy-backend) | API 서버 |
| academy-frontend | 이 레포. 웹 프론트엔드 |
| academy-infra | 운영 docker-compose, Nginx 등 배포 설정 |
| academy-docs | 문서 |

## 실행

```bash
npm ci
npm run dev        # http://localhost:3100
```

백엔드가 `http://localhost:8888`에서 실행 중이어야 한다 (백엔드 레포 README 참고).

## 환경변수

| 변수 | 설명 |
|---|---|
| `NEXT_PUBLIC_API_URL` | 백엔드 API 주소 |

- 로컬에서는 `.env.local`에 둔다. 이 파일은 Git에 올라가지 않는다 (`.gitignore`, `.dockerignore` 대상).
- `NEXT_PUBLIC_*` 값은 **빌드 시점에 번들에 박힌다.** 값을 바꾸려면 다시 빌드해야 하고, 서버에서 환경변수만 바꿔서는 적용되지 않는다.

## 빌드

```bash
npm run lint
npm run build
```

Docker 이미지는 빌드 인자로 API 주소를 받는다.

```bash
docker build --build-arg NEXT_PUBLIC_API_URL=http://localhost:8888 -t academy-frontend:local .
```

## 배포

`main`에 머지되면 GitHub Actions(`.github/workflows/deploy.yml`)가 자동으로 배포한다.

1. ARM 러너에서 `linux/arm64` 이미지를 빌드해 `ghcr.io/heungkuk-academy/academy-frontend`에 `:latest`, `:<커밋 SHA>`로 푸시 (`NEXT_PUBLIC_API_URL`은 시크릿으로 주입)
2. 서버에 SSH로 접속해 frontend(`client` 서비스)만 교체: `./compose.sh pull client && ./compose.sh up -d --no-deps client`
3. 컨테이너가 응답하는지 확인한 뒤 nginx를 reload

필요한 시크릿: `NEXT_PUBLIC_API_URL`, `SERVER_HOST`, `SSH_PRIVATE_KEY`.
compose와 nginx 설정은 academy-infra에서 관리한다.

PR을 열면 `.github/workflows/ci.yml`이 lint와 빌드를 확인한다.

롤백은 서버 `.env`의 `CLIENT_IMAGE`에 이전 커밋 SHA 태그를 지정하고 `./compose.sh up -d --no-deps client`를 실행한다.

## 커밋 금지 항목

`.env`, `.env.*`, 비밀번호/키/토큰, 내부 주소는 커밋하지 않는다.

## 브랜치 전략

- `main`: 운영 배포 기준. 머지되면 배포된다. **직접 커밋 금지**
- `develop`: 개발 기준 브랜치. 작업 브랜치는 여기서 분기하고 PR 대상도 `develop`
- 작업 브랜치: `<type>/<설명>` (예: `feat/reservation-filter`, `fix/login-redirect`, `chore/deploy-ci`)

작업 흐름: 이슈 생성 → `develop`에서 브랜치 분기 → PR → 리뷰 후 머지 → 릴리스 시 `develop` → `main` PR

## 커밋 메시지

[Conventional Commits](https://www.conventionalcommits.org/ko/) 형식을 따른다.

| type | 용도 |
|---|---|
| `feat` | 기능 추가 |
| `fix` | 버그 수정 |
| `docs` | 문서 |
| `chore` | 설정, 빌드, 기타 |
| `refactor` | 구조 개선 |
| `test` | 테스트 |
