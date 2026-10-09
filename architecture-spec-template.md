# [System Name] — Architecture Specification

> **Version**: 1.0  
> **Author**: [Name]  
> **Date**: YYYY-MM-DD  
> **Status**: Draft | Review | Final

---

## 1. Company-Wide Common Requirements

### 1.1 Authentication & Authorization

| ID | Requirement | Detail |
|----|-------------|--------|
| CW-AUTH-001 | Authentication method | JWT-based token auth. Access token expires in 15 min, refresh token in 7 days. |
| CW-AUTH-002 | Password policy | Min 8 chars, at least 1 uppercase + 1 number + 1 special char. Bcrypt hash (cost 12). |
| CW-AUTH-003 | Session management | Redis-backed session store. Concurrent session limit: 3 devices per user. |

### 1.2 Roles & Permissions

| Role | Permissions | Scope |
|------|-------------|-------|
| Super Admin | Full CRUD on all resources | Global |
| Admin | Manage users, view reports, manage content | Organization |
| User | Read content, manage own profile | Self |

### 1.3 Logging & Monitoring

| ID | Requirement | Detail |
|----|-------------|--------|
| CW-LOG-001 | Request logging | Log every API request: method, path, status, duration, user_id. Exclude body for auth endpoints. |
| CW-LOG-002 | Error logging | Structured JSON logs with stack trace. Severity levels: DEBUG, INFO, WARN, ERROR, FATAL. |
| CW-LOG-003 | Audit trail | Log all create/update/delete on core entities with before/after values, actor, timestamp. |

### 1.4 Error Handling

| HTTP Status | Usage | Response Format |
|-------------|-------|-----------------|
| 400 | Validation error | `{ "error": "VALIDATION_ERROR", "message": "...", "details": [...] }` |
| 401 | Not authenticated | `{ "error": "UNAUTHORIZED", "message": "..." }` |
| 403 | Not authorized | `{ "error": "FORBIDDEN", "message": "..." }` |
| 404 | Resource not found | `{ "error": "NOT_FOUND", "message": "..." }` |
| 409 | Conflict (duplicate) | `{ "error": "CONFLICT", "message": "..." }` |
| 500 | Server error | `{ "error": "INTERNAL_ERROR", "message": "An unexpected error occurred" }` |

### 1.5 Security

| ID | Requirement | Detail |
|----|-------------|--------|
| CW-SEC-001 | Input sanitization | Sanitize all user input. Parameterized queries only (no string concatenation). |
| CW-SEC-002 | CORS | Whitelist specific origins. No wildcard `*` in production. |
| CW-SEC-003 | Rate limiting | 100 req/min per IP for public endpoints. 1000 req/min for authenticated. |
| CW-SEC-004 | HTTPS | TLS 1.2+ required. HSTS header enabled. |
| CW-SEC-005 | Sensitive data | Never log passwords, tokens, or PII. Mask in API responses where applicable. |

### 1.6 Internationalization (i18n)

| ID | Requirement | Detail |
|----|-------------|--------|
| CW-I18N-001 | Languages | Korean (ko) primary, English (en) secondary. |
| CW-I18N-002 | Date/time | Store in UTC. Display in user's timezone (default: Asia/Seoul). Format: YYYY-MM-DD HH:mm. |
| CW-I18N-003 | Currency | Store as integer (원). Display with comma separator (e.g., ₩1,000,000). |

### 1.7 Environments & Deployment

| Environment | URL Pattern | Purpose | Deploy Trigger |
|-------------|-------------|---------|----------------|
| Local | localhost:3000 / :8080 | Development | Manual |
| Dev | dev.example.com | Integration testing | Push to `develop` |
| Staging | staging.example.com | UAT / QA | Push to `staging` |
| Production | example.com | Live | Tag `v*` on `main` |

---

## 2. Coding Conventions

### 2.1 Naming

| Element | Convention | Example |
|---------|-----------|---------|
| File (component) | PascalCase | `UserProfile.tsx` |
| File (utility) | camelCase | `formatDate.ts` |
| File (API route) | kebab-case | `user-profile.ts` |
| Variable / function | camelCase | `getUserById()` |
| Constant | SCREAMING_SNAKE | `MAX_RETRY_COUNT` |
| DB table | snake_case, singular | `user_profile` |
| DB column | snake_case | `created_at` |
| API endpoint | kebab-case, plural | `/api/v1/user-profiles` |
| CSS class | kebab-case or BEM | `card-header` / `card__header--active` |
| Env variable | SCREAMING_SNAKE | `DATABASE_URL` |

### 2.2 Folder Structure

> Pick the structure that matches your tech stack. Use only one.

#### Option A — Java (Spring Boot)

```
src/
├── main/
│   ├── java/com/divii/[project]/
│   │   ├── config/           # SecurityConfig, CorsConfig, SwaggerConfig
│   │   ├── controller/       # REST controllers (@RestController)
│   │   ├── dto/
│   │   │   ├── request/      # CreateEventRequest, LoginRequest
│   │   │   └── response/     # EventResponse, PageResponse
│   │   ├── entity/           # JPA entities (@Entity)
│   │   ├── enums/            # Role, EventStatus, CheckInStatus
│   │   ├── exception/        # CustomException, GlobalExceptionHandler
│   │   ├── filter/           # JwtAuthFilter, RateLimitFilter
│   │   ├── mapper/           # Entity ↔ DTO mappers (MapStruct)
│   │   ├── repository/       # JPA repositories (@Repository)
│   │   ├── security/         # JwtProvider, UserDetailsServiceImpl
│   │   ├── service/          # Business logic (@Service)
│   │   │   └── impl/
│   │   └── util/             # DateUtil, QrCodeGenerator
│   └── resources/
│       ├── application.yml
│       ├── application-dev.yml
│       ├── application-prod.yml
│       ├── db/migration/     # Flyway: V1__create_users.sql
│       └── templates/        # Email templates (Thymeleaf)
└── test/
    ├── java/com/divii/[project]/
    │   ├── controller/       # @WebMvcTest
    │   ├── service/          # @SpringBootTest / Mockito
    │   └── repository/       # @DataJpaTest
    └── resources/
        └── application-test.yml
```

#### Option B — NestJS

```
src/
├── common/
│   ├── decorators/          # @CurrentUser, @Roles
│   ├── dto/                 # PaginationDto, ApiResponseDto
│   ├── enums/               # role.enum.ts, status.enum.ts
│   ├── filters/             # http-exception.filter.ts
│   ├── guards/              # jwt-auth.guard.ts, roles.guard.ts
│   ├── interceptors/        # logging.interceptor.ts, transform.interceptor.ts
│   ├── middleware/           # rate-limit.middleware.ts
│   └── pipes/               # validation.pipe.ts
├── config/
│   ├── database.config.ts
│   ├── jwt.config.ts
│   └── app.config.ts
├── modules/
│   ├── auth/
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── auth.module.ts
│   │   ├── dto/             # login.dto.ts, register.dto.ts
│   │   └── strategies/      # jwt.strategy.ts
│   ├── users/
│   │   ├── users.controller.ts
│   │   ├── users.service.ts
│   │   ├── users.module.ts
│   │   ├── dto/
│   │   └── entities/        # user.entity.ts
│   └── [feature]/
│       ├── [feature].controller.ts
│       ├── [feature].service.ts
│       ├── [feature].module.ts
│       ├── dto/
│       └── entities/
├── database/
│   ├── migrations/          # TypeORM: 1696000000-CreateUsers.ts
│   └── seeds/
├── app.module.ts
└── main.ts
test/
├── unit/
├── integration/
└── e2e/
```

#### Option C — Next.js (Full-stack, App Router)

```
src/
├── app/
│   ├── (auth)/              # Route group — no layout nesting
│   │   ├── login/page.tsx
│   │   ├── register/page.tsx
│   │   └── layout.tsx       # Auth layout (no sidebar)
│   ├── (dashboard)/
│   │   ├── events/
│   │   │   ├── page.tsx           # Event list
│   │   │   ├── [id]/page.tsx      # Event detail
│   │   │   └── new/page.tsx       # Create event
│   │   ├── check-in/page.tsx
│   │   └── layout.tsx       # Dashboard layout (sidebar + nav)
│   ├── api/
│   │   ├── auth/
│   │   │   ├── login/route.ts
│   │   │   ├── register/route.ts
│   │   │   └── refresh/route.ts
│   │   ├── events/
│   │   │   ├── route.ts          # GET list, POST create
│   │   │   └── [id]/route.ts     # GET, PATCH, DELETE
│   │   └── check-in/route.ts
│   ├── layout.tsx           # Root layout
│   ├── page.tsx             # Landing
│   └── globals.css
├── components/
│   ├── ui/                  # Button, Modal, Input, Table
│   └── [feature]/           # EventCard, CheckInScanner, QrCode
├── hooks/                   # useAuth, useEvents, usePagination
├── lib/
│   ├── auth.ts              # JWT verify, session helpers
│   ├── db.ts                # Prisma client
│   ├── validations.ts       # Zod schemas
│   └── utils.ts
├── services/                # Server-side business logic
├── types/                   # TypeScript interfaces
└── middleware.ts             # Auth redirect, rate limit
prisma/
├── schema.prisma
├── migrations/
└── seed.ts
```

#### Option D — React (Frontend only, Vite)

```
src/
├── api/                     # Axios instance + endpoint functions
│   ├── client.ts            # Axios config, interceptors, token refresh
│   ├── auth.api.ts          # login(), register(), refresh()
│   ├── events.api.ts        # getEvents(), createEvent()
│   └── check-in.api.ts
├── assets/                  # Images, fonts, icons
├── components/
│   ├── common/              # Button, Modal, Input, Table, Pagination
│   ├── layout/              # Header, Sidebar, Footer, AuthLayout
│   └── [feature]/           # EventCard, QrScanner, AttendeeList
├── contexts/                # AuthContext, ThemeContext
├── hooks/                   # useAuth, useEvents, usePagination, useDebounce
├── pages/
│   ├── auth/
│   │   ├── LoginPage.tsx
│   │   └── RegisterPage.tsx
│   ├── events/
│   │   ├── EventListPage.tsx
│   │   ├── EventDetailPage.tsx
│   │   └── CreateEventPage.tsx
│   ├── check-in/
│   │   └── CheckInPage.tsx
│   └── dashboard/
│       └── DashboardPage.tsx
├── router/                  # React Router config + guards
│   ├── index.tsx
│   └── PrivateRoute.tsx
├── store/                   # Zustand / Redux (if needed)
├── types/                   # TypeScript interfaces
├── utils/                   # formatDate, formatCurrency, validators
├── App.tsx
├── main.tsx
└── vite-env.d.ts
```

#### Option E — Node / Express

```
src/
├── api/
│   └── v1/
│       ├── auth/
│       │   ├── auth.controller.ts
│       │   ├── auth.service.ts
│       │   ├── auth.routes.ts
│       │   └── auth.validation.ts   # Joi / Zod schemas
│       ├── users/
│       │   ├── users.controller.ts
│       │   ├── users.service.ts
│       │   ├── users.routes.ts
│       │   └── users.validation.ts
│       └── [feature]/
│           ├── [feature].controller.ts
│           ├── [feature].service.ts
│           ├── [feature].routes.ts
│           └── [feature].validation.ts
├── config/
│   ├── database.ts
│   ├── redis.ts
│   ├── env.ts               # dotenv + validation
│   └── cors.ts
├── db/
│   ├── models/              # Sequelize/TypeORM entities
│   ├── migrations/
│   └── seeds/
├── middleware/
│   ├── auth.middleware.ts    # JWT verify
│   ├── error.middleware.ts   # Global error handler
│   ├── rate-limit.middleware.ts
│   └── validate.middleware.ts
├── lib/
│   ├── logger.ts            # Winston / Pino
│   ├── mailer.ts
│   ├── qr.ts
│   └── response.ts          # Standard response helper
├── types/
│   └── express.d.ts         # Extend Request with user
├── app.ts                   # Express app setup
└── server.ts                # HTTP server + listen
test/
├── unit/
├── integration/
└── fixtures/
```

### 2.3 Branching & Commit Rules

| Branch | Purpose | Naming |
|--------|---------|--------|
| `main` | Production-ready | Protected, requires PR |
| `develop` | Integration | Default PR target |
| Feature | New feature | `feature/EVENT-123-add-login` |
| Bugfix | Bug fix | `fix/EVENT-456-null-check` |
| Hotfix | Urgent production fix | `hotfix/EVENT-789-crash` |

**Commit message format**:
```
<type>(<scope>): <subject>

<body — optional>

Refs: EVENT-123
```

Types: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`, `style`

### 2.4 Code Review Rules

- Every PR requires at least 1 approval
- Author must not merge their own PR (exception: hotfix with TL approval)
- PR description must reference the task ID
- No `console.log` in production code
- All new functions must have TypeScript types (no `any`)

### 2.5 Lint & Format

| Tool | Config | Rule |
|------|--------|------|
| ESLint | `eslint.config.js` | Extend `@typescript-eslint/recommended` |
| Prettier | `.prettierrc` | `semi: true`, `singleQuote: true`, `tabWidth: 2`, `printWidth: 100` |
| Husky | Pre-commit | Run `lint-staged` |
| lint-staged | `.lintstagedrc` | `*.{ts,tsx}` → eslint --fix + prettier --write |

---

## 3. Requirements Definition

### 3.1 Functional Requirements

| ID | Category | Requirement | Acceptance Criteria | Priority |
|----|----------|-------------|---------------------|----------|
| FR-001 | Auth | User can register with email and password | 1. Email validation (RFC 5322). 2. Password meets CW-AUTH-002. 3. Duplicate email returns 409. 4. Success returns 201 + user object (no password). | Must |
| FR-002 | Auth | User can log in | 1. Valid credentials return access + refresh token. 2. Invalid returns 401. 3. Locked account returns 403 after 5 failed attempts. | Must |
| FR-003 | Auth | User can reset password | 1. Reset email sent within 30s. 2. Token valid for 1 hour. 3. Used token cannot be reused. | Must |
| FR-004 | [Feature] | [Description] | 1. [Criterion 1]. 2. [Criterion 2]. | Must/Should/Could |

### 3.2 Non-Functional Requirements

| ID | Category | Requirement | Acceptance Criteria | Priority |
|----|----------|-------------|---------------------|----------|
| NFR-001 | Performance | API response time < 200ms (p95) | Load test: 100 concurrent users, p95 < 200ms for list endpoints. | Must |
| NFR-002 | Performance | Page load (LCP) < 2.5s | Lighthouse score ≥ 90 on mobile 4G throttle. | Should |
| NFR-003 | Availability | 99.9% uptime (monthly) | Monitoring alert if downtime > 4 min cumulative. | Must |
| NFR-004 | Scalability | Handle 1,000 concurrent users | Horizontal scaling via container orchestration. | Should |
| NFR-005 | Security | OWASP Top 10 compliance | Automated SAST scan passes with 0 critical findings. | Must |

---

## 4. Architecture

### 4.1 System Components

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   Client     │────▶│   API Server │────▶│  Database   │
│  (Next.js)   │     │  (Node.js)   │     │ (PostgreSQL)│
└─────────────┘     └──────┬───────┘     └─────────────┘
                           │
                    ┌──────┴───────┐
                    │    Redis     │
                    │  (Cache +    │
                    │   Session)   │
                    └──────────────┘
```

### 4.2 Component Description

| Component | Tech | Responsibility | Justification |
|-----------|------|---------------|---------------|
| Frontend | Next.js 14 (App Router) | UI rendering, routing, form handling | SSR for SEO, React ecosystem, file-based routing |
| API | Node.js + Express | Business logic, auth, validation | Same language as frontend, large ecosystem |
| Database | PostgreSQL 16 | Persistent data store | ACID compliance, JSON support, mature |
| Cache | Redis 7 | Session store, rate limit counter, query cache | In-memory speed, TTL support, pub/sub |
| Storage | AWS S3 | File uploads (images, documents) | Scalable, CDN-compatible, cost-effective |

### 4.3 Data Flow

```
[User Action] → [Frontend Route] → [API Middleware (auth, rate-limit)]
  → [Controller (validation)] → [Service (business logic)]
  → [Repository (DB query)] → [Database]
  → [Response DTO] → [Client]
```

### 4.4 External Integrations

| Service | Purpose | Auth Method | Endpoint |
|---------|---------|-------------|----------|
| [Email Service] | Transactional email | API key | `https://api.example.com/v1/send` |
| [Payment Gateway] | Payment processing | OAuth2 | `https://pay.example.com/api` |

---

## 5. ERD & DB Design

### 5.1 Entity Relationship Diagram

```
┌──────────┐       ┌──────────────┐       ┌──────────┐
│  users   │──1:N──│ user_roles   │──N:1──│  roles   │
└──────────┘       └──────────────┘       └──────────┘
     │
     │ 1:N
     ▼
┌──────────────┐
│  [entity]    │
└──────────────┘
```

### 5.2 Table Definitions

#### `users`

| Column | Type | Nullable | Default | Key | Index | Description |
|--------|------|----------|---------|-----|-------|-------------|
| id | UUID | NO | gen_random_uuid() | PK | — | |
| email | VARCHAR(255) | NO | — | UQ | idx_users_email | Login email |
| password_hash | VARCHAR(255) | NO | — | — | — | Bcrypt hash |
| name | VARCHAR(100) | NO | — | — | — | Display name |
| status | VARCHAR(20) | NO | 'active' | — | idx_users_status | active/inactive/locked |
| created_at | TIMESTAMPTZ | NO | now() | — | — | |
| updated_at | TIMESTAMPTZ | NO | now() | — | — | |
| deleted_at | TIMESTAMPTZ | YES | NULL | — | — | Soft delete |

**Constraints**:
- `chk_users_status`: status IN ('active', 'inactive', 'locked')
- `chk_users_email`: email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z]{2,}$'

#### `roles`

| Column | Type | Nullable | Default | Key | Index | Description |
|--------|------|----------|---------|-----|-------|-------------|
| id | SERIAL | NO | — | PK | — | |
| name | VARCHAR(50) | NO | — | UQ | — | Role name |
| permissions | JSONB | NO | '{}' | — | — | Permission map |

#### `user_roles`

| Column | Type | Nullable | Default | Key | Index | Description |
|--------|------|----------|---------|-----|-------|-------------|
| user_id | UUID | NO | — | FK → users.id | — | |
| role_id | INT | NO | — | FK → roles.id | — | |
| assigned_at | TIMESTAMPTZ | NO | now() | — | — | |

**PK**: (user_id, role_id)

### 5.3 Indexes

| Table | Index Name | Columns | Type | Reason |
|-------|-----------|---------|------|--------|
| users | idx_users_email | email | UNIQUE B-tree | Login lookup |
| users | idx_users_status | status | B-tree | Filter by status |
| [table] | [name] | [columns] | [type] | [reason] |

### 5.4 Migration Naming

```
YYYYMMDDHHMMSS_[description].sql
Example: 20261007120000_create_users_table.sql
```

---

## 6. API Design

### 6.1 Base URL & Versioning

```
Base: https://api.example.com/v1
Versioning: URL path (/v1, /v2)
Content-Type: application/json
```

### 6.2 Endpoints

#### Auth

| Method | Endpoint | Auth | Description | Req ID |
|--------|----------|------|-------------|--------|
| POST | `/v1/auth/register` | None | Register new user | FR-001 |
| POST | `/v1/auth/login` | None | Login | FR-002 |
| POST | `/v1/auth/refresh` | Refresh token | Refresh access token | FR-002 |
| POST | `/v1/auth/forgot-password` | None | Request password reset | FR-003 |
| POST | `/v1/auth/reset-password` | Reset token | Reset password | FR-003 |

#### Users

| Method | Endpoint | Auth | Description | Req ID |
|--------|----------|------|-------------|--------|
| GET | `/v1/users` | Admin | List users (paginated) | FR-004 |
| GET | `/v1/users/:id` | Admin or Self | Get user detail | FR-004 |
| PATCH | `/v1/users/:id` | Admin or Self | Update user | FR-004 |
| DELETE | `/v1/users/:id` | Admin | Soft delete user | FR-004 |

### 6.3 Request / Response Schema

#### POST `/v1/auth/register`

**Request**:
```json
{
  "email": "user@example.com",
  "password": "SecurePass1!",
  "name": "Hong Gildong"
}
```

**Validation**:
- `email`: required, valid email format, max 255 chars
- `password`: required, min 8 chars, must match CW-AUTH-002
- `name`: required, max 100 chars

**Response 201**:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "name": "Hong Gildong",
  "status": "active",
  "created_at": "2026-10-07T12:00:00Z"
}
```

**Error 409**:
```json
{
  "error": "CONFLICT",
  "message": "Email already registered"
}
```

### 6.4 Pagination

```
GET /v1/users?page=1&limit=20&sort=created_at&order=desc

Response:
{
  "data": [...],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "total_pages": 8
  }
}
```

### 6.5 Error Codes

| Code | Name | HTTP Status | When |
|------|------|-------------|------|
| VALIDATION_ERROR | Validation Error | 400 | Request body fails validation |
| UNAUTHORIZED | Unauthorized | 401 | Missing or expired token |
| FORBIDDEN | Forbidden | 403 | Insufficient permissions |
| NOT_FOUND | Not Found | 404 | Resource does not exist |
| CONFLICT | Conflict | 409 | Duplicate resource |
| RATE_LIMITED | Rate Limited | 429 | Too many requests |
| INTERNAL_ERROR | Internal Error | 500 | Unexpected server error |

---

## 7. Traceability Matrix

| Req ID | Requirement | Architecture Component | DB Tables | API Endpoints |
|--------|-------------|----------------------|-----------|---------------|
| FR-001 | User registration | API Server → Service → Repository | users, user_roles | POST /v1/auth/register |
| FR-002 | User login | API Server → Auth Middleware → Redis | users (read) | POST /v1/auth/login, /refresh |
| FR-003 | Password reset | API Server → Email Service | users (update), password_resets | POST /v1/auth/forgot-password, /reset-password |
| NFR-001 | API response < 200ms | Redis cache layer | — | All endpoints |
| NFR-003 | 99.9% uptime | Health check + load balancer | — | GET /health |

---

## 8. Change Log

| Version | Date | Author | Changes | Affected IDs | Reason |
|---------|------|--------|---------|--------------|--------|
| 1.0 | YYYY-MM-DD | [Name] | Initial spec | All | Initial creation |
| 1.1 | YYYY-MM-DD | [Name] | Added rate limiting to auth endpoints | CW-SEC-003, FR-002 | Security review feedback |
