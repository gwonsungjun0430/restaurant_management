## Note

이 계획은 기존 `2026-07-02-restaurant-reservation.md`(SQLite 기반)를 **대체**한다. 새 요구사항(PostgreSQL, 테이블/좌석 관리, QR 주문, 혼잡도 표시)에 맞춰 데이터 모델과 기술 스택을 처음부터 다시 설계했다. 이 문서는 두 개의 서브프로젝트 계획 중 **1번(기반 시스템 + 예약 관리, "앱기능")**을 다룬다. 2번(테이블/좌석/QR 주문/혼잡도, "웹기능")은 `2026-07-06-restaurant-table-congestion.md`에서 다루며 이 계획이 완료된 후 이어서 진행한다.

**사용자 확인 대기 중인 가정 (응답 없어 기본값으로 진행):**
- ORM: Prisma (PostgreSQL과 궁합이 좋고 마이그레이션 관리가 쉬움)
- 기존 SQLite 계획은 폐기하고 전체 재작성

---

# 음식점 예약 관리 시스템 — 기반 시스템 + 예약 관리 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 여러 음식점(다중 테넌트)을 대상으로, 사장님/직원이 인증 후 음식점 정보(운영시간/위치/전화번호/주차여부)를 등록·관리하고, 고객이 비회원으로 시간대별·인원수 기준 예약을 할 수 있는 웹앱의 기반 시스템과 예약 기능을 구축한다.

**Architecture:** pnpm workspace 모노레포. `Server`는 NestJS + Prisma + PostgreSQL로 REST API를 제공하고, `Client`은 React + Vite + TypeScript SPA로 사장님/직원용 대시보드와 고객용 검색·예약 페이지를 제공한다. 로컬 개발은 Docker Compose로 띄운 PostgreSQL을 사용한다.

**Tech Stack:** TypeScript(strict) 전역, NestJS 10.x, Prisma 5.x + PostgreSQL 16, Passport-JWT, bcryptjs, class-validator, Jest + Supertest(백엔드 테스트), React 18 + Vite, react-router-dom, Vitest + Testing Library(프론트엔드 테스트), Docker Compose(로컬 DB), Nginx(배포 참고 산출물).

## Global Constraints

- 모든 코드는 TypeScript strict 모드로 작성한다 (`"strict": true` in tsconfig).
- 패키지 매니저는 pnpm을 사용한다 (workspace 기능 필요).
- DB는 PostgreSQL만 사용한다 (SQLite/MySQL 금지). 로컬 개발은 Docker Compose로 띄운 컨테이너를 사용한다.
- ORM은 Prisma를 사용한다.
- 이메일/SMS 발송, 결제(실제 카드 결제), 테이블 배치(플로어 플랜) 시각화는 이 계획의 범위 밖이다.
- 비밀번호 해싱은 `bcryptjs`(순수 JS, 네이티브 빌드 불필요)를 사용한다.
- 고객 예약 조회/취소는 로그인 없이 `reservationCode` + `customerPhone` 조합으로만 인증한다.
- 예약은 테이블 배치 없이 시간대별 슬롯 + 총 좌석수(`capacityPerSlot`)로 관리한다. 잔여 좌석이 있으면 승인 절차 없이 자동 확정한다.
- 각 태스크의 커밋 메시지는 Conventional Commits 형식(`feat:`, `test:`, `fix:`, `chore:` 등)을 따른다.

---

## Task 1: 모노레포 & 앱 스캐폴딩 + 로컬 PostgreSQL

**Files:**
- Create: `package.json` (workspace root)
- Create: `pnpm-workspace.yaml`
- Create: `.gitignore`
- Create: `docker-compose.yml`
- Create: `Server/package.json`
- Create: `Server/tsconfig.json`
- Create: `Server/tsconfig.build.json`
- Create: `Server/nest-cli.json`
- Create: `Server/src/main.ts`
- Create: `Server/src/app.module.ts`
- Create: `Client/package.json`
- Create: `Client/tsconfig.json`
- Create: `Client/tsconfig.node.json`
- Create: `Client/vite.config.ts`
- Create: `Client/index.html`
- Create: `Client/src/main.tsx`
- Create: `Client/src/App.tsx`

**Interfaces:**
- Consumes: 없음
- Produces: `Server` — NestJS 앱 진입점(`main.ts`)이 포트 3000에서 기동. `AppModule`은 이후 태스크에서 다른 모듈을 import할 루트 모듈. `Client` — Vite dev server가 포트 5173에서 기동. `App` 컴포넌트는 이후 태스크에서 라우팅을 추가할 루트 컴포넌트. `docker-compose.yml` — `postgres` 서비스, 이후 모든 백엔드 태스크가 여기 접속.

- [ ] **Step 1: 루트 workspace 설정 파일 작성**

`package.json`:
```json
{
  "name": "food-management",
  "private": true,
  "scripts": {
    "db:up": "docker compose up -d postgres",
    "db:down": "docker compose down",
    "dev:api": "pnpm --filter server start:dev",
    "dev:web": "pnpm --filter client dev",
    "test:api": "pnpm --filter server test",
    "test:web": "pnpm --filter client test"
  }
}
```

`pnpm-workspace.yaml`:
```yaml
packages:
  - "Server"
  - "Client"
```

`.gitignore`:
```
node_modules
dist
Client/dist
.env
```

`docker-compose.yml`:
```yaml
services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: restaurant_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

- [ ] **Step 2: NestJS API 앱 스캐폴딩**

`Server/package.json`:
```json
{
  "name": "server",
  "private": true,
  "scripts": {
    "build": "nest build",
    "start": "nest start",
    "start:dev": "nest start --watch",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:e2e": "jest --config ./test/jest-e2e.json"
  },
  "dependencies": {
    "@nestjs/common": "^10.0.0",
    "@nestjs/core": "^10.0.0",
    "@nestjs/platform-express": "^10.0.0",
    "@nestjs/config": "^3.0.0",
    "@nestjs/jwt": "^10.0.0",
    "@nestjs/passport": "^10.0.0",
    "@prisma/client": "^5.0.0",
    "passport": "^0.7.0",
    "passport-jwt": "^4.0.1",
    "bcryptjs": "^2.4.3",
    "class-transformer": "^0.5.1",
    "class-validator": "^0.14.0",
    "reflect-metadata": "^0.1.13",
    "rxjs": "^7.8.0"
  },
  "devDependencies": {
    "@nestjs/cli": "^10.0.0",
    "@nestjs/testing": "^10.0.0",
    "@types/express": "^4.17.17",
    "@types/jest": "^29.5.0",
    "@types/node": "^20.0.0",
    "@types/passport-jwt": "^4.0.0",
    "@types/supertest": "^6.0.0",
    "jest": "^29.5.0",
    "prisma": "^5.0.0",
    "supertest": "^6.3.0",
    "ts-jest": "^29.1.0",
    "ts-node": "^10.9.0",
    "typescript": "^5.1.3"
  },
  "jest": {
    "moduleFileExtensions": ["js", "json", "ts"],
    "rootDir": "src",
    "testRegex": ".*\\.spec\\.ts$",
    "transform": { "^.+\\.(t|j)s$": "ts-jest" },
    "collectCoverageFrom": ["**/*.(t|j)s"],
    "testEnvironment": "node"
  }
}
```

`Server/tsconfig.json`:
```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2021",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "strict": true,
    "skipLibCheck": true,
    "strictNullChecks": true,
    "noImplicitAny": true,
    "esModuleInterop": true
  }
}
```

`Server/tsconfig.build.json`:
```json
{
  "extends": "./tsconfig.json",
  "exclude": ["node_modules", "test", "dist", "**/*spec.ts"]
}
```

`Server/nest-cli.json`:
```json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "sourceRoot": "src"
}
```

`Server/src/app.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true })],
})
export class AppModule {}
```

`Server/src/main.ts`:
```typescript
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableCors();
  app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

- [ ] **Step 3: React + Vite 웹 앱 스캐폴딩**

`Client/package.json`:
```json
{
  "name": "client",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "test": "vitest run"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.22.0"
  },
  "devDependencies": {
    "@testing-library/jest-dom": "^6.4.0",
    "@testing-library/react": "^14.2.0",
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "@vitejs/plugin-react": "^4.2.0",
    "jsdom": "^24.0.0",
    "typescript": "^5.1.3",
    "vite": "^5.1.0",
    "vitest": "^1.3.0"
  }
}
```

`Client/tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "types": ["vitest/globals", "@testing-library/jest-dom"]
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

`Client/tsconfig.node.json`:
```json
{
  "compilerOptions": {
    "composite": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true
  },
  "include": ["vite.config.ts"]
}
```

`Client/vite.config.ts`:
```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
  },
});
```

`Client/index.html`:
```html
<!doctype html>
<html lang="ko">
  <head>
    <meta charset="UTF-8" />
    <title>음식점 예약 관리</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

`Client/src/App.tsx`:
```tsx
export default function App() {
  return <div>음식점 예약 관리</div>;
}
```

`Client/src/main.tsx`:
```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
);
```

- [ ] **Step 4: PostgreSQL 기동 + 의존성 설치 + 두 앱 기동 확인**

Run: `pnpm install`
Expected: 두 워크스페이스(`api`, `web`)의 의존성이 설치됨 (에러 없이 종료)

Run: `pnpm db:up`
Expected: `postgres` 컨테이너가 시작됨 (`docker compose ps`에서 `Up` 상태 확인)

Run: `pnpm dev:api`
Expected: 콘솔에 Nest 부트스트랩 로그 출력, 프로세스가 종료되지 않고 대기. 확인 후 Ctrl+C로 종료.

Run: `pnpm dev:web`
Expected: `VITE ready` 로그와 함께 `http://localhost:5173/` 출력. 확인 후 Ctrl+C로 종료.

- [ ] **Step 5: Commit**

```bash
git add package.json pnpm-workspace.yaml .gitignore docker-compose.yml Server Client
git commit -m "chore: scaffold pnpm workspace with NestJS api, React web apps, and local PostgreSQL"
```

---

## Task 2: Prisma 스키마 (PostgreSQL) + PrismaService

**Files:**
- Create: `Server/prisma/schema.prisma`
- Create: `Server/src/prisma/prisma.service.ts`
- Create: `Server/src/prisma/prisma.module.ts`
- Create: `Server/src/prisma/prisma.service.spec.ts`
- Modify: `Server/src/app.module.ts`
- Create: `Server/.env`
- Modify: `.gitignore`

**Interfaces:**
- Consumes: Task 1의 앱 스캐폴딩 + 실행 중인 `postgres` 컨테이너
- Produces: `PrismaModule` (전역 모듈), `PrismaService` — `PrismaClient`를 상속하며 `onModuleInit`에서 연결. `MemberRole`(`OWNER`|`STAFF`), `ReservationStatus`(`CONFIRMED`|`CANCELLED`|`COMPLETED`|`NO_SHOW`) Prisma enum. 이후 모든 백엔드 태스크에서 `PrismaService`와 이 enum들을 사용.

- [ ] **Step 1: Prisma 스키마 작성**

`Server/prisma/schema.prisma`:
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum MemberRole {
  OWNER
  STAFF
}

enum ReservationStatus {
  CONFIRMED
  CANCELLED
  COMPLETED
  NO_SHOW
}

model User {
  id           String   @id @default(uuid())
  email        String   @unique
  passwordHash String
  name         String
  createdAt    DateTime @default(now())

  ownedRestaurants  Restaurant[]       @relation("RestaurantOwner")
  memberships       RestaurantMember[]
  staffReservations Reservation[]      @relation("ReservationCreatedBy")
}

model Restaurant {
  id                  String   @id @default(uuid())
  name                String
  address             String
  phone               String
  hasParking          Boolean  @default(false)
  ownerId             String
  slotIntervalMinutes Int      @default(30)
  capacityPerSlot     Int      @default(10)
  createdAt           DateTime @default(now())

  owner         User               @relation("RestaurantOwner", fields: [ownerId], references: [id])
  members       RestaurantMember[]
  businessHours BusinessHour[]
  reservations  Reservation[]
}

model RestaurantMember {
  id           String     @id @default(uuid())
  restaurantId String
  userId       String
  role         MemberRole
  createdAt    DateTime   @default(now())

  restaurant Restaurant @relation(fields: [restaurantId], references: [id])
  user       User       @relation(fields: [userId], references: [id])

  @@unique([restaurantId, userId])
}

model BusinessHour {
  id           String @id @default(uuid())
  restaurantId String
  dayOfWeek    Int
  openTime     String
  closeTime    String

  restaurant Restaurant @relation(fields: [restaurantId], references: [id])

  @@unique([restaurantId, dayOfWeek])
}

model Reservation {
  id               String            @id @default(uuid())
  restaurantId     String
  reservationCode  String            @unique
  customerName     String
  customerPhone    String
  partySize        Int
  reservationDate  String
  reservationTime  String
  status           ReservationStatus @default(CONFIRMED)
  createdByStaffId String?
  createdAt        DateTime          @default(now())

  restaurant     Restaurant @relation(fields: [restaurantId], references: [id])
  createdByStaff User?      @relation("ReservationCreatedBy", fields: [createdByStaffId], references: [id])
}
```

- [ ] **Step 2: 환경 변수 파일 작성**

`Server/.env`:
```
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/restaurant_dev?schema=public"
JWT_SECRET="dev-secret-change-in-production"
PORT=3000
```

`.gitignore`에 다음 줄 추가:
```
Server/prisma/dev.db
```
(PostgreSQL 사용으로 실제로는 생성되지 않지만, 과거 SQLite 산출물이 실수로 커밋되는 것을 막기 위한 안전장치로 유지)

- [ ] **Step 3: 마이그레이션 생성 및 Prisma Client 생성**

Run: `pnpm db:up` (이미 떠 있지 않다면)
Run: `pnpm --filter server exec prisma migrate dev --name init`
Expected: `Server/prisma/migrations/<timestamp>_init/migration.sql` 생성, "Your database is now in sync with your schema." 출력, Prisma Client 타입 생성 완료

- [ ] **Step 4: PrismaService 작성**

`Server/src/prisma/prisma.service.ts`:
```typescript
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

`Server/src/prisma/prisma.module.ts`:
```typescript
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

- [ ] **Step 5: PrismaService 연결 테스트 작성**

`Server/src/prisma/prisma.service.spec.ts`:
```typescript
import { Test } from '@nestjs/testing';
import { PrismaService } from './prisma.service';

describe('PrismaService', () => {
  let service: PrismaService;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      providers: [PrismaService],
    }).compile();
    service = moduleRef.get(PrismaService);
    await service.onModuleInit();
  });

  afterAll(async () => {
    await service.onModuleDestroy();
  });

  it('connects and can query the User table', async () => {
    const count = await service.user.count();
    expect(typeof count).toBe('number');
  });
});
```

- [ ] **Step 6: 테스트 실행 확인**

Run: `pnpm --filter server test -- prisma.service.spec.ts`
Expected: PASS (1 passed)

- [ ] **Step 7: AppModule에 PrismaModule 등록**

`Server/src/app.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { PrismaModule } from './prisma/prisma.module';

@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true }), PrismaModule],
})
export class AppModule {}
```

- [ ] **Step 8: Commit**

```bash
git add Server/prisma Server/src/prisma Server/src/app.module.ts Server/.env .gitignore
git commit -m "feat: add Prisma schema (PostgreSQL) and PrismaService"
```

---

## Task 3: Auth — 회원가입/로그인/JWT/비밀번호 변경

**Files:**
- Create: `Server/src/auth/dto/signup.dto.ts`
- Create: `Server/src/auth/dto/login.dto.ts`
- Create: `Server/src/auth/dto/change-password.dto.ts`
- Create: `Server/src/auth/auth.service.ts`
- Create: `Server/src/auth/auth.service.spec.ts`
- Create: `Server/src/auth/auth.controller.ts`
- Create: `Server/src/auth/jwt.strategy.ts`
- Create: `Server/src/auth/jwt-auth.guard.ts`
- Create: `Server/src/auth/current-user.decorator.ts`
- Create: `Server/src/auth/auth.module.ts`
- Modify: `Server/src/app.module.ts`

**Interfaces:**
- Consumes: `PrismaService` (Task 2)
- Produces: `AuthService.signup(dto): Promise<{id, email, name}>`, `AuthService.login(dto): Promise<{accessToken: string}>`, `AuthService.changePassword(userId, dto): Promise<void>`. `JwtAuthGuard`, `CurrentUser()` 데코레이터(`{userId: string; email: string}`) — 이후 모든 인증 필요 라우트에서 사용.

- [ ] **Step 1: DTO 작성**

`Server/src/auth/dto/signup.dto.ts`:
```typescript
import { IsEmail, IsString, MinLength } from 'class-validator';

export class SignupDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;

  @IsString()
  @MinLength(1)
  name: string;
}
```

`Server/src/auth/dto/login.dto.ts`:
```typescript
import { IsEmail, IsString } from 'class-validator';

export class LoginDto {
  @IsEmail()
  email: string;

  @IsString()
  password: string;
}
```

`Server/src/auth/dto/change-password.dto.ts`:
```typescript
import { IsString, MinLength } from 'class-validator';

export class ChangePasswordDto {
  @IsString()
  currentPassword: string;

  @IsString()
  @MinLength(8)
  newPassword: string;
}
```

- [ ] **Step 2: AuthService 실패 테스트 작성**

`Server/src/auth/auth.service.spec.ts`:
```typescript
import { Test } from '@nestjs/testing';
import { ConflictException, UnauthorizedException } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { AuthService } from './auth.service';
import { PrismaService } from '../prisma/prisma.service';

describe('AuthService', () => {
  let service: AuthService;
  let prisma: PrismaService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [JwtModule.register({ secret: 'test-secret', signOptions: { expiresIn: '1h' } })],
      providers: [AuthService, PrismaService],
    }).compile();
    service = moduleRef.get(AuthService);
    prisma = moduleRef.get(PrismaService);
    await prisma.onModuleInit();
    await prisma.reservation.deleteMany();
    await prisma.restaurantMember.deleteMany();
    await prisma.restaurant.deleteMany();
    await prisma.user.deleteMany();
  });

  afterEach(async () => {
    await prisma.onModuleDestroy();
  });

  it('signs up a new user with a hashed password', async () => {
    const result = await service.signup({ email: 'owner@test.com', password: 'password123', name: '사장님' });
    expect(result.email).toBe('owner@test.com');
    const stored = await prisma.user.findUniqueOrThrow({ where: { email: 'owner@test.com' } });
    expect(stored.passwordHash).not.toBe('password123');
  });

  it('rejects signup with a duplicate email', async () => {
    await service.signup({ email: 'dup@test.com', password: 'password123', name: 'A' });
    await expect(
      service.signup({ email: 'dup@test.com', password: 'password456', name: 'B' }),
    ).rejects.toThrow(ConflictException);
  });

  it('logs in with correct credentials and returns a JWT', async () => {
    await service.signup({ email: 'login@test.com', password: 'password123', name: 'A' });
    const result = await service.login({ email: 'login@test.com', password: 'password123' });
    expect(typeof result.accessToken).toBe('string');
  });

  it('rejects login with wrong password', async () => {
    await service.signup({ email: 'wrong@test.com', password: 'password123', name: 'A' });
    await expect(
      service.login({ email: 'wrong@test.com', password: 'incorrect' }),
    ).rejects.toThrow(UnauthorizedException);
  });

  it('changes password when current password is correct', async () => {
    const user = await service.signup({ email: 'change@test.com', password: 'password123', name: 'A' });
    await service.changePassword(user.id, { currentPassword: 'password123', newPassword: 'newpassword456' });
    const result = await service.login({ email: 'change@test.com', password: 'newpassword456' });
    expect(typeof result.accessToken).toBe('string');
  });
});
```

- [ ] **Step 3: 테스트 실행 → 실패 확인**

Run: `pnpm --filter server test -- auth.service.spec.ts`
Expected: FAIL ("Cannot find module './auth.service'")

- [ ] **Step 4: AuthService 구현**

`Server/src/auth/auth.service.ts`:
```typescript
import { ConflictException, Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import * as bcrypt from 'bcryptjs';
import { PrismaService } from '../prisma/prisma.service';
import { SignupDto } from './dto/signup.dto';
import { LoginDto } from './dto/login.dto';
import { ChangePasswordDto } from './dto/change-password.dto';

@Injectable()
export class AuthService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly jwtService: JwtService,
  ) {}

  async signup(dto: SignupDto) {
    const existing = await this.prisma.user.findUnique({ where: { email: dto.email } });
    if (existing) {
      throw new ConflictException('Email already registered');
    }
    const passwordHash = await bcrypt.hash(dto.password, 10);
    const user = await this.prisma.user.create({
      data: { email: dto.email, passwordHash, name: dto.name },
    });
    return { id: user.id, email: user.email, name: user.name };
  }

  async login(dto: LoginDto) {
    const user = await this.prisma.user.findUnique({ where: { email: dto.email } });
    if (!user) {
      throw new UnauthorizedException('Invalid credentials');
    }
    const matches = await bcrypt.compare(dto.password, user.passwordHash);
    if (!matches) {
      throw new UnauthorizedException('Invalid credentials');
    }
    const accessToken = await this.jwtService.signAsync({ sub: user.id, email: user.email });
    return { accessToken };
  }

  async changePassword(userId: string, dto: ChangePasswordDto) {
    const user = await this.prisma.user.findUniqueOrThrow({ where: { id: userId } });
    const matches = await bcrypt.compare(dto.currentPassword, user.passwordHash);
    if (!matches) {
      throw new UnauthorizedException('Current password is incorrect');
    }
    const passwordHash = await bcrypt.hash(dto.newPassword, 10);
    await this.prisma.user.update({ where: { id: userId }, data: { passwordHash } });
  }
}
```

- [ ] **Step 5: 테스트 실행 → 통과 확인**

Run: `pnpm --filter server test -- auth.service.spec.ts`
Expected: PASS (5 passed)

- [ ] **Step 6: JWT 전략, 가드, 데코레이터 작성**

`Server/src/auth/jwt.strategy.ts`:
```typescript
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';

interface JwtPayload {
  sub: string;
  email: string;
}

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get<string>('JWT_SECRET', 'dev-secret-change-in-production'),
    });
  }

  validate(payload: JwtPayload) {
    return { userId: payload.sub, email: payload.email };
  }
}
```

`Server/src/auth/jwt-auth.guard.ts`:
```typescript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

`Server/src/auth/current-user.decorator.ts`:
```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export interface CurrentUserPayload {
  userId: string;
  email: string;
}

export const CurrentUser = createParamDecorator((_: unknown, ctx: ExecutionContext): CurrentUserPayload => {
  const request = ctx.switchToHttp().getRequest();
  return request.user;
});
```

- [ ] **Step 7: AuthController 및 AuthModule 작성**

`Server/src/auth/auth.controller.ts`:
```typescript
import { Body, Controller, Patch, Post, UseGuards } from '@nestjs/common';
import { AuthService } from './auth.service';
import { SignupDto } from './dto/signup.dto';
import { LoginDto } from './dto/login.dto';
import { ChangePasswordDto } from './dto/change-password.dto';
import { JwtAuthGuard } from './jwt-auth.guard';
import { CurrentUser, CurrentUserPayload } from './current-user.decorator';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post('signup')
  signup(@Body() dto: SignupDto) {
    return this.authService.signup(dto);
  }

  @Post('login')
  login(@Body() dto: LoginDto) {
    return this.authService.login(dto);
  }

  @UseGuards(JwtAuthGuard)
  @Patch('password')
  changePassword(@CurrentUser() user: CurrentUserPayload, @Body() dto: ChangePasswordDto) {
    return this.authService.changePassword(user.userId, dto);
  }
}
```

`Server/src/auth/auth.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { JwtStrategy } from './jwt.strategy';

@Module({
  imports: [
    PassportModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        secret: configService.get<string>('JWT_SECRET', 'dev-secret-change-in-production'),
        signOptions: { expiresIn: '12h' },
      }),
    }),
  ],
  providers: [AuthService, JwtStrategy],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule {}
```

- [ ] **Step 8: AppModule에 AuthModule 등록**

`Server/src/app.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { PrismaModule } from './prisma/prisma.module';
import { AuthModule } from './auth/auth.module';

@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true }), PrismaModule, AuthModule],
})
export class AppModule {}
```

- [ ] **Step 9: 전체 테스트 재실행 확인**

Run: `pnpm --filter server test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 10: Commit**

```bash
git add Server/src/auth Server/src/app.module.ts
git commit -m "feat: add signup/login/change-password auth flow with JWT"
```

---

## Task 4: Restaurants — 음식점 생성(위치/전화번호/주차여부 포함) + 멤버십 가드

**Files:**
- Create: `Server/src/restaurants/dto/create-restaurant.dto.ts`
- Create: `Server/src/restaurants/roles.decorator.ts`
- Create: `Server/src/restaurants/restaurant-member.guard.ts`
- Create: `Server/src/restaurants/restaurant-member.guard.spec.ts`
- Create: `Server/src/restaurants/restaurants.service.ts`
- Create: `Server/src/restaurants/restaurants.service.spec.ts`
- Create: `Server/src/restaurants/restaurants.controller.ts`
- Create: `Server/src/restaurants/restaurants.module.ts`
- Modify: `Server/src/app.module.ts`

**Interfaces:**
- Consumes: `PrismaService`(Task 2), `JwtAuthGuard`/`CurrentUser`(Task 3)
- Produces: `RestaurantsService.create(ownerId, dto): Promise<{id, name, address, phone, hasParking, slotIntervalMinutes, capacityPerSlot}>`, `RestaurantsService.findMine(userId): Promise<Restaurant[]>`, `RestaurantMemberGuard`(route param `id`를 restaurantId로 보고 멤버십 확인, `request.membershipRole: 'OWNER'|'STAFF'` 세팅), `@Roles('OWNER')` 데코레이터. 이후 Task 5, 6, 7과 서브프로젝트 2(테이블/QR/혼잡도)에서 이 가드와 `assertMember` 패턴을 재사용.

- [ ] **Step 1: DTO 및 Roles 데코레이터 작성**

`Server/src/restaurants/dto/create-restaurant.dto.ts`:
```typescript
import { IsBoolean, IsInt, IsString, Min, MinLength } from 'class-validator';

export class CreateRestaurantDto {
  @IsString()
  @MinLength(1)
  name: string;

  @IsString()
  @MinLength(1)
  address: string;

  @IsString()
  @MinLength(1)
  phone: string;

  @IsBoolean()
  hasParking: boolean;

  @IsInt()
  @Min(5)
  slotIntervalMinutes: number;

  @IsInt()
  @Min(1)
  capacityPerSlot: number;
}
```

`Server/src/restaurants/roles.decorator.ts`:
```typescript
import { SetMetadata } from '@nestjs/common';

export type MemberRole = 'OWNER' | 'STAFF';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: MemberRole[]) => SetMetadata(ROLES_KEY, roles);
```

- [ ] **Step 2: RestaurantMemberGuard 실패 테스트 작성**

`Server/src/restaurants/restaurant-member.guard.spec.ts`:
```typescript
import { ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Test } from '@nestjs/testing';
import { RestaurantMemberGuard } from './restaurant-member.guard';
import { PrismaService } from '../prisma/prisma.service';

function mockContext(params: Record<string, string>, user: { userId: string }, handler = () => undefined) {
  return {
    switchToHttp: () => ({
      getRequest: () => ({ params, user }),
    }),
    getHandler: () => handler,
    getClass: () => class {},
  } as unknown as ExecutionContext;
}

describe('RestaurantMemberGuard', () => {
  let guard: RestaurantMemberGuard;
  let prisma: PrismaService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      providers: [RestaurantMemberGuard, PrismaService, Reflector],
    }).compile();
    guard = moduleRef.get(RestaurantMemberGuard);
    prisma = moduleRef.get(PrismaService);
    await prisma.onModuleInit();
    await prisma.reservation.deleteMany();
    await prisma.restaurantMember.deleteMany();
    await prisma.restaurant.deleteMany();
    await prisma.user.deleteMany();
  });

  afterEach(async () => {
    await prisma.onModuleDestroy();
  });

  it('throws ForbiddenException when user is not a member of the restaurant', async () => {
    const owner = await prisma.user.create({ data: { email: 'o@t.com', passwordHash: 'x', name: 'O' } });
    const outsider = await prisma.user.create({ data: { email: 'x@t.com', passwordHash: 'x', name: 'X' } });
    const restaurant = await prisma.restaurant.create({
      data: { name: 'R', address: 'Seoul', phone: '010', ownerId: owner.id, slotIntervalMinutes: 30, capacityPerSlot: 10 },
    });
    await prisma.restaurantMember.create({ data: { restaurantId: restaurant.id, userId: owner.id, role: 'OWNER' } });

    const ctx = mockContext({ id: restaurant.id }, { userId: outsider.id });
    await expect(guard.canActivate(ctx)).rejects.toThrow(ForbiddenException);
  });

  it('allows access and attaches membershipRole for a member', async () => {
    const owner = await prisma.user.create({ data: { email: 'o2@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await prisma.restaurant.create({
      data: { name: 'R2', address: 'Seoul', phone: '010', ownerId: owner.id, slotIntervalMinutes: 30, capacityPerSlot: 10 },
    });
    await prisma.restaurantMember.create({ data: { restaurantId: restaurant.id, userId: owner.id, role: 'OWNER' } });

    const request: any = { params: { id: restaurant.id }, user: { userId: owner.id } };
    const ctx = {
      switchToHttp: () => ({ getRequest: () => request }),
      getHandler: () => undefined,
      getClass: () => class {},
    } as unknown as ExecutionContext;

    await expect(guard.canActivate(ctx)).resolves.toBe(true);
    expect(request.membershipRole).toBe('OWNER');
  });
});
```

- [ ] **Step 3: 테스트 실행 → 실패 확인**

Run: `pnpm --filter server test -- restaurant-member.guard.spec.ts`
Expected: FAIL ("Cannot find module './restaurant-member.guard'")

- [ ] **Step 4: RestaurantMemberGuard 구현**

`Server/src/restaurants/restaurant-member.guard.ts`:
```typescript
import { CanActivate, ExecutionContext, ForbiddenException, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { PrismaService } from '../prisma/prisma.service';
import { MemberRole, ROLES_KEY } from './roles.decorator';

@Injectable()
export class RestaurantMemberGuard implements CanActivate {
  constructor(
    private readonly prisma: PrismaService,
    private readonly reflector: Reflector,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const restaurantId: string = request.params.id;
    const userId: string = request.user.userId;

    const membership = await this.prisma.restaurantMember.findUnique({
      where: { restaurantId_userId: { restaurantId, userId } },
    });
    if (!membership) {
      throw new ForbiddenException('Not a member of this restaurant');
    }

    const requiredRoles = this.reflector.getAllAndOverride<MemberRole[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (requiredRoles && requiredRoles.length > 0 && !requiredRoles.includes(membership.role as MemberRole)) {
      throw new ForbiddenException(`Requires role: ${requiredRoles.join(', ')}`);
    }

    request.membershipRole = membership.role;
    return true;
  }
}
```

- [ ] **Step 5: 테스트 실행 → 통과 확인**

Run: `pnpm --filter server test -- restaurant-member.guard.spec.ts`
Expected: PASS (2 passed)

- [ ] **Step 6: RestaurantsService 실패 테스트 작성**

`Server/src/restaurants/restaurants.service.spec.ts`:
```typescript
import { Test } from '@nestjs/testing';
import { RestaurantsService } from './restaurants.service';
import { PrismaService } from '../prisma/prisma.service';

describe('RestaurantsService', () => {
  let service: RestaurantsService;
  let prisma: PrismaService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      providers: [RestaurantsService, PrismaService],
    }).compile();
    service = moduleRef.get(RestaurantsService);
    prisma = moduleRef.get(PrismaService);
    await prisma.onModuleInit();
    await prisma.reservation.deleteMany();
    await prisma.restaurantMember.deleteMany();
    await prisma.restaurant.deleteMany();
    await prisma.user.deleteMany();
  });

  afterEach(async () => {
    await prisma.onModuleDestroy();
  });

  it('creates a restaurant with address/phone/parking and makes the creator an OWNER member', async () => {
    const owner = await prisma.user.create({ data: { email: 'o@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await service.create(owner.id, {
      name: 'My Place',
      address: '서울시 강남구',
      phone: '02-1234-5678',
      hasParking: true,
      slotIntervalMinutes: 30,
      capacityPerSlot: 20,
    });

    expect(restaurant.name).toBe('My Place');
    expect(restaurant.hasParking).toBe(true);
    const membership = await prisma.restaurantMember.findUniqueOrThrow({
      where: { restaurantId_userId: { restaurantId: restaurant.id, userId: owner.id } },
    });
    expect(membership.role).toBe('OWNER');
  });

  it('returns only restaurants the user is a member of', async () => {
    const owner = await prisma.user.create({ data: { email: 'o2@t.com', passwordHash: 'x', name: 'O' } });
    const stranger = await prisma.user.create({ data: { email: 's@t.com', passwordHash: 'x', name: 'S' } });
    await service.create(owner.id, { name: 'Mine', address: 'A', phone: '02', hasParking: false, slotIntervalMinutes: 30, capacityPerSlot: 10 });

    const mine = await service.findMine(owner.id);
    const strangers = await service.findMine(stranger.id);

    expect(mine).toHaveLength(1);
    expect(strangers).toHaveLength(0);
  });
});
```

- [ ] **Step 7: 테스트 실행 → 실패 확인**

Run: `pnpm --filter server test -- restaurants.service.spec.ts`
Expected: FAIL ("Cannot find module './restaurants.service'")

- [ ] **Step 8: RestaurantsService 구현 (create, findMine)**

`Server/src/restaurants/restaurants.service.ts`:
```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { CreateRestaurantDto } from './dto/create-restaurant.dto';

@Injectable()
export class RestaurantsService {
  constructor(private readonly prisma: PrismaService) {}

  async create(ownerId: string, dto: CreateRestaurantDto) {
    return this.prisma.$transaction(async (tx) => {
      const restaurant = await tx.restaurant.create({
        data: {
          name: dto.name,
          address: dto.address,
          phone: dto.phone,
          hasParking: dto.hasParking,
          ownerId,
          slotIntervalMinutes: dto.slotIntervalMinutes,
          capacityPerSlot: dto.capacityPerSlot,
        },
      });
      await tx.restaurantMember.create({
        data: { restaurantId: restaurant.id, userId: ownerId, role: 'OWNER' },
      });
      return restaurant;
    });
  }

  async findMine(userId: string) {
    const memberships = await this.prisma.restaurantMember.findMany({
      where: { userId },
      include: { restaurant: true },
    });
    return memberships.map((m) => m.restaurant);
  }
}
```

- [ ] **Step 9: 테스트 실행 → 통과 확인**

Run: `pnpm --filter server test -- restaurants.service.spec.ts`
Expected: PASS (2 passed)

- [ ] **Step 10: RestaurantsController 및 RestaurantsModule 작성**

`Server/src/restaurants/restaurants.controller.ts`:
```typescript
import { Body, Controller, Get, Post, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/jwt-auth.guard';
import { CurrentUser, CurrentUserPayload } from '../auth/current-user.decorator';
import { RestaurantsService } from './restaurants.service';
import { CreateRestaurantDto } from './dto/create-restaurant.dto';

@Controller('restaurants')
export class RestaurantsController {
  constructor(private readonly restaurantsService: RestaurantsService) {}

  @UseGuards(JwtAuthGuard)
  @Post()
  create(@CurrentUser() user: CurrentUserPayload, @Body() dto: CreateRestaurantDto) {
    return this.restaurantsService.create(user.userId, dto);
  }

  @UseGuards(JwtAuthGuard)
  @Get('mine')
  findMine(@CurrentUser() user: CurrentUserPayload) {
    return this.restaurantsService.findMine(user.userId);
  }
}
```

`Server/src/restaurants/restaurants.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { RestaurantsService } from './restaurants.service';
import { RestaurantsController } from './restaurants.controller';
import { RestaurantMemberGuard } from './restaurant-member.guard';

@Module({
  providers: [RestaurantsService, RestaurantMemberGuard],
  controllers: [RestaurantsController],
  exports: [RestaurantsService, RestaurantMemberGuard],
})
export class RestaurantsModule {}
```

- [ ] **Step 11: AppModule에 RestaurantsModule 등록**

`Server/src/app.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { PrismaModule } from './prisma/prisma.module';
import { AuthModule } from './auth/auth.module';
import { RestaurantsModule } from './restaurants/restaurants.module';

@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true }), PrismaModule, AuthModule, RestaurantsModule],
})
export class AppModule {}
```

- [ ] **Step 12: 전체 테스트 재실행 확인**

Run: `pnpm --filter server test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 13: Commit**

```bash
git add Server/src/restaurants Server/src/app.module.ts
git commit -m "feat: add restaurant creation with address/phone/parking and membership guard"
```

---

## Task 5: Restaurants — 영업시간 CRUD + 직원 초대 + 공개 정보/검색 조회

**Files:**
- Create: `Server/src/restaurants/dto/set-business-hours.dto.ts`
- Create: `Server/src/restaurants/dto/invite-staff.dto.ts`
- Modify: `Server/src/restaurants/restaurants.service.ts`
- Modify: `Server/src/restaurants/restaurants.service.spec.ts`
- Modify: `Server/src/restaurants/restaurants.controller.ts`

**Interfaces:**
- Consumes: `RestaurantsService`, `RestaurantMemberGuard`, `Roles`(Task 4)
- Produces: `RestaurantsService.setBusinessHours(restaurantId, hours): Promise<BusinessHour[]>`, `RestaurantsService.inviteStaff(restaurantId, dto): Promise<{email, tempPassword}>`, `RestaurantsService.getPublic(restaurantId): Promise<{id, name, address, phone, hasParking, slotIntervalMinutes, businessHours}>`, `RestaurantsService.search(query?: string): Promise<PublicRestaurantSummary[]>` — 고객용 검색. Task 6(가용시간 계산)과 서브프로젝트 2(공개 페이지 혼잡도 표시)가 `getPublic`/`search` 구조를 재사용.

- [ ] **Step 1: DTO 작성**

`Server/src/restaurants/dto/set-business-hours.dto.ts`:
```typescript
import { Type } from 'class-transformer';
import { ArrayMinSize, IsArray, IsInt, IsString, Matches, Max, Min, ValidateNested } from 'class-validator';

class BusinessHourEntryDto {
  @IsInt()
  @Min(0)
  @Max(6)
  dayOfWeek: number;

  @IsString()
  @Matches(/^([01]\d|2[0-3]):([0-5]\d)$/)
  openTime: string;

  @IsString()
  @Matches(/^([01]\d|2[0-3]):([0-5]\d)$/)
  closeTime: string;
}

export class SetBusinessHoursDto {
  @IsArray()
  @ArrayMinSize(1)
  @ValidateNested({ each: true })
  @Type(() => BusinessHourEntryDto)
  hours: BusinessHourEntryDto[];
}
```

`Server/src/restaurants/dto/invite-staff.dto.ts`:
```typescript
import { IsEmail, IsString, MinLength } from 'class-validator';

export class InviteStaffDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(1)
  name: string;
}
```

- [ ] **Step 2: 실패 테스트 추가 (`restaurants.service.spec.ts` 하단에 추가)**

```typescript
describe('RestaurantsService - business hours, staff invite, public info, search', () => {
  let service: RestaurantsService;
  let prisma: PrismaService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      providers: [RestaurantsService, PrismaService],
    }).compile();
    service = moduleRef.get(RestaurantsService);
    prisma = moduleRef.get(PrismaService);
    await prisma.onModuleInit();
    await prisma.reservation.deleteMany();
    await prisma.restaurantMember.deleteMany();
    await prisma.restaurant.deleteMany();
    await prisma.user.deleteMany();
  });

  afterEach(async () => {
    await prisma.onModuleDestroy();
  });

  it('replaces business hours for a restaurant', async () => {
    const owner = await prisma.user.create({ data: { email: 'o@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await service.create(owner.id, { name: 'R', address: 'A', phone: '02', hasParking: false, slotIntervalMinutes: 30, capacityPerSlot: 10 });

    await service.setBusinessHours(restaurant.id, [{ dayOfWeek: 1, openTime: '11:00', closeTime: '21:00' }]);
    const second = await service.setBusinessHours(restaurant.id, [
      { dayOfWeek: 1, openTime: '10:00', closeTime: '20:00' },
      { dayOfWeek: 2, openTime: '10:00', closeTime: '20:00' },
    ]);

    expect(second).toHaveLength(2);
  });

  it('invites a staff member and returns a temporary password', async () => {
    const owner = await prisma.user.create({ data: { email: 'o2@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await service.create(owner.id, { name: 'R', address: 'A', phone: '02', hasParking: false, slotIntervalMinutes: 30, capacityPerSlot: 10 });

    const result = await service.inviteStaff(restaurant.id, { email: 'staff@t.com', name: 'Staff' });

    expect(result.email).toBe('staff@t.com');
    expect(typeof result.tempPassword).toBe('string');
    const membership = await prisma.restaurantMember.findFirst({
      where: { restaurantId: restaurant.id, user: { email: 'staff@t.com' } },
    });
    expect(membership?.role).toBe('STAFF');
  });

  it('rejects inviting an email that is already registered', async () => {
    const owner = await prisma.user.create({ data: { email: 'o3@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await service.create(owner.id, { name: 'R', address: 'A', phone: '02', hasParking: false, slotIntervalMinutes: 30, capacityPerSlot: 10 });
    await service.inviteStaff(restaurant.id, { email: 'dup@t.com', name: 'Staff' });

    await expect(service.inviteStaff(restaurant.id, { email: 'dup@t.com', name: 'Staff2' })).rejects.toThrow();
  });

  it('returns public info including address/phone/parking/business hours', async () => {
    const owner = await prisma.user.create({ data: { email: 'o4@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await service.create(owner.id, { name: 'Public Place', address: '서울시', phone: '02', hasParking: true, slotIntervalMinutes: 30, capacityPerSlot: 10 });
    await service.setBusinessHours(restaurant.id, [{ dayOfWeek: 1, openTime: '11:00', closeTime: '21:00' }]);

    const publicInfo = await service.getPublic(restaurant.id);

    expect(publicInfo.name).toBe('Public Place');
    expect(publicInfo.hasParking).toBe(true);
    expect(publicInfo.businessHours).toHaveLength(1);
  });

  it('searches restaurants by name (case-insensitive substring)', async () => {
    const owner = await prisma.user.create({ data: { email: 'o5@t.com', passwordHash: 'x', name: 'O' } });
    await service.create(owner.id, { name: '강남 파스타집', address: 'A', phone: '02', hasParking: false, slotIntervalMinutes: 30, capacityPerSlot: 10 });
    await service.create(owner.id, { name: '홍대 라멘집', address: 'B', phone: '02', hasParking: false, slotIntervalMinutes: 30, capacityPerSlot: 10 });

    const results = await service.search('파스타');

    expect(results).toHaveLength(1);
    expect(results[0].name).toBe('강남 파스타집');
  });

  it('returns all restaurants when search query is omitted', async () => {
    const owner = await prisma.user.create({ data: { email: 'o6@t.com', passwordHash: 'x', name: 'O' } });
    await service.create(owner.id, { name: 'A식당', address: 'A', phone: '02', hasParking: false, slotIntervalMinutes: 30, capacityPerSlot: 10 });
    await service.create(owner.id, { name: 'B식당', address: 'B', phone: '02', hasParking: false, slotIntervalMinutes: 30, capacityPerSlot: 10 });

    const results = await service.search();

    expect(results).toHaveLength(2);
  });
});
```

- [ ] **Step 3: 테스트 실행 → 실패 확인**

Run: `pnpm --filter server test -- restaurants.service.spec.ts`
Expected: FAIL (`service.setBusinessHours is not a function` 등)

- [ ] **Step 4: RestaurantsService에 메서드 추가**

`Server/src/restaurants/restaurants.service.ts` 상단 import를 다음으로 교체:
```typescript
import { ConflictException, Injectable, NotFoundException } from '@nestjs/common';
import * as bcrypt from 'bcryptjs';
import { PrismaService } from '../prisma/prisma.service';
import { CreateRestaurantDto } from './dto/create-restaurant.dto';
import { SetBusinessHoursDto } from './dto/set-business-hours.dto';
import { InviteStaffDto } from './dto/invite-staff.dto';
```

`findMine` 메서드 뒤에 다음 메서드들을 추가:
```typescript
  async setBusinessHours(restaurantId: string, hours: SetBusinessHoursDto['hours']) {
    return this.prisma.$transaction(async (tx) => {
      await tx.businessHour.deleteMany({ where: { restaurantId } });
      await tx.businessHour.createMany({
        data: hours.map((h) => ({ restaurantId, ...h })),
      });
      return tx.businessHour.findMany({ where: { restaurantId }, orderBy: { dayOfWeek: 'asc' } });
    });
  }

  async inviteStaff(restaurantId: string, dto: InviteStaffDto) {
    const existing = await this.prisma.user.findUnique({ where: { email: dto.email } });
    if (existing) {
      throw new ConflictException('Email already registered');
    }
    const tempPassword = Math.random().toString(36).slice(-10);
    const passwordHash = await bcrypt.hash(tempPassword, 10);

    await this.prisma.$transaction(async (tx) => {
      const user = await tx.user.create({ data: { email: dto.email, name: dto.name, passwordHash } });
      await tx.restaurantMember.create({ data: { restaurantId, userId: user.id, role: 'STAFF' } });
    });

    return { email: dto.email, tempPassword };
  }

  async getPublic(restaurantId: string) {
    const restaurant = await this.prisma.restaurant.findUnique({
      where: { id: restaurantId },
      include: { businessHours: true },
    });
    if (!restaurant) {
      throw new NotFoundException('Restaurant not found');
    }
    return {
      id: restaurant.id,
      name: restaurant.name,
      address: restaurant.address,
      phone: restaurant.phone,
      hasParking: restaurant.hasParking,
      slotIntervalMinutes: restaurant.slotIntervalMinutes,
      businessHours: restaurant.businessHours,
    };
  }

  async search(query?: string) {
    const restaurants = await this.prisma.restaurant.findMany({
      where: query ? { name: { contains: query, mode: 'insensitive' } } : undefined,
      orderBy: { name: 'asc' },
    });
    return restaurants.map((r) => ({
      id: r.id,
      name: r.name,
      address: r.address,
      phone: r.phone,
      hasParking: r.hasParking,
    }));
  }
```

- [ ] **Step 5: 테스트 실행 → 통과 확인**

Run: `pnpm --filter server test -- restaurants.service.spec.ts`
Expected: PASS (8 passed)

- [ ] **Step 6: 컨트롤러에 엔드포인트 추가**

`Server/src/restaurants/restaurants.controller.ts` 전체를 다음으로 교체:

```typescript
import { Body, Controller, Get, Param, Post, Put, Query, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/jwt-auth.guard';
import { CurrentUser, CurrentUserPayload } from '../auth/current-user.decorator';
import { RestaurantsService } from './restaurants.service';
import { CreateRestaurantDto } from './dto/create-restaurant.dto';
import { SetBusinessHoursDto } from './dto/set-business-hours.dto';
import { InviteStaffDto } from './dto/invite-staff.dto';
import { RestaurantMemberGuard } from './restaurant-member.guard';
import { Roles } from './roles.decorator';

@Controller('restaurants')
export class RestaurantsController {
  constructor(private readonly restaurantsService: RestaurantsService) {}

  @UseGuards(JwtAuthGuard)
  @Post()
  create(@CurrentUser() user: CurrentUserPayload, @Body() dto: CreateRestaurantDto) {
    return this.restaurantsService.create(user.userId, dto);
  }

  @UseGuards(JwtAuthGuard)
  @Get('mine')
  findMine(@CurrentUser() user: CurrentUserPayload) {
    return this.restaurantsService.findMine(user.userId);
  }

  @Get('search')
  search(@Query('q') q?: string) {
    return this.restaurantsService.search(q);
  }

  @Get(':id/public')
  getPublic(@Param('id') id: string) {
    return this.restaurantsService.getPublic(id);
  }

  @UseGuards(JwtAuthGuard, RestaurantMemberGuard)
  @Roles('OWNER')
  @Put(':id/business-hours')
  setBusinessHours(@Param('id') id: string, @Body() dto: SetBusinessHoursDto) {
    return this.restaurantsService.setBusinessHours(id, dto.hours);
  }

  @UseGuards(JwtAuthGuard, RestaurantMemberGuard)
  @Roles('OWNER')
  @Post(':id/staff/invite')
  inviteStaff(@Param('id') id: string, @Body() dto: InviteStaffDto) {
    return this.restaurantsService.inviteStaff(id, dto);
  }
}
```

> 라우트 순서 주의: `search`, `:id/public`보다 먼저 정의되어야 NestJS가 `/restaurants/search`를 `:id`로 오인하지 않는다 (Express 라우터는 등록 순서대로 매칭).

- [ ] **Step 7: 전체 테스트 재실행 확인**

Run: `pnpm --filter server test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 8: Commit**

```bash
git add Server/src/restaurants
git commit -m "feat: add business hours, staff invite, public info, and search endpoints"
```

---

## Task 6: Reservations — 가용시간(슬롯) 계산 유틸

**Files:**
- Create: `Server/src/reservations/slot.util.ts`
- Create: `Server/src/reservations/slot.util.spec.ts`

**Interfaces:**
- Consumes: 없음 (순수 함수)
- Produces: `generateSlots(openTime: string, closeTime: string, intervalMinutes: number): string[]` — `"HH:mm"` 슬롯 배열. Task 7의 `ReservationsService.getAvailability`가 사용.

- [ ] **Step 1: 실패 테스트 작성**

`Server/src/reservations/slot.util.spec.ts`:
```typescript
import { generateSlots } from './slot.util';

describe('generateSlots', () => {
  it('generates 30-minute slots between open and close time', () => {
    expect(generateSlots('11:00', '13:00', 30)).toEqual(['11:00', '11:30', '12:00', '12:30']);
  });

  it('excludes the close time itself', () => {
    const slots = generateSlots('11:00', '12:00', 30);
    expect(slots).not.toContain('12:00');
  });

  it('handles a partial final interval by not including it', () => {
    expect(generateSlots('11:00', '11:40', 30)).toEqual(['11:00']);
  });

  it('returns an empty array when open equals close', () => {
    expect(generateSlots('11:00', '11:00', 30)).toEqual([]);
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter server test -- slot.util.spec.ts`
Expected: FAIL ("Cannot find module './slot.util'")

- [ ] **Step 3: 구현**

`Server/src/reservations/slot.util.ts`:
```typescript
function toMinutes(time: string): number {
  const [h, m] = time.split(':').map(Number);
  return h * 60 + m;
}

function toTimeString(minutes: number): string {
  const h = Math.floor(minutes / 60)
    .toString()
    .padStart(2, '0');
  const m = (minutes % 60).toString().padStart(2, '0');
  return `${h}:${m}`;
}

export function generateSlots(openTime: string, closeTime: string, intervalMinutes: number): string[] {
  const start = toMinutes(openTime);
  const end = toMinutes(closeTime);
  const slots: string[] = [];
  for (let t = start; t + intervalMinutes <= end; t += intervalMinutes) {
    slots.push(toTimeString(t));
  }
  return slots;
}
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `pnpm --filter server test -- slot.util.spec.ts`
Expected: PASS (4 passed)

- [ ] **Step 5: Commit**

```bash
git add Server/src/reservations/slot.util.ts Server/src/reservations/slot.util.spec.ts
git commit -m "feat: add slot generation utility for reservation availability"
```

---

## Task 7: Reservations — 예약 생성/조회/취소/상태 변경 (고객 + 직원)

**Files:**
- Create: `Server/src/reservations/dto/create-reservation.dto.ts`
- Create: `Server/src/reservations/dto/lookup-reservation.dto.ts`
- Create: `Server/src/reservations/dto/update-reservation-status.dto.ts`
- Create: `Server/src/reservations/reservations.service.ts`
- Create: `Server/src/reservations/reservations.service.spec.ts`
- Create: `Server/src/reservations/reservations.controller.ts`
- Create: `Server/src/reservations/reservations.module.ts`
- Modify: `Server/src/app.module.ts`

**Interfaces:**
- Consumes: `PrismaService`(Task 2), `generateSlots`(Task 6), `RestaurantsService.getPublic`(Task 5), `JwtAuthGuard`/`CurrentUser`(Task 3), `RestaurantMemberGuard`/`Roles`(Task 4)
- Produces: `ReservationsService.getAvailability(restaurantId, date): Promise<{time: string; remaining: number}[]>`, `ReservationsService.create(restaurantId, dto, createdByStaffId?: string): Promise<Reservation>`, `ReservationsService.lookup(code, phone): Promise<Reservation>`, `ReservationsService.cancel(id, code, phone): Promise<Reservation>`, `ReservationsService.listByDate(restaurantId, date): Promise<Reservation[]>`, `ReservationsService.updateStatus(id, status): Promise<Reservation>`. 이 서비스는 서브프로젝트 2 계획에서 직접 참조하지 않는다(테이블/좌석은 별도 독립 좌석 풀).

- [ ] **Step 1: DTO 작성**

`Server/src/reservations/dto/create-reservation.dto.ts`:
```typescript
import { IsInt, IsString, Matches, Min, MinLength } from 'class-validator';

export class CreateReservationDto {
  @IsString()
  @MinLength(1)
  customerName: string;

  @IsString()
  @MinLength(1)
  customerPhone: string;

  @IsInt()
  @Min(1)
  partySize: number;

  @IsString()
  @Matches(/^\d{4}-\d{2}-\d{2}$/)
  reservationDate: string;

  @IsString()
  @Matches(/^([01]\d|2[0-3]):([0-5]\d)$/)
  reservationTime: string;
}
```

`Server/src/reservations/dto/lookup-reservation.dto.ts`:
```typescript
import { IsString, MinLength } from 'class-validator';

export class LookupReservationDto {
  @IsString()
  @MinLength(1)
  code: string;

  @IsString()
  @MinLength(1)
  phone: string;
}
```

`Server/src/reservations/dto/update-reservation-status.dto.ts`:
```typescript
import { IsIn } from 'class-validator';

export type ReservationStatusValue = 'CANCELLED' | 'COMPLETED' | 'NO_SHOW';

export class UpdateReservationStatusDto {
  @IsIn(['CANCELLED', 'COMPLETED', 'NO_SHOW'])
  status: ReservationStatusValue;
}
```

- [ ] **Step 2: ReservationsService 실패 테스트 작성**

`Server/src/reservations/reservations.service.spec.ts`:
```typescript
import { ConflictException, NotFoundException } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import { ReservationsService } from './reservations.service';
import { RestaurantsService } from '../restaurants/restaurants.service';
import { PrismaService } from '../prisma/prisma.service';

describe('ReservationsService', () => {
  let service: ReservationsService;
  let restaurants: RestaurantsService;
  let prisma: PrismaService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      providers: [ReservationsService, RestaurantsService, PrismaService],
    }).compile();
    service = moduleRef.get(ReservationsService);
    restaurants = moduleRef.get(RestaurantsService);
    prisma = moduleRef.get(PrismaService);
    await prisma.onModuleInit();
    await prisma.reservation.deleteMany();
    await prisma.businessHour.deleteMany();
    await prisma.restaurantMember.deleteMany();
    await prisma.restaurant.deleteMany();
    await prisma.user.deleteMany();
  });

  afterEach(async () => {
    await prisma.onModuleDestroy();
  });

  async function setupRestaurant(capacityPerSlot = 10) {
    const owner = await prisma.user.create({ data: { email: `o${Date.now()}${Math.random()}@t.com`, passwordHash: 'x', name: 'O' } });
    const restaurant = await restaurants.create(owner.id, {
      name: 'R',
      address: 'A',
      phone: '02',
      hasParking: false,
      slotIntervalMinutes: 30,
      capacityPerSlot,
    });
    await restaurants.setBusinessHours(restaurant.id, [{ dayOfWeek: 1, openTime: '11:00', closeTime: '13:00' }]);
    return restaurant;
  }

  it('computes remaining seats per slot with no reservations', async () => {
    const restaurant = await setupRestaurant(10);
    // 2026-07-06 is a Monday (dayOfWeek 1)
    const availability = await service.getAvailability(restaurant.id, '2026-07-06');
    expect(availability).toEqual([
      { time: '11:00', remaining: 10 },
      { time: '11:30', remaining: 10 },
      { time: '12:00', remaining: 10 },
      { time: '12:30', remaining: 10 },
    ]);
  });

  it('reduces remaining seats after a confirmed reservation', async () => {
    const restaurant = await setupRestaurant(10);
    await service.create(restaurant.id, {
      customerName: '홍길동',
      customerPhone: '010-1111-2222',
      partySize: 4,
      reservationDate: '2026-07-06',
      reservationTime: '11:00',
    });

    const availability = await service.getAvailability(restaurant.id, '2026-07-06');
    expect(availability.find((s) => s.time === '11:00')?.remaining).toBe(6);
  });

  it('rejects a reservation that exceeds remaining capacity', async () => {
    const restaurant = await setupRestaurant(5);
    await service.create(restaurant.id, {
      customerName: 'A',
      customerPhone: '010-0000-0000',
      partySize: 5,
      reservationDate: '2026-07-06',
      reservationTime: '11:00',
    });

    await expect(
      service.create(restaurant.id, {
        customerName: 'B',
        customerPhone: '010-1111-1111',
        partySize: 1,
        reservationDate: '2026-07-06',
        reservationTime: '11:00',
      }),
    ).rejects.toThrow(ConflictException);
  });

  it('generates a unique reservationCode and allows lookup by code+phone', async () => {
    const restaurant = await setupRestaurant(10);
    const created = await service.create(restaurant.id, {
      customerName: '홍길동',
      customerPhone: '010-1111-2222',
      partySize: 2,
      reservationDate: '2026-07-06',
      reservationTime: '11:00',
    });

    const found = await service.lookup(created.reservationCode, '010-1111-2222');
    expect(found.id).toBe(created.id);
  });

  it('rejects lookup with a mismatched phone number', async () => {
    const restaurant = await setupRestaurant(10);
    const created = await service.create(restaurant.id, {
      customerName: '홍길동',
      customerPhone: '010-1111-2222',
      partySize: 2,
      reservationDate: '2026-07-06',
      reservationTime: '11:00',
    });

    await expect(service.lookup(created.reservationCode, '010-0000-0000')).rejects.toThrow(NotFoundException);
  });

  it('cancels a reservation when code and phone match', async () => {
    const restaurant = await setupRestaurant(10);
    const created = await service.create(restaurant.id, {
      customerName: '홍길동',
      customerPhone: '010-1111-2222',
      partySize: 2,
      reservationDate: '2026-07-06',
      reservationTime: '11:00',
    });

    const cancelled = await service.cancel(created.id, created.reservationCode, '010-1111-2222');
    expect(cancelled.status).toBe('CANCELLED');

    const availability = await service.getAvailability(restaurant.id, '2026-07-06');
    expect(availability.find((s) => s.time === '11:00')?.remaining).toBe(10);
  });

  it('lists reservations by restaurant and date', async () => {
    const restaurant = await setupRestaurant(10);
    await service.create(restaurant.id, {
      customerName: 'A',
      customerPhone: '010-1111-1111',
      partySize: 2,
      reservationDate: '2026-07-06',
      reservationTime: '11:00',
    });

    const list = await service.listByDate(restaurant.id, '2026-07-06');
    expect(list).toHaveLength(1);
  });

  it('updates reservation status', async () => {
    const restaurant = await setupRestaurant(10);
    const created = await service.create(restaurant.id, {
      customerName: 'A',
      customerPhone: '010-1111-1111',
      partySize: 2,
      reservationDate: '2026-07-06',
      reservationTime: '11:00',
    });

    const updated = await service.updateStatus(created.id, 'COMPLETED');
    expect(updated.status).toBe('COMPLETED');
  });
});
```

- [ ] **Step 3: 테스트 실행 → 실패 확인**

Run: `pnpm --filter server test -- reservations.service.spec.ts`
Expected: FAIL ("Cannot find module './reservations.service'")

- [ ] **Step 4: ReservationsService 구현**

`Server/src/reservations/reservations.service.ts`:
```typescript
import { ConflictException, Injectable, NotFoundException } from '@nestjs/common';
import { randomBytes } from 'crypto';
import { PrismaService } from '../prisma/prisma.service';
import { generateSlots } from './slot.util';
import { CreateReservationDto } from './dto/create-reservation.dto';
import { ReservationStatusValue } from './dto/update-reservation-status.dto';

function dateOfWeek(dateStr: string): number {
  return new Date(`${dateStr}T00:00:00Z`).getUTCDay();
}

function generateReservationCode(): string {
  return randomBytes(4).toString('hex').toUpperCase();
}

@Injectable()
export class ReservationsService {
  constructor(private readonly prisma: PrismaService) {}

  async getAvailability(restaurantId: string, date: string) {
    const restaurant = await this.prisma.restaurant.findUniqueOrThrow({ where: { id: restaurantId } });
    const businessHour = await this.prisma.businessHour.findUnique({
      where: { restaurantId_dayOfWeek: { restaurantId, dayOfWeek: dateOfWeek(date) } },
    });
    if (!businessHour) {
      return [];
    }

    const slots = generateSlots(businessHour.openTime, businessHour.closeTime, restaurant.slotIntervalMinutes);
    const reservations = await this.prisma.reservation.findMany({
      where: { restaurantId, reservationDate: date, status: 'CONFIRMED' },
    });

    return slots.map((time) => {
      const booked = reservations
        .filter((r) => r.reservationTime === time)
        .reduce((sum, r) => sum + r.partySize, 0);
      return { time, remaining: restaurant.capacityPerSlot - booked };
    });
  }

  async create(restaurantId: string, dto: CreateReservationDto, createdByStaffId?: string) {
    return this.prisma.$transaction(async (tx) => {
      const restaurant = await tx.restaurant.findUniqueOrThrow({ where: { id: restaurantId } });
      const existing = await tx.reservation.findMany({
        where: {
          restaurantId,
          reservationDate: dto.reservationDate,
          reservationTime: dto.reservationTime,
          status: 'CONFIRMED',
        },
      });
      const booked = existing.reduce((sum, r) => sum + r.partySize, 0);
      if (booked + dto.partySize > restaurant.capacityPerSlot) {
        throw new ConflictException('No remaining seats for this slot');
      }

      return tx.reservation.create({
        data: {
          restaurantId,
          reservationCode: generateReservationCode(),
          customerName: dto.customerName,
          customerPhone: dto.customerPhone,
          partySize: dto.partySize,
          reservationDate: dto.reservationDate,
          reservationTime: dto.reservationTime,
          createdByStaffId: createdByStaffId ?? null,
        },
      });
    });
  }

  async lookup(code: string, phone: string) {
    const reservation = await this.prisma.reservation.findUnique({ where: { reservationCode: code } });
    if (!reservation || reservation.customerPhone !== phone) {
      throw new NotFoundException('Reservation not found');
    }
    return reservation;
  }

  async cancel(id: string, code: string, phone: string) {
    const reservation = await this.lookup(code, phone);
    if (reservation.id !== id) {
      throw new NotFoundException('Reservation not found');
    }
    return this.prisma.reservation.update({ where: { id }, data: { status: 'CANCELLED' } });
  }

  async listByDate(restaurantId: string, date: string) {
    return this.prisma.reservation.findMany({
      where: { restaurantId, reservationDate: date },
      orderBy: { reservationTime: 'asc' },
    });
  }

  async updateStatus(id: string, status: ReservationStatusValue) {
    return this.prisma.reservation.update({ where: { id }, data: { status } });
  }
}
```

- [ ] **Step 5: 테스트 실행 → 통과 확인**

Run: `pnpm --filter server test -- reservations.service.spec.ts`
Expected: PASS (9 passed)

- [ ] **Step 6: ReservationsController 및 ReservationsModule 작성**

`Server/src/reservations/reservations.controller.ts`:
```typescript
import { Body, Controller, Get, Param, Patch, Post, Query, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/jwt-auth.guard';
import { CurrentUser, CurrentUserPayload } from '../auth/current-user.decorator';
import { RestaurantMemberGuard } from '../restaurants/restaurant-member.guard';
import { ReservationsService } from './reservations.service';
import { CreateReservationDto } from './dto/create-reservation.dto';
import { LookupReservationDto } from './dto/lookup-reservation.dto';
import { UpdateReservationStatusDto } from './dto/update-reservation-status.dto';

@Controller()
export class ReservationsController {
  constructor(private readonly reservationsService: ReservationsService) {}

  @Get('restaurants/:id/availability')
  getAvailability(@Param('id') id: string, @Query('date') date: string) {
    return this.reservationsService.getAvailability(id, date);
  }

  @Post('restaurants/:id/reservations')
  createPublic(@Param('id') id: string, @Body() dto: CreateReservationDto) {
    return this.reservationsService.create(id, dto);
  }

  @Get('reservations/lookup')
  lookup(@Query('code') code: string, @Query('phone') phone: string) {
    return this.reservationsService.lookup(code, phone);
  }

  @Patch('reservations/:id/cancel')
  cancel(@Param('id') id: string, @Body() dto: LookupReservationDto) {
    return this.reservationsService.cancel(id, dto.code, dto.phone);
  }

  @UseGuards(JwtAuthGuard, RestaurantMemberGuard)
  @Get('restaurants/:id/reservations')
  listByDate(@Param('id') id: string, @Query('date') date: string) {
    return this.reservationsService.listByDate(id, date);
  }

  @UseGuards(JwtAuthGuard, RestaurantMemberGuard)
  @Post('restaurants/:id/reservations/staff')
  createByStaff(
    @Param('id') id: string,
    @Body() dto: CreateReservationDto,
    @CurrentUser() user: CurrentUserPayload,
  ) {
    return this.reservationsService.create(id, dto, user.userId);
  }

  @UseGuards(JwtAuthGuard, RestaurantMemberGuard)
  @Patch('restaurants/:id/reservations/:resId/status')
  updateStatus(@Param('resId') resId: string, @Body() dto: UpdateReservationStatusDto) {
    return this.reservationsService.updateStatus(resId, dto.status);
  }
}
```

`Server/src/reservations/reservations.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { RestaurantsModule } from '../restaurants/restaurants.module';
import { ReservationsService } from './reservations.service';
import { ReservationsController } from './reservations.controller';

@Module({
  imports: [RestaurantsModule],
  providers: [ReservationsService],
  controllers: [ReservationsController],
  exports: [ReservationsService],
})
export class ReservationsModule {}
```

- [ ] **Step 7: AppModule에 ReservationsModule 등록**

`Server/src/app.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { PrismaModule } from './prisma/prisma.module';
import { AuthModule } from './auth/auth.module';
import { RestaurantsModule } from './restaurants/restaurants.module';
import { ReservationsModule } from './reservations/reservations.module';

@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true }), PrismaModule, AuthModule, RestaurantsModule, ReservationsModule],
})
export class AppModule {}
```

- [ ] **Step 8: 전체 테스트 재실행 확인**

Run: `pnpm --filter server test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 9: Commit**

```bash
git add Server/src/reservations Server/src/app.module.ts
git commit -m "feat: add reservation availability, booking, lookup/cancel, and staff management endpoints"
```

---

## Task 8: Frontend — API 클라이언트 + 라우팅 셸 + 인증 페이지

**Files:**
- Create: `Client/src/api/client.ts`
- Create: `Client/src/auth/AuthContext.tsx`
- Create: `Client/src/auth/SignupPage.tsx`
- Create: `Client/src/auth/LoginPage.tsx`
- Create: `Client/src/auth/LoginPage.test.tsx`
- Modify: `Client/src/App.tsx`
- Create: `Client/.env`

**Interfaces:**
- Consumes: 없음 (Task 1의 스캐폴딩)
- Produces: `apiClient.post<T>(path, body): Promise<T>`, `apiClient.get<T>(path): Promise<T>` — 이후 모든 프론트 태스크가 사용. `AuthContext`(`{token, login(token), logout()}`) — `localStorage`에 JWT 저장. 라우트: `/signup`, `/login`.

- [ ] **Step 1: 환경변수 및 API 클라이언트 작성**

`Client/.env`:
```
VITE_API_BASE_URL=http://localhost:3000
```

`Client/src/api/client.ts`:
```typescript
const BASE_URL = import.meta.env.VITE_API_BASE_URL ?? 'http://localhost:3000';

async function request<T>(path: string, options: RequestInit = {}): Promise<T> {
  const token = localStorage.getItem('accessToken');
  const response = await fetch(`${BASE_URL}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...options.headers,
    },
  });
  if (!response.ok) {
    const body = await response.json().catch(() => ({}));
    throw new Error(body.message ?? `Request failed with status ${response.status}`);
  }
  if (response.status === 204) {
    return undefined as T;
  }
  return response.json() as Promise<T>;
}

export const apiClient = {
  get: <T>(path: string) => request<T>(path, { method: 'GET' }),
  post: <T>(path: string, body?: unknown) =>
    request<T>(path, { method: 'POST', body: body ? JSON.stringify(body) : undefined }),
  put: <T>(path: string, body?: unknown) =>
    request<T>(path, { method: 'PUT', body: body ? JSON.stringify(body) : undefined }),
  patch: <T>(path: string, body?: unknown) =>
    request<T>(path, { method: 'PATCH', body: body ? JSON.stringify(body) : undefined }),
};
```

- [ ] **Step 2: AuthContext 작성**

`Client/src/auth/AuthContext.tsx`:
```tsx
import { createContext, useContext, useMemo, useState, type ReactNode } from 'react';

interface AuthContextValue {
  token: string | null;
  login: (token: string) => void;
  logout: () => void;
}

const AuthContext = createContext<AuthContextValue | undefined>(undefined);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [token, setToken] = useState<string | null>(() => localStorage.getItem('accessToken'));

  const value = useMemo<AuthContextValue>(
    () => ({
      token,
      login: (newToken: string) => {
        localStorage.setItem('accessToken', newToken);
        setToken(newToken);
      },
      logout: () => {
        localStorage.removeItem('accessToken');
        setToken(null);
      },
    }),
    [token],
  );

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (!ctx) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return ctx;
}
```

- [ ] **Step 3: 실패 테스트 작성 (LoginPage)**

`Client/src/auth/LoginPage.test.tsx`:
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { describe, expect, it, vi, beforeEach } from 'vitest';
import { LoginPage } from './LoginPage';
import { AuthProvider } from './AuthContext';
import { apiClient } from '../api/client';

vi.mock('../api/client', () => ({
  apiClient: { post: vi.fn() },
}));

describe('LoginPage', () => {
  beforeEach(() => {
    vi.mocked(apiClient.post).mockReset();
    localStorage.clear();
  });

  it('logs in and stores the access token', async () => {
    vi.mocked(apiClient.post).mockResolvedValue({ accessToken: 'test-token' });

    render(
      <AuthProvider>
        <MemoryRouter>
          <LoginPage />
        </MemoryRouter>
      </AuthProvider>,
    );

    fireEvent.change(screen.getByLabelText('이메일'), { target: { value: 'owner@test.com' } });
    fireEvent.change(screen.getByLabelText('비밀번호'), { target: { value: 'password123' } });
    fireEvent.click(screen.getByRole('button', { name: '로그인' }));

    await waitFor(() => {
      expect(localStorage.getItem('accessToken')).toBe('test-token');
    });
    expect(apiClient.post).toHaveBeenCalledWith('/auth/login', {
      email: 'owner@test.com',
      password: 'password123',
    });
  });
});
```

- [ ] **Step 4: 테스트 실행 → 실패 확인**

Run: `pnpm --filter client test -- LoginPage.test.tsx`
Expected: FAIL ("Cannot find module './LoginPage'")

- [ ] **Step 5: SignupPage, LoginPage 구현**

`Client/src/auth/SignupPage.tsx`:
```tsx
import { FormEvent, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { apiClient } from '../api/client';

export function SignupPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [name, setName] = useState('');
  const [error, setError] = useState<string | null>(null);
  const navigate = useNavigate();

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    setError(null);
    try {
      await apiClient.post('/auth/signup', { email, password, name });
      navigate('/login');
    } catch (err) {
      setError(err instanceof Error ? err.message : '가입에 실패했습니다');
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <h1>사장님 회원가입</h1>
      <label htmlFor="signup-email">이메일</label>
      <input id="signup-email" value={email} onChange={(e) => setEmail(e.target.value)} />
      <label htmlFor="signup-password">비밀번호</label>
      <input id="signup-password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} />
      <label htmlFor="signup-name">이름</label>
      <input id="signup-name" value={name} onChange={(e) => setName(e.target.value)} />
      {error && <p role="alert">{error}</p>}
      <button type="submit">가입하기</button>
    </form>
  );
}
```

`Client/src/auth/LoginPage.tsx`:
```tsx
import { FormEvent, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { apiClient } from '../api/client';
import { useAuth } from './AuthContext';

export function LoginPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState<string | null>(null);
  const { login } = useAuth();
  const navigate = useNavigate();

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    setError(null);
    try {
      const result = await apiClient.post<{ accessToken: string }>('/auth/login', { email, password });
      login(result.accessToken);
      navigate('/dashboard');
    } catch (err) {
      setError(err instanceof Error ? err.message : '로그인에 실패했습니다');
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <h1>로그인</h1>
      <label htmlFor="login-email">이메일</label>
      <input id="login-email" value={email} onChange={(e) => setEmail(e.target.value)} />
      <label htmlFor="login-password">비밀번호</label>
      <input id="login-password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} />
      {error && <p role="alert">{error}</p>}
      <button type="submit">로그인</button>
    </form>
  );
}
```

> 테스트의 `getByLabelText('이메일')`이 매칭되도록 `<label htmlFor="login-email">이메일</label>`을 사용한다 (Testing Library는 `htmlFor`/`id` 연결을 통해 label 텍스트로 input을 찾는다).

- [ ] **Step 6: 테스트 실행 → 통과 확인**

Run: `pnpm --filter client test -- LoginPage.test.tsx`
Expected: PASS (1 passed)

- [ ] **Step 7: App.tsx에 라우팅 추가**

`Client/src/App.tsx`:
```tsx
import { BrowserRouter, Navigate, Route, Routes } from 'react-router-dom';
import { AuthProvider } from './auth/AuthContext';
import { SignupPage } from './auth/SignupPage';
import { LoginPage } from './auth/LoginPage';

export default function App() {
  return (
    <AuthProvider>
      <BrowserRouter>
        <Routes>
          <Route path="/" element={<Navigate to="/login" replace />} />
          <Route path="/signup" element={<SignupPage />} />
          <Route path="/login" element={<LoginPage />} />
        </Routes>
      </BrowserRouter>
    </AuthProvider>
  );
}
```

- [ ] **Step 8: Commit**

```bash
git add Client/src/api Client/src/auth Client/src/App.tsx Client/.env
git commit -m "feat: add API client, auth context, and signup/login pages"
```

---

## Task 9: Frontend — 대시보드: 음식점 설정(위치/전화번호/주차/영업시간) + 직원 초대

**Files:**
- Create: `Client/src/dashboard/DashboardLayout.tsx`
- Create: `Client/src/dashboard/RestaurantSettingsPage.tsx`
- Create: `Client/src/dashboard/RestaurantSettingsPage.test.tsx`
- Modify: `Client/src/App.tsx`

**Interfaces:**
- Consumes: `apiClient`(Task 8), `useAuth`(Task 8)
- Produces: `/dashboard/settings` 라우트 — 음식점 생성 폼(없으면 생성), 있으면 영업시간/직원초대 편집 폼. Task 10, 11이 `DashboardLayout`을 재사용.

- [ ] **Step 1: 실패 테스트 작성**

`Client/src/dashboard/RestaurantSettingsPage.test.tsx`:
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { describe, expect, it, vi, beforeEach } from 'vitest';
import { RestaurantSettingsPage } from './RestaurantSettingsPage';
import { apiClient } from '../api/client';

vi.mock('../api/client', () => ({
  apiClient: { get: vi.fn(), post: vi.fn(), put: vi.fn() },
}));

describe('RestaurantSettingsPage', () => {
  beforeEach(() => {
    vi.mocked(apiClient.get).mockReset();
    vi.mocked(apiClient.post).mockReset();
  });

  it('shows a create-restaurant form when the user has no restaurant yet', async () => {
    vi.mocked(apiClient.get).mockResolvedValue([]);

    render(<RestaurantSettingsPage />);

    await waitFor(() => {
      expect(screen.getByRole('heading', { name: '음식점 등록' })).toBeInTheDocument();
    });
  });

  it('submits the create-restaurant form with address/phone/parking fields', async () => {
    vi.mocked(apiClient.get).mockResolvedValue([]);
    vi.mocked(apiClient.post).mockResolvedValue({ id: 'r1', name: '내 식당' });

    render(<RestaurantSettingsPage />);
    await waitFor(() => screen.getByRole('heading', { name: '음식점 등록' }));

    fireEvent.change(screen.getByLabelText('이름'), { target: { value: '내 식당' } });
    fireEvent.change(screen.getByLabelText('주소'), { target: { value: '서울시 강남구' } });
    fireEvent.change(screen.getByLabelText('전화번호'), { target: { value: '02-1234-5678' } });
    fireEvent.click(screen.getByLabelText('주차 가능'));
    fireEvent.change(screen.getByLabelText('예약 슬롯 간격(분)'), { target: { value: '30' } });
    fireEvent.change(screen.getByLabelText('슬롯당 좌석수'), { target: { value: '20' } });
    fireEvent.click(screen.getByRole('button', { name: '등록하기' }));

    await waitFor(() => {
      expect(apiClient.post).toHaveBeenCalledWith('/restaurants', {
        name: '내 식당',
        address: '서울시 강남구',
        phone: '02-1234-5678',
        hasParking: true,
        slotIntervalMinutes: 30,
        capacityPerSlot: 20,
      });
    });
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter client test -- RestaurantSettingsPage.test.tsx`
Expected: FAIL ("Cannot find module './RestaurantSettingsPage'")

- [ ] **Step 3: RestaurantSettingsPage 구현**

`Client/src/dashboard/RestaurantSettingsPage.tsx`:
```tsx
import { FormEvent, useEffect, useState } from 'react';
import { apiClient } from '../api/client';

interface Restaurant {
  id: string;
  name: string;
  address?: string;
  phone?: string;
  hasParking?: boolean;
  slotIntervalMinutes?: number;
  capacityPerSlot?: number;
}

export function RestaurantSettingsPage() {
  const [restaurants, setRestaurants] = useState<Restaurant[] | null>(null);
  const [name, setName] = useState('');
  const [address, setAddress] = useState('');
  const [phone, setPhone] = useState('');
  const [hasParking, setHasParking] = useState(false);
  const [slotIntervalMinutes, setSlotIntervalMinutes] = useState(30);
  const [capacityPerSlot, setCapacityPerSlot] = useState(10);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    apiClient.get<Restaurant[]>('/restaurants/mine').then(setRestaurants);
  }, []);

  async function handleCreate(e: FormEvent) {
    e.preventDefault();
    setError(null);
    try {
      const created = await apiClient.post<Restaurant>('/restaurants', {
        name,
        address,
        phone,
        hasParking,
        slotIntervalMinutes,
        capacityPerSlot,
      });
      setRestaurants([created]);
    } catch (err) {
      setError(err instanceof Error ? err.message : '등록에 실패했습니다');
    }
  }

  if (restaurants === null) {
    return <p>불러오는 중...</p>;
  }

  if (restaurants.length === 0) {
    return (
      <form onSubmit={handleCreate}>
        <h1>음식점 등록</h1>
        <label htmlFor="rest-name">이름</label>
        <input id="rest-name" value={name} onChange={(e) => setName(e.target.value)} />
        <label htmlFor="rest-address">주소</label>
        <input id="rest-address" value={address} onChange={(e) => setAddress(e.target.value)} />
        <label htmlFor="rest-phone">전화번호</label>
        <input id="rest-phone" value={phone} onChange={(e) => setPhone(e.target.value)} />
        <label htmlFor="rest-parking">주차 가능</label>
        <input
          id="rest-parking"
          type="checkbox"
          checked={hasParking}
          onChange={(e) => setHasParking(e.target.checked)}
        />
        <label htmlFor="rest-slot-interval">예약 슬롯 간격(분)</label>
        <input
          id="rest-slot-interval"
          type="number"
          value={slotIntervalMinutes}
          onChange={(e) => setSlotIntervalMinutes(Number(e.target.value))}
        />
        <label htmlFor="rest-capacity">슬롯당 좌석수</label>
        <input
          id="rest-capacity"
          type="number"
          value={capacityPerSlot}
          onChange={(e) => setCapacityPerSlot(Number(e.target.value))}
        />
        {error && <p role="alert">{error}</p>}
        <button type="submit">등록하기</button>
      </form>
    );
  }

  const restaurant = restaurants[0];
  return (
    <div>
      <h1>{restaurant.name}</h1>
      <p>{restaurant.address}</p>
      <p>{restaurant.phone}</p>
      <p>{restaurant.hasParking ? '주차 가능' : '주차 불가'}</p>
    </div>
  );
}
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `pnpm --filter client test -- RestaurantSettingsPage.test.tsx`
Expected: PASS (2 passed)

- [ ] **Step 5: DashboardLayout 및 라우팅 추가**

`Client/src/dashboard/DashboardLayout.tsx`:
```tsx
import { NavLink, Outlet } from 'react-router-dom';

export function DashboardLayout() {
  return (
    <div>
      <nav>
        <NavLink to="/dashboard/settings">음식점 설정</NavLink>
        <NavLink to="/dashboard/reservations">예약 관리</NavLink>
      </nav>
      <Outlet />
    </div>
  );
}
```

`Client/src/App.tsx`:
```tsx
import { BrowserRouter, Navigate, Route, Routes } from 'react-router-dom';
import { AuthProvider } from './auth/AuthContext';
import { SignupPage } from './auth/SignupPage';
import { LoginPage } from './auth/LoginPage';
import { DashboardLayout } from './dashboard/DashboardLayout';
import { RestaurantSettingsPage } from './dashboard/RestaurantSettingsPage';

export default function App() {
  return (
    <AuthProvider>
      <BrowserRouter>
        <Routes>
          <Route path="/" element={<Navigate to="/login" replace />} />
          <Route path="/signup" element={<SignupPage />} />
          <Route path="/login" element={<LoginPage />} />
          <Route path="/dashboard" element={<DashboardLayout />}>
            <Route index element={<Navigate to="settings" replace />} />
            <Route path="settings" element={<RestaurantSettingsPage />} />
          </Route>
        </Routes>
      </BrowserRouter>
    </AuthProvider>
  );
}
```

- [ ] **Step 6: Commit**

```bash
git add Client/src/dashboard Client/src/App.tsx
git commit -m "feat: add dashboard layout and restaurant settings page"
```

---

## Task 10: Frontend — 대시보드: 예약 목록 + 수동 등록 + 상태 변경

**Files:**
- Create: `Client/src/dashboard/ReservationsPage.tsx`
- Create: `Client/src/dashboard/ReservationsPage.test.tsx`
- Modify: `Client/src/App.tsx`

**Interfaces:**
- Consumes: `apiClient`(Task 8), `DashboardLayout`(Task 9). 이 태스크는 사장님이 등록한 음식점 id를 `localStorage` 또는 `/restaurants/mine`로 조회한다고 가정한다(간단화를 위해 첫 번째 음식점만 사용).
- Produces: `/dashboard/reservations` 라우트.

- [ ] **Step 1: 실패 테스트 작성**

`Client/src/dashboard/ReservationsPage.test.tsx`:
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { describe, expect, it, vi, beforeEach } from 'vitest';
import { ReservationsPage } from './ReservationsPage';
import { apiClient } from '../api/client';

vi.mock('../api/client', () => ({
  apiClient: { get: vi.fn(), post: vi.fn(), patch: vi.fn() },
}));

describe('ReservationsPage', () => {
  beforeEach(() => {
    vi.mocked(apiClient.get).mockReset();
    vi.mocked(apiClient.patch).mockReset();
  });

  it('lists reservations for the restaurant and selected date', async () => {
    vi.mocked(apiClient.get).mockImplementation((path: string) => {
      if (path === '/restaurants/mine') {
        return Promise.resolve([{ id: 'r1', name: '내 식당' }]);
      }
      return Promise.resolve([
        { id: 'res1', customerName: '홍길동', partySize: 4, reservationTime: '11:00', status: 'CONFIRMED' },
      ]);
    });

    render(<ReservationsPage />);

    await waitFor(() => {
      expect(screen.getByText('홍길동')).toBeInTheDocument();
    });
  });

  it('changes a reservation status to COMPLETED', async () => {
    vi.mocked(apiClient.get).mockImplementation((path: string) => {
      if (path === '/restaurants/mine') {
        return Promise.resolve([{ id: 'r1', name: '내 식당' }]);
      }
      return Promise.resolve([
        { id: 'res1', customerName: '홍길동', partySize: 4, reservationTime: '11:00', status: 'CONFIRMED' },
      ]);
    });
    vi.mocked(apiClient.patch).mockResolvedValue({ id: 'res1', status: 'COMPLETED' });

    render(<ReservationsPage />);
    await waitFor(() => screen.getByText('홍길동'));

    fireEvent.click(screen.getByRole('button', { name: '완료 처리' }));

    await waitFor(() => {
      expect(apiClient.patch).toHaveBeenCalledWith('/restaurants/r1/reservations/res1/status', {
        status: 'COMPLETED',
      });
    });
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter client test -- ReservationsPage.test.tsx`
Expected: FAIL ("Cannot find module './ReservationsPage'")

- [ ] **Step 3: ReservationsPage 구현**

`Client/src/dashboard/ReservationsPage.tsx`:
```tsx
import { useEffect, useState } from 'react';
import { apiClient } from '../api/client';

interface Restaurant {
  id: string;
  name: string;
}

interface Reservation {
  id: string;
  customerName: string;
  partySize: number;
  reservationTime: string;
  status: string;
}

function todayIsoDate(): string {
  return new Date().toISOString().slice(0, 10);
}

export function ReservationsPage() {
  const [restaurant, setRestaurant] = useState<Restaurant | null>(null);
  const [date, setDate] = useState(todayIsoDate());
  const [reservations, setReservations] = useState<Reservation[]>([]);

  useEffect(() => {
    apiClient.get<Restaurant[]>('/restaurants/mine').then((list) => {
      if (list.length > 0) setRestaurant(list[0]);
    });
  }, []);

  useEffect(() => {
    if (!restaurant) return;
    apiClient.get<Reservation[]>(`/restaurants/${restaurant.id}/reservations?date=${date}`).then(setReservations);
  }, [restaurant, date]);

  async function updateStatus(resId: string, status: string) {
    if (!restaurant) return;
    await apiClient.patch(`/restaurants/${restaurant.id}/reservations/${resId}/status`, { status });
    setReservations((prev) => prev.map((r) => (r.id === resId ? { ...r, status } : r)));
  }

  if (!restaurant) {
    return <p>음식점을 먼저 등록해주세요.</p>;
  }

  return (
    <div>
      <h1>예약 관리</h1>
      <label htmlFor="reservation-date">날짜</label>
      <input id="reservation-date" type="date" value={date} onChange={(e) => setDate(e.target.value)} />
      <ul>
        {reservations.map((r) => (
          <li key={r.id}>
            <span>{r.reservationTime}</span>
            <span>{r.customerName}</span>
            <span>{r.partySize}명</span>
            <span>{r.status}</span>
            {r.status === 'CONFIRMED' && (
              <>
                <button onClick={() => updateStatus(r.id, 'COMPLETED')}>완료 처리</button>
                <button onClick={() => updateStatus(r.id, 'NO_SHOW')}>노쇼 처리</button>
                <button onClick={() => updateStatus(r.id, 'CANCELLED')}>취소 처리</button>
              </>
            )}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `pnpm --filter client test -- ReservationsPage.test.tsx`
Expected: PASS (2 passed)

- [ ] **Step 5: 라우팅 추가**

`Client/src/App.tsx`의 `Route path="/dashboard"` 블록 안에 추가:
```tsx
            <Route path="reservations" element={<ReservationsPage />} />
```
파일 상단 import에 추가:
```tsx
import { ReservationsPage } from './dashboard/ReservationsPage';
```

- [ ] **Step 6: Commit**

```bash
git add Client/src/dashboard/ReservationsPage.tsx Client/src/dashboard/ReservationsPage.test.tsx Client/src/App.tsx
git commit -m "feat: add dashboard reservations list with status updates"
```

---

## Task 11: Frontend — 고객용 검색 + 음식점 상세/예약 페이지

**Files:**
- Create: `Client/src/customer/SearchPage.tsx`
- Create: `Client/src/customer/SearchPage.test.tsx`
- Create: `Client/src/customer/RestaurantBookingPage.tsx`
- Create: `Client/src/customer/RestaurantBookingPage.test.tsx`
- Modify: `Client/src/App.tsx`

**Interfaces:**
- Consumes: `apiClient`(Task 8)
- Produces: `/search` 라우트(음식점 검색 목록), `/restaurants/:id` 라우트(공개 정보 + 날짜 선택 + 잔여 슬롯 표시 + 예약 폼).

- [ ] **Step 1: SearchPage 실패 테스트 작성**

`Client/src/customer/SearchPage.test.tsx`:
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { describe, expect, it, vi, beforeEach } from 'vitest';
import { SearchPage } from './SearchPage';
import { apiClient } from '../api/client';

vi.mock('../api/client', () => ({
  apiClient: { get: vi.fn() },
}));

describe('SearchPage', () => {
  beforeEach(() => {
    vi.mocked(apiClient.get).mockReset();
  });

  it('lists restaurants matching the search query', async () => {
    vi.mocked(apiClient.get).mockResolvedValue([
      { id: 'r1', name: '강남 파스타집', address: '서울시 강남구', phone: '02', hasParking: true },
    ]);

    render(
      <MemoryRouter>
        <SearchPage />
      </MemoryRouter>,
    );

    fireEvent.change(screen.getByLabelText('음식점 검색'), { target: { value: '파스타' } });
    fireEvent.click(screen.getByRole('button', { name: '검색' }));

    await waitFor(() => {
      expect(screen.getByText('강남 파스타집')).toBeInTheDocument();
    });
    expect(apiClient.get).toHaveBeenCalledWith('/restaurants/search?q=%ED%8C%8C%EC%8A%A4%ED%83%80');
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter client test -- SearchPage.test.tsx`
Expected: FAIL ("Cannot find module './SearchPage'")

- [ ] **Step 3: SearchPage 구현**

`Client/src/customer/SearchPage.tsx`:
```tsx
import { FormEvent, useState } from 'react';
import { Link } from 'react-router-dom';
import { apiClient } from '../api/client';

interface RestaurantSummary {
  id: string;
  name: string;
  address: string;
  phone: string;
  hasParking: boolean;
}

export function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<RestaurantSummary[]>([]);

  async function handleSearch(e: FormEvent) {
    e.preventDefault();
    const results = await apiClient.get<RestaurantSummary[]>(
      `/restaurants/search?q=${encodeURIComponent(query)}`,
    );
    setResults(results);
  }

  return (
    <div>
      <h1>음식점 찾기</h1>
      <form onSubmit={handleSearch}>
        <label htmlFor="search-query">음식점 검색</label>
        <input id="search-query" value={query} onChange={(e) => setQuery(e.target.value)} />
        <button type="submit">검색</button>
      </form>
      <ul>
        {results.map((r) => (
          <li key={r.id}>
            <Link to={`/restaurants/${r.id}`}>{r.name}</Link>
            <span>{r.address}</span>
            <span>{r.hasParking ? '주차 가능' : '주차 불가'}</span>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `pnpm --filter client test -- SearchPage.test.tsx`
Expected: PASS (1 passed)

- [ ] **Step 5: RestaurantBookingPage 실패 테스트 작성**

`Client/src/customer/RestaurantBookingPage.test.tsx`:
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { MemoryRouter, Route, Routes } from 'react-router-dom';
import { describe, expect, it, vi, beforeEach } from 'vitest';
import { RestaurantBookingPage } from './RestaurantBookingPage';
import { apiClient } from '../api/client';

vi.mock('../api/client', () => ({
  apiClient: { get: vi.fn(), post: vi.fn() },
}));

function renderAt(id: string) {
  return render(
    <MemoryRouter initialEntries={[`/restaurants/${id}`]}>
      <Routes>
        <Route path="/restaurants/:id" element={<RestaurantBookingPage />} />
      </Routes>
    </MemoryRouter>,
  );
}

describe('RestaurantBookingPage', () => {
  beforeEach(() => {
    vi.mocked(apiClient.get).mockReset();
    vi.mocked(apiClient.post).mockReset();
  });

  it('shows public info and available slots for the selected date', async () => {
    vi.mocked(apiClient.get).mockImplementation((path: string) => {
      if (path.includes('/public')) {
        return Promise.resolve({ id: 'r1', name: '내 식당', address: 'A', phone: '02', hasParking: true });
      }
      return Promise.resolve([{ time: '11:00', remaining: 4 }]);
    });

    renderAt('r1');

    await waitFor(() => {
      expect(screen.getByText('내 식당')).toBeInTheDocument();
      expect(screen.getByText(/11:00/)).toBeInTheDocument();
    });
  });

  it('submits a booking with the selected slot', async () => {
    vi.mocked(apiClient.get).mockImplementation((path: string) => {
      if (path.includes('/public')) {
        return Promise.resolve({ id: 'r1', name: '내 식당', address: 'A', phone: '02', hasParking: true });
      }
      return Promise.resolve([{ time: '11:00', remaining: 4 }]);
    });
    vi.mocked(apiClient.post).mockResolvedValue({ reservationCode: 'ABCD1234' });

    renderAt('r1');
    await waitFor(() => screen.getByText(/11:00/));

    fireEvent.change(screen.getByLabelText('이름'), { target: { value: '홍길동' } });
    fireEvent.change(screen.getByLabelText('전화번호'), { target: { value: '010-1111-2222' } });
    fireEvent.change(screen.getByLabelText('인원'), { target: { value: '2' } });
    fireEvent.click(screen.getByRole('radio', { name: /11:00/ }));
    fireEvent.click(screen.getByRole('button', { name: '예약하기' }));

    await waitFor(() => {
      expect(screen.getByText(/ABCD1234/)).toBeInTheDocument();
    });
  });
});
```

- [ ] **Step 6: 테스트 실행 → 실패 확인**

Run: `pnpm --filter client test -- RestaurantBookingPage.test.tsx`
Expected: FAIL ("Cannot find module './RestaurantBookingPage'")

- [ ] **Step 7: RestaurantBookingPage 구현**

`Client/src/customer/RestaurantBookingPage.tsx`:
```tsx
import { FormEvent, useEffect, useState } from 'react';
import { useParams } from 'react-router-dom';
import { apiClient } from '../api/client';

interface PublicRestaurant {
  id: string;
  name: string;
  address: string;
  phone: string;
  hasParking: boolean;
}

interface Slot {
  time: string;
  remaining: number;
}

function todayIsoDate(): string {
  return new Date().toISOString().slice(0, 10);
}

export function RestaurantBookingPage() {
  const { id } = useParams<{ id: string }>();
  const [restaurant, setRestaurant] = useState<PublicRestaurant | null>(null);
  const [date, setDate] = useState(todayIsoDate());
  const [slots, setSlots] = useState<Slot[]>([]);
  const [selectedTime, setSelectedTime] = useState<string | null>(null);
  const [name, setName] = useState('');
  const [phone, setPhone] = useState('');
  const [partySize, setPartySize] = useState(2);
  const [reservationCode, setReservationCode] = useState<string | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (!id) return;
    apiClient.get<PublicRestaurant>(`/restaurants/${id}/public`).then(setRestaurant);
  }, [id]);

  useEffect(() => {
    if (!id) return;
    apiClient.get<Slot[]>(`/restaurants/${id}/availability?date=${date}`).then(setSlots);
  }, [id, date]);

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    if (!id || !selectedTime) return;
    setError(null);
    try {
      const result = await apiClient.post<{ reservationCode: string }>(`/restaurants/${id}/reservations`, {
        customerName: name,
        customerPhone: phone,
        partySize,
        reservationDate: date,
        reservationTime: selectedTime,
      });
      setReservationCode(result.reservationCode);
    } catch (err) {
      setError(err instanceof Error ? err.message : '예약에 실패했습니다');
    }
  }

  if (!restaurant) {
    return <p>불러오는 중...</p>;
  }

  if (reservationCode) {
    return (
      <div>
        <h1>예약이 완료되었습니다</h1>
        <p>예약 코드: {reservationCode}</p>
      </div>
    );
  }

  return (
    <div>
      <h1>{restaurant.name}</h1>
      <p>{restaurant.address}</p>
      <p>{restaurant.phone}</p>
      <p>{restaurant.hasParking ? '주차 가능' : '주차 불가'}</p>

      <label htmlFor="booking-date">날짜</label>
      <input id="booking-date" type="date" value={date} onChange={(e) => setDate(e.target.value)} />

      <fieldset>
        <legend>시간 선택</legend>
        {slots.map((slot) => (
          <label key={slot.time}>
            <input
              type="radio"
              name="time-slot"
              value={slot.time}
              checked={selectedTime === slot.time}
              onChange={() => setSelectedTime(slot.time)}
              disabled={slot.remaining <= 0}
            />
            {slot.time} (잔여 {slot.remaining}석)
          </label>
        ))}
      </fieldset>

      <form onSubmit={handleSubmit}>
        <label htmlFor="booking-name">이름</label>
        <input id="booking-name" value={name} onChange={(e) => setName(e.target.value)} />
        <label htmlFor="booking-phone">전화번호</label>
        <input id="booking-phone" value={phone} onChange={(e) => setPhone(e.target.value)} />
        <label htmlFor="booking-party-size">인원</label>
        <input
          id="booking-party-size"
          type="number"
          value={partySize}
          onChange={(e) => setPartySize(Number(e.target.value))}
        />
        {error && <p role="alert">{error}</p>}
        <button type="submit" disabled={!selectedTime}>
          예약하기
        </button>
      </form>
    </div>
  );
}
```

- [ ] **Step 8: 테스트 실행 → 통과 확인**

Run: `pnpm --filter client test -- RestaurantBookingPage.test.tsx`
Expected: PASS (2 passed)

- [ ] **Step 9: 라우팅 추가**

`Client/src/App.tsx`에 다음 import 및 Route 추가:
```tsx
import { SearchPage } from './customer/SearchPage';
import { RestaurantBookingPage } from './customer/RestaurantBookingPage';
```
```tsx
          <Route path="/search" element={<SearchPage />} />
          <Route path="/restaurants/:id" element={<RestaurantBookingPage />} />
```

- [ ] **Step 10: Commit**

```bash
git add Client/src/customer Client/src/App.tsx
git commit -m "feat: add customer search and restaurant booking pages"
```

---

## Task 12: Frontend — 고객용 예약 조회/취소 페이지

**Files:**
- Create: `Client/src/customer/LookupPage.tsx`
- Create: `Client/src/customer/LookupPage.test.tsx`
- Modify: `Client/src/App.tsx`

**Interfaces:**
- Consumes: `apiClient`(Task 8)
- Produces: `/lookup` 라우트 — 코드+전화번호로 예약 조회 후 취소 버튼 제공.

- [ ] **Step 1: 실패 테스트 작성**

`Client/src/customer/LookupPage.test.tsx`:
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { describe, expect, it, vi, beforeEach } from 'vitest';
import { LookupPage } from './LookupPage';
import { apiClient } from '../api/client';

vi.mock('../api/client', () => ({
  apiClient: { get: vi.fn(), patch: vi.fn() },
}));

describe('LookupPage', () => {
  beforeEach(() => {
    vi.mocked(apiClient.get).mockReset();
    vi.mocked(apiClient.patch).mockReset();
  });

  it('looks up a reservation by code and phone', async () => {
    vi.mocked(apiClient.get).mockResolvedValue({
      id: 'res1',
      customerName: '홍길동',
      partySize: 2,
      reservationDate: '2026-07-06',
      reservationTime: '11:00',
      status: 'CONFIRMED',
    });

    render(<LookupPage />);

    fireEvent.change(screen.getByLabelText('예약 코드'), { target: { value: 'ABCD1234' } });
    fireEvent.change(screen.getByLabelText('전화번호'), { target: { value: '010-1111-2222' } });
    fireEvent.click(screen.getByRole('button', { name: '조회하기' }));

    await waitFor(() => {
      expect(screen.getByText('홍길동')).toBeInTheDocument();
    });
    expect(apiClient.get).toHaveBeenCalledWith('/reservations/lookup?code=ABCD1234&phone=010-1111-2222');
  });

  it('cancels the looked-up reservation', async () => {
    vi.mocked(apiClient.get).mockResolvedValue({
      id: 'res1',
      customerName: '홍길동',
      partySize: 2,
      reservationDate: '2026-07-06',
      reservationTime: '11:00',
      status: 'CONFIRMED',
    });
    vi.mocked(apiClient.patch).mockResolvedValue({ id: 'res1', status: 'CANCELLED' });

    render(<LookupPage />);
    fireEvent.change(screen.getByLabelText('예약 코드'), { target: { value: 'ABCD1234' } });
    fireEvent.change(screen.getByLabelText('전화번호'), { target: { value: '010-1111-2222' } });
    fireEvent.click(screen.getByRole('button', { name: '조회하기' }));
    await waitFor(() => screen.getByText('홍길동'));

    fireEvent.click(screen.getByRole('button', { name: '예약 취소' }));

    await waitFor(() => {
      expect(apiClient.patch).toHaveBeenCalledWith('/reservations/res1/cancel', {
        code: 'ABCD1234',
        phone: '010-1111-2222',
      });
    });
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter client test -- LookupPage.test.tsx`
Expected: FAIL ("Cannot find module './LookupPage'")

- [ ] **Step 3: LookupPage 구현**

`Client/src/customer/LookupPage.tsx`:
```tsx
import { FormEvent, useState } from 'react';
import { apiClient } from '../api/client';

interface ReservationDetail {
  id: string;
  customerName: string;
  partySize: number;
  reservationDate: string;
  reservationTime: string;
  status: string;
}

export function LookupPage() {
  const [code, setCode] = useState('');
  const [phone, setPhone] = useState('');
  const [reservation, setReservation] = useState<ReservationDetail | null>(null);
  const [error, setError] = useState<string | null>(null);

  async function handleLookup(e: FormEvent) {
    e.preventDefault();
    setError(null);
    try {
      const result = await apiClient.get<ReservationDetail>(
        `/reservations/lookup?code=${code}&phone=${phone}`,
      );
      setReservation(result);
    } catch (err) {
      setError(err instanceof Error ? err.message : '예약을 찾을 수 없습니다');
    }
  }

  async function handleCancel() {
    if (!reservation) return;
    const updated = await apiClient.patch<ReservationDetail>(`/reservations/${reservation.id}/cancel`, {
      code,
      phone,
    });
    setReservation(updated);
  }

  return (
    <div>
      <h1>예약 조회</h1>
      <form onSubmit={handleLookup}>
        <label htmlFor="lookup-code">예약 코드</label>
        <input id="lookup-code" value={code} onChange={(e) => setCode(e.target.value)} />
        <label htmlFor="lookup-phone">전화번호</label>
        <input id="lookup-phone" value={phone} onChange={(e) => setPhone(e.target.value)} />
        {error && <p role="alert">{error}</p>}
        <button type="submit">조회하기</button>
      </form>

      {reservation && (
        <div>
          <p>{reservation.customerName}</p>
          <p>
            {reservation.reservationDate} {reservation.reservationTime}
          </p>
          <p>{reservation.partySize}명</p>
          <p>상태: {reservation.status}</p>
          {reservation.status === 'CONFIRMED' && <button onClick={handleCancel}>예약 취소</button>}
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `pnpm --filter client test -- LookupPage.test.tsx`
Expected: PASS (2 passed)

- [ ] **Step 5: 라우팅 추가**

`Client/src/App.tsx`에 다음 import 및 Route 추가:
```tsx
import { LookupPage } from './customer/LookupPage';
```
```tsx
          <Route path="/lookup" element={<LookupPage />} />
```

- [ ] **Step 6: Commit**

```bash
git add Client/src/customer/LookupPage.tsx Client/src/customer/LookupPage.test.tsx Client/src/App.tsx
git commit -m "feat: add customer reservation lookup and cancel page"
```

---

## Task 13: Nginx 참고 설정 (배포 산출물)

**Files:**
- Create: `deploy/nginx.conf`

**Interfaces:**
- Consumes: 없음
- Produces: 참고용 Nginx 설정 파일 (실제 배포는 범위 밖, 프로덕션 빌드 산출물을 서빙하고 `/api`를 백엔드로 프록시하는 예시).

- [ ] **Step 1: Nginx 설정 파일 작성**

`deploy/nginx.conf`:
```nginx
server {
    listen 80;
    server_name _;

    root /var/www/restaurant-web/dist;
    index index.html;

    location /api/ {
        proxy_pass http://127.0.0.1:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location / {
        try_files $uri /index.html;
    }
}
```

- [ ] **Step 2: Commit**

```bash
git add deploy/nginx.conf
git commit -m "chore: add reference nginx config for production deployment"
```

---

## 완료 후 확인

- [ ] **Step 1: 전체 백엔드 테스트 실행**

Run: `pnpm test:api`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 2: 전체 프론트엔드 테스트 실행**

Run: `pnpm test:web`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 3: 수동 스모크 테스트**

Run: `pnpm db:up && pnpm dev:api` (터미널 1), `pnpm dev:web` (터미널 2)
Expected: `http://localhost:5173/signup`에서 가입 → `/login`에서 로그인 → `/dashboard/settings`에서 음식점 등록(주소/전화번호/주차여부 포함) → `/dashboard/reservations`에서 예약 관리 → `/search`에서 검색 → `/restaurants/:id`에서 예약 → `/lookup`에서 조회/취소까지 전체 플로우 동작 확인
