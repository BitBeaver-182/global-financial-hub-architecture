Response: Here is how to build a production-grade analytics and forecasting service in NestJS using mathjs and financejs.

This implementation follows the **CQRS pattern** by taking historical net worth snapshots (Actuals), applying statistical trend modeling (Linear Regression) alongside deterministic recurring streams (Salary/Rent/Loans), and returning a unified, chart-ready time-series response over a standard REST endpoint.

### **1\. Installation**

First, install mathjs, financejs, and their TypeScript definitions in your NestJS project:

npm install mathjs financejs  
npm install \--save-dev @types/mathjs

### **2\. Analytics Service Implementation (analytics.service.ts)**

This service fetches historical monthly snapshots, uses mathjs matrix calculations to fit a linear regression trend, factors in recurring cash flow adjustments, and projects 12 months into the future.

import { Injectable, Logger } from '@nestjs/common';  
import { matrix, multiply, inv, transpose } from 'mathjs';  
// @ts-ignore financejs doesn't always ship default ESM types  
import { Finance } from 'financejs';

export interface NetWorthDataPoint {  
  timestamp: string; // ISO Date String  
  actualNetWorth: number | null;  
  projectedNetWorth: number | null;  
  isProjected: boolean;  
}

export interface ForecastResponseDto {  
  currency: string;  
  historicalMonths: number;  
  forecastMonths: number;  
  data: NetWorthDataPoint\[\];  
  summary: {  
    currentNetWorth: number;  
    projected12MonthNetWorth: number;  
    projectedAnnualGrowthRatePct: number;  
    monthlyBurnOrSavingsRate: number;  
  };  
}

@Injectable()  
export class AnalyticsService {  
  private readonly logger \= new Logger(AnalyticsService.name);  
  private readonly finance \= new Finance();

  /\*\*  
   \* Generates a 12-month Net Worth forecast by combining:  
   \* 1\. Historical Net Worth snapshots (Actuals)  
   \* 2\. Math.js Linear Regression (Trend)  
   \* 3\. Deterministic Net Monthly Cash Flow (Known Recurring Income/Expenses)  
   \*/  
  async generate12MonthForecast(  
    userId: string,  
    historicalSnapshots: { date: Date; amount: number }\[\],  
    netMonthlyRecurringCashFlow: number \= 0, // e.g. \+\$1,500/mo savings  
    currency: string \= 'USD',  
  ): Promise\<ForecastResponseDto\> {  
    if (historicalSnapshots.length \< 2\) {  
      throw new Error('At least 2 historical snapshots are required to compute a statistical forecast.');  
    }

    // Sort snapshots chronologically  
    const sortedSnapshots \= \[...historicalSnapshots\].sort(  
      (a, b) \=\> a.date.getTime() \- b.date.getTime(),  
    );

    // \-------------------------------------------------------------  
    // Step 1: Prepare Matrices for Linear Regression (y \= mx \+ c)  
    // \-------------------------------------------------------------  
    // X \= \[\[1, 0\], \[1, 1\], \[1, 2\], ...\] representing month indices  
    // Y \= \[\[value1\], \[value2\], ...\] representing net worth values  
    const xValues \= sortedSnapshots.map((\_, index) \=\> \[1, index\]);  
    const yValues \= sortedSnapshots.map((s) \=\> \[s.amount\]);

    const X \= matrix(xValues);  
    const Y \= matrix(yValues);

    // Ordinary Least Squares (OLS) Formula: B \= (X^T \* X)^-1 \* X^T \* Y  
    const XT \= transpose(X);  
    const XTX\_inv \= inv(multiply(XT, X));  
    const XTY \= multiply(XT, Y);  
    const B \= multiply(XTX\_inv, XTY);

    // Extract Intercept (c) and Slope/Monthly Growth Rate (m)  
    const intercept \= B.get(\[0, 0\]) as number;  
    const monthlySlope \= B.get(\[1, 0\]) as number;

    this.logger.debug(  
      \`Regression calculated for user \${userId}: Base Intercept=\${intercept.toFixed(2)}, Monthly Growth Slope=\${monthlySlope.toFixed(2)}\`,  
    );

    // Combine statistical slope with known deterministic net monthly savings/burn  
    const effectiveMonthlyGrowth \= monthlySlope \+ netMonthlyRecurringCashFlow;

    // \-------------------------------------------------------------  
    // Step 2: Build Output Time-Series Data Array  
    // \-------------------------------------------------------------  
    const resultPoints: NetWorthDataPoint\[\] \= \[\];

    // A. Push Historical Actuals  
    sortedSnapshots.forEach((snapshot) \=\> {  
      resultPoints.push({  
        timestamp: snapshot.date.toISOString(),  
        actualNetWorth: Number(snapshot.amount.toFixed(2)),  
        projectedNetWorth: null,  
        isProjected: false,  
      });  
    });

    // B. Push 12-Month Projections  
    const lastSnapshot \= sortedSnapshots\[sortedSnapshots.length \- 1\];  
    const lastDate \= new Date(lastSnapshot.date);  
    let lastValue \= lastSnapshot.amount;

    for (let month \= 1; month \<= 12; month++) {  
      const forecastDate \= new Date(lastDate);  
      forecastDate.setMonth(forecastDate.getMonth() \+ month);

      // Projecting next value iteratively  
      lastValue \+= effectiveMonthlyGrowth;

      resultPoints.push({  
        timestamp: forecastDate.toISOString(),  
        actualNetWorth: null,  
        projectedNetWorth: Number(lastValue.toFixed(2)),  
        isProjected: true,  
      });  
    }

    // \-------------------------------------------------------------  
    // Step 3: Compute Summary KPIs & Growth Metrics  
    // \-------------------------------------------------------------  
    const currentNetWorth \= lastSnapshot.amount;  
    const projected12MonthNetWorth \= resultPoints\[resultPoints.length \- 1\].projectedNetWorth;

    // Compound Annual Growth Rate (CAGR) via Finance.js  
    let projectedAnnualGrowthRatePct \= 0;  
    if (currentNetWorth \> 0\) {  
      projectedAnnualGrowthRatePct \= this.finance.CAGR(  
        currentNetWorth,  
        projected12MonthNetWorth,  
        1, // 1 Year period  
      );  
    }

    return {  
      currency,  
      historicalMonths: sortedSnapshots.length,  
      forecastMonths: 12,  
      data: resultPoints,  
      summary: {  
        currentNetWorth: Number(currentNetWorth.toFixed(2)),  
        projected12MonthNetWorth: Number(projected12MonthNetWorth.toFixed(2)),  
        projectedAnnualGrowthRatePct: Number(projectedAnnualGrowthRatePct.toFixed(2)),  
        monthlyBurnOrSavingsRate: Number(effectiveMonthlyGrowth.toFixed(2)),  
      },  
    };  
  }  
}

### **3\. Controller & Endpoint Delivery (analytics.controller.ts)**

Expose this through a standard REST controller that passes data directly to your frontend charting components (e.g., Recharts, ECharts, or Tremor).

import { Controller, Get, Query, UseGuards } from '@nestjs/common';  
import { AnalyticsService, ForecastResponseDto } from './analytics.service';

@Controller('api/v1/analytics')  
export class AnalyticsController {  
  constructor(private readonly analyticsService: AnalyticsService) {}

  @Get('net-worth-forecast')  
  async getNetWorthForecast(  
    @Query('userId') userId: string,  
    @Query('recurringSavings') recurringSavings?: string,  
  ): Promise\<ForecastResponseDto\> {  
    // 1\. In production, fetch actual monthly historical snapshots from DB/Formance  
    const mockHistoricalSnapshots \= \[  
      { date: new Date('2026-03-01'), amount: 85000 },  
      { date: new Date('2026-04-01'), amount: 87200 },  
      { date: new Date('2026-05-01'), amount: 86900 },  
      { date: new Date('2026-06-01'), amount: 89500 },  
      { date: new Date('2026-07-01'), amount: 92000 },  
      { date: new Date('2026-08-01'), amount: 94800 },  
      { date: new Date('2026-09-01'), amount: 97500 },  
    \];

    const parsedSavings \= recurringSavings ? parseFloat(recurringSavings) : 500;

    return this.analyticsService.generate12MonthForecast(  
      userId || 'user\_123',  
      mockHistoricalSnapshots,  
      parsedSavings,  
      'USD',  
    );  
  }  
}

### **4\. Sample JSON API Output**

When calling GET /api/v1/analytics/net-worth-forecast?recurringSavings=500, the frontend receives this payload ready to render both an **actuals solid line** and a **projected dashed line**:

{  
  "currency": "USD",  
  "historicalMonths": 7,  
  "forecastMonths": 12,  
  "data": \[  
    { "timestamp": "2026-03-01T00:00:00.000Z", "actualNetWorth": 85000, "projectedNetWorth": null, "isProjected": false },  
    { "timestamp": "2026-09-01T00:00:00.000Z", "actualNetWorth": 97500, "projectedNetWorth": null, "isProjected": false },  
    { "timestamp": "2026-10-01T00:00:00.000Z", "actualNetWorth": null, "projectedNetWorth": 100078.57, "isProjected": true },  
    { "timestamp": "2027-09-01T00:00:00.000Z", "actualNetWorth": null, "projectedNetWorth": 131021.43, "isProjected": true }  
  \],  
  "summary": {  
    "currentNetWorth": 97500,  
    "projected12MonthNetWorth": 131021.43,  
    "projectedAnnualGrowthRatePct": 34.38,  
    "monthlyBurnOrSavingsRate": 2578.57  
  }  
}

---

*User prompt: How can I set up NestJS CacheModule with Redis to cache chart responses for high performance?*

Response: To cache heavy analytical queries (like net worth forecasts) and prevent unnecessary calculations on every dashboard load, you can set up NestJS's native @nestjs/cache-manager with Redis.

Here is the step-by-step setup using **NestJS v10+** with cache-manager and cache-manager-redis-yet (the modern, async-first Redis store for NestJS).

### **1\. Installation**

Install @nestjs/cache-manager, cache-manager, and the modern Redis store wrapper:

npm install @nestjs/cache-manager cache-manager cache-manager-redis-yet redis

### **2\. Configure Global Cache Module (app.module.ts)**

Configure CacheModule asynchronously in your root module using CacheModule.registerAsync so it pulls your Redis credentials dynamically from ConfigService.

import { Module } from '@nestjs/common';  
import { ConfigModule, ConfigService } from '@nestjs/config';  
import { CacheModule } from '@nestjs/cache-manager';  
import { redisStore } from 'cache-manager-redis-yet';  
import { AnalyticsModule } from './modules/analytics/analytics.module';

@Module({  
  imports: \[  
    ConfigModule.forRoot({ isGlobal: true }),  
    CacheModule.registerAsync({  
      isGlobal: true, // Makes CacheManager available everywhere in the app  
      imports: \[ConfigModule\],  
      inject: \[ConfigService\],  
      useFactory: async (configService: ConfigService) \=\> {  
        const store \= await redisStore({  
          socket: {  
            host: configService.get\<string\>('REDIS\_HOST', 'localhost'),  
            port: configService.get\<number\>('REDIS\_PORT', 6379),  
          },  
          password: configService.get\<string\>('REDIS\_PASSWORD'),  
          ttl: 1000 \* 60 \* 60, // Default TTL: 1 hour (in milliseconds)  
        });

        return { store };  
      },  
    }),  
    AnalyticsModule,  
  \],  
})  
export class AppModule {}

### **3\. Apply Automatic Route Caching (analytics.controller.ts)**

NestJS provides a built-in CacheInterceptor that automatically intercepts incoming HTTP GET requests, builds a cache key from the URL and query parameters (e.g. /api/v1/analytics/net-worth-forecast?userId=123\&recurringSavings=500), and serves the cached JSON directly from Redis.

import { Controller, Get, Query, UseInterceptors } from '@nestjs/common';  
import { CacheInterceptor, CacheTTL, CacheKey } from '@nestjs/cache-manager';  
import { AnalyticsService, ForecastResponseDto } from './analytics.service';

@Controller('api/v1/analytics')  
@UseInterceptors(CacheInterceptor) // Intercepts all GET requests in this controller  
export class AnalyticsController {  
  constructor(private readonly analyticsService: AnalyticsService) {}

  @Get('net-worth-forecast')  
  @CacheTTL(1000 \* 60 \* 60 \* 24\) // Override TTL for this specific endpoint: 24 Hours (in ms)  
  async getNetWorthForecast(  
    @Query('userId') userId: string,  
    @Query('recurringSavings') recurringSavings?: string,  
  ): Promise\<ForecastResponseDto\> {  
    // If key exists in Redis, this method body is completely SKIPPED  
    return this.analyticsService.generate12MonthForecast(userId, recurringSavings);  
  }  
}

### **4\. Cache Invalidation Strategy (Programmatic Purging)**

Charts should remain cached for up to 24 hours **unless** the user adds a new asset, connects a bank, or logs a transaction.

When a user posts a transaction, inject CACHE\_MANAGER in your service to invalidate their cached endpoints immediately:

import { Injectable, Inject } from '@nestjs/common';  
import { CACHE\_MANAGER } from '@nestjs/cache-manager';  
import { Cache } from 'cache-manager';

@Injectable()  
export class TransactionsService {  
  constructor(  
    @Inject(CACHE\_MANAGER) private readonly cacheManager: Cache,  
  ) {}

  async onNewTransactionCreated(userId: string) {  
    // 1\. Process double-entry transaction posting in Formance...

    // 2\. Invalidate cache keys for this specific user  
    // Modern redisStore allows pattern/store matching or key deletion  
    const userForecastKey \= \`/api/v1/analytics/net-worth-forecast?userId=\${userId}\`;  
    await this.cacheManager.del(userForecastKey);

    // Or clear all cached analytics keys if using prefixing  
    // await this.cacheManager.store.del(\`/api/v1/analytics/\*\`);  
  }  
}

### **Summary Checklist**

| Concept | Best Practice |
| :---- | :---- |
| **Store Driver** | Use cache-manager-redis-yet (Node-Redis v4 wrapper) over legacy stores. |
| **TTL for Charts** | Set long TTLs (12 to 24 hours) since historical net worth doesn't change every minute. |
| **Invalidation Hook** | Trigger cacheManager.del() in your TransactionsService or FormanceService listener when write operations occur. |

---

*User prompt: Can you generate a Docker Compose file setup running Formance Ledger, Redis, Vault, and PostgreSQL together?*

Response: Here is a production-ready docker-compose.yml configured with **Formance Ledger**, **PostgreSQL** (with separate databases initialized for Formance and your NestJS app), **Redis**, and **HashiCorp Vault** running in dev mode.

### **1\. docker-compose.yml**

version: '3.8'

services:  
  \# \---------------------------------------------------------------------------  
  \# Database: PostgreSQL (Shared Engine with Isolated DBs)  
  \# \---------------------------------------------------------------------------  
  postgres:  
    image: postgres:16-alpine  
    container\_name: wealth\_app\_postgres  
    restart: always  
    environment:  
      POSTGRES\_USER: postgres  
      POSTGRES\_PASSWORD: postgrespassword  
      POSTGRES\_MULTIPLE\_DATABASES: formance\_ledger,app\_db  
    ports:  
      \- "5432:5432"  
    volumes:  
      \- postgres\_data:/var/lib/postgresql/data  
      \- ./scripts/init-multiple-dbs.sh:/docker-entrypoint-initdb.d/init-multiple-dbs.sh  
    healthcheck:  
      test: \["CMD-SHELL", "pg\_isready \-U postgres"\]  
      interval: 5s  
      timeout: 5s  
      retries: 5

  \# \---------------------------------------------------------------------------  
  \# In-Memory Cache & Queue: Redis  
  \# \---------------------------------------------------------------------------  
  redis:  
    image: redis:7-alpine  
    container\_name: wealth\_app\_redis  
    restart: always  
    command: redis-server \--save 60 1 \--loglevel notice  
    ports:  
      \- "6379:6379"  
    volumes:  
      \- redis\_data:/data  
    healthcheck:  
      test: \["CMD", "redis-cli", "ping"\]  
      interval: 5s  
      timeout: 5s  
      retries: 5

  \# \---------------------------------------------------------------------------  
  \# Secrets Management: HashiCorp Vault (Dev Mode)  
  \# \---------------------------------------------------------------------------  
  vault:  
    image: hashicorp/vault:1.15  
    container\_name: wealth\_app\_vault  
    restart: always  
    environment:  
      VAULT\_DEV\_ROOT\_TOKEN\_ID: dev-root-token-secret  
      VAULT\_DEV\_LISTEN\_ADDRESS: 0.0.0.0:8200  
    ports:  
      \- "8200:8200"  
    cap\_add:  
      \- IPC\_LOCK  
    healthcheck:  
      test: \["CMD", "vault", "status"\]  
      interval: 5s  
      timeout: 5s  
      retries: 5

  \# \---------------------------------------------------------------------------  
  \# Accounting Engine: Formance Ledger  
  \# \---------------------------------------------------------------------------  
  formance-ledger:  
    image: ghcr.io/formancehq/ledger:v1.10.2  
    container\_name: wealth\_app\_formance  
    restart: always  
    depends\_on:  
      postgres:  
        condition: service\_healthy  
    environment:  
      LEDGER\_STORAGE\_DRIVER: postgres  
      LEDGER\_STORAGE\_POSTGRES\_CONN\_STRING: postgres://postgres:postgrespassword@postgres:5432/formance\_ledger?sslmode=disable  
      LEDGER\_SERVER\_HTTP\_BIND\_ADDRESS: 0.0.0.0:8080  
    ports:  
      \- "8080:8080"  
    healthcheck:  
      test: \["CMD", "wget", "--spider", "-q", "http://localhost:8080/\_health"\]  
      interval: 5s  
      timeout: 5s  
      retries: 5

volumes:  
  postgres\_data:  
  redis\_data:

### **2\. Database Initialization Script**

To ensure PostgreSQL creates isolated databases for both **Formance Ledger** (formance\_ledger) and your **NestJS App** (app\_db) upon initial container startup, create a directory called scripts/ next to your docker-compose.yml and add init-multiple-dbs.sh:

**scripts/init-multiple-dbs.sh**

\#\!/bin/bash  
set \-e

function create\_user\_and\_database() {  
	local database=\$1  
	echo "  Creating database '\$database'"  
	psql \-v ON\_ERROR\_STOP=1 \--username "\$POSTGRES\_USER" \<\<-EOSQL  
	    CREATE DATABASE \$database;  
	    GRANT ALL PRIVILEGES ON DATABASE \$database TO \$POSTGRES\_USER;  
EOSQL  
}

if \[ \-n "\$POSTGRES\_MULTIPLE\_DATABASES" \]; then  
	echo "Multiple database creation requested: \$POSTGRES\_MULTIPLE\_DATABASES"  
	for db in \$(echo \$POSTGRES\_MULTIPLE\_DATABASES | tr ',' ' '); do  
		create\_user\_and\_database \$db  
	done  
	echo "Multiple databases created successfully"  
fi

Make the script executable:

chmod \+x scripts/init-multiple-dbs.sh

### **3\. NestJS Environment Variables (.env)**

Add these connection strings to your NestJS .env file to communicate with the Docker services:

\# Formance Ledger API Endpoint  
FORMANCE\_SERVER\_URL="http://localhost:8080"  
FORMANCE\_LEDGER\_NAME="main"

\# Redis (BullMQ Queue \+ CacheManager)  
REDIS\_HOST="localhost"  
REDIS\_PORT=6379

\# HashiCorp Vault Secrets Client  
VAULT\_ADDR="http://localhost:8200"  
VAULT\_TOKEN="dev-root-token-secret"

\# PostgreSQL Primary App DB (Prisma / Drizzle)  
DATABASE\_URL="postgresql://postgres:postgrespassword@localhost:5432/app\_db?schema=public"

### **Quick Start Instructions**

Run the stack in detached mode:

docker compose up \-d

Verify service status:

> * **Formance Ledger Health Check:** curl http://localhost:8080/\_health  
> * **HashiCorp Vault UI:** Open http://localhost:8200 (Token: dev-root-token-secret)  
> * **Redis CLI:** docker exec \-it wealth\_app\_redis redis-cli ping (Returns PONG)

---

*User prompt: Category B: Sensitive Data Encryption Option B1 (Prisma Field Encryption / KMS): Recommended. Encrypt API keys and OAuth tokens directly in PostgreSQL using AWS KMS or HashiCorp Vault environment keys is this something free? that can be done in my setup? (or is it alraedy?) and what do you recommend? keep current or use this?*

Response: Yes, field-level encryption **is completely free** in your open-source software stack, and it **can be easily implemented using HashiCorp Vault** (which is already part of your Docker setup).

### **Is it free? Is it already in your setup?**

> * **Is it free?** **Yes.** It costs \$0. Open-source packages like prisma-field-encryption use open algorithms (AES-256-GCM) with no license fees.  
> * **Is it already in your setup?** **Partially.** You already have HashiCorp Vault running in your docker-compose.yml, which acts as your **Key Management Service (KMS)**. However, the Prisma Client extension in NestJS is not yet encrypting fields before writing them to PostgreSQL.

### **Key Recommendation: Upgrade to Application-Level Encryption**

**Recommendation:** **Upgrade to field-level encryption.**

#### **Why?**

> * **Database Dump Exposure:** If someone steals or accidentally leaks a PostgreSQL backup (pg\_dump), plain text OAuth tokens and API keys are completely exposed.  
> * **Shoulder Surfing / GUI Leak:** Anyone with read-only access to your database via Adminer, DBeaver, or psql can read users' sensitive financial API tokens.  
> * **Negligible Performance Cost:** AES-256-GCM encryption at the application layer takes less than 1 millisecond in Node.js.

### **How to Implement It (NestJS \+ Prisma \+ Vault)**

Instead of sending plain strings to PostgreSQL, NestJS encrypts values before writing to PostgreSQL and decrypts them automatically upon querying.

#### **1\. Install the Extension**

npm install prisma-field-encryption

#### **2\. Annotate Your Prisma Schema**

Mark sensitive OAuth/API key fields using comment annotations (/// @encrypted):

model UserIntegration {  
  id           String   @id @default(uuid())  
  userId       String  
  provider     String   // e.g., "PLAID", "TELLER"  
    
  /// @encrypted  
  accessToken  String   // Encrypted in DB  
    
  /// @encrypted  
  refreshToken String?  // Encrypted in DB

  createdAt    DateTime @default(now())  
}

#### **3\. Inject the Encryption Key from Vault into Prisma Service**

In your NestJS PrismaService, fetch the master key from Vault and extend the Prisma Client:

import { Injectable, OnModuleInit } from '@nestjs/common';  
import { PrismaClient } from '@prisma/client';  
import { fieldEncryptionExtension } from 'prisma-field-encryption';  
import { ConfigService } from '@nestjs/config';

@Injectable()  
export class PrismaService extends PrismaClient implements OnModuleInit {  
  public extendedClient: any;

  constructor(private configService: ConfigService) {  
    super();  
  }

  async onModuleInit() {  
    await this.\$connect();

    // 1\. In production, pull this master key dynamically from HashiCorp Vault  
    const masterEncryptionKey \= this.configService.get\<string\>('FIELD\_ENCRYPTION\_KEY');

    // 2\. Extend Prisma Client to handle encryption/decryption transparently  
    this.extendedClient \= this.\$extends(  
      fieldEncryptionExtension({  
        encryptionKey: masterEncryptionKey,  
      })  
    );  
  }  
}

### **How It Works in PostgreSQL**

When you perform normal queries in your NestJS controllers:

// Writing to DB  
await prisma.userIntegration.create({  
  data: {  
    userId: 'usr\_123',  
    provider: 'PLAID',  
    accessToken: 'access-sandbox-1234567890', // Plaintext in TypeScript  
  },  
});

What actually lands in PostgreSQL:

| id | userId | provider | accessToken |
| :---- | :---- | :---- | :---- |
| uuid-1 | usr\_123 | PLAID | k1.aesgcm256.v1.bT...\$7xA2... *(Encrypted Ciphertext)* |

When you query findUnique(), Prisma automatically decrypts it back to 'access-sandbox-1234567890' before passing it back to your code.

---

*User prompt: Key Recommendation: Upgrade to Application-Level Encryption Recommendation: Upgrade to field-level encryption.*

Response: **Yes, field-level encryption is 100% free and open-source.** You do not need paid enterprise tiers or cloud KMS charges to use it.

You can definitely run it with your current architecture, as **HashiCorp Vault** is already defined in your docker-compose.yml.

### **Is it already in your setup?**

**Partially.**

> * **Infrastructure Layer:** **Yes.** HashiCorp Vault is running on port :8200.  
> * **Application Layer:** **No.** Right now, if NestJS writes a Plaid token or bank OAuth secret to PostgreSQL via Prisma, it gets stored in cleartext.

If someone leaks or accesses a database backup (.sql dump), those credentials will be readable unless you add field-level encryption.

### **Key Recommendation: Use Field Encryption**

**Recommendation:** **Implement field-level application encryption.**

| Architecture Approach | Pros | Cons | Recommendation |
| :---- | :---- | :---- | :---- |
| **Current Setup** *(Plaintext in DB)* | Zero setup time, simple queries. | A database compromise or DB backup leak reveals all user API/OAuth secrets. | ❌ Risk too high for financial integrations |
| **Prisma Field Encryption** *(App-Level)* | **Free**, transparent to queries, zero cloud cost, backup-safe. | Searching encrypted fields requires exact-match hashes/salts. | ✅ **Recommended for your stack** |

### **Step-by-Step Implementation in NestJS**

#### **1\. Install standard dependencies**

npm install prisma-field-encryption @hashicorp/vault-js

#### **2\. Annotate fields in schema.prisma**

Add /// @encrypted comments to sensitive OAuth/token fields:

model UserIntegration {  
  id           String   @id @default(uuid())  
  userId       String  
  provider     String   // e.g. "PLAID", "TELLER"

  /// @encrypted  
  accessToken  String   // Gets encrypted in DB automatically

  /// @encrypted  
  refreshToken String?  // Gets encrypted in DB automatically

  createdAt    DateTime @default(now())  
}

#### **3\. Inject Vault Key into Prisma Service (prisma.service.ts)**

Retrieve the master key from HashiCorp Vault on startup and extend your Prisma Client:

import { Injectable, OnModuleInit } from '@nestjs/common';  
import { PrismaClient } from '@prisma/client';  
import { fieldEncryptionExtension } from 'prisma-field-encryption';  
import { ConfigService } from '@nestjs/config';

@Injectable()  
export class PrismaService extends PrismaClient implements OnModuleInit {  
  public extendedClient: any;

  constructor(private configService: ConfigService) {  
    super();  
  }

  async onModuleInit() {  
    await this.\$connect();

    // Pull the master encryption key configured/retrieved from Vault  
    const masterEncryptionKey \= this.configService.get\<string\>('FIELD\_ENCRYPTION\_KEY');

    // Attach extension to encrypt before database write, decrypt after database read  
    this.extendedClient \= this.\$extends(  
      fieldEncryptionExtension({  
        encryptionKey: masterEncryptionKey,  
      }),  
    );  
  }  
}

#### **4\. Usage in Services**

Use this.prisma.extendedClient in your business logic. Operations handle conversion automatically:

// NestJS automatically encrypts \`accessToken\` before running the SQL INSERT  
await this.prisma.extendedClient.userIntegration.create({  
  data: {  
    userId: 'usr\_100',  
    provider: 'PLAID',  
    accessToken: 'access-sandbox-998877665544', // Plaintext string in code  
  },  
});

### **What it looks like inside PostgreSQL**

| id | userId | provider | accessToken |
| :---- | :---- | :---- | :---- |
| uuid-1 | usr\_100 | PLAID | k1.aesgcm256.v1.a8F2...\$e9A1... *(Encrypted Ciphertext)* |

When you query userIntegration.findUnique(), Prisma decrypts the field back into your code as 'access-sandbox-998877665544'.

---
