# Skolo Backend — Architecture

## Stack

| Layer          | Technology                                    |
| -------------- | --------------------------------------------- |
| Language       | TypeScript                                    |
| HTTP Framework | Express.js                                    |
| Real-time      | Socket.io                                     |
| Database       | MongoDB (native driver — `mongodb` package)   |
| Validation     | Zod                                           |
| Auth           | JWT — Access token (15m) + Refresh token (7d) |

---

## Project Structure

```
src/
├── config/                    # App-wide config (env, DB, Socket)
│   ├── env.ts                 # Typed env variables (dotenv + zod validation)
│   ├── database.ts            # MongoDB connection
│   ├── socket.ts              # Socket.io server setup
│   └── permissions.ts         # Role-permission matrix (source of truth for all route guards)
│
├── types/                     # Shared TypeScript interfaces & enums
│   ├── declarations/          # Ambient .d.ts files — picked up by TS automatically, no import needed
│   │   ├── express.d.ts       # adds req.actor: Actor to Express Request
│   │   ├── socket.d.ts        # adds socket.data.user: Actor to Socket.io Socket
│   │   └── global.d.ts        # typed process.env — JWT_SECRET, MONGO_URI, etc.
│   ├── documents/             # DB document interfaces — shape of each MongoDB collection
│   │   ├── user.document.ts
│   │   ├── driver.document.ts
│   │   ├── trip.document.ts
│   │   ├── ...                # one file per collection (mirrors skolo.dbml)
│   │   └── index.ts           # re-export all document types
│   ├── actor.types.ts         # Actor interface — shared JWT payload shape (imported explicitly)
│   ├── repository.types.ts    # PaginatedResult<T>, PaginationOptions, Role
│   └── index.ts               # Re-export all types
│
├── constants/                 # Enums, error codes, messages — never inline strings
│   ├── enums/
│   │   ├── user.enum.ts       # UserType, AdminRole
│   │   ├── trip.enum.ts       # TripStatus, AttendanceStatus
│   │   ├── payment.enum.ts    # PaymentStatus, PaymentType
│   │   └── index.ts
│   ├── messages/              # Nested by domain — MESSAGES.trips.notFound
│   │   ├── auth.messages.ts
│   │   ├── trips.messages.ts
│   │   ├── drivers.messages.ts
│   │   ├── students.messages.ts
│   │   └── index.ts           # merges all into single MESSAGES object
│   ├── collections.ts         # COLLECTIONS.USERS = 'users' — no raw strings
│   ├── error-codes.ts         # ERROR_CODES.TRIP_NOT_FOUND etc.
│   ├── http-status.ts         # HTTP_STATUS.OK, NOT_FOUND — no magic numbers
│   └── index.ts               # re-export everything
│
├── middleware/                # Reusable Express middleware
│   ├── auth.middleware.ts     # JWT verify (users + admins)
│   ├── validate.middleware.ts # Zod request body/param validation
│   ├── error.middleware.ts    # Global error handler
│   └── role.middleware.ts     # Role-based access guard
│
├── modules/                   # Feature modules — one folder per domain
│   │                          # Every module owns its own 5 files:
│   │                          #   routes / controller / service / repository / schema
│   ├── auth/
│   │   ├── auth.routes.ts
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   └── auth.schema.ts     # Zod validation schemas (no repository — auth reads users/admin_portal)
│   ├── trips/
│   │   ├── trips.routes.ts
│   │   ├── trips.controller.ts
│   │   ├── trips.service.ts
│   │   ├── trips.repository.ts  # extends BaseRepository — aggregations & domain queries only
│   │   └── trips.schema.ts
│   ├── tracking/
│   │   ├── tracking.routes.ts
│   │   ├── tracking.controller.ts
│   │   ├── tracking.service.ts
│   │   └── tracking.repository.ts
│   ├── students/
│   │   ├── students.routes.ts
│   │   ├── students.controller.ts
│   │   ├── students.service.ts
│   │   ├── students.repository.ts
│   │   └── students.schema.ts
│   ├── drivers/               # same 5-file pattern
│   ├── schools/
│   ├── parents/
│   ├── notifications/
│   ├── admin/
│   ├── community/
│   └── payments/
│
├── sockets/                   # All WebSocket logic lives here
│   ├── index.ts               # Register all namespaces
│   ├── namespaces/
│   │   ├── driver.namespace.ts   # /driver namespace
│   │   ├── parent.namespace.ts   # /parent namespace
│   │   └── admin.namespace.ts    # /admin namespace
│   ├── handlers/
│   │   ├── location.handler.ts   # GPS update events
│   │   ├── trip.handler.ts       # Trip status events
│   │   └── community.handler.ts  # Chat/post events
│   └── socket.auth.ts            # JWT auth for socket connections
│
├── repositories/
│   └── base.repository.ts     # Shared base — extended by every module repository
│
├── utils/
│   ├── jwt.ts                 # Sign/verify access & refresh tokens
│   ├── otp.ts                 # OTP generation/verification
│   ├── response.ts            # Standardized API response wrapper
│   └── logger.ts              # Winston/Pino logger
│
├── jobs/                      # Cron jobs / background tasks
│   └── trip-cleanup.job.ts    # Auto-close stale trips
│
├── app.ts                     # Express app: middleware + routes registration
└── server.ts                  # HTTP server + Socket.io bind + startup
```

---

## The 4-Layer Rule

Every feature module follows the same strict 4-layer stack. Nothing skips a layer.

| Layer               | File                 | Lives in             | Responsibility                                      |
| ------------------- | -------------------- | -------------------- | --------------------------------------------------- |
| **Route**           | `*.routes.ts`        | `modules/{feature}/` | Declares URL + HTTP method + middleware chain       |
| **Controller**      | `*.controller.ts`    | `modules/{feature}/` | Parses request, calls service, sends response       |
| **Service**         | `*.service.ts`       | `modules/{feature}/` | Business logic — _should I do this, and what next?_ |
| **Repository**      | `*.repository.ts`    | `modules/{feature}/` | DB queries only — _how do I get/save this data?_    |
| **Base Repository** | `base.repository.ts` | `repositories/`      | Shared CRUD inherited by all module repositories    |

> Controllers never touch the DB directly.
> Services never access `req` or `res`.
> Repositories never contain business logic — they only execute what the service asks for.
> `base.repository.ts` is the only file in `src/repositories/` — everything else lives inside its module.

### Service vs Repository

|                 | Service                                        | Repository                                       |
| --------------- | ---------------------------------------------- | ------------------------------------------------ |
| Knows about     | Business rules, other services, external calls | `Collection<T>` only                             |
| Makes decisions | Yes — "can this parent book this driver?"      | No — just executes what it's told                |
| Throws errors   | Yes — business errors (`DRIVER_NOT_APPROVED`)  | Never — returns `null` or empty                  |
| Calls           | Repositories, other services, utils            | DB only (`find`, `aggregate`, `insertOne`, etc.) |

---

## Authentication

### REST Auth Flow

```
POST /auth/login          → { accessToken (15m), refreshToken (7d) }
POST /auth/admin/login    → same shape, from admin_portal collection
POST /auth/refresh        → validates refreshToken → new accessToken
POST /auth/logout         → invalidates refreshToken
```

Two separate login flows:

- `/auth/login` — parents and drivers (`users` collection, phone-based auth)
- `/auth/admin/login` — superadmin, admin, school_admin (`admin_portal` collection, email/password auth)

### JWT Payload

Two token shapes — one per auth source, both normalized to the same `Actor` type:

| Field      | User token           | Admin token                               |
| ---------- | -------------------- | ----------------------------------------- |
| `id`       | `users._id`          | `admin_portal._id`                        |
| `role`     | `parent` \| `driver` | `superadmin` \| `admin` \| `school_admin` |
| `phone`    | ✅                   | —                                         |
| `schoolId` | —                    | only for `school_admin`                   |

The `Actor` interface lives in `src/types/actor.types.ts` and is imported wherever needed. `src/types/declarations/express.d.ts` attaches it globally to `Request` as `req.actor` — no import required in controllers.

### How the Token Flows Through the Architecture

```
Request with Authorization: Bearer <token>
  → auth.middleware.ts     verifies JWT → attaches req.actor = { id, role, ... }
  → role.middleware.ts     checks req.actor.role against PERMISSIONS
  → Controller             reads req.actor.id, passes to service as plain param
  → Service                receives actorId as a function argument — no req/res
  → Repository             receives only filters/data — never sees the token
```

> **Rule:** The token never goes past the controller. Controllers extract `req.actor.id` and `req.actor.role` and pass them as plain arguments. Repositories know nothing about auth.

### How Each Layer Uses It

| Layer                | Uses                                   | Purpose                            |
| -------------------- | -------------------------------------- | ---------------------------------- |
| `auth.middleware.ts` | Full JWT payload                       | Decode + attach `req.actor`        |
| `role.middleware.ts` | `req.actor.role`                       | Allow or 403 against `PERMISSIONS` |
| Controller           | `req.actor.id`, `req.actor.role`       | Pass caller identity to service    |
| Service              | `actorId`, `actorRole` as plain params | Apply business rules               |
| Repository           | Nothing                                | DB queries only                    |

### Example — "Get my trips" (parent)

```
GET /trips/my
  auth.middleware   →  req.actor = { id: 'abc123', role: 'parent' }
  role.middleware   →  PERMISSIONS.trips.listOwn includes 'parent' ✅
  Controller        →  tripsService.getMyTrips(req.actor.id)
  Service           →  tripsRepository.findMany({ parent_id: actorId }, { page, limit })
  Repository        →  runs DB query — token never seen
```

### Socket Auth

Every socket connection must send a valid JWT in the handshake `auth.token` field. The `socket.auth.ts` middleware verifies it and attaches `socket.data.user: Actor` before any handler runs. Connections without a valid token are rejected immediately.

---

## WebSocket Architecture

### 3 Namespaces — one per actor type

| Namespace | Who connects      | Purpose                                          |
| --------- | ----------------- | ------------------------------------------------ |
| `/driver` | Driver mobile app | Emits GPS coordinates, receives trip commands    |
| `/parent` | Parent mobile app | Receives child pickup/drop status, live location |
| `/admin`  | Admin dashboard   | Receives all live events across all trips        |

### Room Strategy

| Room                | Who joins                                     | Used for                             |
| ------------------- | --------------------------------------------- | ------------------------------------ |
| `trip:{tripId}`     | Driver + parents of all students on that trip | All trip lifecycle events            |
| `driver:{driverId}` | Server only                                   | Direct messages to a specific driver |

### Socket Event Naming Convention — `domain:action`

```
location:update        ← driver emits GPS coordinates (high frequency)
trip:started           ← server broadcasts when trip begins
trip:completed         ← server broadcasts when trip ends
student:picked-up      ← server broadcasts after OTP/QR verified
student:dropped-off    ← server broadcasts after drop confirmed
notification:new       ← server pushes notification to parent
community:post-created ← server broadcasts new post to community room
```

---

## Route-Permission Matrix

`src/config/permissions.ts` is the **single source of truth** for all route guards. It is a typed `as const` object, structured by module then action. Every route imports from it — no role string is ever hardcoded in a route file. TypeScript will error if a route references a key that does not exist in the matrix.

Routes always follow the pattern: `authenticate → authorize(PERMISSIONS.module.action) → controller`.

### How `authorize` Middleware Works

`middleware/role.middleware.ts` accepts a `Role[]` from `PERMISSIONS`, reads `req.actor.role` (set by `auth.middleware.ts`), and throws `403` if the role is not in the list.

> Both `users` (parent/driver) and `admin_portal` (superadmin/admin/school_admin) are normalized to a single `req.actor.role` in `auth.middleware.ts`. The `authorize` middleware only cares about the role — it never distinguishes which collection the user came from.

### Permission Table

| Module        | Action        | parent | driver | admin | superadmin | school_admin |
| ------------- | ------------- | :----: | :----: | :---: | :--------: | :----------: |
| **auth**      | login         |   ✅   |   ✅   |   —   |     —      |      —       |
| **auth**      | admin login   |   —    |   —    |  ✅   |     ✅     |      ✅      |
| **trips**     | list          |   ✅   |   ✅   |  ✅   |     ✅     |      ✅      |
| **trips**     | create        |   —    |   —    |  ✅   |     ✅     |      —       |
| **trips**     | delete        |   —    |   —    |   —   |     ✅     |      —       |
| **students**  | list          |   ✅   |   —    |  ✅   |     ✅     |      ✅      |
| **students**  | create        |   ✅   |   —    |  ✅   |     ✅     |      —       |
| **students**  | delete        |   —    |   —    |   —   |     ✅     |      —       |
| **drivers**   | list          |   —    |   —    |  ✅   |     ✅     |      ✅      |
| **drivers**   | approve       |   —    |   —    |  ✅   |     ✅     |      —       |
| **schools**   | list          |   ✅   |   ✅   |  ✅   |     ✅     |      ✅      |
| **schools**   | create        |   —    |   —    |  ✅   |     ✅     |      —       |
| **tracking**  | read location |   ✅   |   —    |  ✅   |     ✅     |      —       |
| **payments**  | list own      |   ✅   |   —    |  ✅   |     ✅     |      —       |
| **admin**     | manage users  |   —    |   —    |  ✅   |     ✅     |      —       |
| **community** | post          |   ✅   |   —    |  ✅   |     ✅     |      —       |

> This table is derived from `src/config/permissions.ts`. If permissions change, update `permissions.ts` first — this table should reflect that.

---

## Base Repository Pattern

`src/repositories/base.repository.ts` is a generic abstract class wrapping the native MongoDB `Collection<T>`. Every child repository extends it and **only adds domain-specific, complex queries**. All standard CRUD is inherited — never repeated.

### `BaseRepository<T>` — Methods

| Method                              | Description                                         |
| ----------------------------------- | --------------------------------------------------- |
| `findById(id, projection?)`         | Find one document by `_id`                          |
| `findOne(filter)`                   | Find one by filter — auto-excludes soft-deleted     |
| `findMany(filter, { page, limit })` | Paginated list — returns `PaginatedResult<T>`       |
| `create(data)`                      | Insert one document                                 |
| `bulkCreate(data[])`                | Insert many documents                               |
| `updateById(id, update)`            | Update one by `_id`                                 |
| `deleteById(id)`                    | Hard delete — use only for full purge               |
| `softDelete(id)`                    | Sets `deletedAt = now` — preferred over hard delete |
| `exists(filter)`                    | Returns `boolean`                                   |
| `count(filter)`                     | Returns document count                              |

**`PaginatedResult<T>` shape:** `{ data, total, page, totalPages, hasNext }`

**Soft delete:** `findOne` and `findMany` automatically append `{ deletedAt: null }` to every filter. Pass `{ includeSoftDeleted: true }` to override.

### Child Repositories — Domain Queries Only

Child repositories extend `BaseRepository` and contain only aggregations, `$lookup` pipelines, and queries that cannot be expressed through the base `findMany`. Standard CRUD is never redeclared.

> **Rule:** If a query can be done with `findMany(filter, opts)`, it belongs in the service — not the repository.

### Layer Flow

```
modules/trips/
  trips.routes.ts
    → trips.controller.ts
        → trips.service.ts
            → trips.repository.ts   (aggregations + domain queries)
                  ↑ extends
            repositories/base.repository.ts   (inherited CRUD)
```

`trips.repository.ts` imports `BaseRepository` from `../../repositories/base.repository`. The service imports the repository from the same folder (`./trips.repository`). No cross-module jumping required.

---

## Types & Declarations

### Three Distinct Shapes — Never Mix Them

Every feature deals with three distinct type shapes. Each lives in a different place.

| Shape        | Where it lives                              | Contains                                                            |
| ------------ | ------------------------------------------- | ------------------------------------------------------------------- |
| **Document** | `src/types/documents/*.document.ts`         | Exact MongoDB document — `_id`, `deletedAt`, all raw DB fields      |
| **Request**  | `src/modules/{feature}/{feature}.schema.ts` | What the client sends — Zod schema + `z.infer<>` type               |
| **Response** | `src/modules/{feature}/{feature}.schema.ts` | What the API returns — may omit `deletedAt`, format `_id` as string |

The **service** bridges these shapes — it receives a request type, calls the repository (which returns documents), and maps the result into a response type before returning to the controller.

### Ambient `.d.ts` Files vs Regular Types

|                         | `.d.ts` files                                                        | Regular `.ts` type files         |
| ----------------------- | -------------------------------------------------------------------- | -------------------------------- |
| Picked up by TypeScript | Automatically — no import needed                                     | Must be imported explicitly      |
| Used for                | Augmenting existing types (`req.actor`), global vars (`process.env`) | Interfaces, enums, utility types |
| Lives in                | `src/types/`                                                         | `src/types/` or module folder    |

**`src/types/` files at a glance:**

| File                  | Kind    | Purpose                                                                                       |
| --------------------- | ------- | --------------------------------------------------------------------------------------------- |
| `express.d.ts`        | ambient | Adds `req.actor: Actor` to every Express `Request`                                            |
| `socket.d.ts`         | ambient | Adds `socket.data.user: Actor` to every Socket.io `Socket`                                    |
| `global.d.ts`         | ambient | Typed `process.env` — `JWT_SECRET`, `MONGO_URI`, etc. are `string`, not `string \| undefined` |
| `actor.types.ts`      | regular | `Actor` interface — shared JWT payload shape, imported where needed                           |
| `repository.types.ts` | regular | `PaginatedResult<T>`, `PaginationOptions`, `Role` union                                       |
| `documents/`          | regular | DB document interfaces — one per collection                                                   |

---

## Constants, Enums & Messages

All enums, error codes, and messages live in `src/constants/`. No string literals are ever inline in routes, controllers, or services.

| File / Folder              | Contains                                                                                   |
| -------------------------- | ------------------------------------------------------------------------------------------ |
| `constants/enums/`         | TypeScript enums — `UserType`, `TripStatus`, `AdminRole`, `PaymentStatus`, etc.            |
| `constants/messages/`      | Nested message objects by domain — `MESSAGES.trips.notFound`, `MESSAGES.auth.invalidToken` |
| `constants/error-codes.ts` | `ERROR_CODES.TRIP_NOT_FOUND` etc. — used in API error responses; clients switch on these   |
| `constants/http-status.ts` | `HTTP_STATUS.OK`, `HTTP_STATUS.NOT_FOUND` — no magic numbers anywhere                      |
| `constants/collections.ts` | `COLLECTIONS.USERS = 'users'` — passed to `getCollection<T>()` instead of raw strings      |

> `MESSAGES` is structured by domain (e.g. `MESSAGES.trips.notFound`) so TypeScript auto-completes the correct key and a typo causes a compile error, not a runtime bug.

---

## Standardized API Response

All REST endpoints return one consistent shape via the `utils/response.ts` helper.

```
// Success
{ success: true, data: { ... } }

// Error
{ success: false, error: { code: "TRIP_NOT_FOUND", message: "..." } }
```

Error `code` always matches a key in `ERROR_CODES`. Error `message` always comes from `MESSAGES`. Neither is ever inline in controller code.

---

## Key Design Decisions

| Decision                 | Choice                                    | Reason                                                                   |
| ------------------------ | ----------------------------------------- | ------------------------------------------------------------------------ |
| API versioning           | None                                      | URL paths start at `/auth`, `/trips`, etc.                               |
| DB layer                 | Native `mongodb` driver                   | No Mongoose — document shapes are plain TypeScript interfaces            |
| Request validation       | Zod                                       | API boundary validation; DB receives already-validated data              |
| Constants/enums/messages | `src/constants/`                          | No inline strings anywhere in routes, controllers, services              |
| Collection names         | `COLLECTIONS.*` constant                  | No raw strings passed to `getCollection<T>()`                            |
| Base repository          | `BaseRepository<T>` wraps `Collection<T>` | Eliminates CRUD repetition across 30+ collections                        |
| Child repositories       | Domain queries only                       | Clear separation — inherited CRUD vs custom aggregations                 |
| Permission matrix        | Central `permissions.ts`                  | One source of truth; TypeScript-enforced sync with route files           |
| Actor normalization      | `req.actor.role` unified                  | `authorize()` works identically for users and admins                     |
| Soft delete              | Opt-in per collection                     | Audit trail preserved; base class handles filter automatically           |
| Socket namespaces        | 3 (driver/parent/admin)                   | Isolates concerns, prevents cross-actor noise                            |
| Module-per-feature       | Yes                                       | An AI agent assigned a feature knows exactly which files to create/touch |

---

## Source of Truth

- Database schema: `agent-backend-lab/database/skolo.dbml`
