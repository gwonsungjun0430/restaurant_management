## Note

이 계획은 서브프로젝트 2번(테이블/좌석 관리 + QR 주문 인식 + 혼잡도 표시, "웹기능")을 다룬다. 서브프로젝트 1번(`2026-07-06-restaurant-foundation-reservation.md` — 기반 시스템 + 예약 관리, "앱기능")이 먼저 완료되어 있어야 한다. 이 계획은 Task 1에서 Plan 1이 만든 `server/prisma/schema.prisma`와 `RestaurantsService`/`RestaurantsController`에 필드를 추가하는 것으로 시작한다.

**사용자 확인 대기 중인 가정 (응답 없어 기본값으로 진행):**
- "메인 매출계산화면에서의 결제내역"은 실제 메뉴/POS 없이, 테이블별 occupied/empty 상태 + "결제완료" 이벤트만 기록하는 경량 방식으로 구현한다.
- "직원호출버튼"은 실제 하드웨어 연동 없이, QR 페이지의 소프트웨어 버튼으로 시뮬레이션한다.
- 이 두 가정이 실제 요구사항과 다르면 알려주면 계획을 다시 조정한다.

---

# 음식점 예약 관리 시스템 — 테이블/좌석 관리 + QR 주문 인식 + 혼잡도 표시 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 음식점의 예약석 이외 좌석(테이블)을 좌석 인원수 타입(1/2/4/6인석)별로 등록하고, 고객이 테이블의 QR을 스캔하면 어느 테이블에서 주문이 들어왔는지 식별하며, 후불/선불 결제 완료 또는 직원호출버튼 연속 두 번 클릭으로 테이블을 비우고, 그 상태를 기반으로 실시간 혼잡도(쾌적/보통/혼잡)와 좌석 타입별 개수를 웹에 표시한다.

**Architecture:** Plan 1이 구축한 `server`(NestJS + Prisma + PostgreSQL) 위에 `Table`/`TableOrderEvent`/`PaymentRecord` 모델과 `TablesModule`을 추가한다. `client`에 사장님용 테이블 관리·플로어 현황 페이지와 고객용 QR 랜딩 페이지를 추가하고, Plan 1의 고객 검색/상세 페이지에 혼잡도·좌석수 배지를 붙인다.

**Tech Stack:** Plan 1과 동일(TypeScript strict, NestJS 10.x, Prisma 5.x + PostgreSQL, Jest + Supertest, React 18 + Vite, Vitest + Testing Library) + `qrcode`(서버 사이드 QR 이미지 생성).

## Global Constraints

- Plan 1의 Global Constraints를 모두 상속한다 (TypeScript strict, pnpm, PostgreSQL, Prisma, bcryptjs, Conventional Commits).
- 실제 메뉴/주문 항목/카드 결제 연동은 이 계획의 범위 밖이다 — 테이블 상태(EMPTY/OCCUPIED)와 결제 완료 "이벤트"만 기록한다.
- 실제 IoT/하드웨어 직원호출 장치 연동은 범위 밖이다 — QR 페이지의 소프트웨어 버튼으로 시뮬레이션한다.
- 테이블 배치(플로어 플랜) 좌표/시각화는 범위 밖이다 — 목록/그리드 형태로만 표시한다.
- 혼잡도는 좌석 인원수 기준 점유율로 계산한다: `occupiedSeats / totalSeats`. `<= 0.33` → 쾌적, `<= 0.66` → 보통, 그 외 → 혼잡. `totalSeats === 0`이면 쾌적으로 취급한다.
- 직원호출버튼은 같은 테이블에서 10초 이내에 두 번째로 눌렸을 때만 "테이블 비우기"로 처리하고, 그 외에는 "직원 호출" 신호로만 취급한다.

---

## Task 1: Prisma 스키마 확장 (Table/TableOrderEvent/PaymentRecord) + 결제방식 설정

**Files:**
- Modify: `server/prisma/schema.prisma`
- Modify: `server/.env`
- Create: `server/src/restaurants/dto/set-payment-mode.dto.ts`
- Modify: `server/src/restaurants/restaurants.service.ts`
- Modify: `server/src/restaurants/restaurants.service.spec.ts`
- Modify: `server/src/restaurants/restaurants.controller.ts`

**Interfaces:**
- Consumes: Plan 1의 `Restaurant` 모델, `RestaurantsService`, `RestaurantMemberGuard`/`Roles`
- Produces: `SeatType`(`SEAT_1`|`SEAT_2`|`SEAT_4`|`SEAT_6`), `TableStatus`(`EMPTY`|`OCCUPIED`), `PaymentMode`(`POSTPAID`|`PREPAID`) Prisma enum. `Restaurant.paymentMode` 필드. `RestaurantsService.setPaymentMode(restaurantId, mode): Promise<Restaurant>`. `Table`/`TableOrderEvent`/`PaymentRecord` 모델 — 이후 모든 태스크가 사용.

- [ ] **Step 1: 스키마에 enum과 모델 추가**

`server/prisma/schema.prisma`의 기존 enum 블록 아래에 추가:
```prisma
enum PaymentMode {
  POSTPAID
  PREPAID
}

enum SeatType {
  SEAT_1
  SEAT_2
  SEAT_4
  SEAT_6
}

enum TableStatus {
  EMPTY
  OCCUPIED
}
```

`Restaurant` 모델에 필드 추가 (기존 `capacityPerSlot` 라인 다음):
```prisma
  capacityPerSlot     Int      @default(10)
  paymentMode         PaymentMode @default(POSTPAID)
```

`Restaurant` 모델의 관계 목록에 추가:
```prisma
  tables        Table[]
  paymentRecords PaymentRecord[]
```

파일 하단에 새 모델 추가:
```prisma
model Table {
  id                String      @id @default(uuid())
  restaurantId      String
  tableNumber       Int
  seatType          SeatType
  status            TableStatus @default(EMPTY)
  qrToken           String      @unique
  lastCallPressedAt DateTime?
  createdAt         DateTime    @default(now())

  restaurant   Restaurant       @relation(fields: [restaurantId], references: [id])
  orderEvents  TableOrderEvent[]
  paymentRecords PaymentRecord[]

  @@unique([restaurantId, tableNumber])
}

model TableOrderEvent {
  id        String   @id @default(uuid())
  tableId   String
  note      String?
  createdAt DateTime @default(now())

  table Table @relation(fields: [tableId], references: [id])
}

model PaymentRecord {
  id           String   @id @default(uuid())
  restaurantId String
  tableId      String
  amount       Int?
  createdAt    DateTime @default(now())

  restaurant Restaurant @relation(fields: [restaurantId], references: [id])
  table      Table      @relation(fields: [tableId], references: [id])
}
```

- [ ] **Step 2: 환경 변수 추가**

`server/.env`에 추가:
```
FRONTEND_BASE_URL="http://localhost:5173"
```

- [ ] **Step 3: 마이그레이션 생성**

Run: `pnpm --filter server exec prisma migrate dev --name add_tables_and_payment_mode`
Expected: `server/prisma/migrations/<timestamp>_add_tables_and_payment_mode/migration.sql` 생성, "Your database is now in sync with your schema." 출력

- [ ] **Step 4: RestaurantsService에 결제방식 설정 실패 테스트 추가**

`server/src/restaurants/restaurants.service.spec.ts` 하단에 다음 `describe` 블록 추가:
```typescript
describe('RestaurantsService - payment mode', () => {
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

  it('defaults to POSTPAID and can be switched to PREPAID', async () => {
    const owner = await prisma.user.create({ data: { email: 'pm@t.com', passwordHash: 'x', name: 'O' } });
    const restaurant = await service.create(owner.id, {
      name: 'R',
      address: 'A',
      phone: '02',
      hasParking: false,
      slotIntervalMinutes: 30,
      capacityPerSlot: 10,
    });
    expect(restaurant.paymentMode).toBe('POSTPAID');

    const updated = await service.setPaymentMode(restaurant.id, 'PREPAID');
    expect(updated.paymentMode).toBe('PREPAID');
  });
});
```

- [ ] **Step 5: 테스트 실행 → 실패 확인**

Run: `pnpm --filter server test -- restaurants.service.spec.ts`
Expected: FAIL (`service.setPaymentMode is not a function`)

- [ ] **Step 6: DTO 및 서비스 메서드 추가**

`server/src/restaurants/dto/set-payment-mode.dto.ts`:
```typescript
import { IsIn } from 'class-validator';

export type PaymentModeValue = 'POSTPAID' | 'PREPAID';

export class SetPaymentModeDto {
  @IsIn(['POSTPAID', 'PREPAID'])
  paymentMode: PaymentModeValue;
}
```

`server/src/restaurants/restaurants.service.ts`의 `search` 메서드 뒤에 추가:
```typescript
  async setPaymentMode(restaurantId: string, paymentMode: 'POSTPAID' | 'PREPAID') {
    return this.prisma.restaurant.update({ where: { id: restaurantId }, data: { paymentMode } });
  }
```

- [ ] **Step 7: 테스트 실행 → 통과 확인**

Run: `pnpm --filter server test -- restaurants.service.spec.ts`
Expected: PASS (9 passed)

- [ ] **Step 8: 컨트롤러에 엔드포인트 추가**

`server/src/restaurants/restaurants.controller.ts`에 import 추가:
```typescript
import { SetPaymentModeDto } from './dto/set-payment-mode.dto';
```
`inviteStaff` 메서드 뒤에 추가:
```typescript
  @UseGuards(JwtAuthGuard, RestaurantMemberGuard)
  @Roles('OWNER')
  @Put(':id/payment-mode')
  setPaymentMode(@Param('id') id: string, @Body() dto: SetPaymentModeDto) {
    return this.restaurantsService.setPaymentMode(id, dto.paymentMode);
  }
```

- [ ] **Step 9: 전체 테스트 재실행 확인**

Run: `pnpm --filter server test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 10: Commit**

```bash
git add server/prisma server/.env server/src/restaurants
git commit -m "feat: add Table/TableOrderEvent/PaymentRecord models and restaurant payment mode setting"
```

---

## Task 2: 좌석 용량 계산 유틸 (혼잡도/좌석수 요약)

**Files:**
- Create: `server/src/tables/seat-capacity.util.ts`
- Create: `server/src/tables/seat-capacity.util.spec.ts`

**Interfaces:**
- Consumes: 없음 (순수 함수)
- Produces: `SEAT_CAPACITY: Record<'SEAT_1'|'SEAT_2'|'SEAT_4'|'SEAT_6', number>`, `computeSeatCounts(tables): Record<SeatTypeValue, number>`, `computeCongestion(tables): {totalSeats: number; occupiedSeats: number; level: '쾌적'|'보통'|'혼잡'}`. Task 3의 `TablesService.getSummary`가 사용.

- [ ] **Step 1: 실패 테스트 작성**

`server/src/tables/seat-capacity.util.spec.ts`:
```typescript
import { computeCongestion, computeSeatCounts } from './seat-capacity.util';

describe('computeSeatCounts', () => {
  it('counts tables grouped by seat type', () => {
    const tables = [
      { seatType: 'SEAT_2' as const, status: 'EMPTY' as const },
      { seatType: 'SEAT_2' as const, status: 'OCCUPIED' as const },
      { seatType: 'SEAT_4' as const, status: 'EMPTY' as const },
    ];
    expect(computeSeatCounts(tables)).toEqual({ SEAT_1: 0, SEAT_2: 2, SEAT_4: 1, SEAT_6: 0 });
  });
});

describe('computeCongestion', () => {
  it('returns 쾌적 when no tables exist', () => {
    expect(computeCongestion([])).toEqual({ totalSeats: 0, occupiedSeats: 0, level: '쾌적' });
  });

  it('returns 쾌적 when occupied ratio is at or below 0.33', () => {
    const tables = [
      { seatType: 'SEAT_4' as const, status: 'OCCUPIED' as const },
      { seatType: 'SEAT_4' as const, status: 'EMPTY' as const },
      { seatType: 'SEAT_4' as const, status: 'EMPTY' as const },
    ];
    // occupied 4 / total 12 = 0.33
    expect(computeCongestion(tables)).toEqual({ totalSeats: 12, occupiedSeats: 4, level: '쾌적' });
  });

  it('returns 보통 when occupied ratio is between 0.34 and 0.66', () => {
    const tables = [
      { seatType: 'SEAT_2' as const, status: 'OCCUPIED' as const },
      { seatType: 'SEAT_2' as const, status: 'EMPTY' as const },
    ];
    // occupied 2 / total 4 = 0.5
    expect(computeCongestion(tables)).toEqual({ totalSeats: 4, occupiedSeats: 2, level: '보통' });
  });

  it('returns 혼잡 when occupied ratio is above 0.66', () => {
    const tables = [
      { seatType: 'SEAT_1' as const, status: 'OCCUPIED' as const },
      { seatType: 'SEAT_1' as const, status: 'OCCUPIED' as const },
      { seatType: 'SEAT_1' as const, status: 'EMPTY' as const },
    ];
    // occupied 2 / total 3 = 0.667
    expect(computeCongestion(tables)).toEqual({ totalSeats: 3, occupiedSeats: 2, level: '혼잡' });
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter server test -- seat-capacity.util.spec.ts`
Expected: FAIL ("Cannot find module './seat-capacity.util'")

- [ ] **Step 3: 구현**

`server/src/tables/seat-capacity.util.ts`:
```typescript
export type SeatTypeValue = 'SEAT_1' | 'SEAT_2' | 'SEAT_4' | 'SEAT_6';
export type TableStatusValue = 'EMPTY' | 'OCCUPIED';

export const SEAT_CAPACITY: Record<SeatTypeValue, number> = {
  SEAT_1: 1,
  SEAT_2: 2,
  SEAT_4: 4,
  SEAT_6: 6,
};

interface TableLike {
  seatType: SeatTypeValue;
  status: TableStatusValue;
}

export function computeSeatCounts(tables: TableLike[]): Record<SeatTypeValue, number> {
  const counts: Record<SeatTypeValue, number> = { SEAT_1: 0, SEAT_2: 0, SEAT_4: 0, SEAT_6: 0 };
  for (const table of tables) {
    counts[table.seatType] += 1;
  }
  return counts;
}

export function computeCongestion(tables: TableLike[]): {
  totalSeats: number;
  occupiedSeats: number;
  level: '쾌적' | '보통' | '혼잡';
} {
  const totalSeats = tables.reduce((sum, t) => sum + SEAT_CAPACITY[t.seatType], 0);
  const occupiedSeats = tables
    .filter((t) => t.status === 'OCCUPIED')
    .reduce((sum, t) => sum + SEAT_CAPACITY[t.seatType], 0);

  if (totalSeats === 0) {
    return { totalSeats: 0, occupiedSeats: 0, level: '쾌적' };
  }

  const ratio = occupiedSeats / totalSeats;
  const level = ratio <= 0.33 ? '쾌적' : ratio <= 0.66 ? '보통' : '혼잡';
  return { totalSeats, occupiedSeats, level };
}
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `pnpm --filter server test -- seat-capacity.util.spec.ts`
Expected: PASS (5 passed)

- [ ] **Step 5: Commit**

```bash
git add server/src/tables/seat-capacity.util.ts server/src/tables/seat-capacity.util.spec.ts
git commit -m "feat: add seat count and congestion calculation utility"
```

---

## Task 3: TablesService — 테이블 등록/목록/QR 코드 생성/요약

**Files:**
- Modify: `server/package.json`
- Create: `server/src/tables/dto/create-table.dto.ts`
- Create: `server/src/tables/tables.service.ts`
- Create: `server/src/tables/tables.service.spec.ts`

**Interfaces:**
- Consumes: `PrismaService`(Plan 1 Task 2), `computeSeatCounts`/`computeCongestion`(Task 2)
- Produces: `TablesService.create(restaurantId, dto): Promise<{id, tableNumber, seatType, status, qrToken, qrCodeDataUrl}>`, `TablesService.list(restaurantId): Promise<Table[]>`, `TablesService.findByQrToken(qrToken): Promise<{table, restaurantName}>`, `TablesService.getSummary(restaurantId): Promise<{seatCounts, totalSeats, occupiedSeats, level}>`. Task 4, 5, 6, 7이 이 서비스에 메서드를 추가한다.

- [ ] **Step 1: `qrcode` 패키지 의존성 추가**

`server/package.json`의 `dependencies`에 추가:
```json
    "qrcode": "^1.5.3",
```
`devDependencies`에 추가:
```json
    "@types/qrcode": "^1.5.5",
```

Run: `pnpm install`
Expected: `qrcode`, `@types/qrcode`가 `server/node_modules`에 설치됨

- [ ] **Step 2: DTO 작성**

`server/src/tables/dto/create-table.dto.ts`:
```typescript
import { IsIn, IsInt, Min } from 'class-validator';

export type SeatTypeValue = 'SEAT_1' | 'SEAT_2' | 'SEAT_4' | 'SEAT_6';

export class CreateTableDto {
  @IsInt()
  @Min(1)
  tableNumber: number;

  @IsIn(['SEAT_1', 'SEAT_2', 'SEAT_4', 'SEAT_6'])
  seatType: SeatTypeValue;
}
```

- [ ] **Step 3: 실패 테스트 작성**

`server/src/tables/tables.service.spec.ts`:
```typescript
import { ConflictException, NotFoundException } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import { ConfigModule } from '@nestjs/config';
import { TablesService } from './tables.service';
import { RestaurantsService } from '../restaurants/restaurants.service';
import { PrismaService } from '../prisma/prisma.service';

describe('TablesService', () => {
  let service: TablesService;
  let restaurants: RestaurantsService;
  let prisma: PrismaService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [ConfigModule.forRoot({ isGlobal: true })],
      providers: [TablesService, RestaurantsService, PrismaService],
    }).compile();
    service = moduleRef.get(TablesService);
    restaurants = moduleRef.get(RestaurantsService);
    prisma = moduleRef.get(PrismaService);
    await prisma.onModuleInit();
    await prisma.paymentRecord.deleteMany();
    await prisma.tableOrderEvent.deleteMany();
    await prisma.table.deleteMany();
    await prisma.reservation.deleteMany();
    await prisma.restaurantMember.deleteMany();
    await prisma.restaurant.deleteMany();
    await prisma.user.deleteMany();
  });

  afterEach(async () => {
    await prisma.onModuleDestroy();
  });

  async function setupRestaurant() {
    const owner = await prisma.user.create({ data: { email: `o${Date.now()}${Math.random()}@t.com`, passwordHash: 'x', name: 'O' } });
    return restaurants.create(owner.id, {
      name: 'R',
      address: 'A',
      phone: '02',
      hasParking: false,
      slotIntervalMinutes: 30,
      capacityPerSlot: 10,
    });
  }

  it('creates a table with a unique qrToken and a QR code data URL', async () => {
    const restaurant = await setupRestaurant();
    const table = await service.create(restaurant.id, { tableNumber: 1, seatType: 'SEAT_4' });

    expect(table.tableNumber).toBe(1);
    expect(table.seatType).toBe('SEAT_4');
    expect(table.status).toBe('EMPTY');
    expect(typeof table.qrToken).toBe('string');
    expect(table.qrCodeDataUrl.startsWith('data:image/png;base64,')).toBe(true);
  });

  it('rejects creating a duplicate table number for the same restaurant', async () => {
    const restaurant = await setupRestaurant();
    await service.create(restaurant.id, { tableNumber: 1, seatType: 'SEAT_2' });

    await expect(service.create(restaurant.id, { tableNumber: 1, seatType: 'SEAT_4' })).rejects.toThrow(
      ConflictException,
    );
  });

  it('lists tables for a restaurant ordered by table number', async () => {
    const restaurant = await setupRestaurant();
    await service.create(restaurant.id, { tableNumber: 2, seatType: 'SEAT_2' });
    await service.create(restaurant.id, { tableNumber: 1, seatType: 'SEAT_4' });

    const list = await service.list(restaurant.id);
    expect(list.map((t) => t.tableNumber)).toEqual([1, 2]);
  });

  it('finds a table by its qrToken along with the restaurant name', async () => {
    const restaurant = await setupRestaurant();
    const table = await service.create(restaurant.id, { tableNumber: 1, seatType: 'SEAT_2' });

    const found = await service.findByQrToken(table.qrToken);
    expect(found.table.id).toBe(table.id);
    expect(found.restaurantName).toBe('R');
  });

  it('rejects lookup for an unknown qrToken', async () => {
    await expect(service.findByQrToken('unknown-token')).rejects.toThrow(NotFoundException);
  });

  it('returns a seat/congestion summary for a restaurant', async () => {
    const restaurant = await setupRestaurant();
    await service.create(restaurant.id, { tableNumber: 1, seatType: 'SEAT_4' });
    await service.create(restaurant.id, { tableNumber: 2, seatType: 'SEAT_4' });

    const summary = await service.getSummary(restaurant.id);
    expect(summary.seatCounts).toEqual({ SEAT_1: 0, SEAT_2: 0, SEAT_4: 2, SEAT_6: 0 });
    expect(summary.totalSeats).toBe(8);
    expect(summary.occupiedSeats).toBe(0);
    expect(summary.level).toBe('쾌적');
  });
});
```

- [ ] **Step 4: 테스트 실행 → 실패 확인**

Run: `pnpm --filter server test -- tables.service.spec.ts`
Expected: FAIL ("Cannot find module './tables.service'")

- [ ] **Step 5: TablesService 구현**

`server/src/tables/tables.service.ts`:
```typescript
import { ConflictException, Injectable, NotFoundException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { randomUUID } from 'crypto';
import * as QRCode from 'qrcode';
import { PrismaService } from '../prisma/prisma.service';
import { CreateTableDto } from './dto/create-table.dto';
import { computeCongestion, computeSeatCounts } from './seat-capacity.util';

@Injectable()
export class TablesService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly configService: ConfigService,
  ) {}

  private async buildQrCodeDataUrl(qrToken: string): Promise<string> {
    const frontendBaseUrl = this.configService.get<string>('FRONTEND_BASE_URL', 'http://localhost:5173');
    return QRCode.toDataURL(`${frontendBaseUrl}/table/${qrToken}`);
  }

  async create(restaurantId: string, dto: CreateTableDto) {
    const existing = await this.prisma.table.findUnique({
      where: { restaurantId_tableNumber: { restaurantId, tableNumber: dto.tableNumber } },
    });
    if (existing) {
      throw new ConflictException('Table number already exists for this restaurant');
    }

    const qrToken = randomUUID();
    const table = await this.prisma.table.create({
      data: { restaurantId, tableNumber: dto.tableNumber, seatType: dto.seatType, qrToken },
    });

    return { ...table, qrCodeDataUrl: await this.buildQrCodeDataUrl(qrToken) };
  }

  async list(restaurantId: string) {
    const tables = await this.prisma.table.findMany({
      where: { restaurantId },
      orderBy: { tableNumber: 'asc' },
    });
    return Promise.all(
      tables.map(async (table) => ({ ...table, qrCodeDataUrl: await this.buildQrCodeDataUrl(table.qrToken) })),
    );
  }

  async findByQrToken(qrToken: string) {
    const table = await this.prisma.table.findUnique({
      where: { qrToken },
      include: { restaurant: true },
    });
    if (!table) {
      throw new NotFoundException('Table not found');
    }
    return { table, restaurantName: table.restaurant.name };
  }

  async getSummary(restaurantId: string) {
    const tables = await this.prisma.table.findMany({ where: { restaurantId } });
    const seatCounts = computeSeatCounts(tables);
    const congestion = computeCongestion(tables);
    return { seatCounts, ...congestion };
  }
}
```

- [ ] **Step 6: 테스트 실행 → 통과 확인**

Run: `pnpm --filter server test -- tables.service.spec.ts`
Expected: PASS (6 passed)

- [ ] **Step 7: Commit**

```bash
git add server/package.json server/src/tables
git commit -m "feat: add TablesService with QR code generation and congestion summary"
```

---

## Task 4: TablesController + TablesModule — 등록/목록/요약/QR 조회 엔드포인트

**Files:**
- Create: `server/src/tables/tables.controller.ts`
- Create: `server/src/tables/tables.module.ts`
- Modify: `server/src/app.module.ts`

**Interfaces:**
- Consumes: `TablesService`(Task 3), `JwtAuthGuard`/`RestaurantMemberGuard`/`Roles`(Plan 1)
- Produces: `POST /restaurants/:id/tables`(OWNER), `GET /restaurants/:id/tables`(member), `GET /restaurants/:id/tables/summary`(공개), `GET /tables/qr/:qrToken`(공개). Task 5, 6, 7이 이 컨트롤러에 엔드포인트를 추가한다.

- [ ] **Step 1: 컨트롤러 작성**

`server/src/tables/tables.controller.ts`:
```typescript
import { Body, Controller, Get, Param, Post, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/jwt-auth.guard';
import { RestaurantMemberGuard } from '../restaurants/restaurant-member.guard';
import { Roles } from '../restaurants/roles.decorator';
import { TablesService } from './tables.service';
import { CreateTableDto } from './dto/create-table.dto';

@Controller()
export class TablesController {
  constructor(private readonly tablesService: TablesService) {}

  @UseGuards(JwtAuthGuard, RestaurantMemberGuard)
  @Roles('OWNER')
  @Post('restaurants/:id/tables')
  create(@Param('id') id: string, @Body() dto: CreateTableDto) {
    return this.tablesService.create(id, dto);
  }

  @UseGuards(JwtAuthGuard, RestaurantMemberGuard)
  @Get('restaurants/:id/tables')
  list(@Param('id') id: string) {
    return this.tablesService.list(id);
  }

  @Get('restaurants/:id/tables/summary')
  getSummary(@Param('id') id: string) {
    return this.tablesService.getSummary(id);
  }

  @Get('tables/qr/:qrToken')
  findByQrToken(@Param('qrToken') qrToken: string) {
    return this.tablesService.findByQrToken(qrToken);
  }
}
```

- [ ] **Step 2: 모듈 작성**

`server/src/tables/tables.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { RestaurantsModule } from '../restaurants/restaurants.module';
import { TablesService } from './tables.service';
import { TablesController } from './tables.controller';

@Module({
  imports: [RestaurantsModule],
  providers: [TablesService],
  controllers: [TablesController],
  exports: [TablesService],
})
export class TablesModule {}
```

- [ ] **Step 3: AppModule에 등록**

`server/src/app.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { PrismaModule } from './prisma/prisma.module';
import { AuthModule } from './auth/auth.module';
import { RestaurantsModule } from './restaurants/restaurants.module';
import { ReservationsModule } from './reservations/reservations.module';
import { TablesModule } from './tables/tables.module';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    PrismaModule,
    AuthModule,
    RestaurantsModule,
    ReservationsModule,
    TablesModule,
  ],
})
export class AppModule {}
```

- [ ] **Step 4: 전체 테스트 재실행 확인**

Run: `pnpm --filter server test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 5: Commit**

```bash
git add server/src/tables/tables.controller.ts server/src/tables/tables.module.ts server/src/app.module.ts
git commit -m "feat: expose table creation, listing, summary, and QR lookup endpoints"
```

---

## Task 5: QR 주문 접수 — 테이블에서 주문이 들어왔음을 식별

**Files:**
- Create: `server/src/tables/dto/place-order.dto.ts`
- Modify: `server/src/tables/tables.service.ts`
- Modify: `server/src/tables/tables.service.spec.ts`
- Modify: `server/src/tables/tables.controller.ts`

**Interfaces:**
- Consumes: `TablesService`(Task 3)
- Produces: `TablesService.placeOrder(qrToken, note?): Promise<{tableId, tableNumber, status}>` — 테이블을 `OCCUPIED`로 표시하고 `TableOrderEvent`를 기록. `POST /tables/qr/:qrToken/order`(공개, 고객이 QR 스캔 후 호출).

- [ ] **Step 1: DTO 작성**

`server/src/tables/dto/place-order.dto.ts`:
```typescript
import { IsOptional, IsString, MaxLength } from 'class-validator';

export class PlaceOrderDto {
  @IsOptional()
  @IsString()
  @MaxLength(500)
  note?: string;
}
```

- [ ] **Step 2: 실패 테스트 추가 (`tables.service.spec.ts` 하단)**

```typescript
describe('TablesService - QR order placement', () => {
  let service: TablesService;
  let restaurants: RestaurantsService;
  let prisma: PrismaService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [ConfigModule.forRoot({ isGlobal: true })],
      providers: [TablesService, RestaurantsService, PrismaService],
    }).compile();
    service = moduleRef.get(TablesService);
    restaurants = moduleRef.get(RestaurantsService);
    prisma = moduleRef.get(PrismaService);
    await prisma.onModuleInit();
    await prisma.paymentRecord.deleteMany();
    await prisma.tableOrderEvent.deleteMany();
    await prisma.table.deleteMany();
    await prisma.reservation.deleteMany();
    await prisma.restaurantMember.deleteMany();
    await prisma.restaurant.deleteMany();
    await prisma.user.deleteMany();
  });

  afterEach(async () => {
    await prisma.onModuleDestroy();
  });

  it('marks the table OCCUPIED and logs an order event when a QR order is placed', async () => {
    const owner = await prisma.user.create({ data: { email: `qo${Date.now()}@t.com`, passwordHash: 'x', name: 'O' } });
    const restaurant = await restaurants.create(owner.id, {
      name: 'R',
      address: 'A',
      phone: '02',
      hasParking: false,
      slotIntervalMinutes: 30,
      capacityPerSlot: 10,
    });
    const table = await service.create(restaurant.id, { tableNumber: 1, seatType: 'SEAT_4' });

    const result = await service.placeOrder(table.qrToken, '음료 추가 요청');

    expect(result.status).toBe('OCCUPIED');
    const stored = await prisma.table.findUniqueOrThrow({ where: { id: table.id } });
    expect(stored.status).toBe('OCCUPIED');
    const events = await prisma.tableOrderEvent.findMany({ where: { tableId: table.id } });
    expect(events).toHaveLength(1);
    expect(events[0].note).toBe('음료 추가 요청');
  });
});
```

- [ ] **Step 3: 테스트 실행 → 실패 확인**

Run: `pnpm --filter server test -- tables.service.spec.ts`
Expected: FAIL (`service.placeOrder is not a function`)

- [ ] **Step 4: TablesService에 메서드 추가**

`server/src/tables/tables.service.ts`의 `getSummary` 메서드 뒤에 추가:
```typescript
  async placeOrder(qrToken: string, note?: string) {
    const { table } = await this.findByQrToken(qrToken);

    return this.prisma.$transaction(async (tx) => {
      const updated = await tx.table.update({ where: { id: table.id }, data: { status: 'OCCUPIED' } });
      await tx.tableOrderEvent.create({ data: { tableId: table.id, note: note ?? null } });
      return { tableId: updated.id, tableNumber: updated.tableNumber, status: updated.status };
    });
  }
```

- [ ] **Step 5: 테스트 실행 → 통과 확인**

Run: `pnpm --filter server test -- tables.service.spec.ts`
Expected: PASS (7 passed)

- [ ] **Step 6: 컨트롤러에 엔드포인트 추가**

`server/src/tables/tables.controller.ts`에 import 추가:
```typescript
import { PlaceOrderDto } from './dto/place-order.dto';
```
`findByQrToken` 메서드 뒤에 추가:
```typescript
  @Post('tables/qr/:qrToken/order')
  placeOrder(@Param('qrToken') qrToken: string, @Body() dto: PlaceOrderDto) {
    return this.tablesService.placeOrder(qrToken, dto.note);
  }
```

- [ ] **Step 7: 전체 테스트 재실행 확인**

Run: `pnpm --filter server test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 8: Commit**

```bash
git add server/src/tables
git commit -m "feat: identify incoming orders by table via QR scan"
```

---

## Task 6: 테이블 비우기 — 후불 결제완료 / 선불 청소완료

**Files:**
- Create: `server/src/tables/dto/complete-payment.dto.ts`
- Modify: `server/src/tables/tables.service.ts`
- Modify: `server/src/tables/tables.service.spec.ts`
- Modify: `server/src/tables/tables.controller.ts`

**Interfaces:**
- Consumes: `TablesService`(Task 3, 5)
- Produces: `TablesService.completePayment(restaurantId, tableId, amount?): Promise<Table>` — 후불(POSTPAID) 모드에서 사용, `PaymentRecord` 기록 후 테이블을 `EMPTY`로 전환. `TablesService.clearTable(restaurantId, tableId): Promise<Table>` — 선불(PREPAID) 모드에서 청소 후 비움 처리. `POST /restaurants/:id/tables/:tableId/complete-payment`, `POST /restaurants/:id/tables/:tableId/clear`(둘 다 인증된 staff/owner).

- [ ] **Step 1: DTO 작성**

`server/src/tables/dto/complete-payment.dto.ts`:
```typescript
import { IsInt, IsOptional, Min } from 'class-validator';

export class CompletePaymentDto {
  @IsOptional()
  @IsInt()
  @Min(0)
  amount?: number;
}
```

- [ ] **Step 2: 실패 테스트 추가 (`tables.service.spec.ts` 하단)**

```typescript
describe('TablesService - clearing a table', () => {
  let service: TablesService;
  let restaurants: RestaurantsService;
  let prisma: PrismaService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [ConfigModule.forRoot({ isGlobal: true })],
      providers: [TablesService, RestaurantsService, PrismaService],
    }).compile();
    service = moduleRef.get(TablesService);
    restaurants = moduleRef.get(RestaurantsService);
    prisma = moduleRef.get(PrismaService);
    await prisma.onModuleInit();
    await prisma.paymentRecord.deleteMany();
    await prisma.tableOrderEvent.deleteMany();
    await prisma.table.deleteMany();
    await prisma.reservation.deleteMany();
    await prisma.restaurantMember.deleteMany();
    await prisma.restaurant.deleteMany();
    await prisma.user.deleteMany();
  });

  afterEach(async () => {
    await prisma.onModuleDestroy();
  });

  async function setupOccupiedTable() {
    const owner = await prisma.user.create({ data: { email: `c${Date.now()}${Math.random()}@t.com`, passwordHash: 'x', name: 'O' } });
    const restaurant = await restaurants.create(owner.id, {
      name: 'R',
      address: 'A',
      phone: '02',
      hasParking: false,
      slotIntervalMinutes: 30,
      capacityPerSlot: 10,
    });
    const table = await service.create(restaurant.id, { tableNumber: 1, seatType: 'SEAT_4' });
    await service.placeOrder(table.qrToken);
    return { restaurant, table };
  }

  it('completes a payment, records it, and empties the table (postpaid)', async () => {
    const { restaurant, table } = await setupOccupiedTable();

    const updated = await service.completePayment(restaurant.id, table.id, 45000);

    expect(updated.status).toBe('EMPTY');
    const records = await prisma.paymentRecord.findMany({ where: { tableId: table.id } });
    expect(records).toHaveLength(1);
    expect(records[0].amount).toBe(45000);
  });

  it('clears a table without a payment record (prepaid)', async () => {
    const { restaurant, table } = await setupOccupiedTable();

    const updated = await service.clearTable(restaurant.id, table.id);

    expect(updated.status).toBe('EMPTY');
    const records = await prisma.paymentRecord.findMany({ where: { tableId: table.id } });
    expect(records).toHaveLength(0);
  });

  it('rejects completing payment for a table that belongs to a different restaurant', async () => {
    const { table } = await setupOccupiedTable();
    const otherOwner = await prisma.user.create({ data: { email: `other${Date.now()}@t.com`, passwordHash: 'x', name: 'Other' } });
    const otherRestaurant = await restaurants.create(otherOwner.id, {
      name: 'Other',
      address: 'B',
      phone: '02',
      hasParking: false,
      slotIntervalMinutes: 30,
      capacityPerSlot: 10,
    });

    await expect(service.completePayment(otherRestaurant.id, table.id, 1000)).rejects.toThrow(NotFoundException);
  });

  it('rejects clearing a table that belongs to a different restaurant', async () => {
    const { table } = await setupOccupiedTable();
    const otherOwner = await prisma.user.create({ data: { email: `other2${Date.now()}@t.com`, passwordHash: 'x', name: 'Other' } });
    const otherRestaurant = await restaurants.create(otherOwner.id, {
      name: 'Other2',
      address: 'B',
      phone: '02',
      hasParking: false,
      slotIntervalMinutes: 30,
      capacityPerSlot: 10,
    });

    await expect(service.clearTable(otherRestaurant.id, table.id)).rejects.toThrow(NotFoundException);
  });
});
```

- [ ] **Step 3: 테스트 실행 → 실패 확인**

Run: `pnpm --filter server test -- tables.service.spec.ts`
Expected: FAIL (`service.completePayment is not a function`)

- [ ] **Step 4: TablesService에 메서드 추가**

`server/src/tables/tables.service.ts`의 `placeOrder` 메서드 뒤에 추가:
```typescript
  private async assertTableBelongsToRestaurant(restaurantId: string, tableId: string) {
    const table = await this.prisma.table.findUnique({ where: { id: tableId } });
    if (!table || table.restaurantId !== restaurantId) {
      throw new NotFoundException('Table not found for this restaurant');
    }
    return table;
  }

  async completePayment(restaurantId: string, tableId: string, amount?: number) {
    await this.assertTableBelongsToRestaurant(restaurantId, tableId);
    return this.prisma.$transaction(async (tx) => {
      await tx.paymentRecord.create({ data: { restaurantId, tableId, amount: amount ?? null } });
      return tx.table.update({ where: { id: tableId }, data: { status: 'EMPTY' } });
    });
  }

  async clearTable(restaurantId: string, tableId: string) {
    await this.assertTableBelongsToRestaurant(restaurantId, tableId);
    return this.prisma.table.update({ where: { id: tableId }, data: { status: 'EMPTY' } });
  }
```

- [ ] **Step 5: 테스트 실행 → 통과 확인**

Run: `pnpm --filter server test -- tables.service.spec.ts`
Expected: PASS (11 passed)

- [ ] **Step 6: 컨트롤러에 엔드포인트 추가**

`server/src/tables/tables.controller.ts`에 import 추가:
```typescript
import { CompletePaymentDto } from './dto/complete-payment.dto';
```
`list` 메서드 뒤에 추가:
```typescript
  @UseGuards(JwtAuthGuard, RestaurantMemberGuard)
  @Post('restaurants/:id/tables/:tableId/complete-payment')
  completePayment(
    @Param('id') id: string,
    @Param('tableId') tableId: string,
    @Body() dto: CompletePaymentDto,
  ) {
    return this.tablesService.completePayment(id, tableId, dto.amount);
  }

  @UseGuards(JwtAuthGuard, RestaurantMemberGuard)
  @Post('restaurants/:id/tables/:tableId/clear')
  clearTable(@Param('id') id: string, @Param('tableId') tableId: string) {
    return this.tablesService.clearTable(id, tableId);
  }
```

- [ ] **Step 7: 전체 테스트 재실행 확인**

Run: `pnpm --filter server test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 8: Commit**

```bash
git add server/src/tables
git commit -m "feat: add postpaid payment-completion and prepaid table-clear endpoints"
```

---

## Task 7: 직원호출버튼 — 연속 두 번 클릭 시 테이블 비우기

**Files:**
- Modify: `server/src/tables/tables.service.ts`
- Modify: `server/src/tables/tables.service.spec.ts`
- Modify: `server/src/tables/tables.controller.ts`

**Interfaces:**
- Consumes: `TablesService`(Task 3, 5, 6)
- Produces: `TablesService.pressCallButton(qrToken): Promise<{action: 'CALL_STAFF' | 'TABLE_CLEARED'}>` — 10초 이내 두 번째 클릭이면 `TABLE_CLEARED`(테이블을 `EMPTY`로 전환), 아니면 `CALL_STAFF`(직원 호출 신호, 상태 변경 없음). `POST /tables/qr/:qrToken/call`(공개).

- [ ] **Step 1: 실패 테스트 추가 (`tables.service.spec.ts` 하단)**

```typescript
describe('TablesService - call button double-press', () => {
  let service: TablesService;
  let restaurants: RestaurantsService;
  let prisma: PrismaService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [ConfigModule.forRoot({ isGlobal: true })],
      providers: [TablesService, RestaurantsService, PrismaService],
    }).compile();
    service = moduleRef.get(TablesService);
    restaurants = moduleRef.get(RestaurantsService);
    prisma = moduleRef.get(PrismaService);
    await prisma.onModuleInit();
    await prisma.paymentRecord.deleteMany();
    await prisma.tableOrderEvent.deleteMany();
    await prisma.table.deleteMany();
    await prisma.reservation.deleteMany();
    await prisma.restaurantMember.deleteMany();
    await prisma.restaurant.deleteMany();
    await prisma.user.deleteMany();
  });

  afterEach(async () => {
    await prisma.onModuleDestroy();
  });

  async function setupOccupiedTable() {
    const owner = await prisma.user.create({ data: { email: `cb${Date.now()}${Math.random()}@t.com`, passwordHash: 'x', name: 'O' } });
    const restaurant = await restaurants.create(owner.id, {
      name: 'R',
      address: 'A',
      phone: '02',
      hasParking: false,
      slotIntervalMinutes: 30,
      capacityPerSlot: 10,
    });
    const table = await service.create(restaurant.id, { tableNumber: 1, seatType: 'SEAT_4' });
    await service.placeOrder(table.qrToken);
    return table;
  }

  it('treats a single press as a call-staff signal without changing table status', async () => {
    const table = await setupOccupiedTable();

    const result = await service.pressCallButton(table.qrToken);

    expect(result.action).toBe('CALL_STAFF');
    const stored = await prisma.table.findUniqueOrThrow({ where: { id: table.id } });
    expect(stored.status).toBe('OCCUPIED');
  });

  it('clears the table when pressed twice within 10 seconds', async () => {
    const table = await setupOccupiedTable();

    await service.pressCallButton(table.qrToken);
    const second = await service.pressCallButton(table.qrToken);

    expect(second.action).toBe('TABLE_CLEARED');
    const stored = await prisma.table.findUniqueOrThrow({ where: { id: table.id } });
    expect(stored.status).toBe('EMPTY');
  });

  it('treats a press after the 10 second window as a new call-staff signal', async () => {
    const table = await setupOccupiedTable();
    const elevenSecondsAgo = new Date(Date.now() - 11_000);
    await prisma.table.update({ where: { id: table.id }, data: { lastCallPressedAt: elevenSecondsAgo } });

    const result = await service.pressCallButton(table.qrToken);

    expect(result.action).toBe('CALL_STAFF');
    const stored = await prisma.table.findUniqueOrThrow({ where: { id: table.id } });
    expect(stored.status).toBe('OCCUPIED');
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter server test -- tables.service.spec.ts`
Expected: FAIL (`service.pressCallButton is not a function`)

- [ ] **Step 3: TablesService에 메서드 추가**

`server/src/tables/tables.service.ts` 상단에 상수 추가:
```typescript
const CALL_DOUBLE_PRESS_WINDOW_MS = 10_000;
```

`clearTable` 메서드 뒤에 추가:
```typescript
  async pressCallButton(qrToken: string): Promise<{ action: 'CALL_STAFF' | 'TABLE_CLEARED' }> {
    const { table } = await this.findByQrToken(qrToken);
    const now = new Date();

    const isDoublePress =
      table.lastCallPressedAt !== null &&
      now.getTime() - table.lastCallPressedAt.getTime() <= CALL_DOUBLE_PRESS_WINDOW_MS;

    if (isDoublePress) {
      await this.prisma.table.update({
        where: { id: table.id },
        data: { status: 'EMPTY', lastCallPressedAt: null },
      });
      return { action: 'TABLE_CLEARED' };
    }

    await this.prisma.table.update({ where: { id: table.id }, data: { lastCallPressedAt: now } });
    return { action: 'CALL_STAFF' };
  }
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `pnpm --filter server test -- tables.service.spec.ts`
Expected: PASS (14 passed)

- [ ] **Step 5: 컨트롤러에 엔드포인트 추가**

`server/src/tables/tables.controller.ts`의 `placeOrder` 메서드 뒤에 추가:
```typescript
  @Post('tables/qr/:qrToken/call')
  pressCallButton(@Param('qrToken') qrToken: string) {
    return this.tablesService.pressCallButton(qrToken);
  }
```

- [ ] **Step 6: 전체 테스트 재실행 확인**

Run: `pnpm --filter server test`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 7: Commit**

```bash
git add server/src/tables
git commit -m "feat: add call-button endpoint with double-press table-clear logic"
```

---

## Task 8: Frontend — 대시보드: 테이블/좌석 관리 페이지 (QR 코드 표시)

**Files:**
- Create: `client/src/dashboard/TablesPage.tsx`
- Create: `client/src/dashboard/TablesPage.test.tsx`
- Modify: `client/src/App.tsx`
- Modify: `client/src/dashboard/DashboardLayout.tsx`

**Interfaces:**
- Consumes: `apiClient`(Plan 1 Task 8), `DashboardLayout`(Plan 1 Task 9)
- Produces: `/dashboard/tables` 라우트 — 테이블 등록 폼 + 목록(좌석타입, 상태, QR 이미지).

- [ ] **Step 1: 실패 테스트 작성**

`client/src/dashboard/TablesPage.test.tsx`:
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { describe, expect, it, vi, beforeEach } from 'vitest';
import { TablesPage } from './TablesPage';
import { apiClient } from '../api/client';

vi.mock('../api/client', () => ({
  apiClient: { get: vi.fn(), post: vi.fn() },
}));

describe('TablesPage', () => {
  beforeEach(() => {
    vi.mocked(apiClient.get).mockReset();
    vi.mocked(apiClient.post).mockReset();
  });

  it('lists existing tables with seat type, status, and QR code image', async () => {
    vi.mocked(apiClient.get).mockImplementation((path: string) => {
      if (path === '/restaurants/mine') {
        return Promise.resolve([{ id: 'r1', name: '내 식당' }]);
      }
      return Promise.resolve([
        {
          id: 't1',
          tableNumber: 1,
          seatType: 'SEAT_4',
          status: 'EMPTY',
          qrCodeDataUrl: 'data:image/png;base64,xyz',
        },
      ]);
    });

    render(<TablesPage />);

    await waitFor(() => {
      expect(screen.getByText('1번 테이블')).toBeInTheDocument();
      expect(screen.getByText('4인석')).toBeInTheDocument();
      expect(screen.getByAltText('1번 테이블 QR 코드')).toHaveAttribute('src', 'data:image/png;base64,xyz');
    });
  });

  it('submits a new table with table number and seat type', async () => {
    vi.mocked(apiClient.get).mockImplementation((path: string) => {
      if (path === '/restaurants/mine') {
        return Promise.resolve([{ id: 'r1', name: '내 식당' }]);
      }
      return Promise.resolve([]);
    });
    vi.mocked(apiClient.post).mockResolvedValue({
      id: 't2',
      tableNumber: 2,
      seatType: 'SEAT_2',
      status: 'EMPTY',
      qrCodeDataUrl: 'data:image/png;base64,abc',
    });

    render(<TablesPage />);
    await waitFor(() => screen.getByLabelText('테이블 번호'));

    fireEvent.change(screen.getByLabelText('테이블 번호'), { target: { value: '2' } });
    fireEvent.change(screen.getByLabelText('좌석 타입'), { target: { value: 'SEAT_2' } });
    fireEvent.click(screen.getByRole('button', { name: '테이블 등록' }));

    await waitFor(() => {
      expect(apiClient.post).toHaveBeenCalledWith('/restaurants/r1/tables', {
        tableNumber: 2,
        seatType: 'SEAT_2',
      });
    });
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter client test -- TablesPage.test.tsx`
Expected: FAIL ("Cannot find module './TablesPage'")

- [ ] **Step 3: TablesPage 구현**

`client/src/dashboard/TablesPage.tsx`:
```tsx
import { FormEvent, useEffect, useState } from 'react';
import { apiClient } from '../api/client';

interface Restaurant {
  id: string;
  name: string;
}

type SeatType = 'SEAT_1' | 'SEAT_2' | 'SEAT_4' | 'SEAT_6';

interface RestaurantTable {
  id: string;
  tableNumber: number;
  seatType: SeatType;
  status: 'EMPTY' | 'OCCUPIED';
  qrCodeDataUrl: string;
}

const SEAT_TYPE_LABEL: Record<SeatType, string> = {
  SEAT_1: '1인석',
  SEAT_2: '2인석',
  SEAT_4: '4인석',
  SEAT_6: '6인석',
};

export function TablesPage() {
  const [restaurant, setRestaurant] = useState<Restaurant | null>(null);
  const [tables, setTables] = useState<RestaurantTable[]>([]);
  const [tableNumber, setTableNumber] = useState(1);
  const [seatType, setSeatType] = useState<SeatType>('SEAT_2');

  useEffect(() => {
    apiClient.get<Restaurant[]>('/restaurants/mine').then((list) => {
      if (list.length > 0) setRestaurant(list[0]);
    });
  }, []);

  useEffect(() => {
    if (!restaurant) return;
    apiClient.get<RestaurantTable[]>(`/restaurants/${restaurant.id}/tables`).then(setTables);
  }, [restaurant]);

  async function handleCreate(e: FormEvent) {
    e.preventDefault();
    if (!restaurant) return;
    const created = await apiClient.post<RestaurantTable>(`/restaurants/${restaurant.id}/tables`, {
      tableNumber,
      seatType,
    });
    setTables((prev) => [...prev, created]);
  }

  if (!restaurant) {
    return <p>음식점을 먼저 등록해주세요.</p>;
  }

  return (
    <div>
      <h1>테이블/좌석 관리</h1>
      <form onSubmit={handleCreate}>
        <label htmlFor="table-number">테이블 번호</label>
        <input
          id="table-number"
          type="number"
          value={tableNumber}
          onChange={(e) => setTableNumber(Number(e.target.value))}
        />
        <label htmlFor="seat-type">좌석 타입</label>
        <select id="seat-type" value={seatType} onChange={(e) => setSeatType(e.target.value as SeatType)}>
          <option value="SEAT_1">1인석</option>
          <option value="SEAT_2">2인석</option>
          <option value="SEAT_4">4인석</option>
          <option value="SEAT_6">6인석</option>
        </select>
        <button type="submit">테이블 등록</button>
      </form>

      <ul>
        {tables.map((table) => (
          <li key={table.id}>
            <span>{table.tableNumber}번 테이블</span>
            <span>{SEAT_TYPE_LABEL[table.seatType]}</span>
            <span>{table.status === 'OCCUPIED' ? '사용중' : '비어있음'}</span>
            <img src={table.qrCodeDataUrl} alt={`${table.tableNumber}번 테이블 QR 코드`} />
          </li>
        ))}
      </ul>
    </div>
  );
}
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `pnpm --filter client test -- TablesPage.test.tsx`
Expected: PASS (2 passed)

- [ ] **Step 5: 라우팅 추가**

`client/src/App.tsx`에 import 추가:
```tsx
import { TablesPage } from './dashboard/TablesPage';
```
`Route path="/dashboard"` 블록 안에 추가:
```tsx
            <Route path="tables" element={<TablesPage />} />
```

`client/src/dashboard/DashboardLayout.tsx`의 `nav`에 링크 추가:
```tsx
        <NavLink to="/dashboard/tables">테이블 관리</NavLink>
```

- [ ] **Step 6: Commit**

```bash
git add client/src/dashboard/TablesPage.tsx client/src/dashboard/TablesPage.test.tsx client/src/App.tsx client/src/dashboard/DashboardLayout.tsx
git commit -m "feat: add dashboard table/seat management page with QR codes"
```

---

## Task 9: Frontend — QR 랜딩 페이지 (고객: 주문 접수 + 직원호출)

**Files:**
- Create: `client/src/customer/TableQrPage.tsx`
- Create: `client/src/customer/TableQrPage.test.tsx`
- Modify: `client/src/App.tsx`

**Interfaces:**
- Consumes: `apiClient`(Plan 1 Task 8)
- Produces: `/table/:qrToken` 라우트 — 테이블 정보 확인, "주문 접수" 버튼, "직원호출" 버튼.

- [ ] **Step 1: 실패 테스트 작성**

`client/src/customer/TableQrPage.test.tsx`:
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { MemoryRouter, Route, Routes } from 'react-router-dom';
import { describe, expect, it, vi, beforeEach } from 'vitest';
import { TableQrPage } from './TableQrPage';
import { apiClient } from '../api/client';

vi.mock('../api/client', () => ({
  apiClient: { get: vi.fn(), post: vi.fn() },
}));

function renderAt(qrToken: string) {
  return render(
    <MemoryRouter initialEntries={[`/table/${qrToken}`]}>
      <Routes>
        <Route path="/table/:qrToken" element={<TableQrPage />} />
      </Routes>
    </MemoryRouter>,
  );
}

describe('TableQrPage', () => {
  beforeEach(() => {
    vi.mocked(apiClient.get).mockReset();
    vi.mocked(apiClient.post).mockReset();
  });

  it('shows the restaurant name and table number', async () => {
    vi.mocked(apiClient.get).mockResolvedValue({
      table: { id: 't1', tableNumber: 3 },
      restaurantName: '내 식당',
    });

    renderAt('token-123');

    await waitFor(() => {
      expect(screen.getByText('내 식당')).toBeInTheDocument();
      expect(screen.getByText(/3번 테이블/)).toBeInTheDocument();
    });
  });

  it('places an order when the order button is clicked', async () => {
    vi.mocked(apiClient.get).mockResolvedValue({
      table: { id: 't1', tableNumber: 3 },
      restaurantName: '내 식당',
    });
    vi.mocked(apiClient.post).mockResolvedValue({ tableId: 't1', tableNumber: 3, status: 'OCCUPIED' });

    renderAt('token-123');
    await waitFor(() => screen.getByText('내 식당'));

    fireEvent.click(screen.getByRole('button', { name: '주문 접수' }));

    await waitFor(() => {
      expect(apiClient.post).toHaveBeenCalledWith('/tables/qr/token-123/order', {});
      expect(screen.getByText('주문이 접수되었습니다')).toBeInTheDocument();
    });
  });

  it('calls staff when the call button is clicked', async () => {
    vi.mocked(apiClient.get).mockResolvedValue({
      table: { id: 't1', tableNumber: 3 },
      restaurantName: '내 식당',
    });
    vi.mocked(apiClient.post).mockResolvedValue({ action: 'CALL_STAFF' });

    renderAt('token-123');
    await waitFor(() => screen.getByText('내 식당'));

    fireEvent.click(screen.getByRole('button', { name: '직원호출' }));

    await waitFor(() => {
      expect(apiClient.post).toHaveBeenCalledWith('/tables/qr/token-123/call');
      expect(screen.getByText('직원을 호출했습니다')).toBeInTheDocument();
    });
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter client test -- TableQrPage.test.tsx`
Expected: FAIL ("Cannot find module './TableQrPage'")

- [ ] **Step 3: TableQrPage 구현**

`client/src/customer/TableQrPage.tsx`:
```tsx
import { useEffect, useState } from 'react';
import { useParams } from 'react-router-dom';
import { apiClient } from '../api/client';

interface TableQrInfo {
  table: { id: string; tableNumber: number };
  restaurantName: string;
}

export function TableQrPage() {
  const { qrToken } = useParams<{ qrToken: string }>();
  const [info, setInfo] = useState<TableQrInfo | null>(null);
  const [message, setMessage] = useState<string | null>(null);

  useEffect(() => {
    if (!qrToken) return;
    apiClient.get<TableQrInfo>(`/tables/qr/${qrToken}`).then(setInfo);
  }, [qrToken]);

  async function handleOrder() {
    if (!qrToken) return;
    await apiClient.post(`/tables/qr/${qrToken}/order`, {});
    setMessage('주문이 접수되었습니다');
  }

  async function handleCall() {
    if (!qrToken) return;
    const result = await apiClient.post<{ action: 'CALL_STAFF' | 'TABLE_CLEARED' }>(`/tables/qr/${qrToken}/call`);
    setMessage(result.action === 'TABLE_CLEARED' ? '테이블을 비웠습니다' : '직원을 호출했습니다');
  }

  if (!info) {
    return <p>불러오는 중...</p>;
  }

  return (
    <div>
      <h1>{info.restaurantName}</h1>
      <p>{info.table.tableNumber}번 테이블</p>
      <button onClick={handleOrder}>주문 접수</button>
      <button onClick={handleCall}>직원호출</button>
      {message && <p>{message}</p>}
    </div>
  );
}
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `pnpm --filter client test -- TableQrPage.test.tsx`
Expected: PASS (3 passed)

- [ ] **Step 5: 라우팅 추가**

`client/src/App.tsx`에 import 추가:
```tsx
import { TableQrPage } from './customer/TableQrPage';
```
Route 추가:
```tsx
          <Route path="/table/:qrToken" element={<TableQrPage />} />
```

- [ ] **Step 6: Commit**

```bash
git add client/src/customer/TableQrPage.tsx client/src/customer/TableQrPage.test.tsx client/src/App.tsx
git commit -m "feat: add customer QR landing page for order placement and staff call"
```

---

## Task 10: Frontend — 대시보드: 플로어 현황 페이지 (혼잡도 + 좌석수 요약 + 테이블 그리드)

**Files:**
- Create: `client/src/dashboard/FloorStatusPage.tsx`
- Create: `client/src/dashboard/FloorStatusPage.test.tsx`
- Modify: `client/src/App.tsx`
- Modify: `client/src/dashboard/DashboardLayout.tsx`

**Interfaces:**
- Consumes: `apiClient`(Plan 1 Task 8), `DashboardLayout`(Plan 1 Task 9)
- Produces: `/dashboard/floor` 라우트 — 혼잡도 배지, 좌석수 요약(1인석/2인석/4인석/6인석), 테이블별 상태 + 결제완료/비우기 버튼.

- [ ] **Step 1: 실패 테스트 작성**

`client/src/dashboard/FloorStatusPage.test.tsx`:
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { describe, expect, it, vi, beforeEach } from 'vitest';
import { FloorStatusPage } from './FloorStatusPage';
import { apiClient } from '../api/client';

vi.mock('../api/client', () => ({
  apiClient: { get: vi.fn(), post: vi.fn() },
}));

function mockGet() {
  return (path: string) => {
    if (path === '/restaurants/mine') {
      return Promise.resolve([{ id: 'r1', name: '내 식당', paymentMode: 'POSTPAID' }]);
    }
    if (path === '/restaurants/r1/tables/summary') {
      return Promise.resolve({
        seatCounts: { SEAT_1: 0, SEAT_2: 1, SEAT_4: 1, SEAT_6: 0 },
        totalSeats: 6,
        occupiedSeats: 2,
        level: '보통',
      });
    }
    if (path === '/restaurants/r1/tables') {
      return Promise.resolve([
        { id: 't1', tableNumber: 1, seatType: 'SEAT_2', status: 'OCCUPIED' },
        { id: 't2', tableNumber: 2, seatType: 'SEAT_4', status: 'EMPTY' },
      ]);
    }
    return Promise.resolve(null);
  };
}

describe('FloorStatusPage', () => {
  beforeEach(() => {
    vi.mocked(apiClient.get).mockReset();
    vi.mocked(apiClient.post).mockReset();
  });

  it('shows the congestion level and seat count summary', async () => {
    vi.mocked(apiClient.get).mockImplementation(mockGet());

    render(<FloorStatusPage />);

    await waitFor(() => {
      expect(screen.getByText('보통')).toBeInTheDocument();
      expect(screen.getByText('1인석: 0개')).toBeInTheDocument();
      expect(screen.getByText('2인석: 1개')).toBeInTheDocument();
      expect(screen.getByText('4인석: 1개')).toBeInTheDocument();
      expect(screen.getByText('6인석: 0개')).toBeInTheDocument();
    });
  });

  it('completes payment for an occupied table in postpaid mode', async () => {
    vi.mocked(apiClient.get).mockImplementation(mockGet());
    vi.mocked(apiClient.post).mockResolvedValue({ id: 't1', status: 'EMPTY' });

    render(<FloorStatusPage />);
    await waitFor(() => screen.getByText('1번 테이블'));

    fireEvent.click(screen.getByRole('button', { name: '결제완료' }));

    await waitFor(() => {
      expect(apiClient.post).toHaveBeenCalledWith('/restaurants/r1/tables/t1/complete-payment', {});
    });
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter client test -- FloorStatusPage.test.tsx`
Expected: FAIL ("Cannot find module './FloorStatusPage'")

- [ ] **Step 3: FloorStatusPage 구현**

`client/src/dashboard/FloorStatusPage.tsx`:
```tsx
import { useEffect, useState } from 'react';
import { apiClient } from '../api/client';

interface Restaurant {
  id: string;
  name: string;
  paymentMode: 'POSTPAID' | 'PREPAID';
}

type SeatType = 'SEAT_1' | 'SEAT_2' | 'SEAT_4' | 'SEAT_6';

interface Summary {
  seatCounts: Record<SeatType, number>;
  totalSeats: number;
  occupiedSeats: number;
  level: '쾌적' | '보통' | '혼잡';
}

interface RestaurantTable {
  id: string;
  tableNumber: number;
  seatType: SeatType;
  status: 'EMPTY' | 'OCCUPIED';
}

const SEAT_TYPE_ORDER: SeatType[] = ['SEAT_1', 'SEAT_2', 'SEAT_4', 'SEAT_6'];
const SEAT_TYPE_LABEL: Record<SeatType, string> = {
  SEAT_1: '1인석',
  SEAT_2: '2인석',
  SEAT_4: '4인석',
  SEAT_6: '6인석',
};

export function FloorStatusPage() {
  const [restaurant, setRestaurant] = useState<Restaurant | null>(null);
  const [summary, setSummary] = useState<Summary | null>(null);
  const [tables, setTables] = useState<RestaurantTable[]>([]);

  useEffect(() => {
    apiClient.get<Restaurant[]>('/restaurants/mine').then((list) => {
      if (list.length > 0) setRestaurant(list[0]);
    });
  }, []);

  useEffect(() => {
    if (!restaurant) return;
    apiClient.get<Summary>(`/restaurants/${restaurant.id}/tables/summary`).then(setSummary);
    apiClient.get<RestaurantTable[]>(`/restaurants/${restaurant.id}/tables`).then(setTables);
  }, [restaurant]);

  async function handleFreeTable(tableId: string) {
    if (!restaurant) return;
    const path =
      restaurant.paymentMode === 'POSTPAID'
        ? `/restaurants/${restaurant.id}/tables/${tableId}/complete-payment`
        : `/restaurants/${restaurant.id}/tables/${tableId}/clear`;
    await apiClient.post(path, {});
    setTables((prev) => prev.map((t) => (t.id === tableId ? { ...t, status: 'EMPTY' } : t)));
  }

  if (!restaurant || !summary) {
    return <p>불러오는 중...</p>;
  }

  return (
    <div>
      <h1>플로어 현황</h1>
      <p>혼잡도: {summary.level}</p>
      <ul>
        {SEAT_TYPE_ORDER.map((seatType) => (
          <li key={seatType}>
            {SEAT_TYPE_LABEL[seatType]}: {summary.seatCounts[seatType]}개
          </li>
        ))}
      </ul>

      <ul>
        {tables.map((table) => (
          <li key={table.id}>
            <span>{table.tableNumber}번 테이블</span>
            <span>{SEAT_TYPE_LABEL[table.seatType]}</span>
            <span>{table.status === 'OCCUPIED' ? '사용중' : '비어있음'}</span>
            {table.status === 'OCCUPIED' && (
              <button onClick={() => handleFreeTable(table.id)}>
                {restaurant.paymentMode === 'POSTPAID' ? '결제완료' : '테이블 비우기'}
              </button>
            )}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `pnpm --filter client test -- FloorStatusPage.test.tsx`
Expected: PASS (2 passed)

- [ ] **Step 5: 라우팅 추가**

`client/src/App.tsx`에 import 추가:
```tsx
import { FloorStatusPage } from './dashboard/FloorStatusPage';
```
`Route path="/dashboard"` 블록 안에 추가:
```tsx
            <Route path="floor" element={<FloorStatusPage />} />
```

`client/src/dashboard/DashboardLayout.tsx`의 `nav`에 링크 추가:
```tsx
        <NavLink to="/dashboard/floor">플로어 현황</NavLink>
```

- [ ] **Step 6: Commit**

```bash
git add client/src/dashboard/FloorStatusPage.tsx client/src/dashboard/FloorStatusPage.test.tsx client/src/App.tsx client/src/dashboard/DashboardLayout.tsx
git commit -m "feat: add dashboard floor status page with congestion and seat summary"
```

---

## Task 11: Frontend — 고객 검색/상세 페이지에 혼잡도·좌석수 배지 추가

**Files:**
- Modify: `client/src/customer/SearchPage.tsx`
- Modify: `client/src/customer/SearchPage.test.tsx`
- Modify: `client/src/customer/RestaurantBookingPage.tsx`
- Modify: `client/src/customer/RestaurantBookingPage.test.tsx`

**Interfaces:**
- Consumes: `apiClient`(Plan 1 Task 8), `GET /restaurants/:id/tables/summary`(Task 4)
- Produces: 검색 결과 목록과 예약 상세 페이지에 혼잡도 배지("쾌적"/"보통"/"혼잡")와 좌석수 요약이 표시됨.

- [ ] **Step 1: SearchPage 테스트에 혼잡도 표시 케이스 추가**

`client/src/customer/SearchPage.test.tsx` 하단에 추가:
```tsx
it('shows the congestion badge for each search result', async () => {
  vi.mocked(apiClient.get).mockImplementation((path: string) => {
    if (path.startsWith('/restaurants/search')) {
      return Promise.resolve([
        { id: 'r1', name: '강남 파스타집', address: '서울시 강남구', phone: '02', hasParking: true },
      ]);
    }
    if (path === '/restaurants/r1/tables/summary') {
      return Promise.resolve({
        seatCounts: { SEAT_1: 0, SEAT_2: 2, SEAT_4: 0, SEAT_6: 0 },
        totalSeats: 4,
        occupiedSeats: 3,
        level: '혼잡',
      });
    }
    return Promise.resolve(null);
  });

  render(
    <MemoryRouter>
      <SearchPage />
    </MemoryRouter>,
  );

  fireEvent.change(screen.getByLabelText('음식점 검색'), { target: { value: '파스타' } });
  fireEvent.click(screen.getByRole('button', { name: '검색' }));

  await waitFor(() => {
    expect(screen.getByText('혼잡')).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `pnpm --filter client test -- SearchPage.test.tsx`
Expected: FAIL (혼잡도 텍스트를 찾을 수 없음)

- [ ] **Step 3: SearchPage에 혼잡도 조회 및 표시 추가**

`client/src/customer/SearchPage.tsx` 전체를 다음으로 교체:
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

interface CongestionSummary {
  level: '쾌적' | '보통' | '혼잡';
}

export function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<RestaurantSummary[]>([]);
  const [congestionByRestaurantId, setCongestionByRestaurantId] = useState<Record<string, CongestionSummary>>({});

  async function handleSearch(e: FormEvent) {
    e.preventDefault();
    const results = await apiClient.get<RestaurantSummary[]>(
      `/restaurants/search?q=${encodeURIComponent(query)}`,
    );
    setResults(results);

    const entries = await Promise.all(
      results.map(async (r) => {
        const summary = await apiClient.get<CongestionSummary>(`/restaurants/${r.id}/tables/summary`);
        return [r.id, summary] as const;
      }),
    );
    setCongestionByRestaurantId(Object.fromEntries(entries));
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
            {congestionByRestaurantId[r.id] && <span>{congestionByRestaurantId[r.id].level}</span>}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `pnpm --filter client test -- SearchPage.test.tsx`
Expected: PASS (2 passed)

- [ ] **Step 5: RestaurantBookingPage 테스트에 좌석수 요약 표시 케이스 추가**

`client/src/customer/RestaurantBookingPage.test.tsx` 하단에 추가:
```tsx
it('shows the seat count summary and congestion level', async () => {
  vi.mocked(apiClient.get).mockImplementation((path: string) => {
    if (path.includes('/public')) {
      return Promise.resolve({ id: 'r1', name: '내 식당', address: 'A', phone: '02', hasParking: true });
    }
    if (path.includes('/tables/summary')) {
      return Promise.resolve({
        seatCounts: { SEAT_1: 1, SEAT_2: 2, SEAT_4: 1, SEAT_6: 0 },
        totalSeats: 9,
        occupiedSeats: 1,
        level: '쾌적',
      });
    }
    return Promise.resolve([]);
  });

  renderAt('r1');

  await waitFor(() => {
    expect(screen.getByText('쾌적')).toBeInTheDocument();
    expect(screen.getByText('1인석: 1개')).toBeInTheDocument();
    expect(screen.getByText('2인석: 2개')).toBeInTheDocument();
    expect(screen.getByText('4인석: 1개')).toBeInTheDocument();
    expect(screen.getByText('6인석: 0개')).toBeInTheDocument();
  });
});
```

- [ ] **Step 6: 테스트 실행 → 실패 확인**

Run: `pnpm --filter client test -- RestaurantBookingPage.test.tsx`
Expected: FAIL (좌석수/혼잡도 텍스트를 찾을 수 없음)

- [ ] **Step 7: RestaurantBookingPage에 좌석수/혼잡도 표시 추가**

`client/src/customer/RestaurantBookingPage.tsx` 상단에 타입 및 상수 추가:
```tsx
type SeatType = 'SEAT_1' | 'SEAT_2' | 'SEAT_4' | 'SEAT_6';

interface CongestionSummary {
  seatCounts: Record<SeatType, number>;
  level: '쾌적' | '보통' | '혼잡';
}

const SEAT_TYPE_ORDER: SeatType[] = ['SEAT_1', 'SEAT_2', 'SEAT_4', 'SEAT_6'];
const SEAT_TYPE_LABEL: Record<SeatType, string> = {
  SEAT_1: '1인석',
  SEAT_2: '2인석',
  SEAT_4: '4인석',
  SEAT_6: '6인석',
};
```

`RestaurantBookingPage` 함수 내부에 상태와 조회 로직 추가 (`restaurant` state 선언 뒤):
```tsx
  const [congestion, setCongestion] = useState<CongestionSummary | null>(null);
```
`useEffect`들 사이에 추가:
```tsx
  useEffect(() => {
    if (!id) return;
    apiClient.get<CongestionSummary>(`/restaurants/${id}/tables/summary`).then(setCongestion);
  }, [id]);
```

`hasParking` 표시하는 `<p>` 태그 뒤에 추가:
```tsx
      {congestion && (
        <div>
          <p>혼잡도: {congestion.level}</p>
          <ul>
            {SEAT_TYPE_ORDER.map((seatType) => (
              <li key={seatType}>
                {SEAT_TYPE_LABEL[seatType]}: {congestion.seatCounts[seatType]}개
              </li>
            ))}
          </ul>
        </div>
      )}
```

- [ ] **Step 8: 테스트 실행 → 통과 확인**

Run: `pnpm --filter client test -- RestaurantBookingPage.test.tsx`
Expected: PASS (3 passed)

- [ ] **Step 9: 전체 프론트엔드 테스트 재실행 확인**

Run: `pnpm test:web`
Expected: PASS (모든 스펙 통과)

- [ ] **Step 10: Commit**

```bash
git add client/src/customer
git commit -m "feat: show congestion level and seat count summary on customer search and booking pages"
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
Expected:
1. `/dashboard/tables`에서 테이블 등록(좌석타입별) 후 QR 이미지 표시 확인
2. QR 이미지가 가리키는 `/table/:qrToken` 페이지에서 "주문 접수" 클릭 → 대시보드 `/dashboard/floor`에서 해당 테이블이 "사용중"으로 표시되는지 확인
3. `/dashboard/floor`에서 "결제완료"(POSTPAID) 또는 "테이블 비우기"(PREPAID) 클릭 후 테이블이 "비어있음"으로 바뀌는지 확인
4. `/table/:qrToken`에서 "직원호출"을 10초 이내 두 번 클릭하면 테이블이 자동으로 비워지는지 확인
5. `/search`, `/restaurants/:id`에서 혼잡도 배지와 좌석수 요약이 표시되는지 확인
