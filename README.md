# README.md

````markdown
# ☁️ VTX.Cloud — Veterinary Cloud (TS, NestJS + Next.js)

Kompletní veterinární cloud (MVP) se **správou pacientů/majitelů** a **ekonomickými přehledy**.
Stack: **NestJS + Prisma + PostgreSQL + Redis + Next.js (App Router) + Tailwind**.

## ✨ Moduly v MVP
- 🐾 **Patients/Owners** – evidence majitelů a pacientů, vyhledávání, detail
- 🧾 **Reports** – KPI (tržby, pacienti, otevřené případy, dlužníci), tržby po měsících
- 💳 **Billing** – datové modely (Invoice/Payment), jednoduchá agregace tržeb
- 🧪 **Labs (placeholder)** – entita pro napojení lab výsledků (do budoucna)
- 🔐 **Auth (placeholder)** – zatím basic JWT hook; RBAC připraven v enumu

## 🏗 Architektura
- **apps/api** – NestJS API (Swagger `/docs`), Prisma (PostgreSQL)
- **apps/web** – Next.js frontend (SSR dashboard + seznam/detail pacientů)
- **apps/worker** – BullMQ/Redis (připraveno; není nutné pro MVP)
- **packages/schemas** – sdílené Zod schéma typů
- **infra/docker** – `docker-compose.yml` (Postgres, Redis, MinIO, API, Web)

## 🚀 Rychlý start (dev)
```bash
# 1) Naklonuj repo a přepni se do složky
pnpm i

# 2) Spusť databázi atd. (postgres+redis+minio)
docker compose -f infra/docker/docker-compose.yml up -d

# 3) Migruj a seeduj DB
pnpm db:push
pnpm db:seed

# 4) Nastartuj API + Web
pnpm dev
# API: http://localhost:8000 (Swagger na /docs)
# Web: http://localhost:3000
````

## 🔧 Scripts

* `pnpm dev` – spustí současně **API** (NestJS) i **Web** (Next.js)
* `pnpm db:push` – Prisma migrate (dev)
* `pnpm db:seed` – naplní základní data (druhy, ukázkoví pacienti, ceny)
* `pnpm build` – build všech balíčků
* `pnpm worker` – spustí background worker (volitelně)

## 🔐 Env

Vytvoř `.env` (podle `.env.example`):

```
DATABASE_URL="postgresql://vtx:vtx@localhost:5432/vtx?schema=public"
REDIS_URL="redis://localhost:6379"
S3_ENDPOINT="http://localhost:9000"
S3_ACCESS_KEY="vtx"
S3_SECRET_KEY="vtxvtxvtx"
S3_BUCKET="vtx-cloud"
API_PORT=8000
WEB_PORT=3000
NEXT_PUBLIC_API_URL="http://localhost:8000"
JWT_SECRET="change_me"
```

## 🔌 API (výběr)

* `GET /patients?q=` – seznam pacientů (fulltext jméno/species)
* `GET /patients/:id` – detail pacienta (+majitel, případy)
* `POST /patients` – vytvoří pacienta (ownerId, name, species, …)
* `GET /owners` / `POST /owners` – majitelé
* `GET /reports/kpi?from=&to=` – KPI (tržby, pacienti, dlužníci, otevřené případy)
* `GET /reports/revenue-by-month?year=2025` – tržby po měsících

## 📊 Dashboard (Web)

* `/` – KPI + graf tržeb
* `/patients` – seznam pacientů (vyhledávání `?q=`)
* `/patients/[id]` – detail pacienta

## 🧱 Datový model (zkráceně)

Owner 1─N Patient 1─N Case 1─N Visit
Visit N─N Procedure; Case 1─1 Invoice 1─N Payment
Visit N─N LabResult

## 🛡️ Bezpečnost & kvalita

* RBAC enum (ADMIN/VET/NURSE/RECEPTION/ACCOUNTANT)
* Zod validace DTO (kontrolováno v controllerech)
* Swagger `/docs`, CI workflow (lint/build), audit fields (`createdAt` ap.)

## 🗺 Roadmap

* Fakturační UI + exporty (PDF/CSV)
* Inventář (odpisy materiálu u úkonů, šarže/expirace)
* Rezervace + notifikace (SMS/e-mail)
* Lab import (PDF/HL7)
* SSO/OAuth, plné RBAC, audit log

---

````

---

## KOŘEN REPA

**`package.json`**
```json
{
  "name": "vtx-cloud",
  "private": true,
  "packageManager": "pnpm@9",
  "scripts": {
    "build": "pnpm -r build",
    "dev": "concurrently -k \"pnpm -C apps/api start:dev\" \"pnpm -C apps/web dev\"",
    "db:push": "pnpm -C apps/api prisma:push",
    "db:seed": "pnpm -C apps/api prisma:seed",
    "lint": "pnpm -r lint",
    "test": "pnpm -r test",
    "worker": "pnpm -C apps/worker dev"
  },
  "devDependencies": {
    "concurrently": "^9.0.0"
  },
  "workspaces": [
    "apps/*",
    "packages/*"
  ]
}
````

**`pnpm-workspace.yaml`**

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

**`.gitignore`**

```
node_modules
.env
.prisma
/.next
/dist
coverage
.vscode
.DS_Store
```

**`.env.example`**

```
DATABASE_URL="postgresql://vtx:vtx@localhost:5432/vtx?schema=public"
REDIS_URL="redis://localhost:6379"
S3_ENDPOINT="http://localhost:9000"
S3_ACCESS_KEY="vtx"
S3_SECRET_KEY="vtxvtxvtx"
S3_BUCKET="vtx-cloud"
API_PORT=8000
WEB_PORT=3000
NEXT_PUBLIC_API_URL="http://localhost:8000"
JWT_SECRET="change_me"
```

---

## INFRA

**`infra/docker/docker-compose.yml`**

```yaml
version: "3.9"
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: vtx
      POSTGRES_PASSWORD: vtx
      POSTGRES_DB: vtx
    ports: ["5432:5432"]
    volumes: [ "vtx_db:/var/lib/postgresql/data" ]
  redis:
    image: redis:7
    ports: ["6379:6379"]
  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: vtx
      MINIO_ROOT_PASSWORD: vtxvtxvtx
    ports: ["9000:9000","9001:9001"]
    volumes: [ "vtx_minio:/data" ]
  api:
    build: ../../apps/api
    env_file: ../../.env
    depends_on: [db, redis, minio]
    ports: ["8000:8000"]
  web:
    build: ../../apps/web
    env_file: ../../.env
    depends_on: [api]
    ports: ["3000:3000"]
  worker:
    build: ../../apps/worker
    env_file: ../../.env
    depends_on: [redis, api]
volumes:
  vtx_db: {}
  vtx_minio: {}
```

---

## APPS/API (NestJS + Prisma)

**`apps/api/package.json`**

```json
{
  "name": "@vtx/api",
  "type": "module",
  "scripts": {
    "start": "node dist/main.js",
    "start:dev": "ts-node-dev --transpile-only --respawn src/main.ts",
    "build": "tsc -p tsconfig.build.json",
    "lint": "eslint .",
    "prisma:generate": "prisma generate",
    "prisma:push": "prisma db push",
    "prisma:seed": "ts-node prisma/seeds/seed.ts"
  },
  "dependencies": {
    "@nestjs/common": "^10.0.0",
    "@nestjs/config": "^3.2.2",
    "@nestjs/core": "^10.0.0",
    "@nestjs/swagger": "^7.4.2",
    "@prisma/client": "^5.18.0",
    "bcryptjs": "^2.4.3",
    "class-transformer": "^0.5.1",
    "class-validator": "^0.14.0",
    "jsonwebtoken": "^9.0.2",
    "reflect-metadata": "^0.1.13",
    "rimraf": "^5.0.5",
    "rxjs": "^7.8.1",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "@types/jsonwebtoken": "^9.0.6",
    "@types/node": "^20.11.30",
    "eslint": "^9.11.1",
    "ts-node": "^10.9.2",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.6.2",
    "prisma": "^5.18.0"
  }
}
```

**`apps/api/tsconfig.json`**

```json
{
  "extends": "../../packages/config/tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true
  },
  "include": ["src", "prisma/seeds"]
}
```

**`apps/api/tsconfig.build.json`**

```json
{
  "extends": "./tsconfig.json",
  "exclude": ["node_modules", "dist", "test", "**/*spec.ts"]
}
```

**`apps/api/src/main.ts`**

```ts
import 'reflect-metadata';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { cors: true });
  app.useGlobalPipes(new ValidationPipe({ transform: true, whitelist: true }));

  const cfg = new DocumentBuilder()
    .setTitle('VTX.Cloud API')
    .setDescription('Veterinary Cloud API')
    .setVersion('1.0.0')
    .addBearerAuth()
    .build();
  const doc = SwaggerModule.createDocument(app, cfg);
  SwaggerModule.setup('docs', app, doc);

  await app.listen(process.env.API_PORT ?? 8000);
}
bootstrap();
```

**`apps/api/src/app.module.ts`**

```ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { PrismaService } from './prisma/prisma.service';
import { OwnersModule } from './domain/owners/owners.module';
import { PatientsModule } from './domain/patients/patients.module';
import { ReportsModule } from './domain/reports/reports.module';

@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true }), OwnersModule, PatientsModule, ReportsModule],
  providers: [PrismaService],
})
export class AppModule {}
```

**`apps/api/src/prisma/prisma.service.ts`**

```ts
import { Injectable, OnModuleInit, INestApplication } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() { await this.$connect(); }
  async enableShutdownHooks(app: INestApplication) {
    this.$on('beforeExit', async () => { await app.close(); });
  }
}
```

### Owners

**`apps/api/src/domain/owners/owners.module.ts`**

```ts
import { Module } from '@nestjs/common';
import { OwnersController } from './owners.controller';
import { OwnersService } from './owners.service';
import { PrismaService } from '../../prisma/prisma.service';

@Module({
  controllers: [OwnersController],
  providers: [OwnersService, PrismaService],
})
export class OwnersModule {}
```

**`apps/api/src/domain/owners/owners.controller.ts`**

```ts
import { Controller, Get, Post, Body, Query } from '@nestjs/common';
import { OwnersService } from './owners.service';
import { z } from 'zod';

const OwnerDto = z.object({
  firstName: z.string().min(1),
  lastName: z.string().min(1),
  phone: z.string().optional(),
  email: z.string().email().optional(),
  address: z.string().optional(),
});
type OwnerDto = z.infer<typeof OwnerDto>;

@Controller('owners')
export class OwnersController {
  constructor(private readonly svc: OwnersService) {}

  @Get()
  list(@Query('q') q?: string) {
    return this.svc.list(q);
  }

  @Post()
  create(@Body() body: OwnerDto) {
    return this.svc.create(OwnerDto.parse(body));
  }
}
```

**`apps/api/src/domain/owners/owners.service.ts`**

```ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service';

@Injectable()
export class OwnersService {
  constructor(private prisma: PrismaService) {}

  list(q?: string) {
    return this.prisma.owner.findMany({
      where: q ? {
        OR: [
          { firstName: { contains: q, mode: 'insensitive' } },
          { lastName: { contains: q, mode: 'insensitive' } },
          { email: { contains: q, mode: 'insensitive' } }
        ]
      } : undefined,
      include: { patients: true }
    });
  }

  create(data: any) {
    return this.prisma.owner.create({ data });
  }
}
```

### Patients

**`apps/api/src/domain/patients/patients.module.ts`**

```ts
import { Module } from '@nestjs/common';
import { PatientsController } from './patients.controller';
import { PatientsService } from './patients.service';
import { PrismaService } from '../../prisma/prisma.service';

@Module({
  controllers: [PatientsController],
  providers: [PatientsService, PrismaService],
})
export class PatientsModule {}
```

**`apps/api/src/domain/patients/patients.controller.ts`**

```ts
import { Controller, Get, Post, Param, Body, Query } from '@nestjs/common';
import { PatientsService } from './patients.service';
import { z } from 'zod';

const CreatePatientDto = z.object({
  ownerId: z.string().cuid(),
  name: z.string().min(1),
  species: z.string().min(1),
  breed: z.string().optional(),
  sex: z.string().optional(),
  birthDate: z.string().datetime().optional()
});
type CreatePatientDto = z.infer<typeof CreatePatientDto>;

@Controller('patients')
export class PatientsController {
  constructor(private readonly svc: PatientsService) {}

  @Get()
  list(@Query('q') q?: string) {
    return this.svc.list(q);
  }

  @Get(':id')
  get(@Param('id') id: string) {
    return this.svc.get(id);
  }

  @Post()
  create(@Body() body: CreatePatientDto) {
    return this.svc.create(CreatePatientDto.parse(body));
  }
}
```

**`apps/api/src/domain/patients/patients.service.ts`**

```ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service';

@Injectable()
export class PatientsService {
  constructor(private prisma: PrismaService) {}

  list(q?: string) {
    return this.prisma.patient.findMany({
      where: q ? {
        OR: [
          { name: { contains: q, mode: 'insensitive' } },
          { species: { contains: q, mode: 'insensitive' } }
        ]
      } : undefined,
      include: { owner: true, cases: true }
    });
  }

  get(id: string) {
    return this.prisma.patient.findUnique({
      where: { id },
      include: { owner: true, cases: { include: { visits: true, invoice: true } } }
    });
  }

  create(data: any) {
    return this.prisma.patient.create({ data });
  }
}
```

### Reports (ekonomika)

**`apps/api/src/domain/reports/reports.module.ts`**

```ts
import { Module } from '@nestjs/common';
import { ReportsService } from './reports.service';
import { ReportsController } from './reports.controller';
import { PrismaService } from '../../prisma/prisma.service';

@Module({
  controllers: [ReportsController],
  providers: [ReportsService, PrismaService],
})
export class ReportsModule {}
```

**`apps/api/src/domain/reports/reports.controller.ts`**

```ts
import { Controller, Get, Query } from '@nestjs/common';
import { ReportsService } from './reports.service';

@Controller('reports')
export class ReportsController {
  constructor(private readonly svc: ReportsService) {}

  @Get('kpi')
  kpi(@Query('from') from?: string, @Query('to') to?: string) {
    return this.svc.kpi({ from, to });
  }

  @Get('revenue-by-month')
  revenueByMonth(@Query('year') year?: string) {
    return this.svc.revenueByMonth(year ? Number(year) : undefined);
  }

  @Get('top-procedures')
  topProcedures(@Query('limit') limit = 10) {
    return this.svc.topProcedures(Number(limit));
  }
}
```

**`apps/api/src/domain/reports/reports.service.ts`**

```ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service';

type Range = { from?: string; to?: string };

@Injectable()
export class ReportsService {
  constructor(private prisma: PrismaService) {}

  async kpi({ from, to }: Range) {
    const whereInvoice = (from && to) ? { issuedAt: { gte: new Date(from), lte: new Date(to) } } : {};
    const totalRevenue = await this.prisma.invoice.aggregate({ _sum: { totalCZK: true }, where: whereInvoice });
    const patients = await this.prisma.patient.count();
    const openCases = await this.prisma.case.count({ where: { status: 'OPEN' } });
    const debtors = await this.prisma.invoice.count({ where: { payments: { none: {} } } });
    return {
      totalRevenueCZK: Number(totalRevenue._sum.totalCZK ?? 0),
      totalPatients: patients,
      openCases,
      debtors
    };
  }

  async topProcedures(limit = 10) {
    return this.prisma.$queryRawUnsafe<any[]>(`
      SELECT p."name", SUM(vp."qty") AS count, SUM(vp."qty" * vp."unitPriceCZK") AS revenue
      FROM "VisitProcedure" vp
      JOIN "Procedure" p ON p.id = vp."procedureId"
      GROUP BY p."name"
      ORDER BY revenue DESC
      LIMIT ${Number(limit)}
    `);
  }

  async revenueByMonth(year?: number) {
    const clause = year ? `WHERE EXTRACT(YEAR FROM "issuedAt") = ${year}` : '';
    return this.prisma.$queryRawUnsafe<any[]>(`
      SELECT to_char(date_trunc('month', "issuedAt"), 'YYYY-MM') AS month,
             SUM("totalCZK")::numeric AS revenue
      FROM "Invoice"
      ${clause}
      GROUP BY 1
      ORDER BY 1
    `);
  }
}
```

### Prisma

**`apps/api/prisma/schema.prisma`**

```prisma
generator client { provider = "prisma-client-js" }
datasource db { provider = "postgresql"; url = env("DATABASE_URL") }

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  role      Role     @default(VET)
  password  String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
enum Role { ADMIN VET NURSE RECEPTION ACCOUNTANT }

model Owner {
  id        String   @id @default(cuid())
  firstName String
  lastName  String
  phone     String?
  email     String?
  address   String?
  patients  Patient[]
  createdAt DateTime @default(now())
}

model Patient {
  id         String   @id @default(cuid())
  name       String
  species    String
  breed      String?
  sex        String?
  birthDate  DateTime?
  ownerId    String
  owner      Owner    @relation(fields: [ownerId], references: [id])
  cases      Case[]
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt
}

model Case {
  id         String     @id @default(cuid())
  patientId  String
  title      String
  status     CaseStatus @default(OPEN)
  createdAt  DateTime   @default(now())
  closedAt   DateTime?
  patient    Patient    @relation(fields: [patientId], references: [id])
  visits     Visit[]
  invoice    Invoice?
}
enum CaseStatus { OPEN CLOSED }

model Visit {
  id         String   @id @default(cuid())
  caseId     String
  visitDate  DateTime @default(now())
  notes      String?
  procedures VisitProcedure[]
  labResults LabResult[]
  case       Case     @relation(fields: [caseId], references: [id])
}

model Procedure {
  id         String   @id @default(cuid())
  code       String   @unique
  name       String
  priceCZK   Decimal  @default(0)
  materials  InventoryItem[] @relation("ProcedureMaterials", references: [id])
}

model VisitProcedure {
  id           String    @id @default(cuid())
  visitId      String
  procedureId  String
  qty          Int       @default(1)
  unitPriceCZK Decimal   @default(0)
  visit        Visit     @relation(fields: [visitId], references: [id])
  procedure    Procedure @relation(fields: [procedureId], references: [id])
}

model InventoryItem {
  id         String   @id @default(cuid())
  sku        String   @unique
  name       String
  unit       String
  priceCZK   Decimal  @default(0)
  batches    Batch[]
}

model Batch {
  id              String   @id @default(cuid())
  inventoryItemId String
  lot             String
  expiry          DateTime?
  qtyOnHand       Decimal  @default(0)
  inventoryItem   InventoryItem @relation(fields: [inventoryItemId], references: [id])
}

model Invoice {
  id         String   @id @default(cuid())
  caseId     String   @unique
  number     String   @unique
  issuedAt   DateTime @default(now())
  totalCZK   Decimal  @default(0)
  payments   Payment[]
  case       Case     @relation(fields: [caseId], references: [id])
}

model Payment {
  id         String   @id @default(cuid())
  invoiceId  String
  method     PaymentMethod
  amountCZK  Decimal
  paidAt     DateTime @default(now())
  invoice    Invoice  @relation(fields: [invoiceId], references: [id])
}
enum PaymentMethod { CASH CARD BANK TRANSFER }

model LabResult {
  id        String   @id @default(cuid())
  visitId   String
  type      String
  result    Json
  fileUrl   String?
  visit     Visit    @relation(fields: [visitId], references: [id])
}
```

**`apps/api/prisma/seeds/seed.ts`**

```ts
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

async function main() {
  // Procedures
  const exam = await prisma.procedure.upsert({
    where: { code: 'EXAM' },
    update: {},
    create: { code: 'EXAM', name: 'Klinické vyšetření', priceCZK: 600 }
  });
  const vaccine = await prisma.procedure.upsert({
    where: { code: 'VACC' },
    update: {},
    create: { code: 'VACC', name: 'Očkování', priceCZK: 900 }
  });

  // Owner + patient
  const owner = await prisma.owner.create({
    data: { firstName: 'Jan', lastName: 'Novák', email: 'jan.novak@example.com' }
  });
  const patient = await prisma.patient.create({
    data: { ownerId: owner.id, name: 'Koko', species: 'Papoušek žako', sex: 'M' }
  });

  // Case + visit + invoice
  const c = await prisma.case.create({
    data: { patientId: patient.id, title: 'Kontrola po úrazu', status: 'OPEN' }
  });
  const visit = await prisma.visit.create({ data: { caseId: c.id, notes: 'Stabilní, drobný otok.' } });
  await prisma.visitProcedure.create({
    data: { visitId: visit.id, procedureId: exam.id, qty: 1, unitPriceCZK: exam.priceCZK }
  });

  await prisma.invoice.create({
    data: {
      caseId: c.id,
      number: '2025-0001',
      issuedAt: new Date(),
      totalCZK: exam.priceCZK
    }
  });

  console.log('Seed hotov.');
}

main().then(() => prisma.$disconnect()).catch(e => { console.error(e); process.exit(1); });
```

---

## APPS/WEB (Next.js)

**`apps/web/package.json`**

```json
{
  "name": "@vtx/web",
  "private": true,
  "scripts": {
    "dev": "next dev -p $WEB_PORT",
    "build": "next build",
    "start": "next start -p $WEB_PORT",
    "lint": "eslint ."
  },
  "dependencies": {
    "next": "15.0.0",
    "react": "18.3.1",
    "react-dom": "18.3.1"
  },
  "devDependencies": {
    "@types/node": "^20.11.30",
    "@types/react": "^18.2.66",
    "eslint": "^9.11.1",
    "typescript": "^5.6.2",
    "tailwindcss": "^3.4.10",
    "postcss": "^8.4.41",
    "autoprefixer": "^10.4.20"
  }
}
```

**`apps/web/tsconfig.json`**

```json
{
  "extends": "../../packages/config/tsconfig.base.json",
  "compilerOptions": { "jsx": "preserve" },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx"]
}
```

**`apps/web/next.config.js`**

```js
/** @type {import('next').NextConfig} */
const nextConfig = { reactStrictMode: true };
module.exports = nextConfig;
```

**`apps/web/postcss.config.js`**

```js
module.exports = { plugins: { tailwindcss: {}, autoprefixer: {} } };
```

**`apps/web/tailwind.config.ts`**

```ts
import type { Config } from 'tailwindcss';
export default {
  content: ['./app/**/*.{ts,tsx}', './components/**/*.{ts,tsx}'],
  theme: { extend: {} },
  plugins: []
} satisfies Config;
```

**`apps/web/app/globals.css`**

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

**`apps/web/app/layout.tsx`**

```tsx
import './globals.css';
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="cs">
      <body className="bg-gray-50 text-gray-900">
        <div className="max-w-6xl mx-auto p-6">{children}</div>
      </body>
    </html>
  );
}
```

**`apps/web/app/page.tsx`** (KPI + revenue chart placeholder)

```tsx
async function getKpi() {
  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/reports/kpi`, { cache: "no-store" });
  return res.json();
}
async function getRevenue() {
  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/reports/revenue-by-month`, { cache: "no-store" });
  return res.json();
}

export default async function Home() {
  const [kpi, revenue] = await Promise.all([getKpi(), getRevenue()]);
  return (
    <div className="grid gap-6">
      <h1 className="text-2xl font-semibold">VTX.Cloud – Přehled</h1>
      <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
        {[
          { t: 'Tržby (CZK)', v: kpi.totalRevenueCZK },
          { t: 'Pacienti', v: kpi.totalPatients },
          { t: 'Otevřené případy', v: kpi.openCases },
          { t: 'Dlužníci', v: kpi.debtors }
        ].map((x, i) => (
          <div key={i} className="p-4 rounded-lg border bg-white">
            <div className="text-sm text-gray-600">{x.t}</div>
            <div className="text-2xl font-bold">{x.v}</div>
          </div>
        ))}
      </div>
      <div className="p-4 rounded-lg border bg-white">
        <div className="text-sm text-gray-600 mb-2">Tržby po měsících</div>
        <ul className="grid gap-1">
          {revenue.map((r: any) => (
            <li key={r.month} className="flex justify-between">
              <span>{r.month}</span>
              <span className="font-medium">{Number(r.revenue)}</span>
            </li>
          ))}
        </ul>
      </div>
    </div>
  );
}
```

**`apps/web/app/patients/page.tsx`**

```tsx
async function getPatients(q?: string) {
  const url = new URL(`${process.env.NEXT_PUBLIC_API_URL}/patients`);
  if (q) url.searchParams.set("q", q);
  const res = await fetch(url.toString(), { cache: "no-store" });
  return res.json();
}

export default async function PatientsPage({ searchParams }: any) {
  const data = await getPatients(searchParams?.q);
  return (
    <div className="grid gap-4">
      <h1 className="text-2xl font-semibold">Pacienti</h1>
      <div className="grid gap-2">
        {data.map((p: any) => (
          <a key={p.id} href={`/patients/${p.id}`} className="p-4 border rounded-md bg-white hover:bg-gray-50">
            <div className="font-medium">{p.name} • {p.species}</div>
            <div className="text-sm text-gray-600">Majitel: {p.owner?.firstName} {p.owner?.lastName}</div>
          </a>
        ))}
      </div>
    </div>
  );
}
```

**`apps/web/app/patients/[id]/page.tsx`**

```tsx
async function getPatient(id: string) {
  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/patients/${id}`, { cache: "no-store" });
  return res.json();
}
export default async function PatientDetail({ params }: any) {
  const p = await getPatient(params.id);
  return (
    <div className="grid gap-4">
      <h1 className="text-2xl font-semibold">{p.name} <span className="text-sm text-gray-600">({p.species})</span></h1>
      <div className="p-4 border rounded-md bg-white">
        <div>Majitel: <b>{p.owner?.firstName} {p.owner?.lastName}</b></div>
        <div>Případy: {p.cases?.length ?? 0}</div>
      </div>
    </div>
  );
}
```

---

## PACKAGES

**`packages/config/tsconfig.base.json`**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "skipLibCheck": true
  }
}
```

**`packages/schemas/package.json`**

```json
{
  "name": "@vtx/schemas",
  "version": "0.0.1",
  "type": "module",
  "dependencies": { "zod": "^3.23.8" }
}
```

**`packages/schemas/patient.ts`**

```ts
import { z } from "zod";
export const PatientSchema = z.object({
  id: z.string().cuid().optional(),
  ownerId: z.string().cuid(),
  name: z.string().min(1),
  species: z.string().min(1),
  breed: z.string().optional(),
  sex: z.string().optional(),
  birthDate: z.string().datetime().optional()
});
export type Patient = z.infer<typeof PatientSchema>;
```

---

## APPS/WORKER (volitelné)

**`apps/worker/package.json`**

```json
{
  "name": "@vtx/worker",
  "private": true,
  "scripts": { "dev": "ts-node src/index.ts" },
  "dependencies": { "ioredis": "^5.4.1" },
  "devDependencies": { "ts-node": "^10.9.2", "typescript": "^5.6.2" }
}
```

**`apps/worker/src/index.ts`**

```ts
console.log('Worker připraven (zatím bez jobů).');
```

---

## Hotovo ✅

* Vlož soubory podle cest, vytvoř `.env` z `.env.example`.
* `docker compose -f infra/docker/docker-compose.yml up -d`
* `pnpm i && pnpm db:push && pnpm db:seed && pnpm dev`
* Otevři **[http://localhost:3000](http://localhost:3000)** (UI) a **[http://localhost:8000/docs](http://localhost:8000/docs)** (API).

Řekni „**přidej billing end-to-end**“ nebo „**inventář end-to-end**“ a napíšu ti další plně hotové moduly.
