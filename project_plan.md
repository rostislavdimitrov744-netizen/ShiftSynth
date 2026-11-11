# ShiftSynth — Project Architecture & Implementation Plan

## 1. Executive Summary & Project Vision

**ShiftSynth** is a collaborative web platform built upon the **MVC (Model–View–Controller)** architectural pattern where freelancers can create and accept gigs, form flat peer-to-peer freelancer organizations to jointly provide and sell services, and rely on an intelligent, automated calendar that populates schedule blocks and prevents booking conflicts the moment a gig is accepted.

### Core Pillars
1. **MVC (Model–View–Controller) Foundation with DTO & DAO Security**:
   - **View (`Frontend/`)**: Angular (TypeScript) Single-Page Application responsible for presentation views, interactive calendars, and user interaction.
   - **Controller (`Backend/src/controllers/` & `routes/`)**: Node.js + Express.js request handlers that govern routing, authentication, and **DTO (Data Transfer Object)** validation/serialization between the View and the Model.
   - **Model (`Backend/src/services/`, `daos/`, & `PostgreSQL`)**: Domain business rules, **DAOs (Data Access Objects)**, Prisma ORM, and the relational PostgreSQL database managing state, transactions, and data integrity.
2. **Gig Marketplace & Service Offerings**: Freelancers (individually or as an organization) can publish services/gigs, review incoming gig requests, and accept work.
3. **Flat Freelancer Organizations (Collectives)**: Groups of freelancers can join forces under a shared organization banner to build and sell services together. Inside an organization, **all members are equals**—every member can review and accept gigs on behalf of the organization without bureaucratic role hierarchies.
4. **Site-Level Authorization Only**: There are only two platform-level roles:
   - `USER`: Standard registered users/freelancers who create profiles, join/create organizations, and publish, review, or accept gigs.
   - `ADMIN`: System administrators / developers with platform moderation, user management, and system oversight permissions.
5. **Automated Conflict-Safe Calendar**: Accepting a gig automatically triggers an atomic Model transaction that verifies schedule availability (conflict detection) and populates the freelancer's (and/or organization members') calendar in the View.

---

## 2. Finalized Technology Stack (MVC Aligned)

| MVC Role | Layer | Technology | Language | Role & Rationale |
| :--- | :--- | :--- | :--- | :--- |
| **VIEW (`V`)** | **Frontend SPA** | **Angular** (Standalone Components) | TypeScript | Renders the user interface, binds user actions to typed Request DTOs, and reactively updates views from Controller Response DTOs using Angular Signals & RxJS. |
| **VIEW (`V`)** | **UI & Calendar** | **Angular Material / TailwindCSS** + **FullCalendar / `angular-calendar`** | TypeScript / SCSS | Accessible UI primitives and rich interactive day/week/month time-grid views for the automated schedule. |
| **CONTROLLER (`C`)** | **Backend API & Routing** | **Node.js + Express.js** | JavaScript (ES6+ Modules) | Receives HTTP requests from the Angular View, enforces site-level role guards (`USER` vs `ADMIN`), validates Inbound DTOs, invokes Model logic, and returns sanitized Outbound DTOs. |
| **MODEL (`M`)** | **Domain Services & DAOs** | **Express Services + DAO Layer + Prisma Client** | JavaScript + Prisma DSL | Encapsulates core business rules (flat organization peer permissions, calendar conflict detection) and isolates all database queries inside Data Access Objects (`DAOs`). |
| **MODEL (`M`)** | **Relational Database** | **PostgreSQL + Prisma Migrate** | SQL + `JSONB` | ACID-compliant relational persistence for users, organizations, gigs, reviews, and indexed time-range calendar conflict queries, managed via versioned SQL migrations. |
| **CROSS-CUTTING** | **Security & DTO Validation** | **JWT (Access + Refresh)**, **bcrypt**, **Zod / Joi**, **Helmet**, **CORS** | JavaScript | Secures the Controller boundary against mass assignment, unauthorized access, and sensitive Model data exposure. |

---

## 3. Visual Platform Architecture (MVC + DTO/DAO Model)

### 3.1 Full-Stack MVC System Architecture (View $\leftrightarrow$ Controller $\leftrightarrow$ Model)

The visual diagram below illustrates the complete **MVC** architecture across `Frontend/` (the **View**) and `Backend/` + `PostgreSQL` (the **Controller** and **Model**), showing how **DTOs** protect the View–Controller boundary and **DAOs** protect the Controller–Model persistence boundary.

```text
+=========================================================================================+
|                   [ V ] VIEW LAYER — FRONTEND WORKSPACE (Angular + TS)                  |
|                                                                                         |
|  +-----------------------+     +-----------------------+     +-----------------------+  |
|  |  Presentation Views   |     |   Route Guards &      |     |   Reactive View State |  |
|  |  - Gig Marketplace    |<--->|   Interceptors        |<--->|   (Signals & RxJS)    |  |
|  |  - Organization Hub   |     |   - AuthGuard         |     +-----------+-----------+  |
|  |  - Auto-Calendar View |     |   - AdminGuard        |                 |              |
|  |  - Site Admin Panel   |     |   - JwtInterceptor    |                 v              |
|  +-----------------------+     +-----------------------+     +-----------------------+  |
|                                                              |  Angular HttpClient   |  |
|                                                              |  + Typed DTO Models   |  |
|                                                              +-----------+-----------+  |
+==========================================================================|==============+
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
|  |     - Blocks Mass Assignment attacks      |   |     - Strips passwordHash & keys  |  |
|  +-------------------------------------------+   +-----------------------------------+  |
|                        |                                                 ^              |
|                        v (Validated Request DTO)                         | (Model Data) |
|  +-----------------------------------------------------------------------------------+  |
|  | 3. EXPRESS ROUTE CONTROLLERS (`src/controllers/` & `src/routes/`)                 |  |
|  |    [ AuthCtrl ]  [ UserCtrl ]  [ OrganizationCtrl ]  [ GigCtrl ]  [ CalendarCtrl ]|  |
|  |    - Translates HTTP actions into Model operations & maps results to Outbound DTOs|  |
|  +-----------------------------------------------------------------------------------+  |
+===========================================|=============================================+
                                            |
                          Controller Invokes Model Business Rules
                                            |
                                            v
+=========================================================================================+
|          [ M ] MODEL LAYER — DOMAIN SERVICES, DAOs & POSTGRESQL DATABASE                |
|                                                                                         |
|  +-----------------------------------------------------------------------------------+  |
|  | 1. MODEL DOMAIN SERVICES (`src/services/`)                                        |  |
|  |    - Enforces Flat Organization Peer Rules (any member can accept/review gigs)    |  |
|  |    - Orchestrates Atomic Gig Acceptance & Calendar Conflict Detection             |  |
|  +-----------------------------------------------------------------------------------+  |
|                                            |                                            |
|                                            v (Encapsulated Persistence Calls)           |
|  +-----------------------------------------------------------------------------------+  |
|  | 2. MODEL DATA ACCESS OBJECTS (`src/daos/`)                                        |  |
|  |    +-----------+   +-------------------+   +-----------+   +-------------------+  |  |
|  |    |  UserDAO  |   |  OrganizationDAO  |   |  GigDAO   |   | CalendarEventDAO  |  |  |
|  |    +-----+-----+   +---------+---------+   +-----+-----+   +---------+---------+  |  |
|  |          +-------------------+-------------------+-------------------+            |  |
|  |                                        |                                          |  |
|  |                                        v                                          |  |
|  |                   [ Prisma Client Singleton & $transaction ]                      |  |
|  +----------------------------------------|------------------------------------------+  |
|                                           |                                             |
|                                           v (TCP / Pooled SQL Connection)               |
|  +-----------------------------------------------------------------------------------+  |
|  | 3. POSTGRESQL RELATIONAL DATABASE (`prisma/`)                                     |  |
|  |    +---------------------------------------+   +-------------------------------+  |  |
|  |    | Relational Model Tables               |   | Prisma Migrate                |  |  |
|  |    | - users              - gigs           |<--| - Versioned SQL Migrations    |  |  |
|  |    | - organizations      - gig_reviews    |   | - ACID Transactions & Indexes |  |  |
|  |    | - organization_members                |   +-------------------------------+  |  |
|  |    | - calendar_events                     |                                      |  |
|  |    +---------------------------------------+                                      |  |
|  +-----------------------------------------------------------------------------------+  |
+=========================================================================================+
```

---

### 3.2 MVC Security Pipeline: How DTOs and DAOs Protect the Model & View

To prevent direct coupling between the **View** (Angular client) and the **Model** (PostgreSQL entities) and protect against OWASP vulnerabilities (Mass Assignment, Sensitive Data Exposure, and Injection), every request and response flows through strict **DTO** and **DAO** gates:

```text
  [ VIEW LAYER ]                  [ CONTROLLER LAYER ]                    [ MODEL LAYER ]
+-----------------+     +------------------------------------+     +-------------------------+
| Angular Client  |     | INBOUND DTO VALIDATOR              |     | DOMAIN SERVICE          |
| Sends Request   |---->| - Validates types & required fields|---->| - Executes business     |
| JSON Payload    |     | - Strips unauthorized properties   |     |   rules & peer org      |
+-----------------+     |   (e.g. blocks `siteRole="ADMIN"`) |     |   membership checks     |
                        +------------------------------------+     +------------+------------+
                                                                                |
                                                                                v
+-----------------+     +------------------------------------+     +-------------------------+
| Angular Client  |     | OUTBOUND DTO MAPPER                |     | DATA ACCESS OBJECT (DAO)|
| Receives Safe   |<----| - Selects only public View fields  |<----| - Encapsulates Prisma   |
| Response JSON   |     | - Strips `passwordHash` & internal |     | - Runs parameterized    |
+-----------------+     |   Model metadata before responding |     |   queries on PostgreSQL |
                        +------------------------------------+     +-------------------------+
```

1. **Inbound DTOs (`Backend/src/dtos/request/`) — View $\rightarrow$ Controller Gate**:
   - Validate request bodies, query parameters, and route parameters before the Controller invokes the Model.
   - Explicitly whitelist allowed fields (e.g., `CreateGigRequestDTO` only permits `title`, `description`, `compensation`, `startTime`, `endTime`, `publishingOrgId`).
2. **Data Access Objects (`Backend/src/daos/`) — Model Persistence Gate**:
   - Controllers **never** talk to Prisma or PostgreSQL directly; they call Domain Services, which delegate all persistence operations to dedicated **DAO** classes (`UserDAO`, `OrganizationDAO`, `GigDAO`, `CalendarEventDAO`).
   - DAOs encapsulate transactional boundaries and keep database queries modular and maintainable.
3. **Outbound DTOs (`Backend/src/dtos/response/`) — Controller $\rightarrow$ View Gate**:
   - Raw Model entities returned from a DAO are **never** sent directly to the View.
   - Every response is mapped through an Outbound DTO (e.g., `UserResponseDTO`), guaranteeing that `passwordHash` or internal flags can never leak to the frontend.

---

### 3.3 Automated Calendar & Conflict Detection Flow (MVC Decision Flow)

When a freelancer (individually or as an equal member of an organization) accepts a gig in the **View**, the **Controller** validates the request and triggers an atomic transaction in the **Model**:

```text
   [ VIEW: Freelancer or Organization Member clicks "Accept Gig" ]
                                  |
                                  v
   [ CONTROLLER: POST /api/v1/gigs/:gigId/accept (Validated via Inbound DTO) ]
                                  |
                                  v
   +------------------------------------------------------------------+
   | MODEL (Service + OrganizationDAO):                               |
   | Accepting on behalf of an Organization?                          |
   |  - YES --> Verify user exists in `organization_members`          |
   |            (All organization members have equal peer authority)  |
   |  - NO  --> Proceed as individual Freelancer                      |
   +------------------------------+-----------------------------------+
                                  |
                                  v
   +------------------------------------------------------------------+
   | MODEL (GigDAO + CalendarEventDAO Atomic Transaction):            |
   | Check `calendar_events` for any existing event where:            |
   |   existing.start_time < new_gig.end_time                         |
   |   AND existing.end_time > new_gig.start_time                     |
   +------------------------------+-----------------------------------+
                                  |
                 +----------------+----------------+
                 |                                 |
       [ Overlap Found (> 0) ]           [ No Overlap (Slot Free) ]
                 |                                 |
                 v                                 v
   +--------------------------+      +------------------------------------+
   | MODEL:                   |      | MODEL:                             |
   | - Rollback Transaction   |      | 1. Update `gigs.status`            |
   | - Leave Gig & Calendar   |      |    -> 'ACCEPTED'                   |
   |   tables unchanged       |      | 2. Auto-Insert `calendar_events`   |
   +-------------+------------+      |    record (`GIG_BOOKING`)          |
                 |                   | 3. Commit Transaction              |
                 v                   +-----------------+------------------+
   +--------------------------+                        |
   | CONTROLLER:              |                        v
   | - Format 409 Conflict    |      +------------------------------------+
   |   Error Outbound DTO     |      | CONTROLLER:                        |
   +-------------+------------+      | - Format 200 OK Outbound DTO       |
                 |                   |   (Sanitized Gig + Calendar Event) |
                 v                   +-----------------+------------------+
   [ VIEW: Displays Schedule  ]                        |
   [ Conflict Alert Modal     ]                        v
                                     [ VIEW: Angular Calendar Automatically ]
                                     [ Renders New Locked Gig Schedule Block]
```

---

## 4. Relational Database Model & Entity-Relationship (ER) Diagram

As part of the **Model (`M`)** layer, PostgreSQL stores six core relational tables managed via Prisma Migrate. Notice the key design rules reflected in the diagram below:
- **Site-Level Roles Only**: `users.site_role` only distinguishes between `USER` (normal freelancer/user) and `ADMIN` (system administrator/developer).
- **Flat Peer Organizations**: `organization_members` is a pure many-to-many join table between `users` and `organizations` **with no role column**—every member of an organization has equal rights to accept and review gigs.
- **Automated Calendar Linkage**: `calendar_events` links directly to `users` (the scheduled freelancer) and optionally to `gigs` (`gig_id` populated automatically when a gig is accepted, or `NULL` when a freelancer adds a manual personal busy block).

### 4.1 Visual Entity-Relationship (ER) Diagram

```text
+-----------------------------------+          +-----------------------------------+
|               users               |          |           organizations           |
+-----------------------------------+          +-----------------------------------+
| PK  id             (UUID)         |          | PK  id             (UUID)         |
|     email          (VARCHAR, UQ)  |          |     name           (VARCHAR, UQ)  |
|     username       (VARCHAR, UQ)  |          |     slug           (VARCHAR, UQ)  |
|     full_name      (VARCHAR)      |          |     description    (TEXT)         |
|     password_hash  (VARCHAR)      |          |     service_tags   (TEXT[])       |
|     bio            (TEXT)         |          |     metadata       (JSONB)        |
|     skills         (TEXT[])       |          |     created_at     (TIMESTAMP)    |
|     site_role      (USER | ADMIN) |          +-----------------+-----------------+
|     is_active      (BOOLEAN)      |                            |
|     created_at     (TIMESTAMP)    |                            | 1
+---+-------------+-------------+---+                            |
    | 1           | 1           | 1                              |
    |             |             |              +-----------------+-----------------+
    |             |             |              |       organization_members        |
    |             |             |              |  (Flat Peer Collective Join Table)|
    |             |             |              +-----------------------------------+
    |             |             +------------->| PK  id              (UUID)        |
    |             |                        N   | FK  user_id         (UUID)        |
    |             |                            | FK  organization_id (UUID)        |<---+
    |             |                            |     joined_at       (TIMESTAMP)   |    |
    |             |                            |     *No intra-org role column*    |    |
    |             |                            +-----------------------------------+    |
    |             |                                                                     |
    |             | 1 (Creator / Assignee)                       1 (Published / Accepted|
    |             +----------------------------+                 by Organization)       |
    |                                          |                 |                      |
    | 1                                        v N               v N                    |
+---+-------------------------------+          +-----------------------------------+    |
|          calendar_events          |          |               gigs                |    |
+-----------------------------------+          +-----------------------------------+    |
| PK  id             (UUID)         |          | PK  id                  (UUID)    |    |
| FK  user_id        (UUID)         |          | FK  creator_user_id     (UUID)    |    |
| FK  gig_id         (UUID, NULL)   |<---------| FK  publishing_org_id   (UUID,NULL)----+
|     title          (VARCHAR)      |  0..N  1 | FK  accepted_by_user_id (UUID,NULL)
|     description    (TEXT)         |          | FK  accepted_by_org_id  (UUID,NULL)
|     start_time     (TIMESTAMP)    |          |     title               (VARCHAR) |
|     end_time       (TIMESTAMP)    |          |     description         (TEXT)    |
|     event_type     (GIG_BOOKING | |          |     compensation        (DECIMAL) |
|                     PERSONAL_BLK) |          |     start_time          (TIMESTAMP)
+-----------------------------------+          |     end_time            (TIMESTAMP)
                                               |     status              (ENUM)    |
                                               |     requirements_meta   (JSONB)   |
                                               +-----------------+-----------------+
                                                                 | 1
                                                                 |
                                                                 v N
                                               +-----------------------------------+
                                               |            gig_reviews            |
                                               +-----------------------------------+
                                               | PK  id                  (UUID)    |
                                               | FK  gig_id              (UUID)    |
                                               | FK  reviewer_user_id    (UUID)    |
                                               |     rating              (INT 1-5) |
                                               |     comment             (TEXT)    |
                                               |     created_at          (TIMESTAMP)
                                               +-----------------------------------+
```

### 4.2 Relational Cardinality & Constraint Summary

| Parent Entity (`1`) | Child Entity (`N`) | Relationship Type | Key Rule & Behavior |
| :--- | :--- | :--- | :--- |
| `users` | `organization_members` | One-to-Many (`1 : N`) | Links a freelancer to one or more organizations (`UNIQUE(organization_id, user_id)`). |
| `organizations` | `organization_members` | One-to-Many (`1 : N`) | Resolves the `N : M` peer collective membership; deleting an org cascades to membership rows. |
| `users` | `gigs` | One-to-Many (`1 : N`) | A user can create many gigs (`creator_user_id`) and accept many gigs (`accepted_by_user_id`). |
| `organizations` | `gigs` | One-to-Many (`1 : N`) | An organization can publish services/gigs (`publishing_org_id`) or accept gigs (`accepted_by_org_id`) via any peer member. |
| `gigs` | `calendar_events` | One-to-Many (`1 : 0..N`) | When a gig is accepted, one or more `GIG_BOOKING` calendar events are automatically created within the same transaction. |
| `users` | `calendar_events` | One-to-Many (`1 : N`) | Indexed on `(user_id, start_time, end_time)` to guarantee instant overlap detection before confirming any gig booking. |
| `gigs` / `users` | `gig_reviews` | One-to-Many (`1 : N`) | Allows freelancers and organization members to review and rate gigs (`1–5` scale). |

---

## 5. Workspace & Folder Structure (`Backend/` & `Frontend/`)

```text
ShiftSynth/
├── project_plan.md
├── README.md
├── docker-compose.yml               # Local PostgreSQL container for development
│
├── Backend/                         # Node.js + Express.js (JavaScript)
│   ├── package.json
│   ├── .env.example
│   ├── prisma/
│   │   ├── schema.prisma            # PostgreSQL schema
│   │   ├── migrations/              # SQL migrations generated by Prisma Migrate
│   │   └── seed.js                  # Initial ADMIN user & sample gigs/orgs
│   └── src/
│       ├── server.js                # Express app bootstrap
│       ├── config/
│       │   ├── db.config.js         # PrismaClient singleton instance
│       │   └── env.config.js        # Validated environment variables
│       ├── middlewares/
│       │   ├── auth.middleware.js   # JWT verification
│       │   ├── role.middleware.js   # Site-level guard: requireRole('ADMIN' | 'USER')
│       │   ├── validate-dto.middleware.js
│       │   └── error.middleware.js  # Centralized error-to-DTO formatter
│       ├── dtos/                    # Security Boundary: Inbound & Outbound DTOs
│       │   ├── request/
│       │   │   ├── auth.request.dto.js
│       │   │   ├── org.request.dto.js
│       │   │   ├── gig.request.dto.js
│       │   │   └── calendar.request.dto.js
│       │   └── response/
│       │       ├── user.response.dto.js
│       │       ├── org.response.dto.js
│       │       ├── gig.response.dto.js
│       │       └── calendar.response.dto.js
│       ├── daos/                    # Data Access Objects (Only layer talking to Prisma)
│       │   ├── user.dao.js
│       │   ├── organization.dao.js
│       │   ├── gig.dao.js
│       │   └── calendar-event.dao.js
│       ├── services/                # Business Logic & Orchestration
│       │   ├── auth.service.js
│       │   ├── user.service.js
│       │   ├── organization.service.js
│       │   ├── gig.service.js
│       │   └── calendar.service.js
│       ├── controllers/             # Express Route Handlers
│       │   ├── auth.controller.js
│       │   ├── user.controller.js
│       │   ├── organization.controller.js
│       │   ├── gig.controller.js
│       │   ├── calendar.controller.js
│       │   └── admin.controller.js
│       └── routes/
│           └── index.js             # API v1 router (/api/v1/*)
│
└── Frontend/                        # Angular (TypeScript)
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
            │   ├── dtos/                    # TypeScript DTO interfaces matching Backend DTOs
            │   │   ├── user.dto.ts
            │   │   ├── organization.dto.ts
            │   │   ├── gig.dto.ts
            │   │   └── calendar-event.dto.ts
            │   └── components/
            │       └── navbar/
            └── features/
                ├── auth/                    # Login / Register views
                ├── dashboard/               # Freelancer overview
                ├── gigs/                    # Browse, Create, Accept, Review gigs
                ├── organizations/           # Create/Join collective, shared org gig queue
                ├── calendar/                # Auto-populated Calendar + Conflict alerts
                └── admin/                   # System Administrator / Developer panel
```

---

## 6. Phased MVC Implementation Roadmap

### Phase 1: Foundation, Model Database Setup & Controller Scaffolding
1. **Model Layer — PostgreSQL & Prisma Initialization**:
   - Set up `docker-compose.yml` for a local PostgreSQL instance.
   - Initialize Prisma in `Backend/` (`npx prisma init`), translate the Section 4 ER diagram into `schema.prisma`, and generate the initial SQL migration (`npx prisma migrate dev --name init_shiftsynth`).
2. **Controller Layer — Express Server & DTO/DAO Base Architecture**:
   - Configure Express with `helmet`, `cors`, JSON body limits, and centralized error handling.
   - Implement base Inbound/Outbound DTO validation middleware and DAO base helpers.
3. **View Layer — Angular Workspace Setup**:
   - Initialize Angular project inside `Frontend/` with strict TypeScript mode, routing, and shared DTO interfaces (`Frontend/src/app/shared/dtos/`).

### Phase 2: Authentication & Site-Level Role Management (`USER` vs `ADMIN`)
1. **Model & Controller Auth Pipeline**:
   - Build `UserDAO` (Model), `RegisterRequestDTO` / `LoginRequestDTO`, `UserResponseDTO` (stripping `passwordHash`), and `AuthController` (Controller).
   - Implement JWT issuance (Access + Refresh tokens) and `requireSiteRole('ADMIN')` middleware.
   - Create seed script to provision the initial System Administrator (`ADMIN`) account.
2. **View Auth & Guards**:
   - Build Login/Register views, JWT HTTP Interceptor, `authGuard`, and `adminGuard`.

### Phase 3: Flat Freelancer Organizations (Collectives)
1. **Model & Controller Organization Module**:
   - Build `OrganizationDAO`, `OrganizationService` (Model), and `OrganizationController` (Controller).
   - Implement endpoints to create an organization, join/invite peers, and list organization services/gigs.
   - Enforce the flat collective rule in the Model: verify via `OrganizationDAO.isUserMember(orgId, userId)` that **any member** of the organization can accept or review gigs associated with the organization.
2. **View Organization Hub**:
   - Build Organization Directory, "Create Collective" view, and shared Organization Workspace where all members see active gigs, reviews, and collective schedules.

### Phase 4: Gig Marketplace, Acceptance & Reviews
1. **Model & Controller Gig Module**:
   - Build `GigDAO`, `GigService` (Model), and `GigController` (Controller) with `CreateGigRequestDTO`, `AcceptGigRequestDTO`, `ReviewGigRequestDTO`, and `GigResponseDTO`.
   - Support creating gigs with start/end timestamps, compensation, and optional organization association.
   - Enable any freelancer or organization member to review gigs and submit ratings/feedback.
2. **View Gig Board**:
   - Build filterable Gig Marketplace view, Gig Creation form, Gig Detail view, and one-click "Accept Gig (Individual or Organization)" action.

### Phase 5: Automated Calendar Engine & Conflict Detection
1. **Model & Controller Transactional Calendar Engine**:
   - Build `CalendarEventDAO` with `findOverlappingEvents(userId, startTime, endTime)`.
   - Wire `GigService.acceptGig` into an atomic Model transaction:
     - Check if the accepting freelancer (or assigned organization members) has any conflicting event during `[gig.startTime, gig.endTime]`.
     - If free, mark `Gig` as `ACCEPTED` and automatically insert `CalendarEvent` (`eventType: GIG_BOOKING`).
     - If overlapping, abort the transaction and return a structured `409 Conflict` DTO via the Controller detailing the conflicting time slot.
   - Allow users to add manual personal busy blocks (`PERSONAL_BLOCK`) so gigs cannot be accidentally accepted during personal time off.
2. **View Interactive Calendar**:
   - Integrate `angular-calendar` / FullCalendar in `features/calendar/`.
   - Render auto-populated `GIG_BOOKING` blocks alongside `PERSONAL_BLOCK` entries in Month, Week, and Day views.
   - Display real-time conflict warnings if a user tries to accept a gig that overlaps with an existing calendar block.

### Phase 6: System Administrator Panel, Testing & Polish
1. **Admin / Developer MVC Module**:
   - Build `AdminController` (protected by `requireSiteRole('ADMIN')`) and Angular `features/admin/` views to inspect users, moderate gigs/organizations, and view system metrics.
2. **Automated Testing & Security Audit**:
   - Unit test Model DAO conflict queries, Controller DTO sanitization (verifying sensitive fields are never exposed), and organization peer permissions.
   - End-to-end test of the core MVC flow: *Create Gig $\rightarrow$ Accept Gig $\rightarrow$ Verify Automatic Calendar Population & Conflict Prevention*.

---

## 7. Future / Optional Stretch Goals (TBD Based on Timeline)
- **External Calendar Sync (Optional / TBD)**: Export calendar events as `.ics` (iCal) feeds or integrate two-way OAuth sync with Google Calendar / Outlook if project timeline permits.
- **Real-Time Notifications**: WebSocket / Server-Sent Events (SSE) notifications when a fellow organization member accepts a new gig or when a gig review is posted.
