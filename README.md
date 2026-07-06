# restaurant_management

음식점을 도와주는 웹앱 — 다중 음식점(SaaS)을 위한 예약 관리 + 테이블/좌석 혼잡도 시스템

## 프로젝트 구조

pnpm workspace 모노레포로 구성되어 있다.

```
server/   NestJS + Prisma + PostgreSQL REST API
client/   React + Vite + TypeScript SPA
```

구현 계획은 `superpowers/plans/`에 있다:
- `2026-07-06-restaurant-foundation-reservation.md` — 기반 시스템(인증/음식점 등록) + 예약 관리
- `2026-07-06-restaurant-table-congestion.md` — 테이블/좌석 관리, QR 주문 인식, 혼잡도 표시

## 사전 요구사항

- Node.js 18.x
- pnpm 9.x — `corepack`으로 설치되는 최신 pnpm(11.x)은 Node 22 이상을 요구하므로, Node 18 환경에서는 `npm install -g pnpm@9`로 pnpm 9를 설치해야 한다.
- Docker Desktop — 로컬 PostgreSQL 실행용 (Prisma 연동 이후 태스크부터 필요)

## 설치

```bash
pnpm install
```

## 로컬 PostgreSQL 실행

```bash
pnpm db:up    # docker compose로 postgres 컨테이너 기동
pnpm db:down  # 컨테이너 종료
```

## 개발 서버 실행

두 개의 터미널에서 각각 실행한다.

```bash
pnpm dev:api   # NestJS 서버, http://localhost:3000
pnpm dev:web   # Vite 개발 서버, http://localhost:5173
```

## 빌드

```bash
pnpm --filter server build
pnpm --filter client build
```

## 테스트

```bash
pnpm test:api
pnpm test:web
```

## 현재 진행 상황

현재는 `server`(NestJS)와 `client`(React + Vite) 앱의 초기 스캐폴딩만 완료된 상태다. Prisma/PostgreSQL 연동, 인증, 예약, 테이블/혼잡도 기능은 위 계획 문서에 따라 순차적으로 구현 예정이다.
