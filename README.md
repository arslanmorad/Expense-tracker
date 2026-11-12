
# SpendWise — Personal Finance & Expense Tracker
🚀 Live Demo: https://expense-tracker-lake-seven-35.vercel.app

A full-stack personal finance tracker that lets users manage multiple accounts, record transactions, transfer money between accounts, set budgets, track recurring payments, and view reports — all with correct money handling, atomic transfers, concurrency control, and a complete audit trail.

**This is not just a CRUD app.** Money-handling systems have real correctness requirements: balances must always be accurate, history must never silently change, and concurrent edits must not corrupt data. The test suite (~100 tests) found and fixed **12+ production bugs** — see [Bugs Found Via Testing](#bugs-found-via-testing).

---

## Features

### Accounts
- Create and manage multiple accounts (Checking, Savings, Credit Card, Cash, Other)
- Each account has a name, type, currency, and balance
- Balance is always derivable from transaction history — never overwritten freely
- Per-user data isolation (one user can never see another user's accounts)

### Transactions
- Record income and expenses with amount, date, category, description
- **Soft delete only** — transactions are never hard-deleted; "deleting" creates a reversing entry and the original remains visible in history
- All monetary values stored as `decimal` (see below for why)

### Transfers Between Accounts
- Move money between accounts (e.g., Checking → Savings)
- Modeled as two linked entries (debit + credit) that always net to zero
- **Created atomically** — if one side fails, neither persists
- **Cross-currency transfers rejected** — no silent USD → EUR conversion

### Categories
- User-defined categories for income and expenses
- Each transaction belongs to exactly one category
- Category ownership validated on every transaction create/update

### Recurring Transactions
- Define rules like "Rent, $1200, Checking, 1st of the month"
- Explicit **`Type` field** (`Income` / `Expense`) — no sign-of-amount convention
- Amounts always positive; `Type` determines direction
- Background service generates real transactions when the scheduled date arrives
- Supports **Daily / Weekly / Bi-Weekly / Monthly / Yearly** frequencies

### Budgets & Alerts
- Set monthly budgets per category (e.g., "$500/month on Dining")
- Track actual **expense** spend against budget (income excluded)
- Background service detects overspend and generates in-app alerts
- **One alert per category per month** — no duplicate notifications

### CSV Import
- Import bank statements from CSV
- Duplicate detection (same date + amount + description)
- Malformed rows reported individually — the whole import never crashes
- Preview parsed transactions before committing
- Import is **transactional** — all-or-nothing per batch

### Reporting
- Monthly spend by category
- Balance-over-time / net worth trend
- Income vs. expense summary for a selected date range

### Authentication & Authorization
- JWT-based authentication
- All financial data scoped to the logged-in user
- Users can never query another user's data, even by guessing IDs

### Concurrency Control
- **Optimistic concurrency** on accounts via a `RowVersion` concurrency token
- A `SaveChangesInterceptor` automatically bumps the token on every save — services can't forget
- Concurrent edits throw `DbUpdateConcurrencyException`, caught by services and surfaced as a clean error instead of silently overwriting
- **Verified by tests**: two concurrent writes → exactly one succeeds, the other fails with a concurrency error

### Audit Trail
- Every transaction change (create, edit, soft-delete) is recorded with a timestamp
- Includes old and new values, the acting user, and the entity ID

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Backend | ASP.NET Core 10 |
| ORM | Entity Framework Core |
| Database | PostgreSQL 15 |
| Auth | JWT + ASP.NET Identity |
| Validation | FluentValidation |
| Background Jobs | IHostedService / BackgroundService |
| Frontend | React (Vite) |
| Styling | CSS |
| Containerization | Docker + Docker Compose |
| Backend Hosting | Render |
| Frontend Hosting | Vercel |
| Database Hosting | Render Postgres |
| Testing | xUnit + FluentAssertions + Moq + EF Core InMemory + SQLite |

---

## Architecture
┌──────────────────────────────────────────┐
│ Frontend (React + Vite) │
│ Deployed on Vercel │
└──────────────────┬───────────────────────┘
│ HTTPS (JWT in header)
▼
┌──────────────────────────────────────────┐
│ Backend (ASP.NET Core Web API) │
│ Deployed on Render (Docker container) │
│ │
│ ├── Controllers │
│ ├── Services (business logic) │
│ ├── Background Services │
│ │ ├── BudgetAlertService │
│ │ └── RecurringRuleService │
│ └── Entity Framework Core │
└──────────────────┬───────────────────────┘
│ Npgsql
▼
┌──────────────────────────────────────────┐
│ PostgreSQL 15 │
│ Managed on Render │
└──────────────────────────────────────────┘

text

---

## Why I Used `decimal` (Not `float`/`double`) For Money

All monetary values use C#'s `decimal` type — never `float` or `double`.

`float` and `double` are binary floating-point types. They can't represent most decimal fractions exactly, so tiny rounding errors creep in:

```csharp
double a = 0.1;
double b = 0.2;
double sum = a + b;

Console.WriteLine(sum);  // 0.30000000000000004
The result should be 0.3, but double returns 0.30000000000000004. That tiny error is invisible in a single calculation — but over thousands of transactions, it compounds into real money being wrong.

decimal is a base-10 type designed for financial math. It represents 0.1 and 0.2 exactly, so 0.1 + 0.2 equals 0.3 — always.

Result: Account balances are always exact. Reports never drift. Money doesn't silently lose fractions of a cent.

## How To Run Locally
Prerequisites
Docker Desktop (for the database)

.NET SDK 10

Node.js 20+

1. Start the database
From the project root:

bash
docker-compose up postgres -d
This starts PostgreSQL 15 on port 5433.

2. Run the backend
bash
cd Backend
dotnet restore
dotnet build
dotnet run
Backend runs on http://localhost:7000.

3. Run the frontend
bash
cd Fronted
npm install
npm run dev
Frontend runs on http://localhost:5173 (or 3000 — check the terminal output).

4. Open the app
text
http://localhost:5173
Register a user, log in, and start using the app.

## Running With Docker Compose (Full Stack)
bash
docker-compose up --build
Then open http://localhost:3000.

To stop:

bash
docker-compose down
To reset the database:

bash
docker-compose down -v
docker-compose up -d --build
Environment Variables
Backend (.env or Render dashboard)
Variable	Description
ConnectionStrings__DefaultConnection	PostgreSQL connection string
JwtSettings__Key	JWT signing key (32+ chars)
JwtSettings__Issuer	JWT issuer
JwtSettings__Audience	JWT audience
EmailSettings__SmtpHost	SMTP host
EmailSettings__SmtpPort	SMTP port
EmailSettings__SmtpUsername	SMTP username
EmailSettings__SmtpPassword	SMTP password / app password
EmailSettings__FromEmail	Sender email
EmailSettings__FromName	Sender display name
ASPNETCORE_ENVIRONMENT	Production in prod, Development locally
Frontend (.env or Vercel dashboard)
Variable	Description
VITE_API_URL	Backend API base URL
Note: Secrets are never committed to this repo. appsettings.json contains placeholders only; real values live in appsettings.Development.json (gitignored) or platform environment variables.

## Database Migrations
Migrations are applied automatically on backend startup. To create a new migration:

bash
cd Backend
dotnet ef migrations add <MigrationName>
To apply manually:

bash
dotnet ef database update

## Project Structure
text
Expense-tracker/
├── Backend/
│   ├── Attributes/
│   ├── Controllers/
│   ├── DTOs/
│   ├── Migrations/
│   ├── Models/
│   ├── Services/
│   ├── Validators/
│   ├── Program.cs
│   ├── appsettings.json
│   └── Dockerfile
├── Fronted/
│   ├── src/
│   ├── public/
│   ├── package.json
│   ├── vercel.json
│   └── Dockerfile
├── Tests/
│   ├── Services/          ← service-level tests
│   ├── Controllers/       ← HTTP integration tests
│   └── Infrastructure/    ← test base classes (InMemory + SQLite)
├── docker-compose.yml
└── README.md

## Testing
The project includes ~100 automated tests across service, controller, and integration layers.

Test infrastructure
Provider	Used for	Why
EF Core InMemory	Service logic tests, controller tests	Fast, no DB setup
SQLite (in-memory, shared connection)	Concurrency tests, transaction-requiring tests	Real transaction + concurrency token enforcement
Moq	UserManager mocking	Isolates services from Identity
FluentAssertions	All assertions	Readable failure messages
Why two providers? InMemory doesn't enforce IsConcurrencyToken or support BeginTransactionAsync. For tests that require real database behavior, we use SQLite in-memory (shared connection per test class).

Test coverage
Area	What's tested	~Tests
AccountService	CRUD, per-user isolation (IDOR), soft-delete, audit trail, balance immutability on update, same-user multiple accounts	~19
TransactionService	Income/expense creation, balance update, zero/negative rejection, invalid type rejection, category validation, type-change on update, soft-delete reversal, IDOR	~20
TransferService	Double-entry atomicity, both balances updated, insufficient funds, same-account rejection, ownership, currency mismatch, soft-deleted account guards, exact-balance boundary	~18
Balance invariants	balance == starting + Σ(signed non-deleted transactions) across create / update / delete / soft-delete	3
Concurrency	Two concurrent transfers never overdraft, stale write throws DbUpdateConcurrencyException, balance invariant under mixed concurrent ops	3
Recurring rules	All frequency branches (Daily/Weekly/Bi-Weekly/Monthly/Yearly/unknown), IDOR on update/delete, deleted-account guard, create-doesn't-touch-balance, type validation	~19
Budget alerts	Threshold exceeded creates alert, under-threshold silent, exactly-at-limit silent, multiple categories, no duplicate alerts per month, income doesn't count as spending	~6
CSV import	Valid parse, malformed rows reported individually, duplicates skipped, empty file, null description, invalid date, negative amount handling	~7
Controllers (HTTP)	Auth (401 without token), happy path (200), validation (400), IDOR (other user's resource), round-trip (create → list), list isolation	~15
Total: ~100 tests, all green.

## How to run
bash
cd Tests
dotnet test
Or filtered:

bash
# Just concurrency tests
dotnet test --filter "FullyQualifiedName~ConcurrencyTests"

# Just recurring rule tests
dotnet test --filter "FullyQualifiedName~RecurringRuleTests"
Bugs Found Via Testing
12+ production bugs found and fixed via tests. This is why the test suite exists — not "coverage for coverage's sake," but because money-handling code breaks in subtle ways.

#	Bug	Found by	Severity
1	Missing IsConcurrencyToken() configuration — RowVersion existed but EF ignored it. Two concurrent updates silently overwrote each other (lost-update race). The catch (DbUpdateConcurrencyException) blocks in services could never fire.	ConcurrencyTests	🔴 Critical
2	Zero/negative amounts accepted on TransactionService — a negative "expense" would increase the account balance (money created from nothing)	TransactionsServiceTests	🔴 Critical
3	Zero/negative amounts accepted on TransferService — same class of bug	TransferServiceTests	🔴 Critical
4	Cross-currency transfers silently allowed — USD → EUR with no conversion, corrupting both balances	TransferServiceTests	🔴 Critical
5	Missing category validation on transactions — CategoryId 999 would persist as an orphaned FK	TransactionsServiceTests	🟠 Medium
6	Type-change balance corruption in UpdateAsync — changing a transaction from Income to Expense didn't reverse the old amount before applying the new	TransactionsServiceTests	🔴 Critical
7	Recurring rule AddMonths(30) for Monthly frequency — a monthly rent rule would fire once and then not for 2.5 years	RecurringRuleTests	🔴 Critical
8	Recurring-rule transactions missing UserId — existed in DB but invisible to the user (all list queries filter by UserId)	RecurringRuleTests	🔴 Critical
9	Recurring-rule balance never updated — transactions created but account.Balance unchanged	RecurringRuleTests	🔴 Critical
10	Recurring-rule creation mutated balance — creating a rule immediately moved money (money from nothing at rule creation)	RecurringRuleTests	🔴 Critical
11	Budget alert duplicates — background service created a new notification every hour while over budget	BudgetAlertTests	🟠 Medium
12	Budget treated income as spending — Sum(t.Amount) without Type == "Expense" filter meant income inflated budget usage	BudgetAlertTests	🟠 Medium
13	CSV import persisted rejected transactions — insufficient-balance rows were added to the context before the balance check, so they saved anyway	CsvImportConfirmTests	🟠 Medium
Result: each fix is now covered by a regression test. Changing any of this logic would fail a test before it reaches production.

Why these tests matter
Money-handling systems have real correctness requirements that code review alone cannot catch:

Balance calculation — proves account.Balance == Σ signed non-deleted transactions, across creates, edits, deletes, and transfers

Transfer atomicity — verifies both sides of a transfer persist, or neither does

Soft-delete preservation — deleted transactions remain in history; balances still compute correctly

Concurrency — optimistic concurrency returns a clean error instead of silently overwriting

IDOR prevention — a user cannot read, update, or delete another user's resources even by guessing IDs

Input validation — zero/negative amounts, invalid types, cross-currency transfers, and non-existent categories are rejected before they corrupt data

Deployment
Backend: Deployed on Render as a Docker container, auto-deploying from main

Frontend: Deployed on Vercel, auto-deploying from main

Database: Render-managed PostgreSQL 15

Every push to main triggers automatic redeployment of both services.

Known Limitations
Render free tier: backend sleeps after 15 minutes of inactivity — the first request after sleep takes 30–60 seconds

In-memory cache (if used) resets on backend restart. For production scale, use Redis

Rate limiter (if used) is in-memory and resets on restart. For multi-instance deployments, use a shared store

License
MIT
