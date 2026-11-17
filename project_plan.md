# ShiftSynth — Project Architecture & Implementation Plan

## 1. Executive Summary & Project Vision

**ShiftSynth** is a comprehensive web platform built upon the **MVC (Model–View–Controller)** architectural pattern for freelancers to manage their gigs, deadlines, task execution, client communication, and financial transactions in one unified workspace.

### Mandatory Core Pillars (Primary Scope)
1. **Identity, Authentication & Account Semantics**:
   - Full account lifecycle including **Site Registration**, **Email Confirmation** (verification link sent via email), **Login** (JWT Access & Refresh tokens), **Authenticated Password Change**, and **Forgot / Reset Password** (via tokenized email link).
   - **Site-Level Authorization Only**:
     - `USER`: Standard registered freelancers who manage gigs, unified calendars, per-gig Kanban boards, client contacts, payments, and email invoices.
     - `ADMIN`: System administrators / developers with platform oversight, user moderation, and system diagnostic permissions.
2. **Unified Deadline Calendar + Per-Gig Kanban Boards**:
   - **Unified Deadline Calendar**: A single master calendar that displays **all** accepted gigs and deadlines across the freelancer's entire workload. Accepting a gig automatically triggers conflict detection and populates the deadline block onto the unified calendar.
   - **Dedicated Per-Gig Kanban Board (`1 Gig : 1 Kanban Board`)**: While the calendar is unified, **each accepted gig automatically receives its own separate Kanban board** (`TODO`, `IN_PROGRESS`, `REVIEW`, `DONE`) to aid the freelancer in dividing and tracking tasks specific to that gig.
3. **Client Contacts & Communication Hub (Contacts Page)**:
   - A dedicated Contacts page where freelancers maintain a directory of their clients (contact details, company info, notes) and communicate directly with them through logged messages and email dispatch linked to their gigs.
4. **Payment & Banking Transaction System**:
   - Processes and records structured banking/payment transaction data (`amount`, `currency`, `bank_ref_code`, `masked_iban_acct`, `status`, timestamps) tied to a specific gig and client contact.
   - Supports **Payments to the Freelancer** as a core mandatory feature, with **Client Refunds** architecturally supported in the data model as a **Planned Feature**.
5. **Automated Email Invoicing / Receipt System**:
   - Whenever a payment transaction (or planned refund) is recorded, the platform automatically generates a structured invoice/receipt containing the banking-transaction details and emails it directly to the freelancer's client.
6. **MVC Architecture Hardened with DTOs & DAOs**:
   - **View (`Frontend/`)**: Angular (TypeScript) SPA rendering the UI, Unified Calendar, Per-Gig Kanban boards, Contacts hub, and Payment/Invoice views.
   - **Controller (`Backend/src/controllers/` & `dtos/`)**: Express.js route handlers protected by **Inbound & Outbound DTOs (Data Transfer Objects)** to prevent mass assignment and sensitive field exposure.
   - **Model (`Backend/src/services/`, `daos/`, & `PostgreSQL`)**: Domain business rules and **DAOs (Data Access Objects)** encapsulating Prisma ORM transactions and PostgreSQL persistence.

---

## 2. Finalized Technology Stack (MVC Aligned)

| MVC Role | Layer | Technology | Language | Role & Rationale |
| :--- | :--- | :--- | :--- | :--- |
| **VIEW (`V`)** | **Frontend SPA** | **Angular** (Standalone Architecture) | TypeScript | Enterprise-grade SPA framework with strict TypeScript DTO contracts, Angular Signals, and RxJS for reactive state across calendar, Kanban, contacts, and payment views. |
| **VIEW (`V`)** | **Calendar & Kanban UI** | **FullCalendar / `angular-calendar`** + **Angular CDK Drag-and-Drop (`@angular/cdk/drag-drop`)** | TypeScript / SCSS | Powers the **Unified Deadline Calendar** (Month/Week/Day views) and interactive drag-and-drop **Per-Gig Kanban Boards**. |
| **CONTROLLER (`C`)** | **Backend API & Routing** | **Node.js + Express.js** | JavaScript (ES6+ Modules) | Handles RESTful HTTP requests from the Angular View, enforces site-level role guards (`USER` vs `ADMIN`), validates Inbound DTOs, and serializes Outbound DTOs. |
| **MODEL (`M`)** | **Domain Services & DAOs** | **Express Services + DAO Layer + Prisma Client** | JavaScript + Prisma DSL | Encapsulates business rules (conflict detection, per-gig Kanban provisioning, payment recording, invoice generation) and isolates all SQL queries inside `DAOs`. |
| **MODEL (`M`)** | **Relational Database** | **PostgreSQL + Prisma Migrate** | SQL + `JSONB` | ACID-compliant relational database ensuring atomic transactions across gigs, calendar events, Kanban boards, banking transactions, and invoices. |
| **INFRA / EMAIL** | **Transactional Email Engine** | **Nodemailer (SMTP / SendGrid / Mailtrap)** | JavaScript + HTML Templates | Dispatches **Automated Client Email Invoices/Receipts**, **Account Email Confirmation links**, **Forgot Password Reset links**, and **Client Contact messages**. |
| **SECURITY** | **Auth & DTO Validation** | **JWT**, **bcrypt**, **crypto**, **Zod / Joi**, **Helmet**, **CORS** | JavaScript | Secures password hashing, email verification/reset tokens, HTTP headers, and strict DTO payload whitelisting. |

---

## 3. Visual Platform Architecture (MVC + DTO/DAO Model)

### 3.1 Full-Stack MVC System Architecture (View $\leftrightarrow$ Controller $\leftrightarrow$ Model)

The visual diagram below illustrates the complete **MVC** architecture across `Frontend/` (**View**), `Backend/` (**Controller** & **Model**), **PostgreSQL**, and the **Transactional Email Gateway**:

```text
+=========================================================================================+
|                   [ V ] VIEW LAYER — FRONTEND WORKSPACE (Angular + TS)                  |
|                                                                                         |
|  +---------------------------------------------------+   +---------------------------+  |
|  | Presentation Views (`src/app/features/`)          |   | Core Guards & State       |  |
|  | - Auth Views (Register, Verify, Login, Reset Pwd) |   | - AuthGuard & AdminGuard  |  |
|  | - Gig Marketplace & Gig Workspace                 |<->| - JwtInterceptor          |  |
|  | - Unified Deadline Calendar (All Gigs)            |   | - Signals & RxJS Services |  |
|  | - Per-Gig Kanban Board (CDK Drag & Drop Tasks)    |   +-------------+-------------+  |
|  | - Client Contacts & Communication Hub             |                 |                |
|  | - Payments & Email Invoice History                |                 v                |
|  | - Site Administrator Panel (`ADMIN` only)         |   +---------------------------+  |
|  +---------------------------------------------------+   | Angular HttpClient        |  |
|                                                          | + Typed Request/Resp DTOs |  |
|                                                          +-------------+-------------+  |
+========================================================================|================+
                                                                         |
                                  HTTPS JSON                             |
                 (View <-> Controller via Request/Response DTOs)         |
                                                                         v
+=========================================================================================+
|          [ C ] CONTROLLER LAYER — BACKEND ENTRYPOINT (Node.js + Express.js)             |
|                                                                                         |
|  +-----------------------------------------------------------------------------------+  |
|  | 1. API EDGE & SECURITY MIDDLEWARE                                                 |  |
|  |    [ Helmet / CORS / Rate Limiter ] ---> [ JWT Auth ] ---> [ Site Role Guard ]    |  |
|  |                                                            (USER vs ADMIN only)   |  |
|  +-----------------------------------------------------------------------------------+  |
|                        |                                                 ^              |
|                        v (Raw Request JSON)                              | (Safe JSON)  |
|  +-------------------------------------------+   +-----------------------------------+  |
|  | 2A. INBOUND DTO VALIDATION LAYER          |   | 2B. OUTBOUND DTO SANITIZER LAYER  |  |
|  |     - Whitelists allowed request fields   |   |     - Formats Model data for View |  |
|  |     - Blocks Mass Assignment attacks      |   |     - Strips passwordHash, token  |  |
|  |     - Validates Banking & Contact inputs  |   |       hashes & raw bank secrets   |  |
|  +-------------------------------------------+   +-----------------------------------+  |
|                        |                                                 ^              |
|                        v (Validated Request DTO)                         | (Model Data) |
|  +-----------------------------------------------------------------------------------+  |
|  | 3. EXPRESS ROUTE CONTROLLERS (`src/controllers/` & `src/routes/`)                 |  |
|  |    [ AuthCtrl ] [ GigCtrl ] [ CalendarCtrl ] [ KanbanCtrl ] [ ContactCtrl ]       |  |
|  |    [ PaymentCtrl ] [ InvoiceCtrl ] [ AdminCtrl ]                                  |  |
|  +-----------------------------------------------------------------------------------+  |
+===========================================|=============================================+
                                            |
                          Controller Invokes Model Business Rules
                                            |
                                            v
+=========================================================================================+
|          [ M ] MODEL LAYER — DOMAIN SERVICES, DAOs, EMAIL ENGINE & POSTGRESQL           |
|                                                                                         |
|  +--------------------------------------------------------+  +-----------------------+  |
|  | 1. MODEL DOMAIN SERVICES (`src/services/`)             |  | EMAIL / SMTP GATEWAY  |  |
|  |    - AuthService (Verify Email, Forgot/Change Password)|  | (`mailer.util.js`)    |  |
|  |    - GigService (Accept Gig -> Calendar + Kanban Init) |->| - Email Confirmation  |  |
|  |    - CalendarService (Unified Deadlines & Conflict Ck) |  | - Password Reset Link |  |
|  |    - KanbanService (Per-Gig Board & Task Division)     |  | - Client Banking      |  |
|  |    - ContactService (Client Directory & Messaging)     |  |   Invoice Receipts    |  |
|  |    - PaymentService & InvoiceService (Tx & Receipts)   |  | - Client Contact Mail |  |
|  +---------------------------+----------------------------+  +-----------------------+  |
|                              |                                                          |
|                              v (Encapsulated Persistence Calls Only)                    |
|  +-----------------------------------------------------------------------------------+  |
|  | 2. MODEL DATA ACCESS OBJECTS (`src/daos/`)                                        |  |
|  |    [ UserDAO ] [ AuthTokenDAO ] [ GigDAO ] [ CalendarEventDAO ] [ KanbanDAO ]     |  |
|  |    [ ClientContactDAO ] [ ContactMessageDAO ] [ TransactionDAO ] [ InvoiceDAO ]   |  |
|  |                                        |                                          |  |
|  |                                        v                                          |  |
|  |                   [ Prisma Client Singleton & $transaction ]                      |  |
|  +----------------------------------------|------------------------------------------+  |
|                                           |                                             |
|                                           v (TCP / Pooled SQL Connection)               |
|  +-----------------------------------------------------------------------------------+  |
|  | 3. POSTGRESQL RELATIONAL DATABASE (`prisma/`)                                     |  |
|  |    - Tables: users, auth_tokens, client_contacts, contact_messages, gigs,         |  |
|  |              kanban_boards, kanban_tasks, calendar_events, transactions,          |  |
|  |              email_invoices                                                       |  |
|  |    - Managed via Versioned SQL Migrations (`prisma migrate`)                      |  |
|  +-----------------------------------------------------------------------------------+  |
+=========================================================================================+
```

---

### 3.2 MVC Security Pipeline: How DTOs and DAOs Protect Sensitive Auth & Banking Data

Because **ShiftSynth** handles authentication tokens, client contact details, and banking/payment transaction records, every request and response must pass through strict **DTO** and **DAO** security boundaries:

```text
  [ VIEW LAYER ]                  [ CONTROLLER LAYER ]                    [ MODEL LAYER ]
+-----------------+     +------------------------------------+     +-------------------------+
| Angular Client  |     | INBOUND DTO VALIDATOR              |     | DOMAIN SERVICE          |
| Sends Request   |---->| - Validates types & required fields|---->| - Enforces ownership    |
| JSON Payload    |     | - Strips unauthorized properties   |     |   (Freelancer owns Gig, |
+-----------------+     |   (e.g. blocks `siteRole="ADMIN"`, |     |   Kanban, Contact & Tx) |
                        |   `isEmailVerified=true`)          |     +------------+------------+
                        +------------------------------------+                  |
                                                                                v
+-----------------+     +------------------------------------+     +-------------------------+
| Angular Client  |     | OUTBOUND DTO MAPPER                |     | DATA ACCESS OBJECT (DAO)|
| Receives Safe   |<----| - Selects only public View fields  |<----| - Encapsulates Prisma   |
| Response JSON   |     | - Masks sensitive banking numbers  |     | - Runs parameterized    |
+-----------------+     | - Strips `passwordHash` & tokens   |     |   queries on PostgreSQL |
                        +------------------------------------+     +-------------------------+
```

---

### 3.3 Core Workflow 1: Gig Acceptance $\rightarrow$ Unified Deadline Calendar + Per-Gig Kanban Provisioning

When a freelancer accepts a gig in the **View**, the **Model** executes an atomic database transaction (`prisma.$transaction`) that verifies schedule availability, adds the gig's deadline block to the **Unified Calendar**, and automatically provisions a **dedicated Kanban Board** exclusively for that gig:

```text
   [ VIEW: Freelancer clicks "Accept Gig" ]
                      |
                      v
   [ CONTROLLER: POST /api/v1/gigs/:gigId/accept (Inbound DTO Validated) ]
                      |
                      v
   +-----------------------------------------------------------------------+
   | MODEL (Atomic Transaction via GigDAO + CalendarEventDAO + KanbanDAO): |
   | 1. Query `calendar_events` on the Unified Calendar for time overlap:  |
   |      existing.start_time < gig.deadline_end_time                      |
   |      AND existing.end_time > gig.start_time                           |
   +------------------+----------------------------------------------------+
                      |
        +-------------+-------------+
        |                           |
  [ Conflict ]               [ Slot Free ]
        |                           |
        v                           v
   +----------+       +----------------------------------------------------+
   | ROLLBACK |       | 1. Update `gigs.status` -> 'ACCEPTED'              |
   | Return   |       | 2. Insert `calendar_events` entry into the         |
   | 409 DTO  |       |    UNIFIED DEADLINE CALENDAR (all gigs displayed)  |
   +----------+       | 3. Auto-Create a separate `kanban_boards` record   |
                      |    linked 1:1 to this `gig_id` (`TODO`, `IN_PROG`, |
                      |    `REVIEW`, `DONE` task columns)                  |
                      | 4. COMMIT TRANSACTION                              |
                      +-----------------------+----------------------------+
                                              |
                                              v
                      [ VIEW: Unified Calendar displays the new Gig        ]
                      [       & Gig's Dedicated Kanban Board is unlocked   ]
```

---

### 3.4 Core Workflow 2: Payment System & Automated Client Email Invoicing

When a payment transaction for a gig is processed, the **Model** persists the banking transaction data, generates an official invoice receipt record, and dispatches an HTML email receipt directly to the freelancer's client:

```text
   [ VIEW: Freelancer records/processes Gig Payment in Payments View ]
                                   |
                                   v
   [ CONTROLLER: POST /api/v1/payments (Validated via CreatePaymentRequestDTO) ]
                                   |
                                   v
   +-----------------------------------------------------------------------+
   | MODEL (PaymentService + TransactionDAO + InvoiceDAO Transaction):     |
   | 1. Verify Gig ownership & fetch linked `client_contacts` record       |
   | 2. Insert banking record into `transactions` table:                   |
   |    - `transaction_type`: 'PAYMENT' (or Planned Feature: 'REFUND')     |
   |    - `amount`, `currency`, `bank_ref_code`, `masked_iban_acct`        |
   |    - `status`: 'COMPLETED'                                            |
   | 3. Insert receipt record into `email_invoices` table:                 |
   |    - Generates unique `invoice_number` (e.g. "INV-2026-00042")        |
   |    - Stores immutable `receipt_snapshot` (JSONB banking & gig details)|
   +-------------------------------+---------------------------------------+
                                   |
                                   v
   +-----------------------------------------------------------------------+
   | MODEL (EmailService / Nodemailer SMTP Gateway):                       |
   | 1. Populate HTML Receipt Template with banking transaction info,      |
   |    freelancer details, client details, gig title, and amount paid     |
   | 2. Dispatch email to `client_contact.client_email`                    |
   | 3. Update `email_invoices.delivery_status` -> 'SENT' (`sent_at`)      |
   +-------------------------------+---------------------------------------+
                                   |
                                   v
   [ CONTROLLER -> VIEW: Return 201 Created (Sanitized Transaction + Invoice DTO) ]
```

---

## 4. Relational Database Model & Entity-Relationship (ER) Diagram

As part of the **Model (`M`)** layer, PostgreSQL stores ten core relational tables managed via Prisma Migrate:

1. `users`: Registered freelancers and site administrators (`site_role`: `USER` or `ADMIN`, plus `is_email_verified`).
2. `auth_tokens`: Stores hashed tokens for **Email Confirmation** (`VERIFY_EMAIL`), **Forgot Password** (`RESET_PASSWORD`), and **Session Refresh** (`REFRESH_TOKEN`).
3. `client_contacts`: Powers the **Contacts Page**, storing each freelancer's client directory (name, email, phone, company, notes).
4. `contact_messages`: Logs communication messages and emails sent between the freelancer and their clients on the Contacts page.
5. `gigs`: Stores gig details, compensation, start/deadline timestamps, status, and links to the freelancer (`users`) and client (`client_contacts`).
6. `calendar_events`: Powers the **Unified Deadline Calendar**—stores auto-generated gig schedule/deadline blocks across all accepted gigs (`GIG_BOOKING`) and manual busy blocks (`PERSONAL_BLOCK`).
7. `kanban_boards`: Enforces a strict **1-to-1 (`1 : 1`) relationship with `gigs`** (`UNIQUE(gig_id)`), ensuring every gig has its own separate Kanban board.
8. `kanban_tasks`: Stores individual task cards (`TODO`, `IN_PROGRESS`, `REVIEW`, `DONE`) belonging to a specific gig's Kanban board.
9. `transactions`: Powers the **Payment System**, storing banking transaction metadata (`PAYMENT` and future planned `REFUND`).
10. `email_invoices`: Powers the **Email Invoicing System**, storing the generated invoice number, banking receipt snapshot (`JSONB`), recipient client email, and email delivery status (`PENDING`, `SENT`, `FAILED`).

### 4.1 Visual Entity-Relationship (ER) Diagram

```text
+-----------------------------------+          +-----------------------------------+
|               users               |          |            auth_tokens            |
+-----------------------------------+          +-----------------------------------+
| PK  id                (UUID)      | 1      N | PK  id                (UUID)      |
|     email             (VARCHAR,UQ)|--------->| FK  user_id           (UUID)      |
|     username          (VARCHAR,UQ)|          |     token_hash        (VARCHAR)   |
|     full_name         (VARCHAR)   |          |     token_type        (VERIFY_MAIL|
|     password_hash     (VARCHAR)   |          |                        RESET_PWD | |
|     is_email_verified (BOOLEAN)   |          |                        REFRESH)   |
|     site_role         (USER|ADMIN)|          |     expires_at        (TIMESTAMP) |
|     created_at        (TIMESTAMP) |          |     used_at           (TIMESTAMP) |
+---+---------+---------+---------+----------+ +-----------------------------------+
    | 1       | 1       | 1
    |         |         +-----------------------------------------+
    |         |                                                   | 1
    |         | 1                                                 v N
    |         |                                +-----------------------------------+
    |         |                                |          client_contacts          |
    |         |                                |     (Freelancer's Client Hub)     |
    |         |                                +-----------------------------------+
    |         |                                | PK  id                (UUID)      |
    |         |                                | FK  freelancer_id     (UUID)      |
    |         |                                |     client_name       (VARCHAR)   |
    |         |                                |     client_email      (VARCHAR)   |
    |         |                                |     client_phone      (VARCHAR)   |
    |         |                                |     company_name      (VARCHAR)   |
    |         |                                |     notes             (TEXT)      |
    |         |                                +-----+-----------------------+-----+
    |         |                                      | 1                     | 1
    |         |                                      |                       |
    |         | 1 (Freelancer)                       | 1 (Client)            |
    |         +--------------------+                 |                       |
    |                              |                 |                       |
    | 1                            v N               v N                     v N
+---+-------------------+      +-----------------------------------+   +-----------------------------------+
|    calendar_events    |      |               gigs                |   |         contact_messages          |
|  (Unified Calendar)   |      +-----------------------------------+   |   (Client Communication Log)      |
+-----------------------+      | PK  id                (UUID)      |   +-----------------------------------+
| PK  id        (UUID)  |      | FK  freelancer_id     (UUID)      |   | PK  id                (UUID)      |
| FK  user_id   (UUID)  |      | FK  client_contact_id (UUID)      |   | FK  contact_id        (UUID)      |
| FK  gig_id    (UUID)  |<-----|     title             (VARCHAR)   |   | FK  gig_id            (UUID, NULL)|
|     title     (VARCHAR| 0..N |     description       (TEXT)      |   |     subject           (VARCHAR)   |
|     start_time(TIME)  |      |     compensation      (DECIMAL)   |   |     message_body      (TEXT)      |
|     end_time  (TIME)  |      |     start_time        (TIMESTAMP) |   |     sent_via_email    (BOOLEAN)   |
|     event_type(ENUM)  |      |     deadline_end_time (TIMESTAMP) |   |     created_at        (TIMESTAMP) |
+-----------------------+      |     status            (ENUM)      |   +-----------------------------------+
                               +-----+-----------------------+-----+
                                     | 1                     | 1
                                     | (1:1 Per-Gig Board)   | (1:N Gig Transactions)
                                     v 1                     v N
+------------------------------------+--+      +-----------------------------------+
|             kanban_boards             |      |           transactions            |
|      (1 Separate Board Per Gig)       |      |    (Payment & Banking System)     |
+---------------------------------------+      +-----------------------------------+
| PK  id                    (UUID)      |      | PK  id                (UUID)      |
| FK  gig_id                (UUID, UQ)  |      | FK  gig_id            (UUID)      |
|     board_title           (VARCHAR)   |      | FK  freelancer_id     (UUID)      |
|     created_at            (TIMESTAMP) |      | FK  client_contact_id (UUID)      |
+-------------------+-------------------+      |     transaction_type  (PAYMENT |  |
                    | 1                        |                        *REFUND*)  |
                    |                          |     amount            (DECIMAL)   |
                    v N                        |     currency          (VARCHAR)   |
+---------------------------------------+      |     bank_ref_code     (VARCHAR,UQ)|
|             kanban_tasks              |      |     masked_iban_acct  (VARCHAR)   |
|     (Task Division For That Gig)      |      |     status            (ENUM)      |
+---------------------------------------+      |     processed_at      (TIMESTAMP) |
| PK  id                    (UUID)      |      +-----------------+-----------------+
| FK  board_id              (UUID)      |                        | 1
|     title                 (VARCHAR)   |                        | (1:1 Email Receipt)
|     description           (TEXT)      |                        v 1
|     column_status         (TODO |     |      +-----------------------------------+
|                            IN_PROG |  |      |          email_invoices           |
|                            REVIEW |   |      | (Automated Client Email Receipts) |
|                            DONE)      |      +-----------------------------------+
|     priority              (LOW|MED|HI)|      | PK  id                (UUID)      |
|     position_order        (INT)       |      | FK  transaction_id    (UUID, UQ)  |
|     due_date              (TIMESTAMP) |      | FK  client_contact_id (UUID)      |
+---------------------------------------+      |     invoice_number    (VARCHAR,UQ)|
                                               |     recipient_email   (VARCHAR)   |
                                               |     receipt_snapshot  (JSONB)     |
                                               |     delivery_status   (SENT|FAIL) |
                                               |     sent_at           (TIMESTAMP) |
                                               +-----------------------------------+
```

### 4.2 Relational Cardinality & Constraint Summary

| Parent Entity (`1`) | Child Entity (`N` / `1`) | Relationship Type | Key Rule & Behavior |
| :--- | :--- | :--- | :--- |
| `users` | `auth_tokens` | One-to-Many (`1 : N`) | Manages expiring tokens for email confirmation, forgot-password resets, and refresh sessions. |
| `users` | `client_contacts` | One-to-Many (`1 : N`) | Each freelancer maintains their own private directory of clients on the Contacts page. |
| `client_contacts` | `contact_messages` | One-to-Many (`1 : N`) | Tracks all communication messages and outbound emails between the freelancer and a client. |
| `users` & `client_contacts` | `gigs` | One-to-Many (`1 : N`) | Links each gig to the freelancer (`freelancer_id`) and the client (`client_contact_id`). |
| `users` & `gigs` | `calendar_events` | One-to-Many (`1 : N`) | **Unified Deadline Calendar**: all accepted gigs populate into one unified calendar per user, indexed on `(user_id, start_time, end_time)` for conflict detection. |
| `gigs` | `kanban_boards` | **One-to-One (`1 : 1`)** | **Per-Gig Kanban Board**: `gig_id` is `UNIQUE`, guaranteeing a separate Kanban board for each individual gig. |
| `kanban_boards` | `kanban_tasks` | One-to-Many (`1 : N`) | Holds the task division cards (`TODO`, `IN_PROGRESS`, `REVIEW`, `DONE`) for that specific gig's board. |
| `gigs` & `client_contacts` | `transactions` | One-to-Many (`1 : N`) | Stores banking payment records (`PAYMENT` now; `REFUND` reserved as a Planned Feature). |
| `transactions` | `email_invoices` | **One-to-One (`1 : 1`)** | Generates a unique banking receipt (`invoice_number` + `receipt_snapshot`) emailed automatically to the client. |

---

## 5. Workspace & Folder Structure (`Backend/` & `Frontend/`)

The project structure below is organized around the **MVC** pattern with dedicated **DTO** and **DAO** layers for all mandatory modules:

```text
ShiftSynth/
├── project_plan.md
├── README.md
├── docker-compose.yml                       # Local PostgreSQL + Mailpit/Mailtrap SMTP container
│
├── Backend/                                 # [C] CONTROLLER & [M] MODEL (Node.js + Express.js)
│   ├── package.json
│   ├── .env.example                         # DB_URL, JWT_SECRET, SMTP_HOST, SMTP_USER, SMTP_PASS
│   ├── prisma/
│   │   ├── schema.prisma                    # PostgreSQL relational schema (10 core tables)
│   │   ├── migrations/                      # Versioned SQL migrations via Prisma Migrate
│   │   └── seed.js                          # Initial ADMIN user & sample data
│   └── src/
│       ├── server.js                        # Express application bootstrap
│       ├── config/
│       │   ├── db.config.js                 # PrismaClient singleton instance
│       │   ├── env.config.js                # Environment variable validation
│       │   └── mail.config.js               # Nodemailer SMTP transport configuration
│       ├── utils/
│       │   ├── mailer.util.js               # Email dispatcher (Confirmation, Reset Pwd, Invoices, Contacts)
│       │   └── email-templates/             # HTML templates for receipts & auth emails
│       │       ├── confirm-email.template.js
│       │       ├── reset-password.template.js
│       │       ├── invoice-receipt.template.js
│       │       └── client-message.template.js
│       ├── middlewares/
│       │   ├── auth.middleware.js           # JWT verification & verified-email check
│       │   ├── role.middleware.js           # Site-level guard: requireRole('ADMIN' | 'USER')
│       │   ├── validate-dto.middleware.js   # Inbound DTO validator middleware
│       │   └── error.middleware.js          # Centralized error-to-DTO formatter
│       ├── dtos/                            # Security Boundary: Inbound & Outbound DTOs
│       │   ├── request/
│       │   │   ├── auth.request.dto.js      # Register, Login, VerifyEmail, ForgotPwd, ResetPwd, ChangePwd
│       │   │   ├── gig.request.dto.js       # Create, Update, Accept Gig DTOs
│       │   │   ├── calendar.request.dto.js  # Personal block & date-range query DTOs
│       │   │   ├── kanban.request.dto.js    # Create Task, Move Task Column/Order DTOs
│       │   │   ├── contact.request.dto.js   # Create Client Contact, Send Client Message DTOs
│       │   │   └── payment.request.dto.js   # Record Payment Transaction & Trigger Invoice DTOs
│       │   └── response/
│       │       ├── user.response.dto.js
│       │       ├── gig.response.dto.js
│       │       ├── calendar.response.dto.js
│       │       ├── kanban.response.dto.js
│       │       ├── contact.response.dto.js
│       │       ├── payment.response.dto.js
│       │       └── invoice.response.dto.js
│       ├── daos/                            # [M] Data Access Objects (Only layer querying Prisma)
│       │   ├── user.dao.js
│       │   ├── auth-token.dao.js
│       │   ├── gig.dao.js
│       │   ├── calendar-event.dao.js
│       │   ├── kanban.dao.js
│       │   ├── client-contact.dao.js
│       │   ├── contact-message.dao.js
│       │   ├── transaction.dao.js
│       │   └── email-invoice.dao.js
│       ├── services/                        # [M] Domain Business Logic & Orchestration
│       │   ├── auth.service.js              # Registration, email confirmation, password lifecycle
│       │   ├── user.service.js
│       │   ├── gig.service.js               # Gig CRUD & atomic acceptance orchestration
│       │   ├── calendar.service.js          # Unified calendar aggregation & conflict detection
│       │   ├── kanban.service.js            # Per-gig Kanban board & task division management
│       │   ├── contact.service.js           # Client directory & client email/message dispatch
│       │   ├── payment.service.js           # Banking transaction processing
│       │   └── invoice.service.js           # Automated receipt generation & client email delivery
│       ├── controllers/                     # [C] Express Route Controllers
│       │   ├── auth.controller.js
│       │   ├── user.controller.js
│       │   ├── gig.controller.js
│       │   ├── calendar.controller.js
│       │   ├── kanban.controller.js
│       │   ├── contact.controller.js
│       │   ├── payment.controller.js
│       │   ├── invoice.controller.js
│       │   └── admin.controller.js
│       └── routes/
│           └── index.js                     # REST API v1 router (/api/v1/*)
│
└── Frontend/                                # [V] VIEW LAYER (Angular + TypeScript)
    ├── package.json
    ├── angular.json
    ├── tsconfig.json
    └── src/
        ├── main.ts
        ├── styles.scss
        └── app/
            ├── app.config.ts
            ├── app.routes.ts
            ├── core/
            │   ├── guards/
            │   │   ├── auth.guard.ts
            │   │   └── admin.guard.ts       # Site-level ADMIN guard
            │   ├── interceptors/
            │   │   └── jwt.interceptor.ts
            │   └── services/
            │       └── auth.service.ts
            ├── shared/
            │   ├── dtos/                    # TypeScript DTO interfaces mirroring Backend DTOs
            │   │   ├── auth.dto.ts
            │   │   ├── user.dto.ts
            │   │   ├── gig.dto.ts
            │   │   ├── calendar-event.dto.ts
            │   │   ├── kanban.dto.ts
            │   │   ├── contact.dto.ts
            │   │   ├── payment.dto.ts
            │   │   └── invoice.dto.ts
            │   └── components/
            │       └── navbar/
            └── features/
                ├── auth/                    # Login, Register, Email Confirmation, Forgot/Change Password
                ├── dashboard/               # Freelancer overview (upcoming deadlines, recent payments)
                ├── gigs/                    # Gig creation, acceptance & gig workspace shell
                ├── calendar/                # Unified Deadline Calendar (displays all accepted gigs)
                ├── kanban/                  # Per-Gig Kanban Board (`@angular/cdk/drag-drop` task columns)
                ├── contacts/                # Client Contacts Directory & Client Communication page
                ├── payments/                # Payment Transaction form, banking log & Email Invoice status
                └── admin/                   # System Administrator / Developer panel
```

---

## 6. Phased MVC Implementation Roadmap (Mandatory Priorities First)

### Phase 1: Foundation, Database Model & SMTP Infrastructure
1. **Model Layer — PostgreSQL & Prisma Setup**:
   - Configure `docker-compose.yml` with PostgreSQL and a local SMTP test server (e.g., Mailpit / Mailtrap) for testing transactional emails.
   - Initialize Prisma in `Backend/`, implement the 10-table relational model from Section 4, and run `npx prisma migrate dev --name init_shiftsynth`.
2. **Controller & Email Infrastructure**:
   - Set up Express with security middleware (`helmet`, `cors`), Inbound/Outbound DTO base helpers, and `mailer.util.js` (Nodemailer).
3. **View Layer — Angular Workspace Setup**:
   - Initialize Angular inside `Frontend/` with routing, Angular Material / TailwindCSS, `@angular/cdk/drag-drop`, calendar libraries, and shared TypeScript DTO contracts.

### Phase 2: Identity & Full Authentication Semantics (Mandatory)
1. **Model & Controller Auth Pipeline**:
   - Implement `UserDAO` and `AuthTokenDAO`.
   - Build **Registration** (`POST /api/v1/auth/register`) $\rightarrow$ hashes password, creates unverified user, generates `VERIFY_EMAIL` token, and dispatches **Email Confirmation** link.
   - Build **Email Confirmation** (`POST /api/v1/auth/verify-email`) $\rightarrow$ validates token and marks `is_email_verified = true`.
   - Build **Login** (`POST /api/v1/auth/login`) $\rightarrow$ issues JWT Access & Refresh tokens and enforces site-level roles (`USER` vs `ADMIN`).
   - Build **Password Change** (`PATCH /api/v1/auth/change-password` for logged-in users) and **Forgot / Reset Password** (`POST /api/v1/auth/forgot-password` & `POST /api/v1/auth/reset-password` via emailed reset token).
2. **View Auth Pages**:
   - Build Register, Check-Your-Email / Verify Email callback page, Login, Forgot Password, Reset Password, and Account Security (Change Password) views.

### Phase 3: Client Contacts & Communication Hub (Mandatory #1)
1. **Model & Controller Contacts Module**:
   - Build `ClientContactDAO`, `ContactMessageDAO`, `ContactService`, and `ContactController` with strict Inbound/Outbound DTOs.
   - Allow freelancers to create, update, search, and manage client profiles (`client_contacts`).
   - Implement direct client communication (`POST /api/v1/contacts/:contactId/messages`): logs the message in `contact_messages` and dispatches an email directly to the client via `mailer.util.js`.
2. **View Contacts Page**:
   - Build `features/contacts/` featuring a searchable Client Contacts list, client detail drawer, linked gigs history, and an integrated "Message / Email Client" communication thread.

### Phase 4: Gigs, Unified Deadline Calendar & Per-Gig Kanban Boards (Mandatory #2)
1. **Model & Controller — Gigs, Unified Calendar & Per-Gig Kanban**:
   - Build `GigDAO`, `CalendarEventDAO`, and `KanbanDAO`.
   - Implement `GigService.acceptGig` inside an atomic `prisma.$transaction`:
     1. Run **Conflict Detection** on `calendar_events` to ensure the freelancer has no overlapping gig during `[start_time, deadline_end_time]`.
     2. Mark the gig as `ACCEPTED`.
     3. Insert the gig's schedule/deadline block into `calendar_events` so it appears on the **Unified Deadline Calendar**.
     4. Automatically create a dedicated `kanban_boards` record (`1 : 1` with `gig_id`) so that specific gig has its own isolated Kanban board.
   - Build `KanbanController` & `KanbanService` for creating, editing, reordering, and dragging tasks across columns (`TODO`, `IN_PROGRESS`, `REVIEW`, `DONE`) within a gig's board.
2. **View — Unified Calendar & Per-Gig Kanban Workspace**:
   - Build `features/calendar/` displaying **all accepted gigs and deadlines unified on one master calendar** with conflict alerts.
   - Build `features/kanban/` (`/gigs/:gigId/kanban`) using `@angular/cdk/drag-drop` so clicking any gig opens **that gig's dedicated Kanban board** for task division.

### Phase 5: Payment System & Automated Email Invoicing (Mandatory #3 & #4)
1. **Model & Controller — Payments & Email Receipts**:
   - Build `TransactionDAO`, `EmailInvoiceDAO`, `PaymentService`, and `InvoiceService`.
   - When a payment is submitted (`POST /api/v1/payments`):
     1. Validate the payload via `CreatePaymentRequestDTO` and mask sensitive account numbers (`masked_iban_acct`).
     2. Persist the banking transaction record in `transactions` (`transaction_type: 'PAYMENT'`, `bank_ref_code`, `amount`, `status: 'COMPLETED'`).
     3. Generate an `email_invoices` record with a unique `invoice_number` and a `JSONB` banking receipt snapshot.
     4. Automatically trigger `mailer.util.js` using `invoice-receipt.template.js` to email the banking-transaction receipt to the client (`client_contact.client_email`).
     5. Record `delivery_status = 'SENT'` and return the sanitized `PaymentResponseDTO` and `InvoiceResponseDTO`.
2. **View — Payments & Invoicing Dashboard**:
   - Build `features/payments/` where freelancers can process/record gig payments, inspect banking transaction logs, view generated receipt details, and verify that the client's email invoice was sent (with a "Resend Receipt Email" action if needed).

### Phase 6: System Administrator Panel, Security Audit & Verification
1. **Admin MVC Module**:
   - Build `AdminController` (guarded by `requireRole('ADMIN')`) and Angular `features/admin/` views for platform oversight and user management.
2. **End-to-End Verification**:
   - Verify Auth flow (Register $\rightarrow$ Confirm Email $\rightarrow$ Login $\rightarrow$ Forgot/Change Password).
   - Verify Gig Acceptance flow (Accept Gig $\rightarrow$ Conflict Check $\rightarrow$ Unified Calendar updates + Dedicated Per-Gig Kanban board initializes).
   - Verify Client Contacts communication and Payment $\rightarrow$ Automated Client Email Invoice receipt delivery.

---

## 7. Planned Future Features (Deferred Scope)

To keep the initial implementation focused on the mandatory core workflows above, the following capabilities are documented as **Planned Features** for subsequent releases:

1. **Freelancer Organizations (Peer Collectives) — *[Planned Feature]***:
   - Ability for groups of freelancers to band together into flat, peer-to-peer organizations (`organizations` and `organization_members` tables) to jointly offer services, review incoming gig requests, and accept gigs on behalf of the collective.
2. **Client Refund Processing & Refund Email Receipts — *[Planned Feature]***:
   - Extending the Payment and Email Invoicing systems (`transactions.transaction_type = 'REFUND'`) so freelancers or administrators can issue full or partial refunds and automatically email a banking refund receipt to the client.
3. **External Calendar Sync (iCal / Google Calendar) — *[Planned Feature / TBD]***:
   - Exporting the Unified Deadline Calendar as an `.ics` feed or syncing with external calendar providers.
