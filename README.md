SpendWise — Personal Finance & Expense Tracker
🚀 Live Demo: https://expense-tracker-lake-seven-35.vercel.app

A full-stack personal finance tracker that lets users manage multiple accounts, record transactions, transfer money between accounts, set budgets, track recurring payments, and view reports — all with correct money handling, atomic transfers, concurrency control, and a complete audit trail.

This is not just a CRUD app. Money-handling systems have real correctness requirements: balances must always be accurate, history must never silently change, and concurrent edits must not corrupt data.

Features
Accounts
Create and manage multiple accounts (Checking, Savings, Credit Card, Cash, Other)
Each account has a name, type, currency, and balance
Balance is always derivable from transaction history — never overwritten freely
Per-user data isolation (one user can never see another user's accounts)
Transactions
Record income and expenses with amount, date, category, description
Soft delete only — transactions are never hard-deleted; "deleting" creates a reversing entry and the original remains visible in history
All monetary values stored as decimal (see below for why)
Transfers Between Accounts
Move money between accounts (e.g., Checking → Savings)
Modeled as two linked entries (debit + credit) that always net to zero
Created atomically — if one side fails, neither persists
Categories
User-defined categories for income and expenses
Each transaction belongs to exactly one category
Recurring Transactions
Define rules like "Rent, $1200, Checking, 1st of the month"
Projected upcoming transactions visible without pre-inserting future rows
Background service generates real transactions when the scheduled date arrives
Budgets & Alerts
Set monthly budgets per category (e.g., "$500/month on Dining")
Track actual spend against budget in real time
Background service detects overspend and generates in-app alerts
CSV Import
Import bank statements from CSV
Duplicate detection (same date + amount + description)
Malformed rows reported individually — the whole import never crashes
Preview parsed transactions before committing
Reporting
Monthly spend by category
Balance-over-time / net worth trend
Income vs. expense summary for a selected date range
Authentication & Authorization
JWT-based authentication
All financial data scoped to the logged-in user
Users can never query another user's data, even by guessing IDs
Concurrency Control
Optimistic concurrency on accounts via RowVersion token
Concurrent edits return a clean error instead of silently overwriting
Audit Trail
Every transaction change (create, edit, soft-delete) is recorded with a timestamp
Tech Stack
Layer	Technology
Backend	ASP.NET Core 10
ORM	Entity Framework Core
Database	PostgreSQL 15
Auth	JWT + ASP.NET Identity
Validation	FluentValidation
Background Jobs	IHostedService / BackgroundService
Frontend	React (Vite)
Styling	CSS
Containerization	Docker + Docker Compose
Backend Hosting	Render
Frontend Hosting	Vercel
Database Hosting	Render Postgres
Architecture
┌──────────────────────────────────────────┐
│  Frontend (React + Vite)                 │
│  Deployed on Vercel                      │
└──────────────────┬───────────────────────┘
                   │ HTTPS (JWT in header)
                   ▼
┌──────────────────────────────────────────┐
│  Backend (ASP.NET Core Web API)          │
│  Deployed on Render (Docker container)   │
│                                          │
│  ├── Controllers                         │
│  ├── Services (business logic)           │
│  ├── Background Services                 │
│  │   ├── BudgetAlertService              │
│  │   └── RecurringRuleService            │
│  └── Entity Framework Core               │
└──────────────────┬───────────────────────┘
                   │ Npgsql
                   ▼
┌──────────────────────────────────────────┐
│  PostgreSQL 15                           │
│  Managed on Render                       │
└──────────────────────────────────────────┘
Why I Used decimal (Not float/double) For Money
All monetary values use C#'s decimal type — never float or double.

float and double are binary floating-point types. They can't represent most decimal fractions exactly, so tiny rounding errors creep in:

double a = 0.1;
double b = 0.2;
double sum = a + b;

Console.WriteLine(sum);  // 0.30000000000000004
The result should be 0.3, but double returns 0.30000000000000004. That tiny error is invisible in a single calculation — but over thousands of transactions, it compounds into real money being wrong.

decimal is a base-10 type designed for financial math. It represents 0.1 and 0.2 exactly, so 0.1 + 0.2 equals 0.3 — always.

Result: Account balances are always exact. Reports never drift. Money doesn't silently lose fractions of a cent.

How To Run Locally
Prerequisites
Docker Desktop (for the database)
.NET SDK 10
Node.js 20+
1. Start the database
From the project root:

docker-compose up postgres -d
This starts PostgreSQL 15 on port 5433.

2. Run the backend
cd Backend
dotnet restore
dotnet build
dotnet run
Backend runs on http://localhost:7000.

3. Run the frontend
cd Fronted
npm install
npm run dev
Frontend runs on http://localhost:5173 (or 3000 — check the terminal output).

4. Open the app
http://localhost:5173
Register a user, log in, and start using the app.

Running With Docker Compose (Full Stack)
To run everything in containers:

docker-compose up --build
Then open http://localhost:3000.

To stop:

docker-compose down
To reset the database:

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

Database Migrations
Migrations are applied automatically on backend startup. To create a new migration:

cd Backend
dotnet ef migrations add <MigrationName>
To apply manually:

dotnet ef database update
Project Structure
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
├── docker-compose.yml
└── README.md
Testing
The project includes 100+ automated tests covering the core financial logic and API surface.

Test coverage
Area	What's tested
AccountController	CRUD operations, per-user data isolation, validation, error responses
TransferController	Atomic transfers, insufficient funds, cross-account consistency
TransactionController	Income/expense creation, soft delete, history preservation
AccountService	Balance calculation from transaction history, concurrency handling
TransferService	Double-entry integrity, rollback on failure, two-sided consistency
TransactionService	Audit trail, reversal logic, soft-delete behavior
Why these tests matter
Money-handling systems have real correctness requirements:

Balance calculation — proves an account's balance always equals the sum of its transaction history, across creates, edits, deletes, and transfers
Transfer atomicity — verifies both sides of a transfer persist, or neither does
Soft-delete preservation — deleted transactions remain in history; balances still compute correctly
Concurrency — optimistic concurrency returns a clean error instead of silently overwriting
Run tests:

cd Tests
dotnet run
Deployment
Backend: Deployed on Render as a Docker container, auto-deploying from main
Frontend: Deployed on Vercel, auto-deploying from main
Database: Render-managed PostgreSQL 15
Every push to main triggers automatic redeployment of both services.

Known Limitations
In-memory cache (if used) resets on backend restart
Render free tier: backend sleeps after 15 minutes of inactivity — first request after sleep takes 30–60 seconds
License
MIT
