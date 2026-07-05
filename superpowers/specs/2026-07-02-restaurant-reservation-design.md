# 음식점 예약 관리 시스템 — 설계 문서 (서브프로젝트 1)

## 배경 및 범위

음식점 관리 SaaS 웹 애플리케이션의 첫 번째 서브프로젝트. 전체 목표는 여러 음식점을 대상으로 예약 관리와 주문(POS) 관리를 지원하는 것이지만, 두 기능은 독립적으로 개발 가능하므로 분리했다. 이 문서는 **기반 시스템(인증, 음식점/직원 관리) + 예약 관리**를 다룬다. 주문(POS) 관리는 이후 별도의 설계 문서/계획으로 진행한다.

### 핵심 요구사항

- 여러 음식점이 각자 독립적으로 가입해 사용하는 다중 SaaS
- 사장님(음식점 운영자)과 직원이 관리 화면을 사용하고, 고객은 비회원으로 예약
- 예약은 테이블 배치(플로어 플랜) 없이 시간대별 슬롯 + 총 좌석수로 관리
- 예약은 잔여 좌석이 있으면 자동 확정 (승인 절차 없음)

- 이메일/SMS 알림은 MVP 범위에서 제외

## 아키텍처 & 기술 스택

모노레포(pnpm workspace)로 구성한다.

- **`apps/web`** — React + Vite + TypeScript (SPA). 사장님/직원용 관리 대시보드와 고객용 예약 페이지를 모두 포함한다.
- **`apps/api`** — Nest.js + TypeScript. REST API 서버.
- **DB** — SQLite. **ORM** — Prisma (SQLite와 궁합이 좋고 마이그레이션이 간단하다).
- **인증** — 사장님/직원은 JWT 기반 인증 (Nest.js + Passport-JWT). 고객은 비회원이므로 로그인 없이 이름+전화번호로 예약/조회한다.
- **Nginx** — 프로덕션 배포 시 리버스 프록시로 사용. React 정적 빌드 파일을 서빙하고 `/api` 요청을 Nest.js로 프록시한다. 개발 환경에서는 Vite dev server를 그대로 사용하며 Nginx는 배포 산출물(설정 파일)로만 다룬다.

이 서브프로젝트는 로컬 개발 환경 동작을 최우선으로 하며, 실제 배포(서버 프로비저닝 등)는 범위에서 제외한다. Nginx 설정 파일은 참고용 산출물로 작성한다.

## 데이터 모델 (Prisma schema)

- **User** — `id`, `email`(unique), `passwordHash`, `name`, `createdAt`
- **Restaurant** — `id`, `name`, `ownerId`(→User), `slotIntervalMinutes`(기본 30), `capacityPerSlot`(시간대별 총 좌석수), `createdAt`
- **RestaurantMember** — `id`, `restaurantId`(→Restaurant), `userId`(→User), `role`(`OWNER` | `STAFF`), `createdAt` — 사용자가 어느 음식점에 속하는지와 권한을 나타내는 조인 테이블. 유니크 제약: `(restaurantId, userId)`
- **BusinessHour** — `id`, `restaurantId`(→Restaurant), `dayOfWeek`(0=일요일~6=토요일), `openTime`(HH:mm), `closeTime`(HH:mm)
- **Reservation** — `id`, `restaurantId`(→Restaurant), `reservationCode`(unique, 고객 조회용 랜덤 문자열), `customerName`, `customerPhone`, `partySize`, `reservationDate`(YYYY-MM-DD), `reservationTime`(HH:mm), `status`(`CONFIRMED` | `CANCELLED` | `COMPLETED` | `NO_SHOW`), `createdByStaffId`(→User, nullable — 직원이 대신 등록한 경우), `createdAt`

**용량 체크 로직:** 예약 생성 시 `restaurantId + reservationDate + reservationTime`이 일치하고 `status=CONFIRMED`인 예약들의 `partySize` 합계를 구하고, `합계 + 신규 partySize <= capacityPerSlot`이면 확정, 아니면 409 Conflict로 거절한다.

## 인증 & 권한

- **사장님 가입:** `POST /auth/signup` (email, password, name) → User 생성. 로그인 후 `POST /restaurants` (name, slotIntervalMinutes, capacityPerSlot)로 음식점을 등록하면 자동으로 `RestaurantMember(role=OWNER)`가 생성된다.
- **로그인:** `POST /auth/login` (email, password) → JWT 발급. 이후 요청은 `Authorization: Bearer <token>` 헤더를 사용한다.
- **직원 초대:** 사장님이 `POST /restaurants/:id/staff/invite` (email, name)을 호출하면 서버가 임시 비밀번호를 생성해 응답으로 반환한다 (이메일 발송 없음 — MVP이므로 사장님이 직접 전달). `User` + `RestaurantMember(role=STAFF)`가 생성된다. 직원은 로그인 후 `PATCH /auth/password` (currentPassword, newPassword)로 비밀번호를 변경할 수 있다.
- **권한 가드:** Nest.js Guard가 JWT의 `userId`로 대상 `restaurantId`의 `RestaurantMember` 존재 여부를 확인한다. 직원 초대, 영업시간/용량 설정 등 일부 액션은 `role=OWNER`만 허용한다.

## 예약 흐름

### 고객(비회원) 예약

1. `GET /restaurants/:id/public` — 음식점 이름, 영업시간 등 공개 정보 조회
2. `GET /restaurants/:id/availability?date=YYYY-MM-DD` — 해당 날짜의 영업시간을 `slotIntervalMinutes` 단위로 나눠 슬롯 목록을 만들고, 각 슬롯의 `capacityPerSlot - 기존 CONFIRMED 예약 partySize 합계`를 잔여 좌석으로 반환한다.
3. `POST /restaurants/:id/reservations` (customerName, customerPhone, partySize, date, time) — 잔여 좌석을 트랜잭션 내에서 재검증한 뒤 확정하고 `reservationCode`를 생성해 응답한다.
4. `GET /reservations/lookup?code=&phone=` — 코드와 전화번호가 모두 일치할 때만 예약 상세를 반환한다 (불일치 시 404).
5. `PATCH /reservations/:id/cancel` (code, phone 포함) — 본인 확인 후 상태를 `CANCELLED`로 변경한다.

### 직원/사장님 예약 관리 (인증 필요)

- `GET /restaurants/:id/reservations?date=` — 목록 조회
- `POST /restaurants/:id/reservations` (인증된 staff/owner가 호출, phone 검증 없이 직접 등록) — `createdByStaffId`에 호출자 id를 기록
- `PATCH /restaurants/:id/reservations/:resId/status` (body: `CANCELLED` | `COMPLETED` | `NO_SHOW`)

## 프론트엔드 페이지 (React + Vite)

- `/signup`, `/login` — 사장님 인증
- `/dashboard` — 음식점 설정(영업시간, 슬롯 간격, 좌석수), 직원 초대, 예약 목록 조회/수동 등록/상태 변경
- `/book/:restaurantId` — 고객용 예약 페이지 (날짜 선택 → 잔여 슬롯 표시 → 예약 폼)
- `/book/:restaurantId/lookup` — 고객용 예약 조회/취소 페이지

## 에러 처리

- **동시성:** 같은 슬롯에 동시 예약 요청이 몰릴 경우를 대비해 예약 생성은 Prisma 트랜잭션 내에서 좌석 재검증 후 insert한다. SQLite는 단일 파일 DB로 기본적으로 쓰기가 직렬화되므로 트랜잭션만으로 충분하다.
- **잔여 좌석 부족:** 409 Conflict 응답 (다른 슬롯 선택 유도는 프론트에서 `availability` 재조회로 처리)
- **잘못된 code/phone 조합:** 404 Not Found
- **권한 없는 접근:** 403 Forbidden (다른 음식점의 리소스에 접근하거나 STAFF가 OWNER 전용 액션을 시도하는 경우)

## 테스트 전략

- **Nest.js(백엔드):** Jest로 서비스 단위 테스트(용량 계산 로직, 권한 가드 로직) + 주요 API 플로우에 대한 e2e 테스트 (Nest.js 공식 e2e 테스트 방식, SQLite 인메모리/임시 파일 DB 사용)
- **React(프론트엔드):** Vitest + Testing Library로 예약 폼, 가용 시간 슬롯 표시, 대시보드 예약 목록 컴포넌트 테스트

## 범위 밖 (Out of Scope)

- 주문(POS) 관리 — 다음 서브프로젝트에서 다룸
- 테이블 배치(플로어 플랜) 시각화
- 이메일/SMS 알림
- 실 배포(서버 프로비저닝, CI/CD)
- 결제 기능
