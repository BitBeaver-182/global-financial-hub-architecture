
Response: Financial apps like Monarch, Copilot, and YNAB implement custom auto-categorization through a **Rule Engine pipeline**. Every time a raw bank stream or CSV row enters the backend, it passes through a multi-stage data-cleaning and pattern-matching pipeline before hitting the user's ledger.

### **1\. The 4-Stage Categorization Pipeline**

Raw bank transaction data is notoriously dirty (e.g., TST\* SBUX 02419 ONLINE SEATTLE WA). Before applying user rules, the app standardizes the data.

\[ Raw Bank Payload \]  
         │  
         ▼  
\[ Stage 1: Data Normalization \] ── (Cleans string, extracts merchant name)  
         │  
         ▼  
\[ Stage 2: System ML / Global Rules \] ── (Default categorization fallback)  
         │  
         ▼  
\[ Stage 3: User Custom Rule Engine \] ── (Overrides default categorization)  
         │  
         ▼  
\[ Stage 4: Immutable Transaction Record \]

### **2\. Database Schema for Custom Rules**

To build a rule system like Monarch's, you need to support both **conditions** (IF statements) and **actions** (THEN outcomes).

#### **Prisma / PostgreSQL Schema Example**

model Rule {  
  id          String   @id @default(uuid())  
  userId      String  
  name        String?  
  priority    Int      @default(0) // Higher priority runs first  
  isActive    Boolean  @default(true)  
    
  // Logic matching mode: AND (all conditions match) vs OR (any condition matches)  
  matchType   MatchType @default(AND) 

  conditions  RuleCondition\[\]  
  actions     RuleAction\[\]

  createdAt   DateTime @default(now())  
  updatedAt   DateTime @updatedAt

  @@index(\[userId, priority\])  
}

enum MatchType {  
  AND  
  OR  
}

model RuleCondition {  
  id          String         @id @default(uuid())  
  ruleId      String  
  rule        Rule           @relation(fields: \[ruleId\], references: \[id\], onDelete: Cascade)  
    
  field       ConditionField // MERCHANT, AMOUNT, ACCOUNT, RAW\_DESCRIPTION, TYPE  
  operator    Operator       // EQUALS, CONTAINS, STARTS\_WITH, ENDS\_WITH, GREATER\_THAN, LESS\_THAN  
  value       String         // e.g., "Starbucks", "100.00"  
}

enum ConditionField {  
  MERCHANT  
  AMOUNT  
  ACCOUNT\_ID  
  RAW\_TEXT  
}

enum Operator {  
  EQUALS  
  CONTAINS  
  STARTS\_WITH  
  ENDS\_WITH  
  GREATER\_THAN  
  LESS\_THAN  
}

model RuleAction {  
  id          String      @id @default(uuid())  
  ruleId      String  
  rule        Rule        @relation(fields: \[ruleId\], references: \[id\], onDelete: Cascade)  
    
  type        ActionType  // SET\_CATEGORY, RENAME\_MERCHANT, ADD\_TAG, HIDE\_TRANSACTION  
  value       String      // e.g., category\_id\_123 or "Starbucks Coffee"  
}

enum ActionType {  
  SET\_CATEGORY  
  RENAME\_MERCHANT  
  ADD\_TAG  
  HIDE  
}

### **3\. Implementation of the Rule Engine (TypeScript)**

Here is how the engine evaluates incoming transactions against stored user rules:

type Transaction \= {  
  rawDescription: string;  
  merchantName: string;  
  amount: number;  
  accountId: string;  
  categoryId?: string;  
  tags: string\[\];  
  hidden: boolean;  
};

type RuleCondition \= {  
  field: 'MERCHANT' | 'AMOUNT' | 'ACCOUNT\_ID' | 'RAW\_TEXT';  
  operator: 'EQUALS' | 'CONTAINS' | 'STARTS\_WITH' | 'GREATER\_THAN' | 'LESS\_THAN';  
  value: string;  
};

type RuleAction \= {  
  type: 'SET\_CATEGORY' | 'RENAME\_MERCHANT' | 'ADD\_TAG' | 'HIDE';  
  value: string;  
};

type Rule \= {  
  matchType: 'AND' | 'OR';  
  conditions: RuleCondition\[\];  
  actions: RuleAction\[\];  
};

export function evaluateRules(transaction: Transaction, rules: Rule\[\]): Transaction {  
  // Clone transaction to avoid side effects  
  const result \= { ...transaction };

  for (const rule of rules) {  
    const isMatch \= checkRuleMatch(result, rule);

    if (isMatch) {  
      // Execute all actions in the matching rule  
      for (const action of rule.actions) {  
        switch (action.type) {  
          case 'SET\_CATEGORY':  
            result.categoryId \= action.value;  
            break;  
          case 'RENAME\_MERCHANT':  
            result.merchantName \= action.value;  
            break;  
          case 'ADD\_TAG':  
            if (\!result.tags.includes(action.value)) {  
              result.tags.push(action.value);  
            }  
            break;  
          case 'HIDE':  
            result.hidden \= action.value \=== 'true';  
            break;  
        }  
      }  
    }  
  }

  return result;  
}

function checkRuleMatch(tx: Transaction, rule: Rule): boolean {  
  const matches \= rule.conditions.map((condition) \=\> {  
    let fieldValue: string | number \= '';

    switch (condition.field) {  
      case 'MERCHANT':  
        fieldValue \= tx.merchantName.toLowerCase();  
        break;  
      case 'RAW\_TEXT':  
        fieldValue \= tx.rawDescription.toLowerCase();  
        break;  
      case 'ACCOUNT\_ID':  
        fieldValue \= tx.accountId;  
        break;  
      case 'AMOUNT':  
        fieldValue \= tx.amount;  
        break;  
    }

    const targetValue \= condition.value.toLowerCase();

    switch (condition.operator) {  
      case 'CONTAINS':  
        return String(fieldValue).includes(targetValue);  
      case 'EQUALS':  
        return String(fieldValue) \=== targetValue;  
      case 'STARTS\_WITH':  
        return String(fieldValue).startsWith(targetValue);  
      case 'GREATER\_THAN':  
        return Number(fieldValue) \> Number(targetValue);  
      case 'LESS\_THAN':  
        return Number(fieldValue) \< Number(targetValue);  
      default:  
        return false;  
    }  
  });

  // Apply AND / OR logic across conditions  
  return rule.matchType \=== 'AND'  
    ? matches.every(Boolean)  
    : matches.some(Boolean);  
}

### **4\. How Financial Apps Make This UX Seamless**

> 1. **"Create Rule From Transaction" Modal:** When a user manually edits a category on a transaction, the UI pops up a checkbox: *"Always categorize \[Merchant\] as \[Category\] in the future?"* Pre-filling the rule modal cuts user effort to one click.  
> 2. **Retroactive Rule Application:** When a user creates or modifies a rule, ask: *"Apply to existing transactions?"*. Running the engine on past database records ensures consistency.  
> 3. **Regex / Wildcard Support for Power Users:** For complex bank descriptions (e.g., matching all variants of Amazon AWS vs. Amazon Retail), offering regex matching allows power users to write one catch-all rule.  
> 4. **Merchant Normalization Table:** Before user rules run, apps maintain a canonical merchant mapping (UBER \*TRIP 1823 → Uber). Matching rules against the normalized merchant name is significantly more reliable than matching raw bank strings.

---

*User prompt: I see there's goCardless (EU/UK) and SimpleFIN (Canada/US) to sync (securely???) with the banks. is there any in South America like those 2? are they free and open source?*

### **Response: 1\. Are GoCardless and SimpleFIN "Secure"?**

**Yes, but they operate under completely different security models:**

> * **GoCardless (GoCardless Bank Account Data, formerly Nordigen):** Operates under official **EU/UK Open Banking regulations (PSD2)**. It uses OAuth redirects—meaning you never give your password or bank credentials to GoCardless or your finance app. You log into your bank directly, grant temporary permission (typically valid for 90–180 days), and the bank gives GoCardless a read-only access token.  
> * **SimpleFIN:** Operates as a lightweight middleware layer in North America (powered by MX underneath). While secure via encryption, open banking laws in the US/Canada are less standardized than in Europe, so some connections use OAuth while others rely on credential vaulting or screen scraping.

### **2\. What Is Available in South America / Latin America (LatAm)?**

South America is experiencing a massive Open Banking boom (led heavily by **Brazil's Banco Central Open Finance framework**, as well as developments in **Colombia, Mexico, and Chile**).

However, because LatAm banking systems are localized, global providers like GoCardless do not cover the region. Instead, LatAm has its own dedicated open banking aggregators:

| Provider | Primary Regional Coverage | How it Works |
| :---- | :---- | :---- |
| **Pluggy** | Brazil, Argentina, Colombia, Mexico | Built specifically for LatAm. Connects to 50+ major LatAm banks, credit cards, and investment accounts (like Itaú, Bradesco, Santander, Mercado Pago, Nubank). |
| **Belvo** | Mexico, Brazil, Colombia | The largest "Plaid for LatAm". Offers deep Open Finance APIs for bank statements, employment verification, and fiscal invoices (SAT/DIAN). |
| **Mono** | Colombia, Mexico | Focuses heavily on business banking accounts and automated bank feed syncs. |
| **OFDA / Open Finance Brasil** | Brazil | Government-regulated Open Banking framework. Free direct OAuth APIs provided natively by major Brazilian banks if you register as an authorized fintech entity. |

### **3\. Are They Free and Open Source?**

**No, there is virtually no free, open-source bank aggregator for South America (or globally).**

#### **Why Open Source Aggregators Don't Exist for Banks**

Bank APIs require constant maintenance: banks frequently change anti-scraping measures, alter OAuth flows, update security protocols, or enforce mandatory regulatory audits (like SOC 2 / ISO 27001). Paid aggregators charge fees because they employ engineering teams to maintain these breakable integrations 24/7.

#### **Pricing Realities for LatAm Options**

> * **Commercial LatAm APIs (Belvo / Pluggy):**  
  * **Free Tier:** Offer sandbox environments and free trial credits (usually 25–100 live connected accounts for development).  
  * **Paid Tier:** Pay-per-connected-account or pay-per-API-call (typically \~\$0.10–\$0.30 per active connection per month).  
> * **The "Truly Free" Alternative (Manual File Import):**  
  * If you want a 100% free, open-source workflow for users in South America, the standard path is to build **OFX, QFX, and CSV statement parsers** (using open-source packages like node-ofx-parser or PapaParse).  
  * Almost every bank in South America allows users to download monthly .csv or .ofx bank files for free.

---

*User prompt: Is it possible to track instead of transactions, the balance of an account instead? would that work with out ledger thingie?*

Response: **Yes, absolutely.** Tracking balance snapshots instead of individual line-item transactions is a completely valid model—popularized by net-worth tracking apps like **Kubera**, **Empower (Personal Capital)**, and **ProjectionLab**.

It also works inside a double-entry ledger engine, but it changes **how** you construct your journal entries.

### **1\. How Balance-Only Tracking Works in a Ledger**

In double-entry accounting:

Assets=Liabilities+Equity

When tracking transactions, your categories (Groceries, Salary, Rent) act as the details that explain why your bank balance changed.

If you **skip transactions** and only record periodic account balances, you handle changes in wealth through an **Equity / Adjustment** entry rather than categorized spending.

#### **Example: Balance Reconciliation Entry**

Suppose your checking account balance was **\$5,000** on Jan 1\. On Feb 1, you check your balance and it's **\$4,200**. You lost \$800, but you don't care to log whether it went to coffee, rent, or utilities.

To bring your ledger balance in line with reality, the system generates an **Uncategorized Adjustment** entry:

2026-02-01 Balance Reconciliation  
    Assets:Bank:Checking                 \-\$800.00  
    Equity:Uncategorized Adjustments      \+\$800.00

By balancing against an **Equity** account (or an Expenses:Uncategorized catch-all), your ledger stays mathematically balanced without knowing individual line items.

### **2\. Database Schema: Transaction vs. Balance Snapshots**

If you're building a balance-first app, your data layer becomes much simpler than a full transaction engine.

model Account {  
  id          String            @id @default(uuid())  
  name        String            // "Chase Checking", "Bitcoin Wallet", "Real Estate"  
  type        AccountType       // ASSET, LIABILITY  
  snapshots   BalanceSnapshot\[\]  
}

model BalanceSnapshot {  
  id          String   @id @default(uuid())  
  accountId   String  
  account     Account  @relation(fields: \[accountId\], references: \[id\])  
    
  balance     Decimal  // e.g. 10450.50  
  currency    String   @default("USD")  
  timestamp   DateTime @default(now())

  @@index(\[accountId, timestamp\])  
}

enum AccountType {  
  ASSET  
  LIABILITY  
}

Instead of running continuous aggregations (SUM(amount) across millions of transactions), calculating your Net Worth or account balance over time becomes a simple query for the **latest snapshot** per account.

### **3\. Trade-offs: Balance Snapshots vs. Transaction Logs**

| Feature | Balance-Only Engine | Transaction-Based Engine (Monarch/YNAB) |
| :---- | :---- | :---- |
| **Primary Focus** | Net Worth & Wealth Tracking over time. | Budgeting, Cash Flow, and Category Spending. |
| **API Integration** | **Easier.** Works with basic read-only APIs or manual user inputs. | **Harder.** Requires webhooks for real-time pending/posted transactions. |
| **Insights Gained** | "Am I richer or poorer than last month?" | "Did I spend too much on restaurants this week?" |
| **Privacy & Effort** | High privacy, low maintenance. | Lower privacy, requires continuous rule setup and maintenance. |

### **4\. Hybrid Approach (Best of Both Worlds)**

Many modern personal finance engines use **Balance Snapshots as the Source of Truth**, while using transactions purely as metadata:

> 1. **The Anchor:** The engine logs periodic account balances directly from bank snapshots.  
> 2. **The Fill-In:** If transaction feeds are available, it attaches them underneath to explain the gap between Snapshot *A* and Snapshot *B*.  
> 3. **The Fallback:** If transactions fail to sync or are missing, the system simply creates a balancing delta entry so your Net Worth chart remains 100% accurate.

---

*User prompt: going back to Double-Entry Accounting Engines, since it;s going to be in the core of the app, rather than making it from scratch i rather use something proven. my backend is in node (not sure yet if plain or if i want to use nestjs or what, maybe suggest something as well)*

Response: Rather than writing a double-entry accounting engine from scratch in Node.js, leveraging open-source core engines or established relational database patterns gives you a battle-tested architecture that guarantees balanced books.

### **1\. The Reality of Node.js Libraries for Accounting**

Unlike Python (which has **Beancount**) or Rust (which has **Ledger** ports), the NPM ecosystem does **not** have a single, dominant, turnkey Node.js double-entry framework. Most NPM packages are either thin number formatters or unmaintained hobby utilities.

Instead, production applications typically choose one of two proven approaches:

#### **Option A: Embed an Engine via WebAssembly / CLI**

If you want strict, battle-tested double-entry math, run **Beancount** (Python) or **Ledger** (C++) inside Node using WebAssembly bindings or a child-process sidecar service.

#### **Option B: Build on a Proven Database Core (Recommended)**

Because double-entry math is fundamentally about **atomic SQL constraints**, building your domain service on top of a proven PostgreSQL double-entry schema using an ORM like **Prisma** or **Drizzle** is often the cleanest approach.

### **2\. NestJS vs. Express vs. Fastify**

For a financial application centered on double-entry accounting, **NestJS** is the strongest choice.

┌──────────────────────────────────────────────────────────┐  
│                      NestJS App                          │  
│                                                          │  
│  \[ Controller \]  ──►  \[ Ledger Service \]                 │  
│                              │                           │  
│                              ▼                           │  
│                   \[ PostgreSQL Transaction \]             │  
│                   ├── 1\. Lock Accounts                   │  
│                   ├── 2\. Validate Debits \== Credits     │  
│                   └── 3\. Insert Journal & Ledger Entries │  
└──────────────────────────────────────────────────────────┘

#### **Why NestJS Wins for Financial Apps**

> 1. **Enforced Architecture:** Financial apps require strict layer separation (Controllers → Services → Repositories). NestJS enforces this through Modules and Dependency Injection out of the box.  
> 2. **Atomic Transaction Management:** NestJS cleanly handles database unit-of-work patterns (e.g., using cls-hooked or Prisma/Drizzle transaction contexts) to prevent partial ledger commits.  
> 3. **TypeScript First:** Strict type safety is essential for financial arithmetic, preventing bugs like passing strings into balance calculations.

*Alternative:* If NestJS feels too boilerplate-heavy, use **Fastify \+ TypeScript** instead of plain Express. Fastify provides built-in schema validation (via TypeBox/Zod), better async handling, and significantly higher HTTP performance.

### **3\. Production Double-Entry Engine Schema (PostgreSQL \+ Prisma)**

This schema implements double-entry principles directly at the database level:

datasource db {  
  provider \= "postgresql"  
  url      \= env("DATABASE\_URL")  
}

generator client {  
  provider \= "prisma-client-js"  
}

enum AccountType {  
  ASSET  
  LIABILITY  
  EQUITY  
  REVENUE  
  EXPENSE  
}

model Account {  
  id          String         @id @default(uuid())  
  code        String         // e.g., "1010" for Cash, "4010" for Salary  
  name        String  
  type        AccountType  
  currency    String         @default("USD")  
    
  entries     LedgerEntry\[\]  
    
  createdAt   DateTime       @default(now())  
  updatedAt   DateTime       @updatedAt

  @@unique(\[code\])  
}

model JournalTransaction {  
  id          String        @id @default(uuid())  
  description String  
  postedAt    DateTime      @default(now())  
    
  entries     LedgerEntry\[\]  
    
  createdAt   DateTime      @default(now())  
}

model LedgerEntry {  
  id                   String             @id @default(uuid())  
  journalTransactionId String  
  journalTransaction   JournalTransaction @relation(fields: \[journalTransactionId\], references: \[id\], onDelete: Cascade)  
    
  accountId            String  
  account              Account            @relation(fields: \[accountId\], references: \[id\])  
    
  // Positives represent DEBIT, Negatives represent CREDIT (or vice versa based on convention)  
  // Store amounts in integer cents/minor units to avoid float precision loss  
  amount               BigInt  
    
  createdAt            DateTime           @default(now())

  @@index(\[accountId\])  
  @@index(\[journalTransactionId\])  
}

### **4\. NestJS Core Accounting Service**

This service executes transactions atomically, guaranteeing that ∑Debits−∑Credits=0:

import { Injectable, BadRequestException } from '@nestjs/common';  
import { PrismaService } from './prisma.service';

export interface EntryInput {  
  accountId: string;  
  amount: bigint; // Cent amount: \+1000 \= \$10.00 Debit, \-1000 \= \$10.00 Credit  
}

@Injectable()  
export class LedgerService {  
  constructor(private readonly prisma: PrismaService) {}

  async postTransaction(description: string, entries: EntryInput\[\]) {  
    // 1\. Double-Entry Rule Check: Sum of all entry amounts MUST equal 0  
    const totalDelta \= entries.reduce((sum, entry) \=\> sum \+ entry.amount, BigInt(0));  
      
    if (totalDelta \!== BigInt(0)) {  
      throw new BadRequestException(  
        \`Unbalanced Journal Entry: Sum of entries must be zero. Current imbalance: \${totalDelta}\`  
      );  
    }

    if (entries.length \< 2\) {  
      throw new BadRequestException('A double-entry transaction requires at least 2 legs.');  
    }

    // 2\. Atomic Database Execution  
    return this.prisma.\$transaction(async (tx) \=\> {  
      const journal \= await tx.journalTransaction.create({  
        data: {  
          description,  
          entries: {  
            create: entries.map((e) \=\> ({  
              accountId: e.accountId,  
              amount: e.amount,  
            })),  
          },  
        },  
        include: {  
          entries: true,  
        },  
      });

      return journal;  
    });  
  }

  // Calculate real-time balance for any account  
  async getAccountBalance(accountId: string): Promise\<bigint\> {  
    const aggregate \= await this.prisma.ledgerEntry.aggregate({  
      where: { accountId },  
      \_sum: {  
        amount: true,  
      },  
    });

    return aggregate.\_sum.amount ?? BigInt(0);  
  }  
}

---

*User prompt: so.. i don't know id you didn't undertsand or i didn't. i get node js doesn't have this accounting packages, but since we use nestjs, we can host it as a microservice right? can't we then build somethiing on top of it or pointless? i will be doing my own interface, but what i would like to avod is having to recreate the accounting for it. do you undertsand?*

Response: Yes, I understand completely\! You want to avoid writing the core accounting logic—the ledger math, entry validation, account chart structures, and balance rollups—from scratch, and instead plug into a dedicated, battle-tested system while building your own frontend and product UI on top.

Using a dedicated accounting microservice is **not pointless at all**. In fact, that is how most modern fintechs, banks, and apps like Monarch or Stripe build their architectures.

Here are the best ways to adopt an existing, proven engine as a standalone service rather than re-inventing accounting rules yourself:

### **1\. Ready-Made Microservices You Can Run Standalone**

Instead of embedding a library directly in your NestJS app, you can host one of these dedicated accounting services in Docker alongside your backend and talk to it via gRPC or REST APIs:

#### **A. Formance Ledger (Go / Open Source Engine)**

> * **What it is:** A cloud-native, open-source double-entry ledger database built specifically for fintechs and financial platforms.  
> * **How it works:** You run the formance-ledger container in Docker. It exposes a simple REST and gRPC API.  
> * **Why it fits your setup:** You send JSON transaction payloads from your NestJS app directly to Formance. Formance handles double-entry constraints, multi-currency conversions, complex multi-leg splits, and immutable ledger hashing automatically. You never write a line of accounting code.  
> * **Tech:** Open Source (Apache 2.0), Go runtime, PostgreSQL backend.

#### **B. Apache Fineract (Java / REST API)**

> * **What it is:** The open-source banking engine backed by the Apache Foundation.  
> * **How it works:** Hosts a full double-entry general ledger microservice complete with asset/holding hierarchies, accounts, and journal entry endpoints.  
> * **Pros/Cons:** Extremely battle-tested and compliant, but heavier to deploy compared to Formance.

### **2\. The Microservice Sidecar Pattern (Embedded CLI/Wasm Engine)**

If you don't want to run a complex second server like Formance, another popular industry pattern is the **Sidecar Pattern**. You run a light wrapper service around a C++, Rust, or Python double-entry engine that speaks JSON to your NestJS app:

┌──────────────────────────────┐         HTTP / gRPC          ┌─────────────────────────────────────────┐  
│     Your NestJS Backend      │ ───────────────────────────► │      Accounting Engine Sidecar          │  
│                              │                              │  (Formance / Beancount / Ledger CLI)   │  
│  • User Management           │ ◄─────────────────────────── │                                         │  
│  • API Gateway               │         JSON Response        │  • Validates Debits \== Credits          │  
│  • Custom Product UI & Rules │                              │  • Maintains Ledger Balance Integrity   │  
└──────────────────────────────┘                              └─────────────────────────────────────────┘

#### **Example using beancount or hledger as a Sidecar:**

> 1. You deploy a minimal Python/FastAPI container wrapping **Beancount** (the gold standard plain-text double-entry engine).  
> 2. When a user creates a transaction or balance adjustment in your app, NestJS POSTs the data to the sidecar: POST http://ledger-service/transactions  
> 3. The sidecar validates the entries, posts them to the ledger, and returns calculated balances back to NestJS as JSON.

### **3\. Which Path Should You Take?**

| Approach | Setup Effort | Accounting Maintenance Needed | Best For |
| :---- | :---- | :---- | :---- |
| **Formance Ledger (Microservice)** | Low (Docker Compose) | **Zero.** The engine owns all debits, credits, and rules. | Production apps where you want a ready-to-use API out of the box. |
| **Python Beancount / hledger (Sidecar)** | Medium | **Zero.** Rely on proven C++/Python math via a microservice wrapper. | Complete control over ledger files with zero math implementation. |
| **Native NestJS \+ SQL Schema** | Medium | **Minimal.** Requires basic SQL constraints (Debits \== Credits). | Keeping your infrastructure to a single codebase/database. |

### **Recommendation**

If you want to **avoid writing accounting logic completely** while building a TypeScript/NestJS app, spin up **Formance Ledger** in Docker. Your NestJS app will strictly manage user accounts, authentication, and UI views, offloading 100% of the financial math to the Formance microservice.

---

*User prompt: yes\! 100% i want to use Formance Ledger. Make a small summary of the charts and stack we've spoken so far as well of a bit of the architecture (very very briefly since we haven't decided everything yet). Identify missing pieces and provide suggestions (max 3 options for me to decide)*

### **Response: 1\. Architectural Summary & Stack**

Using **Formance Ledger** offloads all double-entry arithmetic, Numscript transaction logic, immutable logging, and multi-asset balance rollups out of your application backend.

┌─────────────────────────┐          ┌──────────────────────────┐          ┌───────────────────────────────┐  
│     User Client         │          │   NestJS Gateway / API   │          │   Formance Ledger (Docker)    │  
│  (Custom Frontend UI)   │ ───────► │   • Auth / Permissions   │ ───────► │   • Core Double-Entry Engine  │  
│                         │   HTTP   │   • Business Logic       │   REST   │   • Numscript Execution       │  
└─────────────────────────┘          │   • User/Account Metadata│   / gRPC │   • Multi-Asset / Currencies  │  
                                     └─────────────┬────────────┘          └───────────────┬───────────────┘  
                                                   │                                       │  
                                                   ▼                                       ▼  
                                     ┌──────────────────────────┐          ┌───────────────────────────────┐  
                                     │ App Database (PostgreSQL)│          │ Ledger Database (PostgreSQL)  │  
                                     │ (Users, App Settings)    │          │ (Immutable Postings & Logs)   │  
                                     └──────────────────────────┘          └───────────────────────────────┘

#### **Updated Tech Stack**

> * **Core Accounting Engine:** Formance Ledger *(Open Source / Docker Container)*  
> * **Backend API Framework:** NestJS *(TypeScript, Modular Architecture, Strict Services)*  
> * **App Data Store / ORM:** PostgreSQL \+ Prisma or Drizzle *(For user profiles, auth, app metadata)*  
> * **Bank Integration Strategy:** Open Banking Aggregators (Belvo/Pluggy for LatAm, GoCardless/SimpleFIN elsewhere) OR Manual CSV/OFX Parsers.

### **2\. Identified Missing Architecture Pieces**

To bring this setup to production, three critical infrastructural components remain unassigned:

#### **Gap A: Syncing & Background Job Orchestration**

Financial apps cannot process bank webhooks or scheduled statement imports on the main HTTP API thread without blocking traffic or risking half-processed state updates.

#### **Gap B: Bank Credential & Sensitive Token Storage**

Open banking OAuth tokens, refresh keys, and account identifiers must be encrypted at rest with field-level cryptography, isolated from normal database logs.

#### **Gap C: Analytical Reporting Layer (OLAP vs. OLTP)**

Formance Ledger is optimized for real-time transactional writes (OLTP). Generating complex multi-year net worth graphs or spending trends directly from Formance can become slow as posting history scales into millions of rows.

### **3\. Decisions Required: Suggestions to Bridge the Gaps**
