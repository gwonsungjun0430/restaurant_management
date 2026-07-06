# 음식점 예약 관리 시스템 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 여러 음식점(다중 테넌트)을 대상으로, 사장님/직원이 인증 후 예약을 관리하고 고객이 비회원으로 예약할 수 있는 웹앱을 구축한다.

**Architecture:** pnpm workspace 모노레포. `apps/api`는 Nest.js + Prisma + SQLite로 REST API를 제공하고, `apps/web`은 React + Vite SPA로 관리 대시보드와 고객용 예약 페이지를 제공한다. 프로덕션 배포 시 Nginx가 정적 파일 서빙과 `/api` 리버스 프록시를 담당한다 (개발 중엔 Vite dev server 사용).

**Tech Stack:** TypeScript(strict) 전역, Nest.js 10.x, Prisma + SQLite, Passport-JWT, bcryptjs, class-validator, Jest + Supertest(백엔드 테스트), React 18 + Vite, react-router-dom, Vitest + Testing Library(프론트엔드 테스트), Nginx.

## Global Constraints

- 모든 코드는 TypeScript strict 모드로 작성한다 (`"strict": true` in tsconfig).
- 패키지 매니저는 pnpm을 사용한다 (workspace 기능 필요).
- DB는 SQLite 파일(`apps/api/prisma/dev.db`)만 사용한다 (PostgreSQL/MySQL 사용 금지).
- 이메일/SMS 발송, 결제, 테이블 배치(플로어 플랜) 기능은 이번 계획의 범위 밖이다 — 구현하지 않는다.
- 비밀번호 해싱은 `bcryptjs`(순수 JS, 네이티브 빌드 불필요)를 사용한다.
- 고객 예약 조회/취소는 로그인 없이 `reservationCode` + `customerPhone` 조합으로만 인증한다.
- 각 태스크의 커밋 메시지는 Conventional Commits 형식(`feat:`, `test:`, `fix:` 등)을 따른다.

---

## Task 1: 모노레포 & 앱 스캐폴딩

**Files:**
- Create: `package.json` (workspace root)
- Create: `pnpm-workspace.yaml`
- Create: `.gitignore`
- Create: `apps/api/package.json`
- Create: `apps/api/tsconfig.json`
- Create: `apps/api/tsconfig.build.json`
- Create: `apps/api/nest-cli.json`
- Create: `apps/api/src/main.ts`
- Create: `apps/api/src/app.module.ts`
- Create: `apps/web/package.json`
- Create: `apps/web/tsconfig.json`
- Create: `apps/web/tsconfig.node.json`
- Create: `apps/web/vite.config.ts`
- Create: `apps/web/index.html`
- Create: `apps/web/src/main.tsx`
- Create: `apps/web/src/App.tsx`

**Interfaces:**
- Produces: `apps/api` — Nest.js 앱 진입점(`main.ts`)이 포트 3000에서 기동. `AppModule`은 이후 태스크에서 다른 모듈을 import할 루트 모듈.
- Produces: `apps/web` — Vite dev server가 포트 5173에서 기동. `App` 컴포넌트는 이후 태스크에서 라우팅을 추가할 루트 컴포넌트.

- [ ] **Step 1: 루트 workspace 설정 파일 작성**

`package.json`:
```json
{
  "name": "food-management",
  "private": true,
  "scripts": {
    "dev:api": "pnpm --filter api start:dev",
    "dev:web": "pnpm --filter web dev",
    "test:api": "pnpm --filter api test",
    "test:web": "pnpm --filter web test"
  }
}
```

`pnpm-workspace.yaml`:
```yaml
packages:
  - "apps/*"
```

`.gitignore`:
```
node_modules
dist
apps/api/prisma/dev.db
apps/api/prisma/*.db-journal
apps/web/dist
.env
```

- [ ] **Step 2: Nest.js API 앱 스캐폴딩**

`apps/api/package.json`:
```json
{
  "name": "api",
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

`apps/api/tsconfig.json`:
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

`apps/api/tsconfig.build.json`:
```json
{
  "extends": "./tsconfig.json",
  "exclude": ["node_modules", "test", "dist", "**/*spec.ts"]
}
```

`apps/api/nest-cli.json`:
```json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "sourceRoot": "src"
}
```

`apps/api/src/app.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true })],
})
export class AppModule {}
```

`apps/api/src/main.ts`:
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

`apps/web/package.json`:
```json
{
  "name": "web",
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

`apps/web/tsconfig.json`:
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

`apps/web/tsconfig.node.json`:
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

`apps/web/vite.config.ts`:
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

`apps/web/index.html`:
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

`apps/web/src/App.tsx`:
```tsx
export default function App() {
  return <div>음식점 예약 관리</div>;
}
```

`apps/web/src/main.tsx`:
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

- [ ] **Step 4: 의존성 설치 및 두 앱이 기동되는지 확인**

Run: `pnpm install`
Expected: 두 워크스페이스(`api`, `web`)의 의존성이 설치됨 (에러 없이 종료)

Run: `pnpm dev:api`
Expected: 콘솔에 Nest 부트스트랩 로그 출력, 프로세스가 종료되지 않고 대기. 확인 후 Ctrl+C로 종료.

Run: `pnpm dev:web`
Expected: `VITE ready` 로그와 함께 `http://localhost:5173/` 출력. 확인 후 Ctrl+C로 종료.

- [ ] **Step 5: Commit**

```bash
git add package.json pnpm-workspace.yaml .gitignore apps/api apps/web
git commit -m "chore: scaffold pnpm workspace with Nest.js api and React web apps"
```

---

## Task 2: Prisma 스키마 + PrismaService

**Files:**
- Create: `apps/api/prisma/schema.prisma`
- Create: `apps/api/src/prisma/prisma.service.ts`
- Create: `apps/api/src/prisma/prisma.module.ts`
- Create: `apps/api/src/prisma/prisma.service.spec.ts`
- Modify: `apps/api/src/app.module.ts`
- Create: `apps/api/.env`

**Interfaces:**
- Consumes: 없음 (Task 1의 앱 스캐폴딩만 사용)
- Produces: `PrismaModule` (전역 모듈), `PrismaService` — `PrismaClient`를 상속하며 `onModuleInit`에서 연결. 이후 모든 백엔드 태스크에서 `PrismaService`를 주입받아 사용.

- [ ] **Step 1: Prisma 스키마 작성**

`apps/api/prisma/schema.prisma`:
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
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
  id           String   @id @default(uuid())
  restaurantId String
  userId       String
  role         String
  createdAt    DateTime @default(now())

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
  id               String   @id @default(uuid())
  restaurantId     String
  reservationCode  String   @unique
  customerName     String
  customerPhone    String
  partySize        Int
  reservationDate  String
  reservationTime  String
  status           String   @default("CONFIRMED")
  createdByStaffId String?
  createdAt        DateTime @default(now())

  restaurant     Restaurant @relation(fields: [restaurantId], references: [id])
  createdByStaff User?      @relation("ReservationCreatedBy", fields: [createdByStaffId], references: [id])
}
```

> `role`과 `status`는 SQLite의 enum 제약 이슈를 피하기 위해 `String`으로 저장하고, 애플리케이션 코드(TypeScript union 타입 + class-validator `@IsIn`)에서 값을 제한한다. 허용 값: `role` ∈ `{OWNER, STAFF}`, `status` ∈ `{CONFIRMED, CANCELLED, COMPLETED, NO_SHOW}`.

- [ ] **Step 2: 환경 변수 파일 작성**

`apps/api/.env`:
```
DATABASE_URL="file:./dev.db"
JWT_SECRET="dev-secret-change-in-production"
PORT=3000
```

- [ ] **Step 3: 마이그레이션 생성 및 Prisma client 생성**

Run: `pnpm --filter api exec prisma migrate dev --name init`
Expected: `apps/api/prisma/migrations/<timestamp>_init/migration.sql` 생성, `apps/api/prisma/dev.db` 파일 생성, "Your database is now in sync with your schema." 출력

- [ ] **Step 4: PrismaService 작성**

`apps/api/src/prisma/prisma.service.ts`:
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

`apps/api/src/prisma/prisma.module.ts`:
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

`apps/api/src/prisma/prisma.service.spec.ts`:
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

Run: `pnpm --filter api test -- prisma.service.spec.ts`
Expected: PASS (1 passed)

- [ ] **Step 7: AppModule에 PrismaModule 등록**

`apps/api/src/app.module.ts`:
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
git add apps/api/prisma apps/api/src/prisma apps/api/src/app.module.ts apps/api/.env apps/api/.gitignore
git commit -m "feat: add Prisma schema and PrismaService with SQLite datasource"
```

---

## Task 3: Auth — 회원가입/로그인/JWT/비밀번호 변경

**Files:**
- Create: `apps/api/src/auth/dto/signup.dto.ts`
- Create: `apps/api/src/auth/dto/login.dto.ts`
- Create: `apps/api/src/auth/dto/change-password.dto.ts`
- Create: `apps/api/src/auth/auth.service.ts`
- Create: `apps/api/src/auth/auth.service.spec.ts`
- Create: `apps/api/src/auth/auth.controller.ts`
- Create: `apps/api/src/auth/jwt.strategy.ts`
- Create: `apps/api/src/auth/jwt-auth.guard.ts`
- Create: `apps/api/src/auth/current-user.decorator.ts`
- Create: `apps/api/src/auth/auth.module.ts`
- Modify: `apps/api/src/app.module.ts`

**Interfaces:**
- Consumes: `PrismaService` (Task 2) — `prisma.user.create/findUnique/update`
- Produces: `AuthService.signup(dto): Promise<{ id: string; email: string; name: string }>`, `AuthService.login(dto): Promise<{ accessToken: string }>`, `AuthService.changePassword(userId, dto): Promise<void>`. `JwtAuthGuard` — 이후 모든 인증 필요 라우트에서 사용. `CurrentUser()` 데코레이터 — 컨트롤러에서 `{ userId: string; email: string }`을 주입받는 데 사용.

- [ ] **Step 1: DTO 작성**

`apps/api/src/auth/dto/signup.dto.ts`:
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

`apps/api/src/auth/dto/login.dto.ts`:
```typescript
import { IsEmail, IsString } from 'class-validator';

export class LoginDto {
  @IsEmail()
  email: string;

  @IsString()
  password: string;
}
```

`apps/api/src/auth/dto/change-password.dto.ts`:
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

`apps/api/src/auth/auth.service.spec.ts`:
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

Run: `pnpm --filter api test -- auth.service.spec.ts`
Expected: FAIL ("Cannot find module './auth.service'")

- [ ] **Step 4: AuthService 구현**

`apps/api/src/auth/auth.service.ts`:
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

Run: `pnpm --filter api test -- auth.service.spec.ts`
Expected: PASS (5 passed)

- [ ] **Step 6: JWT 전략, 가드, 데코레이터 작성**

`apps/api/src/auth/jwt.strategy.ts`:
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

`apps/api/src/auth/jwt-auth.guard.ts`:
```typescript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

`apps/api/src/auth/current-user.decorator.ts`:
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

`apps/api/src/auth/auth.controller.ts`:
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

`apps/api/src/auth/auth.module.ts`:
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

`apps/api/src/app.module.ts`:
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

Run: `pnpm --filter api test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 10: Commit**

```bash
git add apps/api/src/auth apps/api/src/app.module.ts
git commit -m "feat: add signup/login/change-password auth flow with JWT"
```

---

## Task 4: Restaurants — 음식점 생성 + 멤버십 가드

**Files:**
- Create: `apps/api/src/restaurants/dto/create-restaurant.dto.ts`
- Create: `apps/api/src/restaurants/roles.decorator.ts`
- Create: `apps/api/src/restaurants/restaurant-member.guard.ts`
- Create: `apps/api/src/restaurants/restaurant-member.guard.spec.ts`
- Create: `apps/api/src/restaurants/restaurants.service.ts`
- Create: `apps/api/src/restaurants/restaurants.service.spec.ts`
- Create: `apps/api/src/restaurants/restaurants.controller.ts`
- Create: `apps/api/src/restaurants/restaurants.module.ts`
- Modify: `apps/api/src/app.module.ts`

**Interfaces:**
- Consumes: `PrismaService` (Task 2), `JwtAuthGuard`, `CurrentUser`/`CurrentUserPayload` (Task 3)
- Produces: `RestaurantsService.create(ownerId, dto): Promise<{id, name, slotIntervalMinutes, capacityPerSlot}>`, `RestaurantsService.findMine(userId): Promise<Restaurant[]>`, `RestaurantMemberGuard` — route param `id`를 restaurantId로 보고 요청자의 멤버십을 확인해 `request.membershipRole: 'OWNER'|'STAFF'`를 세팅. `@Roles('OWNER')` 데코레이터 + 이 가드가 role 검증까지 함께 수행. 이후 Task 5, 6, 7, 8, 9에서 이 가드와 서비스의 `assertMember`/`assertOwner` 헬퍼를 재사용.

- [ ] **Step 1: DTO 및 Roles 데코레이터 작성**

`apps/api/src/restaurants/dto/create-restaurant.dto.ts`:
```typescript
import { IsInt, IsString, Min, MinLength } from 'class-validator';

export class CreateRestaurantDto {
  @IsString()
  @MinLength(1)
  name: string;

  @IsInt()
  @Min(5)
  slotIntervalMinutes: number;

  @IsInt()
  @Min(1)
  capacityPerSlot: number;
}
```

`apps/api/src/restaurants/roles.decorator.ts`:
```typescript
import { SetMetadata } from '@nestjs/common';

export type MemberRole = 'OWNER' | 'STAFF';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: MemberRole[]) => SetMetadata(ROLES_KEY, roles);
```

- [ ] **Step 2: RestaurantMemberGuard 실패 테스트 작성**

`apps/api/src/restaurants/restaurant-member.guard.spec.ts`:
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
      data: { name: 'R', ownerId: owner.id, slotIntervalMinutes: 30, capacityPerSlot: 10 },
    });
    await prisma.restaurantMember.create({ data: { restaurantId: restaurant.id, userId: owner.id, role: 'OWNER' } });

    const ctx = mockContext({ id: restaurant.id }, { userId: outsider.id });
    await expect(guard.canActivate(ctx)).rejects.toThrow(ForbiddenException);
  });

  it('allows access and attaches membershipRole for a member', async () => {
    const owner = await prisma.user.create({ data: { email: 'o2@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await prisma.restaurant.create({
      data: { name: 'R2', ownerId: owner.id, slotIntervalMinutes: 30, capacityPerSlot: 10 },
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

Run: `pnpm --filter api test -- restaurant-member.guard.spec.ts`
Expected: FAIL ("Cannot find module './restaurant-member.guard'")

- [ ] **Step 4: RestaurantMemberGuard 구현**

`apps/api/src/restaurants/restaurant-member.guard.ts`:
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

Run: `pnpm --filter api test -- restaurant-member.guard.spec.ts`
Expected: PASS (2 passed)

- [ ] **Step 6: RestaurantsService 실패 테스트 작성**

`apps/api/src/restaurants/restaurants.service.spec.ts`:
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

  it('creates a restaurant and makes the creator an OWNER member', async () => {
    const owner = await prisma.user.create({ data: { email: 'o@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await service.create(owner.id, { name: 'My Place', slotIntervalMinutes: 30, capacityPerSlot: 20 });

    expect(restaurant.name).toBe('My Place');
    const membership = await prisma.restaurantMember.findUniqueOrThrow({
      where: { restaurantId_userId: { restaurantId: restaurant.id, userId: owner.id } },
    });
    expect(membership.role).toBe('OWNER');
  });

  it('returns only restaurants the user is a member of', async () => {
    const owner = await prisma.user.create({ data: { email: 'o2@t.com', passwordHash: 'x', name: 'O' } });
    const stranger = await prisma.user.create({ data: { email: 's@t.com', passwordHash: 'x', name: 'S' } });
    await service.create(owner.id, { name: 'Mine', slotIntervalMinutes: 30, capacityPerSlot: 10 });

    const mine = await service.findMine(owner.id);
    const strangers = await service.findMine(stranger.id);

    expect(mine).toHaveLength(1);
    expect(strangers).toHaveLength(0);
  });
});
```

- [ ] **Step 7: 테스트 실행 → 실패 확인**

Run: `pnpm --filter api test -- restaurants.service.spec.ts`
Expected: FAIL ("Cannot find module './restaurants.service'")

- [ ] **Step 8: RestaurantsService 구현 (create, findMine)**

`apps/api/src/restaurants/restaurants.service.ts`:
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

Run: `pnpm --filter api test -- restaurants.service.spec.ts`
Expected: PASS (2 passed)

- [ ] **Step 10: RestaurantsController 및 RestaurantsModule 작성**

`apps/api/src/restaurants/restaurants.controller.ts`:
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

`apps/api/src/restaurants/restaurants.module.ts`:
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

`apps/api/src/app.module.ts`:
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

Run: `pnpm --filter api test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 13: Commit**

```bash
git add apps/api/src/restaurants apps/api/src/app.module.ts
git commit -m "feat: add restaurant creation and membership guard"
```

---

## Task 5: Restaurants — 영업시간 CRUD + 직원 초대 + 공개 정보 조회

**Files:**
- Create: `apps/api/src/restaurants/dto/set-business-hours.dto.ts`
- Create: `apps/api/src/restaurants/dto/invite-staff.dto.ts`
- Modify: `apps/api/src/restaurants/restaurants.service.ts`
- Modify: `apps/api/src/restaurants/restaurants.service.spec.ts`
- Modify: `apps/api/src/restaurants/restaurants.controller.ts`

**Interfaces:**
- Consumes: `RestaurantsService`, `RestaurantMemberGuard`, `Roles` 데코레이터 (Task 4)
- Produces: `RestaurantsService.setBusinessHours(restaurantId, hours): Promise<BusinessHour[]>`, `RestaurantsService.inviteStaff(restaurantId, dto): Promise<{ email, tempPassword }>`, `RestaurantsService.getPublic(restaurantId): Promise<{ id, name, slotIntervalMinutes, businessHours }>`. Task 6(가용시간 계산)이 `getPublic`과 `businessHours` 데이터 구조를 재사용.

- [ ] **Step 1: DTO 작성**

`apps/api/src/restaurants/dto/set-business-hours.dto.ts`:
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

`apps/api/src/restaurants/dto/invite-staff.dto.ts`:
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

- [ ] **Step 2: 실패 테스트 추가 (`restaurants.service.spec.ts`에 추가)**

기존 파일 하단에 다음 `describe` 블록을 추가한다:

```typescript
describe('RestaurantsService - business hours, staff invite, public info', () => {
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
    const restaurant = await service.create(owner.id, { name: 'R', slotIntervalMinutes: 30, capacityPerSlot: 10 });

    await service.setBusinessHours(restaurant.id, [{ dayOfWeek: 1, openTime: '11:00', closeTime: '21:00' }]);
    const second = await service.setBusinessHours(restaurant.id, [
      { dayOfWeek: 1, openTime: '10:00', closeTime: '20:00' },
      { dayOfWeek: 2, openTime: '10:00', closeTime: '20:00' },
    ]);

    expect(second).toHaveLength(2);
  });

  it('invites a staff member and returns a temporary password', async () => {
    const owner = await prisma.user.create({ data: { email: 'o2@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await service.create(owner.id, { name: 'R', slotIntervalMinutes: 30, capacityPerSlot: 10 });

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
    const restaurant = await service.create(owner.id, { name: 'R', slotIntervalMinutes: 30, capacityPerSlot: 10 });
    await service.inviteStaff(restaurant.id, { email: 'dup@t.com', name: 'Staff' });

    await expect(service.inviteStaff(restaurant.id, { email: 'dup@t.com', name: 'Staff2' })).rejects.toThrow();
  });

  it('returns public info including business hours', async () => {
    const owner = await prisma.user.create({ data: { email: 'o4@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await service.create(owner.id, { name: 'Public Place', slotIntervalMinutes: 30, capacityPerSlot: 10 });
    await service.setBusinessHours(restaurant.id, [{ dayOfWeek: 1, openTime: '11:00', closeTime: '21:00' }]);

    const publicInfo = await service.getPublic(restaurant.id);

    expect(publicInfo.name).toBe('Public Place');
    expect(publicInfo.businessHours).toHaveLength(1);
  });
});
```

- [ ] **Step 3: 테스트 실행 → 실패 확인**

Run: `pnpm --filter api test -- restaurants.service.spec.ts`
Expected: FAIL (`service.setBusinessHours is not a function` 등)

- [ ] **Step 4: RestaurantsService에 메서드 추가**

`apps/api/src/restaurants/restaurants.service.ts`의 `RestaurantsService` 클래스 안, `findMine` 메서드 뒤에 아래 메서드들을 추가하고 파일 상단 import에 필요한 항목을 더한다:

```typescript
import { ConflictException, Injectable, NotFoundException } from '@nestjs/common';
import * as bcrypt from 'bcryptjs';
import { PrismaService } from '../prisma/prisma.service';
import { CreateRestaurantDto } from './dto/create-restaurant.dto';
import { SetBusinessHoursDto } from './dto/set-business-hours.dto';
import { InviteStaffDto } from './dto/invite-staff.dto';
```

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
      slotIntervalMinutes: restaurant.slotIntervalMinutes,
      businessHours: restaurant.businessHours,
    };
  }
```

- [ ] **Step 5: 테스트 실행 → 통과 확인**

Run: `pnpm --filter api test -- restaurants.service.spec.ts`
Expected: PASS (6 passed)

- [ ] **Step 6: 컨트롤러에 엔드포인트 추가**

`apps/api/src/restaurants/restaurants.controller.ts` 전체를 다음으로 교체한다:

```typescript
import { Body, Controller, Get, Param, Post, Put, UseGuards } from '@nestjs/common';
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

- [ ] **Step 7: 전체 테스트 재실행 확인**

Run: `pnpm --filter api test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/restaurants
git commit -m "feat: add business hours, staff invite, and public restaurant info endpoints"
```

---

## Task 6: Reservations — 가용시간(슬롯) 계산

**Files:**
- Create: `apps/api/src/reservations/slot.util.ts`
- Create: `apps/api/src/reservations/slot.util.spec.ts`
- Create: `apps/api/src/reservations/reservations.service.ts`
- Create: `apps/api/src/reservations/reservations.service.spec.ts`
- Create: `apps/api/src/reservations/reservations.controller.ts`
- Create: `apps/api/src/reservations/reservations.module.ts`
- Modify: `apps/api/src/app.module.ts`

**Interfaces:**
- Consumes: `PrismaService` (Task 2), `RestaurantsModule`에서 export된 `RestaurantMemberGuard` (Task 4)
- Produces: `generateSlots(openTime, closeTime, intervalMinutes): string[]`, `ReservationsService.getAvailability(restaurantId, date): Promise<{ time: string; remaining: number }[]>`. Task 7, 9가 이 서비스와 `generateSlots`를 재사용.

- [ ] **Step 1: 슬롯 생성 순수 함수 실패 테스트 작성**

`apps/api/src/reservations/slot.util.spec.ts`:
```typescript
import { generateSlots } from './slot.util';

describe('generateSlots', () => {
  it('generates time slots at the given interval within business hours', () => {
    expect(generateSlots('11:00', '13:00', 30)).toEqual(['11:00', '11:30', '12:00', '12:30']);
  });

  it('excludes the closing time itself', () => {
    expect(generateSlots('11:00', '12:00', 60)).toEqual(['11:00']);
  });

  it('returns an empty array when open equals close', () => {
    expect(generateSlots('11:00', '11:00', 30)).toEqual([]);
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter api test -- slot.util.spec.ts`
Expected: FAIL ("Cannot find module './slot.util'")

- [ ] **Step 3: generateSlots 구현**

`apps/api/src/reservations/slot.util.ts`:
```typescript
function toMinutes(time: string): number {
  const [h, m] = time.split(':').map(Number);
  return h * 60 + m;
}

function toTimeString(minutes: number): string {
  const h = Math.floor(minutes / 60).toString().padStart(2, '0');
  const m = (minutes % 60).toString().padStart(2, '0');
  return `${h}:${m}`;
}

export function generateSlots(openTime: string, closeTime: string, intervalMinutes: number): string[] {
  const open = toMinutes(openTime);
  const close = toMinutes(closeTime);
  const slots: string[] = [];
  for (let t = open; t < close; t += intervalMinutes) {
    slots.push(toTimeString(t));
  }
  return slots;
}

export function generateReservationCode(): string {
  const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789';
  let code = '';
  for (let i = 0; i < 6; i++) {
    code += chars[Math.floor(Math.random() * chars.length)];
  }
  return code;
}

export function dayOfWeekFromDate(date: string): number {
  return new Date(`${date}T00:00:00`).getDay();
}
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `pnpm --filter api test -- slot.util.spec.ts`
Expected: PASS (3 passed)

- [ ] **Step 5: ReservationsService 실패 테스트 작성 (가용시간)**

`apps/api/src/reservations/reservations.service.spec.ts`:
```typescript
import { Test } from '@nestjs/testing';
import { ReservationsService } from './reservations.service';
import { RestaurantsService } from '../restaurants/restaurants.service';
import { PrismaService } from '../prisma/prisma.service';

describe('ReservationsService - availability', () => {
  let reservations: ReservationsService;
  let restaurants: RestaurantsService;
  let prisma: PrismaService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      providers: [ReservationsService, RestaurantsService, PrismaService],
    }).compile();
    reservations = moduleRef.get(ReservationsService);
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

  it('returns full capacity for every slot when there are no reservations yet', async () => {
    const owner = await prisma.user.create({ data: { email: 'o@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await restaurants.create(owner.id, { name: 'R', slotIntervalMinutes: 60, capacityPerSlot: 10 });
    const monday = 1;
    await restaurants.setBusinessHours(restaurant.id, [{ dayOfWeek: monday, openTime: '11:00', closeTime: '13:00' }]);

    const availability = await reservations.getAvailability(restaurant.id, '2026-07-06');

    expect(availability).toEqual([
      { time: '11:00', remaining: 10 },
      { time: '12:00', remaining: 10 },
    ]);
  });

  it('subtracts confirmed reservations from remaining capacity', async () => {
    const owner = await prisma.user.create({ data: { email: 'o2@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await restaurants.create(owner.id, { name: 'R', slotIntervalMinutes: 60, capacityPerSlot: 10 });
    await restaurants.setBusinessHours(restaurant.id, [{ dayOfWeek: 1, openTime: '11:00', closeTime: '13:00' }]);
    await prisma.reservation.create({
      data: {
        restaurantId: restaurant.id,
        reservationCode: 'ABC123',
        customerName: 'Kim',
        customerPhone: '010-0000-0000',
        partySize: 4,
        reservationDate: '2026-07-06',
        reservationTime: '11:00',
        status: 'CONFIRMED',
      },
    });

    const availability = await reservations.getAvailability(restaurant.id, '2026-07-06');

    expect(availability).toEqual([
      { time: '11:00', remaining: 6 },
      { time: '12:00', remaining: 10 },
    ]);
  });

  it('returns an empty list when the restaurant is closed on that day', async () => {
    const owner = await prisma.user.create({ data: { email: 'o3@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await restaurants.create(owner.id, { name: 'R', slotIntervalMinutes: 60, capacityPerSlot: 10 });
    await restaurants.setBusinessHours(restaurant.id, [{ dayOfWeek: 1, openTime: '11:00', closeTime: '13:00' }]);

    const availability = await reservations.getAvailability(restaurant.id, '2026-07-07');

    expect(availability).toEqual([]);
  });
});
```

> 2026-07-06은 월요일(dayOfWeek=1), 2026-07-07은 화요일이다.

- [ ] **Step 6: 테스트 실행 → 실패 확인**

Run: `pnpm --filter api test -- reservations.service.spec.ts`
Expected: FAIL ("Cannot find module './reservations.service'")

- [ ] **Step 7: ReservationsService 구현 (getAvailability)**

`apps/api/src/reservations/reservations.service.ts`:
```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { dayOfWeekFromDate, generateSlots } from './slot.util';

@Injectable()
export class ReservationsService {
  constructor(private readonly prisma: PrismaService) {}

  async getAvailability(restaurantId: string, date: string) {
    const restaurant = await this.prisma.restaurant.findUniqueOrThrow({ where: { id: restaurantId } });
    const dayOfWeek = dayOfWeekFromDate(date);
    const businessHour = await this.prisma.businessHour.findUnique({
      where: { restaurantId_dayOfWeek: { restaurantId, dayOfWeek } },
    });
    if (!businessHour) {
      return [];
    }

    const slots = generateSlots(businessHour.openTime, businessHour.closeTime, restaurant.slotIntervalMinutes);
    const reservations = await this.prisma.reservation.findMany({
      where: { restaurantId, reservationDate: date, status: 'CONFIRMED' },
    });

    const bookedBySlot = new Map<string, number>();
    for (const r of reservations) {
      bookedBySlot.set(r.reservationTime, (bookedBySlot.get(r.reservationTime) ?? 0) + r.partySize);
    }

    return slots.map((time) => ({
      time,
      remaining: restaurant.capacityPerSlot - (bookedBySlot.get(time) ?? 0),
    }));
  }
}
```

- [ ] **Step 8: 테스트 실행 → 통과 확인**

Run: `pnpm --filter api test -- reservations.service.spec.ts`
Expected: PASS (3 passed)

- [ ] **Step 9: 컨트롤러 및 모듈 작성**

`apps/api/src/reservations/reservations.controller.ts`:
```typescript
import { Controller, Get, Param, Query } from '@nestjs/common';
import { ReservationsService } from './reservations.service';

@Controller('restaurants/:id/availability')
export class ReservationsController {
  constructor(private readonly reservationsService: ReservationsService) {}

  @Get()
  getAvailability(@Param('id') id: string, @Query('date') date: string) {
    return this.reservationsService.getAvailability(id, date);
  }
}
```

`apps/api/src/reservations/reservations.module.ts`:
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

- [ ] **Step 10: AppModule에 ReservationsModule 등록**

`apps/api/src/app.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { PrismaModule } from './prisma/prisma.module';
import { AuthModule } from './auth/auth.module';
import { RestaurantsModule } from './restaurants/restaurants.module';
import { ReservationsModule } from './reservations/reservations.module';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    PrismaModule,
    AuthModule,
    RestaurantsModule,
    ReservationsModule,
  ],
})
export class AppModule {}
```

- [ ] **Step 11: 전체 테스트 재실행 확인**

Run: `pnpm --filter api test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 12: Commit**

```bash
git add apps/api/src/reservations apps/api/src/app.module.ts
git commit -m "feat: add reservation availability calculation"
```

---

## Task 7: Reservations — 고객 예약 생성

**Files:**
- Create: `apps/api/src/reservations/dto/create-reservation.dto.ts`
- Modify: `apps/api/src/reservations/reservations.service.ts`
- Modify: `apps/api/src/reservations/reservations.service.spec.ts`
- Modify: `apps/api/src/reservations/reservations.controller.ts`

**Interfaces:**
- Consumes: `ReservationsService.getAvailability`, `generateSlots`, `generateReservationCode`, `dayOfWeekFromDate` (Task 6)
- Produces: `ReservationsService.createReservation(restaurantId, dto, createdByStaffId?: string): Promise<Reservation>`. Task 8, 9가 재사용.

- [ ] **Step 1: DTO 작성**

`apps/api/src/reservations/dto/create-reservation.dto.ts`:
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

- [ ] **Step 2: 실패 테스트 추가 (`reservations.service.spec.ts`에 추가)**

파일 하단에 새 `describe` 블록을 추가한다:

```typescript
describe('ReservationsService - createReservation', () => {
  let reservations: ReservationsService;
  let restaurants: RestaurantsService;
  let prisma: PrismaService;
  let restaurantId: string;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      providers: [ReservationsService, RestaurantsService, PrismaService],
    }).compile();
    reservations = moduleRef.get(ReservationsService);
    restaurants = moduleRef.get(RestaurantsService);
    prisma = moduleRef.get(PrismaService);
    await prisma.onModuleInit();
    await prisma.reservation.deleteMany();
    await prisma.businessHour.deleteMany();
    await prisma.restaurantMember.deleteMany();
    await prisma.restaurant.deleteMany();
    await prisma.user.deleteMany();

    const owner = await prisma.user.create({ data: { email: 'o@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await restaurants.create(owner.id, { name: 'R', slotIntervalMinutes: 60, capacityPerSlot: 10 });
    await restaurants.setBusinessHours(restaurant.id, [{ dayOfWeek: 1, openTime: '11:00', closeTime: '13:00' }]);
    restaurantId = restaurant.id;
  });

  afterEach(async () => {
    await prisma.onModuleDestroy();
  });

  it('creates a confirmed reservation with a unique code when capacity is available', async () => {
    const result = await reservations.createReservation(restaurantId, {
      customerName: 'Kim',
      customerPhone: '010-1111-2222',
      partySize: 4,
      reservationDate: '2026-07-06',
      reservationTime: '11:00',
    });

    expect(result.status).toBe('CONFIRMED');
    expect(result.reservationCode).toHaveLength(6);
  });

  it('rejects a reservation that would exceed remaining capacity', async () => {
    await reservations.createReservation(restaurantId, {
      customerName: 'Kim',
      customerPhone: '010-1111-2222',
      partySize: 8,
      reservationDate: '2026-07-06',
      reservationTime: '11:00',
    });

    await expect(
      reservations.createReservation(restaurantId, {
        customerName: 'Lee',
        customerPhone: '010-3333-4444',
        partySize: 3,
        reservationDate: '2026-07-06',
        reservationTime: '11:00',
      }),
    ).rejects.toThrow();
  });

  it('rejects a reservation time that is not a valid slot', async () => {
    await expect(
      reservations.createReservation(restaurantId, {
        customerName: 'Kim',
        customerPhone: '010-1111-2222',
        partySize: 2,
        reservationDate: '2026-07-06',
        reservationTime: '15:00',
      }),
    ).rejects.toThrow();
  });

  it('records createdByStaffId when a staff member creates the reservation', async () => {
    const staff = await prisma.user.create({ data: { email: 's@t.com', passwordHash: 'x', name: 'S' } });
    const result = await reservations.createReservation(
      restaurantId,
      { customerName: 'Kim', customerPhone: '010-1111-2222', partySize: 2, reservationDate: '2026-07-06', reservationTime: '11:00' },
      staff.id,
    );

    expect(result.createdByStaffId).toBe(staff.id);
  });
});
```

- [ ] **Step 3: 테스트 실행 → 실패 확인**

Run: `pnpm --filter api test -- reservations.service.spec.ts`
Expected: FAIL (`reservations.createReservation is not a function`)

- [ ] **Step 4: ReservationsService에 createReservation 추가**

`apps/api/src/reservations/reservations.service.ts` 상단 import를 다음으로 교체하고:

```typescript
import { BadRequestException, ConflictException, Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { CreateReservationDto } from './dto/create-reservation.dto';
import { dayOfWeekFromDate, generateReservationCode, generateSlots } from './slot.util';
```

클래스 하단에 다음 메서드를 추가한다:

```typescript
  async createReservation(restaurantId: string, dto: CreateReservationDto, createdByStaffId?: string) {
    const restaurant = await this.prisma.restaurant.findUniqueOrThrow({ where: { id: restaurantId } });
    const dayOfWeek = dayOfWeekFromDate(dto.reservationDate);
    const businessHour = await this.prisma.businessHour.findUnique({
      where: { restaurantId_dayOfWeek: { restaurantId, dayOfWeek } },
    });
    if (!businessHour) {
      throw new BadRequestException('Restaurant is closed on this date');
    }
    const validSlots = generateSlots(businessHour.openTime, businessHour.closeTime, restaurant.slotIntervalMinutes);
    if (!validSlots.includes(dto.reservationTime)) {
      throw new BadRequestException('Invalid reservation time slot');
    }

    return this.prisma.$transaction(async (tx) => {
      const existing = await tx.reservation.findMany({
        where: {
          restaurantId,
          reservationDate: dto.reservationDate,
          reservationTime: dto.reservationTime,
          status: 'CONFIRMED',
        },
      });
      const bookedCount = existing.reduce((sum, r) => sum + r.partySize, 0);
      if (bookedCount + dto.partySize > restaurant.capacityPerSlot) {
        throw new ConflictException('Not enough remaining capacity for this time slot');
      }

      let reservationCode = generateReservationCode();
      for (let attempt = 0; attempt < 5; attempt++) {
        const codeTaken = await tx.reservation.findUnique({ where: { reservationCode } });
        if (!codeTaken) break;
        reservationCode = generateReservationCode();
      }

      return tx.reservation.create({
        data: {
          restaurantId,
          reservationCode,
          customerName: dto.customerName,
          customerPhone: dto.customerPhone,
          partySize: dto.partySize,
          reservationDate: dto.reservationDate,
          reservationTime: dto.reservationTime,
          status: 'CONFIRMED',
          createdByStaffId: createdByStaffId ?? null,
        },
      });
    });
  }
```

- [ ] **Step 5: 테스트 실행 → 통과 확인**

Run: `pnpm --filter api test -- reservations.service.spec.ts`
Expected: PASS (7 passed)

- [ ] **Step 6: 컨트롤러에 고객용 예약 생성 엔드포인트 추가**

`apps/api/src/reservations/reservations.controller.ts` 전체를 다음으로 교체한다:

```typescript
import { Body, Controller, Get, Param, Post, Query } from '@nestjs/common';
import { ReservationsService } from './reservations.service';
import { CreateReservationDto } from './dto/create-reservation.dto';

@Controller('restaurants/:id')
export class ReservationsController {
  constructor(private readonly reservationsService: ReservationsService) {}

  @Get('availability')
  getAvailability(@Param('id') id: string, @Query('date') date: string) {
    return this.reservationsService.getAvailability(id, date);
  }

  @Post('reservations')
  createReservation(@Param('id') id: string, @Body() dto: CreateReservationDto) {
    return this.reservationsService.createReservation(id, dto);
  }
}
```

- [ ] **Step 7: 전체 테스트 재실행 확인**

Run: `pnpm --filter api test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/reservations
git commit -m "feat: add customer reservation creation with capacity validation"
```

---

## Task 8: Reservations — 고객 예약 조회 & 취소

**Files:**
- Modify: `apps/api/src/reservations/reservations.service.ts`
- Modify: `apps/api/src/reservations/reservations.service.spec.ts`
- Modify: `apps/api/src/reservations/reservations.controller.ts`

**Interfaces:**
- Consumes: `ReservationsService.createReservation` (Task 7)
- Produces: `ReservationsService.lookup(code, phone): Promise<Reservation>`, `ReservationsService.cancelByCustomer(code, phone): Promise<Reservation>`

- [ ] **Step 1: 실패 테스트 추가 (`reservations.service.spec.ts`에 추가)**

파일 하단에 새 `describe` 블록을 추가한다:

```typescript
describe('ReservationsService - lookup and customer cancel', () => {
  let reservations: ReservationsService;
  let restaurants: RestaurantsService;
  let prisma: PrismaService;
  let restaurantId: string;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      providers: [ReservationsService, RestaurantsService, PrismaService],
    }).compile();
    reservations = moduleRef.get(ReservationsService);
    restaurants = moduleRef.get(RestaurantsService);
    prisma = moduleRef.get(PrismaService);
    await prisma.onModuleInit();
    await prisma.reservation.deleteMany();
    await prisma.businessHour.deleteMany();
    await prisma.restaurantMember.deleteMany();
    await prisma.restaurant.deleteMany();
    await prisma.user.deleteMany();

    const owner = await prisma.user.create({ data: { email: 'o@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await restaurants.create(owner.id, { name: 'R', slotIntervalMinutes: 60, capacityPerSlot: 10 });
    await restaurants.setBusinessHours(restaurant.id, [{ dayOfWeek: 1, openTime: '11:00', closeTime: '13:00' }]);
    restaurantId = restaurant.id;
  });

  afterEach(async () => {
    await prisma.onModuleDestroy();
  });

  it('looks up a reservation by matching code and phone', async () => {
    const created = await reservations.createReservation(restaurantId, {
      customerName: 'Kim',
      customerPhone: '010-1111-2222',
      partySize: 2,
      reservationDate: '2026-07-06',
      reservationTime: '11:00',
    });

    const found = await reservations.lookup(created.reservationCode, '010-1111-2222');
    expect(found.id).toBe(created.id);
  });

  it('throws when phone does not match the code', async () => {
    const created = await reservations.createReservation(restaurantId, {
      customerName: 'Kim',
      customerPhone: '010-1111-2222',
      partySize: 2,
      reservationDate: '2026-07-06',
      reservationTime: '11:00',
    });

    await expect(reservations.lookup(created.reservationCode, '010-9999-9999')).rejects.toThrow();
  });

  it('cancels a reservation when code and phone match', async () => {
    const created = await reservations.createReservation(restaurantId, {
      customerName: 'Kim',
      customerPhone: '010-1111-2222',
      partySize: 2,
      reservationDate: '2026-07-06',
      reservationTime: '11:00',
    });

    const cancelled = await reservations.cancelByCustomer(created.reservationCode, '010-1111-2222');
    expect(cancelled.status).toBe('CANCELLED');
  });

  it('frees capacity after a customer cancellation', async () => {
    const created = await reservations.createReservation(restaurantId, {
      customerName: 'Kim',
      customerPhone: '010-1111-2222',
      partySize: 10,
      reservationDate: '2026-07-06',
      reservationTime: '11:00',
    });
    await reservations.cancelByCustomer(created.reservationCode, '010-1111-2222');

    const availability = await reservations.getAvailability(restaurantId, '2026-07-06');
    expect(availability.find((a) => a.time === '11:00')?.remaining).toBe(10);
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter api test -- reservations.service.spec.ts`
Expected: FAIL (`reservations.lookup is not a function`)

- [ ] **Step 3: ReservationsService에 lookup, cancelByCustomer 추가**

`apps/api/src/reservations/reservations.service.ts` 상단 import를 다음으로 교체하고:

```typescript
import { BadRequestException, ConflictException, Injectable, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { CreateReservationDto } from './dto/create-reservation.dto';
import { dayOfWeekFromDate, generateReservationCode, generateSlots } from './slot.util';
```

클래스 하단에 다음 메서드를 추가한다:

```typescript
  async lookup(reservationCode: string, customerPhone: string) {
    const reservation = await this.prisma.reservation.findFirst({
      where: { reservationCode, customerPhone },
    });
    if (!reservation) {
      throw new NotFoundException('Reservation not found');
    }
    return reservation;
  }

  async cancelByCustomer(reservationCode: string, customerPhone: string) {
    const reservation = await this.lookup(reservationCode, customerPhone);
    return this.prisma.reservation.update({
      where: { id: reservation.id },
      data: { status: 'CANCELLED' },
    });
  }
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `pnpm --filter api test -- reservations.service.spec.ts`
Expected: PASS (11 passed)

- [ ] **Step 5: 컨트롤러에 조회/취소 엔드포인트 추가**

`apps/api/src/reservations/reservations.controller.ts` 전체를 다음으로 교체한다:

```typescript
import { Body, Controller, Get, Param, Patch, Post, Query } from '@nestjs/common';
import { ReservationsService } from './reservations.service';
import { CreateReservationDto } from './dto/create-reservation.dto';
import { IsString } from 'class-validator';

class LookupCancelDto {
  @IsString()
  reservationCode: string;

  @IsString()
  customerPhone: string;
}

@Controller('restaurants/:id')
export class ReservationsController {
  constructor(private readonly reservationsService: ReservationsService) {}

  @Get('availability')
  getAvailability(@Param('id') id: string, @Query('date') date: string) {
    return this.reservationsService.getAvailability(id, date);
  }

  @Post('reservations')
  createReservation(@Param('id') id: string, @Body() dto: CreateReservationDto) {
    return this.reservationsService.createReservation(id, dto);
  }
}

@Controller('reservations')
export class ReservationLookupController {
  constructor(private readonly reservationsService: ReservationsService) {}

  @Get('lookup')
  lookup(@Query('code') code: string, @Query('phone') phone: string) {
    return this.reservationsService.lookup(code, phone);
  }

  @Patch('cancel')
  cancel(@Body() dto: LookupCancelDto) {
    return this.reservationsService.cancelByCustomer(dto.reservationCode, dto.customerPhone);
  }
}
```

`apps/api/src/reservations/reservations.module.ts`을 다음으로 교체한다:

```typescript
import { Module } from '@nestjs/common';
import { RestaurantsModule } from '../restaurants/restaurants.module';
import { ReservationsService } from './reservations.service';
import { ReservationsController, ReservationLookupController } from './reservations.controller';

@Module({
  imports: [RestaurantsModule],
  providers: [ReservationsService],
  controllers: [ReservationsController, ReservationLookupController],
  exports: [ReservationsService],
})
export class ReservationsModule {}
```

- [ ] **Step 6: 전체 테스트 재실행 확인**

Run: `pnpm --filter api test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/reservations
git commit -m "feat: add customer reservation lookup and cancellation"
```

---

## Task 9: Reservations — 직원 예약 관리 (목록/수동등록/상태변경)

**Files:**
- Create: `apps/api/src/reservations/dto/staff-create-reservation.dto.ts`
- Create: `apps/api/src/reservations/dto/update-status.dto.ts`
- Modify: `apps/api/src/reservations/reservations.service.ts`
- Modify: `apps/api/src/reservations/reservations.service.spec.ts`
- Modify: `apps/api/src/reservations/reservations.controller.ts`
- Modify: `apps/api/src/reservations/reservations.module.ts`

**Interfaces:**
- Consumes: `ReservationsService.createReservation` (Task 7), `RestaurantMemberGuard`, `Roles` (Task 4)
- Produces: `ReservationsService.listForRestaurant(restaurantId, date?): Promise<Reservation[]>`, `ReservationsService.updateStatus(restaurantId, reservationId, status): Promise<Reservation>`. `POST /restaurants/:id/reservations/staff`, `GET /restaurants/:id/reservations`, `PATCH /restaurants/:id/reservations/:resId/status` — 인증+멤버십 필요.

- [ ] **Step 1: DTO 작성**

`apps/api/src/reservations/dto/staff-create-reservation.dto.ts`:
```typescript
export { CreateReservationDto as StaffCreateReservationDto } from './create-reservation.dto';
```

`apps/api/src/reservations/dto/update-status.dto.ts`:
```typescript
import { IsIn } from 'class-validator';

export type ReservationStatus = 'CONFIRMED' | 'CANCELLED' | 'COMPLETED' | 'NO_SHOW';

export class UpdateStatusDto {
  @IsIn(['CONFIRMED', 'CANCELLED', 'COMPLETED', 'NO_SHOW'])
  status: ReservationStatus;
}
```

- [ ] **Step 2: 실패 테스트 추가 (`reservations.service.spec.ts`에 추가)**

파일 하단에 새 `describe` 블록을 추가한다:

```typescript
describe('ReservationsService - staff management', () => {
  let reservations: ReservationsService;
  let restaurants: RestaurantsService;
  let prisma: PrismaService;
  let restaurantId: string;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      providers: [ReservationsService, RestaurantsService, PrismaService],
    }).compile();
    reservations = moduleRef.get(ReservationsService);
    restaurants = moduleRef.get(RestaurantsService);
    prisma = moduleRef.get(PrismaService);
    await prisma.onModuleInit();
    await prisma.reservation.deleteMany();
    await prisma.businessHour.deleteMany();
    await prisma.restaurantMember.deleteMany();
    await prisma.restaurant.deleteMany();
    await prisma.user.deleteMany();

    const owner = await prisma.user.create({ data: { email: 'o@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await restaurants.create(owner.id, { name: 'R', slotIntervalMinutes: 60, capacityPerSlot: 10 });
    await restaurants.setBusinessHours(restaurant.id, [{ dayOfWeek: 1, openTime: '11:00', closeTime: '13:00' }]);
    restaurantId = restaurant.id;
  });

  afterEach(async () => {
    await prisma.onModuleDestroy();
  });

  it('lists reservations for a restaurant filtered by date', async () => {
    await reservations.createReservation(restaurantId, {
      customerName: 'Kim', customerPhone: '010-1', partySize: 2, reservationDate: '2026-07-06', reservationTime: '11:00',
    });
    await reservations.createReservation(restaurantId, {
      customerName: 'Lee', customerPhone: '010-2', partySize: 2, reservationDate: '2026-07-13', reservationTime: '11:00',
    });

    const list = await reservations.listForRestaurant(restaurantId, '2026-07-06');
    expect(list).toHaveLength(1);
    expect(list[0].customerName).toBe('Kim');
  });

  it('updates a reservation status', async () => {
    const created = await reservations.createReservation(restaurantId, {
      customerName: 'Kim', customerPhone: '010-1', partySize: 2, reservationDate: '2026-07-06', reservationTime: '11:00',
    });

    const updated = await reservations.updateStatus(restaurantId, created.id, 'NO_SHOW');
    expect(updated.status).toBe('NO_SHOW');
  });

  it('throws when updating a reservation that belongs to a different restaurant', async () => {
    const owner2 = await prisma.user.create({ data: { email: 'o2@t.com', passwordHash: 'x', name: 'O2' } });
    const otherRestaurant = await restaurants.create(owner2.id, { name: 'Other', slotIntervalMinutes: 60, capacityPerSlot: 10 });
    const created = await reservations.createReservation(restaurantId, {
      customerName: 'Kim', customerPhone: '010-1', partySize: 2, reservationDate: '2026-07-06', reservationTime: '11:00',
    });

    await expect(reservations.updateStatus(otherRestaurant.id, created.id, 'CANCELLED')).rejects.toThrow();
  });
});
```

- [ ] **Step 3: 테스트 실행 → 실패 확인**

Run: `pnpm --filter api test -- reservations.service.spec.ts`
Expected: FAIL (`reservations.listForRestaurant is not a function`)

- [ ] **Step 4: ReservationsService에 listForRestaurant, updateStatus 추가**

`apps/api/src/reservations/reservations.service.ts` 클래스 하단에 다음 메서드를 추가한다:

```typescript
  async listForRestaurant(restaurantId: string, date?: string) {
    return this.prisma.reservation.findMany({
      where: { restaurantId, ...(date ? { reservationDate: date } : {}) },
      orderBy: [{ reservationDate: 'asc' }, { reservationTime: 'asc' }],
    });
  }

  async updateStatus(restaurantId: string, reservationId: string, status: string) {
    const reservation = await this.prisma.reservation.findFirst({ where: { id: reservationId, restaurantId } });
    if (!reservation) {
      throw new NotFoundException('Reservation not found for this restaurant');
    }
    return this.prisma.reservation.update({ where: { id: reservationId }, data: { status } });
  }
```

- [ ] **Step 5: 테스트 실행 → 통과 확인**

Run: `pnpm --filter api test -- reservations.service.spec.ts`
Expected: PASS (14 passed)

- [ ] **Step 6: 컨트롤러에 직원용 엔드포인트 추가**

`apps/api/src/reservations/reservations.controller.ts` 상단 import에 아래를 추가하고:

```typescript
import { UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/jwt-auth.guard';
import { CurrentUser, CurrentUserPayload } from '../auth/current-user.decorator';
import { RestaurantMemberGuard } from '../restaurants/restaurant-member.guard';
import { UpdateStatusDto } from './dto/update-status.dto';
```

`ReservationsController` 클래스 안에 다음 메서드들을 추가한다:

```typescript
  @UseGuards(JwtAuthGuard, RestaurantMemberGuard)
  @Get('reservations')
  list(@Param('id') id: string, @Query('date') date?: string) {
    return this.reservationsService.listForRestaurant(id, date);
  }

  @UseGuards(JwtAuthGuard, RestaurantMemberGuard)
  @Post('reservations/staff')
  createByStaff(
    @Param('id') id: string,
    @Body() dto: CreateReservationDto,
    @CurrentUser() user: CurrentUserPayload,
  ) {
    return this.reservationsService.createReservation(id, dto, user.userId);
  }

  @UseGuards(JwtAuthGuard, RestaurantMemberGuard)
  @Patch('reservations/:resId/status')
  updateStatus(
    @Param('id') id: string,
    @Param('resId') resId: string,
    @Body() dto: UpdateStatusDto,
  ) {
    return this.reservationsService.updateStatus(id, resId, dto.status);
  }
```

- [ ] **Step 7: 전체 테스트 재실행 확인**

Run: `pnpm --filter api test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/reservations
git commit -m "feat: add staff reservation listing, manual entry, and status updates"
```

---

## Task 10: 백엔드 E2E 테스트 — 전체 예약 플로우

**Files:**
- Create: `apps/api/test/jest-e2e.json`
- Create: `apps/api/test/reservation-flow.e2e-spec.ts`

**Interfaces:**
- Consumes: `AppModule` (Task 1-9 전체)
- Produces: 없음 (검증 전용 테스트)

- [ ] **Step 1: e2e Jest 설정 작성**

`apps/api/test/jest-e2e.json`:
```json
{
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": ".",
  "testEnvironment": "node",
  "testRegex": ".e2e-spec.ts$",
  "transform": { "^.+\\.(t|j)s$": "ts-jest" }
}
```

- [ ] **Step 2: e2e 테스트 작성 (사장님 가입 → 음식점 생성 → 영업시간 설정 → 고객 예약 → 직원 취소)**

`apps/api/test/reservation-flow.e2e-spec.ts`:
```typescript
import { INestApplication, ValidationPipe } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';
import { PrismaService } from '../src/prisma/prisma.service';

describe('Reservation flow (e2e)', () => {
  let app: INestApplication;
  let prisma: PrismaService;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({ imports: [AppModule] }).compile();
    app = moduleRef.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
    await app.init();
    prisma = moduleRef.get(PrismaService);
  });

  afterAll(async () => {
    await app.close();
  });

  beforeEach(async () => {
    await prisma.reservation.deleteMany();
    await prisma.businessHour.deleteMany();
    await prisma.restaurantMember.deleteMany();
    await prisma.restaurant.deleteMany();
    await prisma.user.deleteMany();
  });

  it('supports the full owner + customer reservation lifecycle', async () => {
    const signupRes = await request(app.getHttpServer())
      .post('/auth/signup')
      .send({ email: 'owner@e2e.com', password: 'password123', name: 'Owner' })
      .expect(201);
    expect(signupRes.body.email).toBe('owner@e2e.com');

    const loginRes = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email: 'owner@e2e.com', password: 'password123' })
      .expect(201);
    const token = loginRes.body.accessToken;

    const restaurantRes = await request(app.getHttpServer())
      .post('/restaurants')
      .set('Authorization', `Bearer ${token}`)
      .send({ name: 'E2E Restaurant', slotIntervalMinutes: 60, capacityPerSlot: 4 })
      .expect(201);
    const restaurantId = restaurantRes.body.id;

    await request(app.getHttpServer())
      .put(`/restaurants/${restaurantId}/business-hours`)
      .set('Authorization', `Bearer ${token}`)
      .send({ hours: [{ dayOfWeek: 1, openTime: '11:00', closeTime: '13:00' }] })
      .expect(200);

    const availabilityRes = await request(app.getHttpServer())
      .get(`/restaurants/${restaurantId}/availability?date=2026-07-06`)
      .expect(200);
    expect(availabilityRes.body).toEqual([
      { time: '11:00', remaining: 4 },
      { time: '12:00', remaining: 4 },
    ]);

    const reservationRes = await request(app.getHttpServer())
      .post(`/restaurants/${restaurantId}/reservations`)
      .send({
        customerName: '고객',
        customerPhone: '010-1234-5678',
        partySize: 4,
        reservationDate: '2026-07-06',
        reservationTime: '11:00',
      })
      .expect(201);
    const { reservationCode } = reservationRes.body;

    await request(app.getHttpServer())
      .post(`/restaurants/${restaurantId}/reservations`)
      .send({
        customerName: '다른고객',
        customerPhone: '010-9999-0000',
        partySize: 1,
        reservationDate: '2026-07-06',
        reservationTime: '11:00',
      })
      .expect(409);

    const lookupRes = await request(app.getHttpServer())
      .get(`/reservations/lookup?code=${reservationCode}&phone=010-1234-5678`)
      .expect(200);
    expect(lookupRes.body.customerName).toBe('고객');

    const staffListRes = await request(app.getHttpServer())
      .get(`/restaurants/${restaurantId}/reservations`)
      .set('Authorization', `Bearer ${token}`)
      .expect(200);
    expect(staffListRes.body).toHaveLength(1);

    await request(app.getHttpServer())
      .patch(`/restaurants/${restaurantId}/reservations/${staffListRes.body[0].id}/status`)
      .set('Authorization', `Bearer ${token}`)
      .send({ status: 'COMPLETED' })
      .expect(200);

    const freedAvailabilityRes = await request(app.getHttpServer())
      .get(`/restaurants/${restaurantId}/availability?date=2026-07-06`)
      .expect(200);
    expect(freedAvailabilityRes.body.find((s: { time: string }) => s.time === '11:00').remaining).toBe(4);
  });
});
```

> 마지막 단계에서 예약 상태를 `COMPLETED`로 바꾸면 `getAvailability`가 `status='CONFIRMED'`인 예약만 집계하므로 잔여 좌석이 다시 4로 회복된다.

- [ ] **Step 3: e2e 테스트 실행 확인**

Run: `pnpm --filter api test:e2e`
Expected: PASS (1 passed)

- [ ] **Step 4: Commit**

```bash
git add apps/api/test
git commit -m "test: add end-to-end reservation lifecycle test"
```

---

## Task 11: 프론트엔드 라우팅 + API 클라이언트 + 인증 컨텍스트

**Files:**
- Create: `apps/web/src/api/client.ts`
- Create: `apps/web/src/auth/AuthContext.tsx`
- Modify: `apps/web/src/App.tsx`
- Create: `apps/web/src/App.spec.tsx`
- Create: `apps/web/src/setupTests.ts`
- Modify: `apps/web/vite.config.ts`
- Create: `apps/web/.env`

**Interfaces:**
- Produces: `apiFetch<T>(path: string, options?: RequestInit): Promise<T>` — `VITE_API_URL` 기준, `localStorage`의 토큰을 자동 첨부. `AuthProvider`, `useAuth(): { token: string | null; login(token): void; logout(): void }`. Task 12~16의 모든 페이지가 이 두 모듈을 사용.

- [ ] **Step 1: 환경 변수 및 테스트 셋업 작성**

`apps/web/.env`:
```
VITE_API_URL=http://localhost:3000
```

`apps/web/src/setupTests.ts`:
```typescript
import '@testing-library/jest-dom/vitest';
```

`apps/web/vite.config.ts`:
```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/setupTests.ts',
  },
});
```

- [ ] **Step 2: API 클라이언트 작성**

`apps/web/src/api/client.ts`:
```typescript
const API_URL = import.meta.env.VITE_API_URL ?? 'http://localhost:3000';

export class ApiError extends Error {
  constructor(
    public status: number,
    message: string,
  ) {
    super(message);
  }
}

export async function apiFetch<T>(path: string, options: RequestInit = {}): Promise<T> {
  const token = localStorage.getItem('accessToken');
  const headers: Record<string, string> = {
    'Content-Type': 'application/json',
    ...(options.headers as Record<string, string>),
  };
  if (token) {
    headers.Authorization = `Bearer ${token}`;
  }

  const response = await fetch(`${API_URL}${path}`, { ...options, headers });
  const contentType = response.headers.get('content-type') ?? '';
  const body = contentType.includes('application/json') ? await response.json() : undefined;

  if (!response.ok) {
    throw new ApiError(response.status, body?.message ?? 'Request failed');
  }
  return body as T;
}
```

- [ ] **Step 3: AuthContext 작성**

`apps/web/src/auth/AuthContext.tsx`:
```tsx
import { createContext, ReactNode, useContext, useState } from 'react';

interface AuthContextValue {
  token: string | null;
  login: (token: string) => void;
  logout: () => void;
}

const AuthContext = createContext<AuthContextValue | undefined>(undefined);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [token, setToken] = useState<string | null>(localStorage.getItem('accessToken'));

  const login = (newToken: string) => {
    localStorage.setItem('accessToken', newToken);
    setToken(newToken);
  };

  const logout = () => {
    localStorage.removeItem('accessToken');
    setToken(null);
  };

  return <AuthContext.Provider value={{ token, login, logout }}>{children}</AuthContext.Provider>;
}

export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (!ctx) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return ctx;
}
```

- [ ] **Step 4: App 라우팅 실패 테스트 작성**

`apps/web/src/App.spec.tsx`:
```tsx
import { render, screen } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import App from './App';

describe('App routing', () => {
  it('renders the login page at /login', () => {
    render(<App initialEntries={['/login']} />);
    expect(screen.getByRole('heading', { name: '로그인' })).toBeInTheDocument();
  });
});
```

- [ ] **Step 5: 테스트 실행 → 실패 확인**

Run: `pnpm --filter web test`
Expected: FAIL (`App` 컴포넌트가 `initialEntries` prop을 지원하지 않음 / `로그인` 텍스트 없음)

- [ ] **Step 6: App에 라우팅 뼈대 작성 (플레이스홀더 페이지는 Task 12~16에서 대체)**

`apps/web/src/App.tsx`:
```tsx
import { MemoryRouter, BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { AuthProvider } from './auth/AuthContext';

function LoginPagePlaceholder() {
  return <h1>로그인</h1>;
}

interface AppProps {
  initialEntries?: string[];
}

export default function App({ initialEntries }: AppProps) {
  const Router = initialEntries ? MemoryRouter : BrowserRouter;
  const routerProps = initialEntries ? { initialEntries } : {};

  return (
    <AuthProvider>
      <Router {...routerProps}>
        <Routes>
          <Route path="/login" element={<LoginPagePlaceholder />} />
          <Route path="*" element={<Navigate to="/login" replace />} />
        </Routes>
      </Router>
    </AuthProvider>
  );
}
```

- [ ] **Step 7: 테스트 실행 → 통과 확인**

Run: `pnpm --filter web test`
Expected: PASS (1 passed)

- [ ] **Step 8: Commit**

```bash
git add apps/web/src apps/web/vite.config.ts apps/web/.env
git commit -m "feat: add API client, auth context, and routing skeleton"
```

---

## Task 12: 프론트엔드 — 회원가입/로그인 페이지

**Files:**
- Create: `apps/web/src/pages/SignupPage.tsx`
- Create: `apps/web/src/pages/SignupPage.spec.tsx`
- Create: `apps/web/src/pages/LoginPage.tsx`
- Create: `apps/web/src/pages/LoginPage.spec.tsx`
- Modify: `apps/web/src/App.tsx`
- Modify: `apps/web/src/App.spec.tsx`

**Interfaces:**
- Consumes: `apiFetch` (Task 11의 `api/client.ts`), `useAuth` (Task 11의 `AuthContext`)
- Produces: `/signup`, `/login` 라우트. 로그인 성공 시 `useAuth().login(token)` 호출 후 `/dashboard`로 이동.

- [ ] **Step 1: LoginPage 실패 테스트 작성**

`apps/web/src/pages/LoginPage.spec.tsx`:
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { vi } from 'vitest';
import LoginPage from './LoginPage';
import { AuthProvider } from '../auth/AuthContext';

describe('LoginPage', () => {
  it('logs in and stores the access token on submit', async () => {
    vi.stubGlobal(
      'fetch',
      vi.fn().mockResolvedValue({
        ok: true,
        headers: { get: () => 'application/json' },
        json: async () => ({ accessToken: 'fake-token' }),
      }),
    );

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
      expect(localStorage.getItem('accessToken')).toBe('fake-token');
    });
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter web test -- LoginPage`
Expected: FAIL ("Cannot find module './LoginPage'")

- [ ] **Step 3: LoginPage 구현**

`apps/web/src/pages/LoginPage.tsx`:
```tsx
import { FormEvent, useState } from 'react';
import { useNavigate, Link } from 'react-router-dom';
import { apiFetch, ApiError } from '../api/client';
import { useAuth } from '../auth/AuthContext';

export default function LoginPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState<string | null>(null);
  const { login } = useAuth();
  const navigate = useNavigate();

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    setError(null);
    try {
      const result = await apiFetch<{ accessToken: string }>('/auth/login', {
        method: 'POST',
        body: JSON.stringify({ email, password }),
      });
      login(result.accessToken);
      navigate('/dashboard');
    } catch (err) {
      setError(err instanceof ApiError ? err.message : '로그인에 실패했습니다');
    }
  }

  return (
    <div>
      <h1>로그인</h1>
      <form onSubmit={handleSubmit}>
        <label htmlFor="email">이메일</label>
        <input id="email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} />
        <label htmlFor="password">비밀번호</label>
        <input id="password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} />
        {error && <p role="alert">{error}</p>}
        <button type="submit">로그인</button>
      </form>
      <Link to="/signup">회원가입</Link>
    </div>
  );
}
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `pnpm --filter web test -- LoginPage`
Expected: PASS (1 passed)

- [ ] **Step 5: SignupPage 실패 테스트 작성**

`apps/web/src/pages/SignupPage.spec.tsx`:
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { vi } from 'vitest';
import SignupPage from './SignupPage';

describe('SignupPage', () => {
  it('submits signup details to the API', async () => {
    const fetchMock = vi.fn().mockResolvedValue({
      ok: true,
      headers: { get: () => 'application/json' },
      json: async () => ({ id: '1', email: 'owner@test.com', name: 'Owner' }),
    });
    vi.stubGlobal('fetch', fetchMock);

    render(
      <MemoryRouter>
        <SignupPage />
      </MemoryRouter>,
    );

    fireEvent.change(screen.getByLabelText('이메일'), { target: { value: 'owner@test.com' } });
    fireEvent.change(screen.getByLabelText('비밀번호'), { target: { value: 'password123' } });
    fireEvent.change(screen.getByLabelText('이름'), { target: { value: 'Owner' } });
    fireEvent.click(screen.getByRole('button', { name: '회원가입' }));

    await waitFor(() => {
      expect(fetchMock).toHaveBeenCalledWith(
        expect.stringContaining('/auth/signup'),
        expect.objectContaining({ method: 'POST' }),
      );
    });
  });
});
```

- [ ] **Step 6: 테스트 실행 → 실패 확인**

Run: `pnpm --filter web test -- SignupPage`
Expected: FAIL ("Cannot find module './SignupPage'")

- [ ] **Step 7: SignupPage 구현**

`apps/web/src/pages/SignupPage.tsx`:
```tsx
import { FormEvent, useState } from 'react';
import { useNavigate, Link } from 'react-router-dom';
import { apiFetch, ApiError } from '../api/client';

export default function SignupPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [name, setName] = useState('');
  const [error, setError] = useState<string | null>(null);
  const navigate = useNavigate();

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    setError(null);
    try {
      await apiFetch('/auth/signup', {
        method: 'POST',
        body: JSON.stringify({ email, password, name }),
      });
      navigate('/login');
    } catch (err) {
      setError(err instanceof ApiError ? err.message : '회원가입에 실패했습니다');
    }
  }

  return (
    <div>
      <h1>회원가입</h1>
      <form onSubmit={handleSubmit}>
        <label htmlFor="email">이메일</label>
        <input id="email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} />
        <label htmlFor="password">비밀번호</label>
        <input id="password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} />
        <label htmlFor="name">이름</label>
        <input id="name" type="text" value={name} onChange={(e) => setName(e.target.value)} />
        {error && <p role="alert">{error}</p>}
        <button type="submit">회원가입</button>
      </form>
      <Link to="/login">로그인</Link>
    </div>
  );
}
```

- [ ] **Step 8: 테스트 실행 → 통과 확인**

Run: `pnpm --filter web test -- SignupPage`
Expected: PASS (1 passed)

- [ ] **Step 9: App 라우팅에 실제 페이지 연결, App.spec.tsx 갱신**

`apps/web/src/App.tsx`:
```tsx
import { MemoryRouter, BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { AuthProvider } from './auth/AuthContext';
import LoginPage from './pages/LoginPage';
import SignupPage from './pages/SignupPage';

interface AppProps {
  initialEntries?: string[];
}

export default function App({ initialEntries }: AppProps) {
  const Router = initialEntries ? MemoryRouter : BrowserRouter;
  const routerProps = initialEntries ? { initialEntries } : {};

  return (
    <AuthProvider>
      <Router {...routerProps}>
        <Routes>
          <Route path="/login" element={<LoginPage />} />
          <Route path="/signup" element={<SignupPage />} />
          <Route path="*" element={<Navigate to="/login" replace />} />
        </Routes>
      </Router>
    </AuthProvider>
  );
}
```

`apps/web/src/App.spec.tsx`은 변경 없이 그대로 통과해야 한다 (로그인 페이지의 `h1` 텍스트는 동일).

- [ ] **Step 10: 전체 프론트엔드 테스트 재실행 확인**

Run: `pnpm --filter web test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 11: Commit**

```bash
git add apps/web/src
git commit -m "feat: add signup and login pages"
```

---

## Task 13: 프론트엔드 — 대시보드 (음식점 설정 + 영업시간 + 직원 초대)

**Files:**
- Create: `apps/web/src/pages/DashboardPage.tsx`
- Create: `apps/web/src/pages/DashboardPage.spec.tsx`
- Create: `apps/web/src/components/RestaurantSetupForm.tsx`
- Create: `apps/web/src/components/BusinessHoursEditor.tsx`
- Create: `apps/web/src/components/StaffInviteForm.tsx`
- Modify: `apps/web/src/App.tsx`

**Interfaces:**
- Consumes: `apiFetch`, `useAuth` (Task 11)
- Produces: `/dashboard` 라우트 (인증 필요, 토큰 없으면 `/login`으로 리다이렉트). `RestaurantSetupForm`, `BusinessHoursEditor`, `StaffInviteForm` — Task 14에서 같은 대시보드 페이지 내부에 예약 목록 컴포넌트를 추가할 때 재사용하는 레이아웃 컨벤션.

- [ ] **Step 1: DashboardPage 실패 테스트 작성**

`apps/web/src/pages/DashboardPage.spec.tsx`:
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { vi } from 'vitest';
import DashboardPage from './DashboardPage';
import { AuthProvider } from '../auth/AuthContext';

function renderDashboard() {
  localStorage.setItem('accessToken', 'fake-token');
  return render(
    <AuthProvider>
      <MemoryRouter>
        <DashboardPage />
      </MemoryRouter>
    </AuthProvider>,
  );
}

describe('DashboardPage', () => {
  it('creates a restaurant when no restaurant exists yet', async () => {
    const fetchMock = vi.fn().mockImplementation((url: string) => {
      if (url.includes('/restaurants/mine') && !url.includes('POST')) {
        return Promise.resolve({
          ok: true,
          headers: { get: () => 'application/json' },
          json: async () => [],
        });
      }
      return Promise.resolve({
        ok: true,
        headers: { get: () => 'application/json' },
        json: async () => ({ id: 'r1', name: 'My Place', slotIntervalMinutes: 30, capacityPerSlot: 10 }),
      });
    });
    vi.stubGlobal('fetch', fetchMock);

    renderDashboard();

    await waitFor(() => expect(screen.getByLabelText('음식점 이름')).toBeInTheDocument());
    fireEvent.change(screen.getByLabelText('음식점 이름'), { target: { value: 'My Place' } });
    fireEvent.change(screen.getByLabelText('예약 단위(분)'), { target: { value: '30' } });
    fireEvent.change(screen.getByLabelText('시간대별 좌석수'), { target: { value: '10' } });
    fireEvent.click(screen.getByRole('button', { name: '음식점 등록' }));

    await waitFor(() => {
      expect(fetchMock).toHaveBeenCalledWith(
        expect.stringContaining('/restaurants'),
        expect.objectContaining({ method: 'POST' }),
      );
    });
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter web test -- DashboardPage`
Expected: FAIL ("Cannot find module './DashboardPage'")

- [ ] **Step 3: RestaurantSetupForm 구현**

`apps/web/src/components/RestaurantSetupForm.tsx`:
```tsx
import { FormEvent, useState } from 'react';
import { apiFetch } from '../api/client';

interface Restaurant {
  id: string;
  name: string;
  slotIntervalMinutes: number;
  capacityPerSlot: number;
}

export default function RestaurantSetupForm({ onCreated }: { onCreated: (r: Restaurant) => void }) {
  const [name, setName] = useState('');
  const [slotIntervalMinutes, setSlotIntervalMinutes] = useState(30);
  const [capacityPerSlot, setCapacityPerSlot] = useState(10);

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    const restaurant = await apiFetch<Restaurant>('/restaurants', {
      method: 'POST',
      body: JSON.stringify({ name, slotIntervalMinutes, capacityPerSlot }),
    });
    onCreated(restaurant);
  }

  return (
    <form onSubmit={handleSubmit}>
      <h2>음식점 등록</h2>
      <label htmlFor="restaurant-name">음식점 이름</label>
      <input id="restaurant-name" value={name} onChange={(e) => setName(e.target.value)} />
      <label htmlFor="slot-interval">예약 단위(분)</label>
      <input
        id="slot-interval"
        type="number"
        value={slotIntervalMinutes}
        onChange={(e) => setSlotIntervalMinutes(Number(e.target.value))}
      />
      <label htmlFor="capacity">시간대별 좌석수</label>
      <input id="capacity" type="number" value={capacityPerSlot} onChange={(e) => setCapacityPerSlot(Number(e.target.value))} />
      <button type="submit">음식점 등록</button>
    </form>
  );
}
```

- [ ] **Step 4: BusinessHoursEditor, StaffInviteForm 구현**

`apps/web/src/components/BusinessHoursEditor.tsx`:
```tsx
import { useState } from 'react';
import { apiFetch } from '../api/client';

const DAY_LABELS = ['일', '월', '화', '수', '목', '금', '토'];

interface Hour {
  dayOfWeek: number;
  openTime: string;
  closeTime: string;
}

export default function BusinessHoursEditor({ restaurantId }: { restaurantId: string }) {
  const [hours, setHours] = useState<Hour[]>(
    DAY_LABELS.map((_, dayOfWeek) => ({ dayOfWeek, openTime: '11:00', closeTime: '21:00' })),
  );
  const [enabledDays, setEnabledDays] = useState<Set<number>>(new Set());
  const [saved, setSaved] = useState(false);

  function toggleDay(dayOfWeek: number) {
    setEnabledDays((prev) => {
      const next = new Set(prev);
      if (next.has(dayOfWeek)) next.delete(dayOfWeek);
      else next.add(dayOfWeek);
      return next;
    });
  }

  function updateHour(dayOfWeek: number, field: 'openTime' | 'closeTime', value: string) {
    setHours((prev) => prev.map((h) => (h.dayOfWeek === dayOfWeek ? { ...h, [field]: value } : h)));
  }

  async function handleSave() {
    const activeHours = hours.filter((h) => enabledDays.has(h.dayOfWeek));
    await apiFetch(`/restaurants/${restaurantId}/business-hours`, {
      method: 'PUT',
      body: JSON.stringify({ hours: activeHours }),
    });
    setSaved(true);
  }

  return (
    <div>
      <h2>영업시간</h2>
      {hours.map((h) => (
        <div key={h.dayOfWeek}>
          <label>
            <input
              type="checkbox"
              checked={enabledDays.has(h.dayOfWeek)}
              onChange={() => toggleDay(h.dayOfWeek)}
              aria-label={`${DAY_LABELS[h.dayOfWeek]}요일 영업`}
            />
            {DAY_LABELS[h.dayOfWeek]}요일
          </label>
          <input
            type="time"
            value={h.openTime}
            onChange={(e) => updateHour(h.dayOfWeek, 'openTime', e.target.value)}
            aria-label={`${DAY_LABELS[h.dayOfWeek]}요일 오픈시간`}
          />
          <input
            type="time"
            value={h.closeTime}
            onChange={(e) => updateHour(h.dayOfWeek, 'closeTime', e.target.value)}
            aria-label={`${DAY_LABELS[h.dayOfWeek]}요일 마감시간`}
          />
        </div>
      ))}
      <button type="button" onClick={handleSave}>
        영업시간 저장
      </button>
      {saved && <p>저장되었습니다</p>}
    </div>
  );
}
```

`apps/web/src/components/StaffInviteForm.tsx`:
```tsx
import { FormEvent, useState } from 'react';
import { apiFetch } from '../api/client';

export default function StaffInviteForm({ restaurantId }: { restaurantId: string }) {
  const [email, setEmail] = useState('');
  const [name, setName] = useState('');
  const [tempPassword, setTempPassword] = useState<string | null>(null);

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    const result = await apiFetch<{ email: string; tempPassword: string }>(
      `/restaurants/${restaurantId}/staff/invite`,
      { method: 'POST', body: JSON.stringify({ email, name }) },
    );
    setTempPassword(result.tempPassword);
  }

  return (
    <form onSubmit={handleSubmit}>
      <h2>직원 초대</h2>
      <label htmlFor="staff-email">이메일</label>
      <input id="staff-email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} />
      <label htmlFor="staff-name">이름</label>
      <input id="staff-name" value={name} onChange={(e) => setName(e.target.value)} />
      <button type="submit">초대</button>
      {tempPassword && <p>임시 비밀번호: {tempPassword} (직원에게 직접 전달하세요)</p>}
    </form>
  );
}
```

- [ ] **Step 5: DashboardPage 구현**

`apps/web/src/pages/DashboardPage.tsx`:
```tsx
import { useEffect, useState } from 'react';
import { Navigate } from 'react-router-dom';
import { apiFetch } from '../api/client';
import { useAuth } from '../auth/AuthContext';
import RestaurantSetupForm from '../components/RestaurantSetupForm';
import BusinessHoursEditor from '../components/BusinessHoursEditor';
import StaffInviteForm from '../components/StaffInviteForm';

interface Restaurant {
  id: string;
  name: string;
  slotIntervalMinutes: number;
  capacityPerSlot: number;
}

export default function DashboardPage() {
  const { token } = useAuth();
  const [restaurant, setRestaurant] = useState<Restaurant | null>(null);
  const [loaded, setLoaded] = useState(false);

  useEffect(() => {
    if (!token) return;
    apiFetch<Restaurant[]>('/restaurants/mine').then((list) => {
      setRestaurant(list[0] ?? null);
      setLoaded(true);
    });
  }, [token]);

  if (!token) {
    return <Navigate to="/login" replace />;
  }

  if (!loaded) {
    return <p>불러오는 중...</p>;
  }

  if (!restaurant) {
    return <RestaurantSetupForm onCreated={setRestaurant} />;
  }

  return (
    <div>
      <h1>{restaurant.name} 관리</h1>
      <BusinessHoursEditor restaurantId={restaurant.id} />
      <StaffInviteForm restaurantId={restaurant.id} />
    </div>
  );
}
```

- [ ] **Step 6: 테스트 실행 → 통과 확인**

Run: `pnpm --filter web test -- DashboardPage`
Expected: PASS (1 passed)

- [ ] **Step 7: App 라우팅에 /dashboard 추가**

`apps/web/src/App.tsx`의 import와 `<Routes>` 내부를 수정한다:

```tsx
import { MemoryRouter, BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { AuthProvider } from './auth/AuthContext';
import LoginPage from './pages/LoginPage';
import SignupPage from './pages/SignupPage';
import DashboardPage from './pages/DashboardPage';

interface AppProps {
  initialEntries?: string[];
}

export default function App({ initialEntries }: AppProps) {
  const Router = initialEntries ? MemoryRouter : BrowserRouter;
  const routerProps = initialEntries ? { initialEntries } : {};

  return (
    <AuthProvider>
      <Router {...routerProps}>
        <Routes>
          <Route path="/login" element={<LoginPage />} />
          <Route path="/signup" element={<SignupPage />} />
          <Route path="/dashboard" element={<DashboardPage />} />
          <Route path="*" element={<Navigate to="/login" replace />} />
        </Routes>
      </Router>
    </AuthProvider>
  );
}
```

- [ ] **Step 8: 전체 프론트엔드 테스트 재실행 확인**

Run: `pnpm --filter web test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 9: Commit**

```bash
git add apps/web/src
git commit -m "feat: add dashboard with restaurant setup, business hours, and staff invite"
```

---

## Task 14: 프론트엔드 — 대시보드 예약 목록 + 수동 등록 + 상태 변경

**Files:**
- Create: `apps/web/src/components/ReservationList.tsx`
- Create: `apps/web/src/components/ReservationList.spec.tsx`
- Create: `apps/web/src/components/ManualReservationForm.tsx`
- Modify: `apps/web/src/pages/DashboardPage.tsx`

**Interfaces:**
- Consumes: `apiFetch` (Task 11)
- Produces: `ReservationList` — `GET /restaurants/:id/reservations` 결과를 표로 렌더링하고 상태 변경 버튼 제공. `ManualReservationForm` — `POST /restaurants/:id/reservations/staff` 호출.

- [ ] **Step 1: ReservationList 실패 테스트 작성**

`apps/web/src/components/ReservationList.spec.tsx`:
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { vi } from 'vitest';
import ReservationList from './ReservationList';

describe('ReservationList', () => {
  it('renders reservations and updates status on button click', async () => {
    const reservations = [
      { id: 'res1', customerName: 'Kim', partySize: 2, reservationTime: '11:00', status: 'CONFIRMED' },
    ];
    const fetchMock = vi.fn().mockImplementation((url: string, options?: RequestInit) => {
      if (options?.method === 'PATCH') {
        return Promise.resolve({
          ok: true,
          headers: { get: () => 'application/json' },
          json: async () => ({ ...reservations[0], status: 'NO_SHOW' }),
        });
      }
      return Promise.resolve({
        ok: true,
        headers: { get: () => 'application/json' },
        json: async () => reservations,
      });
    });
    vi.stubGlobal('fetch', fetchMock);

    render(<ReservationList restaurantId="r1" date="2026-07-06" />);

    await waitFor(() => expect(screen.getByText('Kim')).toBeInTheDocument());
    fireEvent.click(screen.getByRole('button', { name: '노쇼 처리' }));

    await waitFor(() => {
      expect(fetchMock).toHaveBeenCalledWith(
        expect.stringContaining('/reservations/res1/status'),
        expect.objectContaining({ method: 'PATCH' }),
      );
    });
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter web test -- ReservationList`
Expected: FAIL ("Cannot find module './ReservationList'")

- [ ] **Step 3: ReservationList 구현**

`apps/web/src/components/ReservationList.tsx`:
```tsx
import { useEffect, useState } from 'react';
import { apiFetch } from '../api/client';

interface Reservation {
  id: string;
  customerName: string;
  partySize: number;
  reservationTime: string;
  status: string;
}

export default function ReservationList({ restaurantId, date }: { restaurantId: string; date: string }) {
  const [reservations, setReservations] = useState<Reservation[]>([]);

  async function load() {
    const list = await apiFetch<Reservation[]>(`/restaurants/${restaurantId}/reservations?date=${date}`);
    setReservations(list);
  }

  useEffect(() => {
    load();
  }, [restaurantId, date]);

  async function updateStatus(id: string, status: string) {
    await apiFetch(`/restaurants/${restaurantId}/reservations/${id}/status`, {
      method: 'PATCH',
      body: JSON.stringify({ status }),
    });
    await load();
  }

  return (
    <table>
      <thead>
        <tr>
          <th>시간</th>
          <th>고객명</th>
          <th>인원</th>
          <th>상태</th>
          <th>액션</th>
        </tr>
      </thead>
      <tbody>
        {reservations.map((r) => (
          <tr key={r.id}>
            <td>{r.reservationTime}</td>
            <td>{r.customerName}</td>
            <td>{r.partySize}</td>
            <td>{r.status}</td>
            <td>
              <button type="button" onClick={() => updateStatus(r.id, 'NO_SHOW')}>
                노쇼 처리
              </button>
              <button type="button" onClick={() => updateStatus(r.id, 'COMPLETED')}>
                완료 처리
              </button>
              <button type="button" onClick={() => updateStatus(r.id, 'CANCELLED')}>
                취소
              </button>
            </td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `pnpm --filter web test -- ReservationList`
Expected: PASS (1 passed)

- [ ] **Step 5: ManualReservationForm 구현**

`apps/web/src/components/ManualReservationForm.tsx`:
```tsx
import { FormEvent, useState } from 'react';
import { apiFetch } from '../api/client';

export default function ManualReservationForm({
  restaurantId,
  onCreated,
}: {
  restaurantId: string;
  onCreated: () => void;
}) {
  const [customerName, setCustomerName] = useState('');
  const [customerPhone, setCustomerPhone] = useState('');
  const [partySize, setPartySize] = useState(2);
  const [reservationDate, setReservationDate] = useState('');
  const [reservationTime, setReservationTime] = useState('');
  const [error, setError] = useState<string | null>(null);

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    setError(null);
    try {
      await apiFetch(`/restaurants/${restaurantId}/reservations/staff`, {
        method: 'POST',
        body: JSON.stringify({ customerName, customerPhone, partySize, reservationDate, reservationTime }),
      });
      onCreated();
    } catch {
      setError('예약 등록에 실패했습니다 (잔여 좌석을 확인하세요)');
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <h2>전화 예약 등록</h2>
      <label htmlFor="manual-name">고객명</label>
      <input id="manual-name" value={customerName} onChange={(e) => setCustomerName(e.target.value)} />
      <label htmlFor="manual-phone">전화번호</label>
      <input id="manual-phone" value={customerPhone} onChange={(e) => setCustomerPhone(e.target.value)} />
      <label htmlFor="manual-party-size">인원</label>
      <input id="manual-party-size" type="number" value={partySize} onChange={(e) => setPartySize(Number(e.target.value))} />
      <label htmlFor="manual-date">날짜</label>
      <input id="manual-date" type="date" value={reservationDate} onChange={(e) => setReservationDate(e.target.value)} />
      <label htmlFor="manual-time">시간</label>
      <input id="manual-time" type="time" value={reservationTime} onChange={(e) => setReservationTime(e.target.value)} />
      {error && <p role="alert">{error}</p>}
      <button type="submit">등록</button>
    </form>
  );
}
```

- [ ] **Step 6: DashboardPage에 예약 관리 섹션 연결**

`apps/web/src/pages/DashboardPage.tsx`의 import와 반환 JSX를 수정한다:

```tsx
import { useEffect, useState } from 'react';
import { Navigate } from 'react-router-dom';
import { apiFetch } from '../api/client';
import { useAuth } from '../auth/AuthContext';
import RestaurantSetupForm from '../components/RestaurantSetupForm';
import BusinessHoursEditor from '../components/BusinessHoursEditor';
import StaffInviteForm from '../components/StaffInviteForm';
import ReservationList from '../components/ReservationList';
import ManualReservationForm from '../components/ManualReservationForm';

interface Restaurant {
  id: string;
  name: string;
  slotIntervalMinutes: number;
  capacityPerSlot: number;
}

export default function DashboardPage() {
  const { token } = useAuth();
  const [restaurant, setRestaurant] = useState<Restaurant | null>(null);
  const [loaded, setLoaded] = useState(false);
  const [selectedDate, setSelectedDate] = useState(new Date().toISOString().slice(0, 10));
  const [refreshKey, setRefreshKey] = useState(0);

  useEffect(() => {
    if (!token) return;
    apiFetch<Restaurant[]>('/restaurants/mine').then((list) => {
      setRestaurant(list[0] ?? null);
      setLoaded(true);
    });
  }, [token]);

  if (!token) {
    return <Navigate to="/login" replace />;
  }

  if (!loaded) {
    return <p>불러오는 중...</p>;
  }

  if (!restaurant) {
    return <RestaurantSetupForm onCreated={setRestaurant} />;
  }

  return (
    <div>
      <h1>{restaurant.name} 관리</h1>
      <BusinessHoursEditor restaurantId={restaurant.id} />
      <StaffInviteForm restaurantId={restaurant.id} />
      <section>
        <h2>예약 관리</h2>
        <label htmlFor="dashboard-date">날짜</label>
        <input
          id="dashboard-date"
          type="date"
          value={selectedDate}
          onChange={(e) => setSelectedDate(e.target.value)}
        />
        <ManualReservationForm restaurantId={restaurant.id} onCreated={() => setRefreshKey((k) => k + 1)} />
        <ReservationList key={refreshKey} restaurantId={restaurant.id} date={selectedDate} />
      </section>
    </div>
  );
}
```

- [ ] **Step 7: 전체 프론트엔드 테스트 재실행 확인**

Run: `pnpm --filter web test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 8: Commit**

```bash
git add apps/web/src
git commit -m "feat: add reservation list and manual reservation entry to dashboard"
```

---

## Task 15: 프론트엔드 — 고객용 예약 페이지

**Files:**
- Create: `apps/web/src/pages/BookPage.tsx`
- Create: `apps/web/src/pages/BookPage.spec.tsx`
- Create: `apps/web/src/components/AvailabilitySlots.tsx`
- Create: `apps/web/src/components/CustomerReservationForm.tsx`
- Modify: `apps/web/src/App.tsx`

**Interfaces:**
- Consumes: `apiFetch` (Task 11)
- Produces: `/book/:restaurantId` 라우트 (인증 불필요)

- [ ] **Step 1: BookPage 실패 테스트 작성**

`apps/web/src/pages/BookPage.spec.tsx`:
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { MemoryRouter, Route, Routes } from 'react-router-dom';
import { vi } from 'vitest';
import BookPage from './BookPage';

describe('BookPage', () => {
  it('shows available slots for the selected date and books a reservation', async () => {
    const fetchMock = vi.fn().mockImplementation((url: string) => {
      if (url.includes('/public')) {
        return Promise.resolve({
          ok: true,
          headers: { get: () => 'application/json' },
          json: async () => ({ id: 'r1', name: '테스트 식당', slotIntervalMinutes: 30, businessHours: [] }),
        });
      }
      if (url.includes('/availability')) {
        return Promise.resolve({
          ok: true,
          headers: { get: () => 'application/json' },
          json: async () => [{ time: '11:00', remaining: 4 }],
        });
      }
      return Promise.resolve({
        ok: true,
        headers: { get: () => 'application/json' },
        json: async () => ({ reservationCode: 'ABC123' }),
      });
    });
    vi.stubGlobal('fetch', fetchMock);

    render(
      <MemoryRouter initialEntries={['/book/r1']}>
        <Routes>
          <Route path="/book/:restaurantId" element={<BookPage />} />
        </Routes>
      </MemoryRouter>,
    );

    await waitFor(() => expect(screen.getByText('테스트 식당')).toBeInTheDocument());

    fireEvent.change(screen.getByLabelText('날짜 선택'), { target: { value: '2026-07-06' } });
    await waitFor(() => expect(screen.getByText(/11:00/)).toBeInTheDocument());

    fireEvent.click(screen.getByText(/11:00/));
    fireEvent.change(screen.getByLabelText('이름'), { target: { value: '고객' } });
    fireEvent.change(screen.getByLabelText('전화번호'), { target: { value: '010-1111-2222' } });
    fireEvent.change(screen.getByLabelText('인원'), { target: { value: '2' } });
    fireEvent.click(screen.getByRole('button', { name: '예약하기' }));

    await waitFor(() => expect(screen.getByText(/ABC123/)).toBeInTheDocument());
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter web test -- BookPage`
Expected: FAIL ("Cannot find module './BookPage'")

- [ ] **Step 3: AvailabilitySlots, CustomerReservationForm 구현**

`apps/web/src/components/AvailabilitySlots.tsx`:
```tsx
interface Slot {
  time: string;
  remaining: number;
}

export default function AvailabilitySlots({
  slots,
  selectedTime,
  onSelect,
}: {
  slots: Slot[];
  selectedTime: string | null;
  onSelect: (time: string) => void;
}) {
  if (slots.length === 0) {
    return <p>선택한 날짜에는 예약 가능한 시간이 없습니다</p>;
  }

  return (
    <ul>
      {slots.map((slot) => (
        <li key={slot.time}>
          <button
            type="button"
            disabled={slot.remaining <= 0}
            aria-pressed={selectedTime === slot.time}
            onClick={() => onSelect(slot.time)}
          >
            {slot.time} (잔여 {slot.remaining}석)
          </button>
        </li>
      ))}
    </ul>
  );
}
```

`apps/web/src/components/CustomerReservationForm.tsx`:
```tsx
import { FormEvent, useState } from 'react';
import { apiFetch, ApiError } from '../api/client';

export default function CustomerReservationForm({
  restaurantId,
  date,
  time,
  onBooked,
}: {
  restaurantId: string;
  date: string;
  time: string;
  onBooked: (reservationCode: string) => void;
}) {
  const [customerName, setCustomerName] = useState('');
  const [customerPhone, setCustomerPhone] = useState('');
  const [partySize, setPartySize] = useState(2);
  const [error, setError] = useState<string | null>(null);

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    setError(null);
    try {
      const result = await apiFetch<{ reservationCode: string }>(`/restaurants/${restaurantId}/reservations`, {
        method: 'POST',
        body: JSON.stringify({
          customerName,
          customerPhone,
          partySize,
          reservationDate: date,
          reservationTime: time,
        }),
      });
      onBooked(result.reservationCode);
    } catch (err) {
      setError(err instanceof ApiError ? err.message : '예약에 실패했습니다');
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <p>선택한 시간: {time}</p>
      <label htmlFor="customer-name">이름</label>
      <input id="customer-name" value={customerName} onChange={(e) => setCustomerName(e.target.value)} />
      <label htmlFor="customer-phone">전화번호</label>
      <input id="customer-phone" value={customerPhone} onChange={(e) => setCustomerPhone(e.target.value)} />
      <label htmlFor="customer-party-size">인원</label>
      <input
        id="customer-party-size"
        type="number"
        value={partySize}
        onChange={(e) => setPartySize(Number(e.target.value))}
      />
      {error && <p role="alert">{error}</p>}
      <button type="submit">예약하기</button>
    </form>
  );
}
```

- [ ] **Step 4: BookPage 구현**

`apps/web/src/pages/BookPage.tsx`:
```tsx
import { useEffect, useState } from 'react';
import { useParams } from 'react-router-dom';
import { apiFetch } from '../api/client';
import AvailabilitySlots from '../components/AvailabilitySlots';
import CustomerReservationForm from '../components/CustomerReservationForm';

interface RestaurantPublic {
  id: string;
  name: string;
  slotIntervalMinutes: number;
}

interface Slot {
  time: string;
  remaining: number;
}

export default function BookPage() {
  const { restaurantId } = useParams<{ restaurantId: string }>();
  const [restaurant, setRestaurant] = useState<RestaurantPublic | null>(null);
  const [date, setDate] = useState('');
  const [slots, setSlots] = useState<Slot[]>([]);
  const [selectedTime, setSelectedTime] = useState<string | null>(null);
  const [reservationCode, setReservationCode] = useState<string | null>(null);

  useEffect(() => {
    if (!restaurantId) return;
    apiFetch<RestaurantPublic>(`/restaurants/${restaurantId}/public`).then(setRestaurant);
  }, [restaurantId]);

  useEffect(() => {
    if (!restaurantId || !date) return;
    apiFetch<Slot[]>(`/restaurants/${restaurantId}/availability?date=${date}`).then(setSlots);
    setSelectedTime(null);
  }, [restaurantId, date]);

  if (!restaurant) {
    return <p>불러오는 중...</p>;
  }

  if (reservationCode) {
    return (
      <div>
        <h1>예약이 완료되었습니다</h1>
        <p>예약번호: {reservationCode}</p>
        <p>예약번호와 전화번호로 나중에 예약을 조회/취소할 수 있습니다.</p>
      </div>
    );
  }

  return (
    <div>
      <h1>{restaurant.name}</h1>
      <label htmlFor="book-date">날짜 선택</label>
      <input id="book-date" type="date" value={date} onChange={(e) => setDate(e.target.value)} />
      {date && <AvailabilitySlots slots={slots} selectedTime={selectedTime} onSelect={setSelectedTime} />}
      {selectedTime && (
        <CustomerReservationForm
          restaurantId={restaurant.id}
          date={date}
          time={selectedTime}
          onBooked={setReservationCode}
        />
      )}
    </div>
  );
}
```

- [ ] **Step 5: 테스트 실행 → 통과 확인**

Run: `pnpm --filter web test -- BookPage`
Expected: PASS (1 passed)

- [ ] **Step 6: App 라우팅에 /book/:restaurantId 추가**

`apps/web/src/App.tsx`의 import와 `<Routes>` 내부를 수정한다:

```tsx
import { MemoryRouter, BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { AuthProvider } from './auth/AuthContext';
import LoginPage from './pages/LoginPage';
import SignupPage from './pages/SignupPage';
import DashboardPage from './pages/DashboardPage';
import BookPage from './pages/BookPage';

interface AppProps {
  initialEntries?: string[];
}

export default function App({ initialEntries }: AppProps) {
  const Router = initialEntries ? MemoryRouter : BrowserRouter;
  const routerProps = initialEntries ? { initialEntries } : {};

  return (
    <AuthProvider>
      <Router {...routerProps}>
        <Routes>
          <Route path="/login" element={<LoginPage />} />
          <Route path="/signup" element={<SignupPage />} />
          <Route path="/dashboard" element={<DashboardPage />} />
          <Route path="/book/:restaurantId" element={<BookPage />} />
          <Route path="*" element={<Navigate to="/login" replace />} />
        </Routes>
      </Router>
    </AuthProvider>
  );
}
```

- [ ] **Step 7: 전체 프론트엔드 테스트 재실행 확인**

Run: `pnpm --filter web test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 8: Commit**

```bash
git add apps/web/src
git commit -m "feat: add customer-facing booking page"
```

---

## Task 16: 프론트엔드 — 고객용 예약 조회/취소 페이지

**Files:**
- Create: `apps/web/src/pages/LookupPage.tsx`
- Create: `apps/web/src/pages/LookupPage.spec.tsx`
- Modify: `apps/web/src/App.tsx`

**Interfaces:**
- Consumes: `apiFetch` (Task 11), `GET /reservations/lookup`, `PATCH /reservations/cancel` (Task 8)
- Produces: `/book/:restaurantId/lookup` 라우트 (인증 불필요)

- [ ] **Step 1: LookupPage 실패 테스트 작성**

`apps/web/src/pages/LookupPage.spec.tsx`:
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { vi } from 'vitest';
import LookupPage from './LookupPage';

describe('LookupPage', () => {
  it('looks up and cancels a reservation', async () => {
    const fetchMock = vi.fn().mockImplementation((url: string, options?: RequestInit) => {
      if (options?.method === 'PATCH') {
        return Promise.resolve({
          ok: true,
          headers: { get: () => 'application/json' },
          json: async () => ({ id: 'res1', status: 'CANCELLED' }),
        });
      }
      return Promise.resolve({
        ok: true,
        headers: { get: () => 'application/json' },
        json: async () => ({
          id: 'res1',
          customerName: '고객',
          reservationDate: '2026-07-06',
          reservationTime: '11:00',
          partySize: 2,
          status: 'CONFIRMED',
        }),
      });
    });
    vi.stubGlobal('fetch', fetchMock);

    render(
      <MemoryRouter>
        <LookupPage />
      </MemoryRouter>,
    );

    fireEvent.change(screen.getByLabelText('예약번호'), { target: { value: 'ABC123' } });
    fireEvent.change(screen.getByLabelText('전화번호'), { target: { value: '010-1111-2222' } });
    fireEvent.click(screen.getByRole('button', { name: '조회' }));

    await waitFor(() => expect(screen.getByText('고객')).toBeInTheDocument());

    fireEvent.click(screen.getByRole('button', { name: '예약 취소' }));

    await waitFor(() => expect(screen.getByText(/취소되었습니다/)).toBeInTheDocument());
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter web test -- LookupPage`
Expected: FAIL ("Cannot find module './LookupPage'")

- [ ] **Step 3: LookupPage 구현**

`apps/web/src/pages/LookupPage.tsx`:
```tsx
import { FormEvent, useState } from 'react';
import { apiFetch, ApiError } from '../api/client';

interface Reservation {
  id: string;
  customerName: string;
  reservationDate: string;
  reservationTime: string;
  partySize: number;
  status: string;
}

export default function LookupPage() {
  const [code, setCode] = useState('');
  const [phone, setPhone] = useState('');
  const [reservation, setReservation] = useState<Reservation | null>(null);
  const [cancelled, setCancelled] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function handleLookup(e: FormEvent) {
    e.preventDefault();
    setError(null);
    setCancelled(false);
    try {
      const result = await apiFetch<Reservation>(`/reservations/lookup?code=${code}&phone=${phone}`);
      setReservation(result);
    } catch (err) {
      setReservation(null);
      setError(err instanceof ApiError ? err.message : '예약을 찾을 수 없습니다');
    }
  }

  async function handleCancel() {
    await apiFetch(`/reservations/cancel`, {
      method: 'PATCH',
      body: JSON.stringify({ reservationCode: code, customerPhone: phone }),
    });
    setCancelled(true);
  }

  return (
    <div>
      <h1>예약 조회</h1>
      <form onSubmit={handleLookup}>
        <label htmlFor="lookup-code">예약번호</label>
        <input id="lookup-code" value={code} onChange={(e) => setCode(e.target.value)} />
        <label htmlFor="lookup-phone">전화번호</label>
        <input id="lookup-phone" value={phone} onChange={(e) => setPhone(e.target.value)} />
        <button type="submit">조회</button>
      </form>
      {error && <p role="alert">{error}</p>}
      {reservation && !cancelled && (
        <div>
          <p>{reservation.customerName}</p>
          <p>{reservation.reservationDate} {reservation.reservationTime}</p>
          <p>인원: {reservation.partySize}</p>
          <p>상태: {reservation.status}</p>
          <button type="button" onClick={handleCancel}>
            예약 취소
          </button>
        </div>
      )}
      {cancelled && <p>예약이 취소되었습니다</p>}
    </div>
  );
}
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `pnpm --filter web test -- LookupPage`
Expected: PASS (1 passed)

- [ ] **Step 5: App 라우팅에 /book/:restaurantId/lookup 추가**

`apps/web/src/App.tsx`의 import와 `<Routes>` 내부를 수정한다:

```tsx
import { MemoryRouter, BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { AuthProvider } from './auth/AuthContext';
import LoginPage from './pages/LoginPage';
import SignupPage from './pages/SignupPage';
import DashboardPage from './pages/DashboardPage';
import BookPage from './pages/BookPage';
import LookupPage from './pages/LookupPage';

interface AppProps {
  initialEntries?: string[];
}

export default function App({ initialEntries }: AppProps) {
  const Router = initialEntries ? MemoryRouter : BrowserRouter;
  const routerProps = initialEntries ? { initialEntries } : {};

  return (
    <AuthProvider>
      <Router {...routerProps}>
        <Routes>
          <Route path="/login" element={<LoginPage />} />
          <Route path="/signup" element={<SignupPage />} />
          <Route path="/dashboard" element={<DashboardPage />} />
          <Route path="/book/:restaurantId" element={<BookPage />} />
          <Route path="/book/:restaurantId/lookup" element={<LookupPage />} />
          <Route path="*" element={<Navigate to="/login" replace />} />
        </Routes>
      </Router>
    </AuthProvider>
  );
}
```

- [ ] **Step 6: 전체 프론트엔드 테스트 재실행 확인**

Run: `pnpm --filter web test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 7: Commit**

```bash
git add apps/web/src
git commit -m "feat: add customer reservation lookup and cancellation page"
```

---

## Task 17: Nginx 리버스 프록시 설정 (배포 참고 산출물)

**Files:**
- Create: `nginx/nginx.conf`
- Create: `nginx/README.md`

**Interfaces:**
- Consumes: 없음 (독립적인 배포 설정 파일)
- Produces: 없음 (프로덕션 배포 시 참고용 설정)

- [ ] **Step 1: Nginx 설정 파일 작성**

`nginx/nginx.conf`:
```nginx
server {
    listen 80;
    server_name _;

    root /var/www/food-management/web;
    index index.html;

    location /api/ {
        rewrite ^/api/(.*)$ /$1 break;
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

- [ ] **Step 2: 배포 방법 문서화**

`nginx/README.md`:
```markdown
# Nginx 배포 설정

이 설정은 단일 리눅스 서버에 이 프로젝트를 배포할 때 사용하는 참고 구성이다.

## 사전 준비
1. `pnpm --filter web build` 실행 후 생성되는 `apps/web/dist` 폴더를 서버의 `/var/www/food-management/web`에 복사한다.
2. `pnpm --filter api build` 실행 후 `apps/api/dist`를 서버에 복사하고, `pm2` 등으로 `node dist/main.js`를 3000번 포트에서 상시 구동한다.
3. `nginx.conf`를 `/etc/nginx/sites-available/food-management`에 복사하고 `sites-enabled`에 심볼릭 링크를 건다.
4. `nginx -t`로 설정 문법을 검증한 뒤 `systemctl reload nginx`로 반영한다.

## 프론트엔드 API 호출 경로
프로덕션에서는 `apps/web/.env`의 `VITE_API_URL`을 `/api`로 설정해 빌드하면, 브라우저는 같은 오리진의 `/api/*`로 요청하고 Nginx가 이를 백엔드(3000번 포트)로 프록시한다.
```

- [ ] **Step 3: 설정 파일 문법 확인 (Nginx가 로컬에 설치되어 있는 경우)**

Run: `nginx -t -c "$(pwd)/nginx/nginx.conf"` (Nginx가 설치되어 있지 않다면 이 단계는 건너뛰고 파일 내용을 육안으로 검토한다)
Expected: `syntax is ok` / `test is successful` (또는 Nginx 미설치 시 스킵)

- [ ] **Step 4: Commit**

```bash
git add nginx
git commit -m "docs: add nginx reverse proxy config for production deployment"
```

---

## 최종 검증

모든 태스크 완료 후 아래를 실행해 전체가 정상 동작하는지 확인한다.

- [ ] **Step 1: 백엔드 전체 테스트 + e2e**

Run: `pnpm --filter api test && pnpm --filter api test:e2e`
Expected: 모든 테스트 PASS

- [ ] **Step 2: 프론트엔드 전체 테스트**

Run: `pnpm --filter web test`
Expected: 모든 테스트 PASS

- [ ] **Step 3: 두 앱을 동시에 기동해 수동 확인**

Run: `pnpm dev:api` (터미널 1), `pnpm dev:web` (터미널 2)
Expected: `http://localhost:5173/signup`에서 회원가입 → 로그인 → `/dashboard`에서 음식점 등록·영업시간 설정 → `/book/:restaurantId`에서 고객 예약 → `/dashboard`에서 예약 확인 → `/book/:restaurantId/lookup`에서 고객이 예약 취소, 전 과정이 에러 없이 동작
