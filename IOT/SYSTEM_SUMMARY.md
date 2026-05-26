# TeamPlatform — System Summary
> Generated: 2026-05-26  
> Purpose: Blueprint for inheriting this codebase's architecture, patterns, and reusable modules when building a new IoT-domain web application.

---

## Table of Contents
1. [System Rules & Conventions](#1-system-rules--conventions)
2. [Tech Stack](#2-tech-stack)
3. [Project Folder Tree](#3-project-folder-tree)
4. [Code Mapping (Architecture)](#4-code-mapping-architecture)
5. [Existing Features](#5-existing-features)
6. [Reusable Functions & Components](#6-reusable-functions--components)
7. [Web UI Layouts & Design System](#7-web-ui-layouts--design-system)
8. [Database Structure](#8-database-structure)
9. [Permission & Role System](#9-permission--role-system)
10. [Shared Types & API Contract](#10-shared-types--api-contract)
11. [Linked Variables & Enums](#11-linked-variables--enums)
12. [IoT Inheritance Recommendations](#12-iot-inheritance-recommendations)

---

## 1. System Rules & Conventions

### Database Safety (CRITICAL)
- **NEVER** run `prisma db push --force-reset`, `migrate reset`, `DROP DATABASE`, or `TRUNCATE` without a backup.
- Before any schema change: `docker exec team-platform-postgres pg_dump -U team_platform team_platform > backup_$(date +%Y%m%d_%H%M%S).sql`
- Use `prisma migrate deploy` with manually-written ADD-only SQL migrations, not `migrate dev`.
- Ask the user before any data-destructive operation.

### Backend Conventions
- Every module lives in `apps/api/src/modules/<domain>/` and exports exactly these files:
  - `<domain>.routes.ts` — Express router
  - `<domain>.service.ts` — business logic
  - `<domain>.repository.ts` — Prisma queries (optional, used when queries are complex)
  - `<domain>.schemas.ts` — Zod request/response schemas
- Routes always wrap handlers with `asyncHandler()` from `src/utils/async-handler.ts`.
- Errors are thrown as `new AppError(statusCode, message)` from `src/utils/app-error.ts`.
- Auth middleware chain: `requireAuth` → `requirePermission("perm.code")` → handler.
- `req.user` is populated with `AuthenticatedRequestUser` (id, email, roles[], permissions[]).
- Admin role bypasses all permission checks automatically (see `require-permission.ts`).

### Frontend Conventions
- **Pages** live in `apps/web/app/<route>/page.tsx` — they are RSC (React Server Components) that import a `*-client.tsx` feature component.
- **Feature components** live in `apps/web/features/<domain>/*-client.tsx` — they are `"use client"` and contain all interactive state.
- **Reusable UI** lives in `apps/web/components/ui/` — generic, domain-agnostic.
- File naming: `kebab-case` everywhere. `*-client.tsx` suffix = client component.
- Language: bilingual EN/VI via `useLanguage()` hook from `lib/i18n.tsx`. All user-facing strings go through `t("key")`.
- Auth is stored in `localStorage` (`accessToken`, `currentUser`). `AppShell` performs client-side auth guard.

### Git / Code Style
- TypeScript strict mode throughout.
- No `any` types; use Zod schemas for external data.
- Tailwind for all styling — no CSS modules or styled-components.
- Vitest for API unit tests (calculator logic, utility functions).
- ESLint enforced on both `api` and `web`.

---

## 2. Tech Stack

| Layer | Technology | Version |
|---|---|---|
| **API Runtime** | Node.js | ≥ 20 |
| **API Framework** | Express.js | 4.18 |
| **ORM** | Prisma | 5.13 |
| **Database** | PostgreSQL (Docker: `team-platform-postgres`) | — |
| **Queue / Jobs** | BullMQ + Redis (ioredis) | 5.7 / 5.4 |
| **Auth** | JWT (jsonwebtoken) | 9.0 |
| **Password hashing** | bcryptjs | 2.4 |
| **Validation** | Zod | 3.23 |
| **PDF rendering** | Puppeteer | 22.7 |
| **PDF parsing** | pdf-parse, pdfjs-dist | — |
| **Excel** | ExcelJS | 4.4 |
| **Fuzzy search** | Fuse.js | 7.3 |
| **Email** | Nodemailer | 6.9 |
| **File uploads** | Multer | 2.1 |
| **API Security** | Helmet, CORS, compression | — |
| **Frontend** | Next.js (App Router) | 14.2 |
| **UI Framework** | React | 18.2 |
| **Styling** | Tailwind CSS | 3.4 |
| **Icons** | Lucide React | 0.468 |
| **Language** | TypeScript | 5.4 |
| **Test runner** | Vitest | 1.5 |

**Ports:** API → `4000`, Web → `3000`  
**DB connection string:** `postgresql://team_platform:team_platform@localhost:5432/team_platform`  
**API start:** `cd apps/api && npm run dev` (tsx watch src/server.ts)

---

## 3. Project Folder Tree

```
TeamPlatform/                          # Monorepo root
├── apps/
│   ├── api/                           # Express API
│   │   ├── prisma/
│   │   │   ├── schema.prisma          # Single Prisma schema (all models)
│   │   │   ├── seed.ts                # Seed script
│   │   │   └── migrations/            # SQL migration files
│   │   ├── src/
│   │   │   ├── server.ts              # HTTP server entry point
│   │   │   ├── app.ts                 # Express app factory
│   │   │   ├── routes.ts              # Central router (mounts all module routers)
│   │   │   ├── config/
│   │   │   │   └── env.ts             # Validated env vars (Zod)
│   │   │   ├── middleware/
│   │   │   │   ├── authenticate.ts    # JWT bearer → req.user
│   │   │   │   ├── require-permission.ts  # RBAC gate
│   │   │   │   ├── error-handler.ts   # Centralized error responses
│   │   │   │   └── not-found-handler.ts
│   │   │   ├── utils/
│   │   │   │   ├── app-error.ts       # AppError class
│   │   │   │   └── async-handler.ts   # Express async wrapper
│   │   │   └── modules/               # Feature modules (see §4)
│   │   │       ├── auth/
│   │   │       ├── users/
│   │   │       ├── roles/
│   │   │       ├── permissions/
│   │   │       ├── employees/
│   │   │       ├── departments/
│   │   │       ├── positions/
│   │   │       ├── attendance/
│   │   │       ├── allowances/
│   │   │       ├── payroll/
│   │   │       ├── salary-advances/
│   │   │       ├── tax/
│   │   │       ├── projects/
│   │   │       ├── project-documents/
│   │   │       ├── audit/
│   │   │       ├── tasks/
│   │   │       ├── email/
│   │   │       ├── inventory/
│   │   │       ├── inventory-import-agent/
│   │   │       ├── material-operations/
│   │   │       ├── quotations/
│   │   │       ├── temples/
│   │   │       ├── device-local/
│   │   │       ├── imports/
│   │   │       ├── exports/
│   │   │       ├── reports/
│   │   │       ├── statistics/
│   │   │       ├── settings/
│   │   │       └── hr/                # Shared HR utilities (no direct routes)
│   │   └── uploads/                   # Uploaded file storage
│   │
│   └── web/                           # Next.js frontend
│       ├── app/                       # App Router pages
│       │   ├── layout.tsx             # Root layout (AppShell + Providers)
│       │   ├── providers.tsx          # React context providers
│       │   ├── globals.css            # Global styles
│       │   ├── page.tsx               # Dashboard (/)
│       │   ├── login/
│       │   ├── forgot-password/
│       │   ├── reset-password/
│       │   ├── change-password/
│       │   ├── employees/[id]/edit/
│       │   ├── departments/
│       │   ├── positions/
│       │   ├── attendance/
│       │   ├── payroll/[id]/
│       │   ├── projects/[id]/
│       │   ├── inventory/
│       │   │   ├── categories/
│       │   │   ├── items/[id]/
│       │   │   ├── movements/
│       │   │   ├── reports/
│       │   │   ├── settings/
│       │   │   ├── purchase-requests/
│       │   │   └── import-agent/
│       │   ├── quotations/[id]/
│       │   ├── material-operations/
│       │   ├── tasks/settings/
│       │   ├── device-local/
│       │   ├── reports/
│       │   ├── import-export/
│       │   ├── email/
│       │   ├── permissions/
│       │   ├── document-permissions/
│       │   └── admin/
│       │       ├── temples/
│       │       └── document-types/
│       ├── components/
│       │   ├── layout/
│       │   │   └── app-shell.tsx      # Sidebar + header + main layout
│       │   ├── brand/
│       │   │   └── team-logo.tsx
│       │   ├── module-card.tsx        # Dashboard feature card
│       │   └── ui/                    # Design system (see §6)
│       │       ├── button.tsx
│       │       ├── input.tsx
│       │       ├── textarea.tsx
│       │       ├── select.tsx
│       │       ├── combo-input.tsx
│       │       ├── currency-price-input.tsx
│       │       ├── controls.tsx       # FilterBar, ToolbarButton, fieldClassName
│       │       ├── data-table.tsx
│       │       ├── table-pagination.tsx
│       │       ├── status-badge.tsx
│       │       ├── tabs.tsx
│       │       ├── feedback.tsx       # Toast notifications
│       │       ├── alert.tsx
│       │       ├── dialog.tsx
│       │       ├── modal.tsx
│       │       ├── page-header.tsx
│       │       ├── page-skeleton.tsx
│       │       ├── smart-search.tsx
│       │       └── permission-gate.tsx
│       ├── features/                  # Domain feature components
│       │   ├── auth/
│       │   ├── hr/
│       │   ├── payroll/
│       │   ├── projects/
│       │   ├── project-documents/
│       │   ├── inventory/
│       │   ├── inventory-import-agent/
│       │   ├── quotations/
│       │   ├── material-operations/
│       │   ├── tasks/
│       │   ├── device-local/
│       │   ├── reports/
│       │   ├── email/
│       │   ├── import-export/
│       │   ├── permissions/
│       │   ├── temples/
│       │   └── document-types/
│       ├── lib/
│       │   ├── api-client.ts          # HTTP client (fetch wrapper + auth headers)
│       │   ├── auth.ts                # Token storage, user context helpers
│       │   └── i18n.tsx               # EN/VI translation provider + useLanguage hook
│       ├── services/                  # API call functions (per module)
│       └── tailwind.config.ts         # Design tokens
│
└── packages/
    └── shared/                        # Shared by both api and web
        └── src/
            ├── types.ts               # ApiResponse<T>, PaginationQuery, AuthUser
            ├── permissions.ts         # coreRoles[], permissions[] (source of truth)
            ├── list.ts                # Shared list utilities
            └── index.ts               # Re-exports
```

---

## 4. Code Mapping (Architecture)

### Backend Module Anatomy

Each module follows this exact pattern:

```
modules/<domain>/
├── <domain>.routes.ts      # Express Router — mounts endpoints, applies middleware
├── <domain>.service.ts     # Business logic — calls repository, throws AppError
├── <domain>.repository.ts  # Prisma queries — raw DB access only, no logic
└── <domain>.schemas.ts     # Zod schemas — validate req.body / req.query
```

**Route template:**
```typescript
// <domain>.routes.ts
import { Router } from "express";
import { requireAuth } from "../../middleware/authenticate";
import { requirePermission } from "../../middleware/require-permission";
import { asyncHandler } from "../../utils/async-handler";
import * as service from "./<domain>.service";

export const domainRouter = Router();

domainRouter.get("/", requireAuth, requirePermission("domain.view"), asyncHandler(async (req, res) => {
  const data = await service.list(req.query);
  res.json({ status: "ok", data });
}));
```

**Service template:**
```typescript
// <domain>.service.ts
import { AppError } from "../../utils/app-error";
import * as repo from "./<domain>.repository";

export async function list(query: unknown) {
  // validate, call repo, return
}

export async function getById(id: string) {
  const record = await repo.findById(id);
  if (!record) throw new AppError(404, "Not found");
  return record;
}
```

### Frontend Page Anatomy

Every page is a thin RSC wrapper that delegates to a `*-client.tsx`:

```tsx
// app/<route>/page.tsx  (Server Component — no "use client")
import { FeatureNameClient } from "../../features/<domain>/feature-name-client";

export default function Page() {
  return <FeatureNameClient />;
}
```

```tsx
// features/<domain>/feature-name-client.tsx
"use client";
import { useState, useEffect } from "react";
import { PageHeader } from "../../components/ui/page-header";
import { useLanguage } from "../../lib/i18n";

export function FeatureNameClient() {
  const { t } = useLanguage();
  const [data, setData] = useState([]);

  useEffect(() => { /* fetch from API */ }, []);

  return (
    <div className="space-y-6">
      <PageHeader title={t("Feature Name")} actions={<>...</>} />
      {/* content */}
    </div>
  );
}
```

### API Route Mounting (apps/api/src/routes.ts → app.ts)

All module routers are imported and mounted under `/api`:

```
GET  /api/auth/…
GET  /api/employees/…
GET  /api/inventory/…
GET  /api/quotations/…
…etc
```

---

## 5. Existing Features

### Authentication & Identity
| Feature | Description |
|---|---|
| JWT login | Email + password → access token stored in localStorage |
| Session replacement | SSE stream detects duplicate login; old session auto-expires |
| Password reset | Email link → token hash → new password |
| Change password | In-app form with current password confirmation |
| RBAC | Roles (admin, director, hr, accountant, pm, etc.) + granular permissions |
| Audit logging | All mutations logged with actor, action, target, old/new values |

### HR Management
| Feature | Description |
|---|---|
| Employees | Full profile: personal info, bank, ID card, emergency contact, linked user account |
| Departments | Hierarchy with code, status |
| Positions | Base salary, salary step amount |
| Salary advances | Track advance payments per employee |

### Attendance
| Feature | Description |
|---|---|
| Weekly grid | 7-day grid per employee: status (present/leave_paid/leave_unpaid/rest_day) |
| OT tracking | Regular OT hours, holiday OT hours, travel km per day |
| OT review workflow | Employees submit → HR approves/rejects per cell |
| Period locking | Past weeks auto-lock; admin can unlock |
| Monthly summary | Aggregate attendance stats |

### Payroll
| Feature | Description |
|---|---|
| Monthly calculation | Auto-calculates from attendance, allowances, tax, insurance |
| Tax brackets | Progressive tax bands, personal/dependent deductions |
| Insurance | BHXH/BHYT/BHTN rates (employee + employer) |
| Allowance types | Fixed, per-day, by attendance rate, manual bonus, deduction |
| Payslips | Per-employee PDF payslip (publish → employee can view own) |
| Status lifecycle | draft → finalized → published → locked |

### Project Management
| Feature | Description |
|---|---|
| Projects | Full project profile: customer info, budget, dates, location, status |
| Plans | Hierarchical plans with start/end dates, progress |
| Tasks | Assignment, priority, status, deadline, progress |
| Issues | Severity, assignee, root cause, solution |
| Materials | Planned vs used quantities, estimated/actual pricing |
| Costs | Labor, material, overhead cost entries |
| Members | Role-based project membership |
| Documents | File uploads with document types, security levels, approval workflow |

### Inventory Management
| Feature | Description |
|---|---|
| Item catalog | 3-level hierarchy: Category → ItemType → Item |
| Multi-warehouse | Items linked to warehouses and customers |
| Stock movements | Receipt, issue, adjustment, return, reservation, release |
| Purchase requests | Request → approve workflow |
| Import agent | AI-assisted PDF supplier quotation import |
| Supplier mapping | Learns column layouts per supplier |
| Reports | Stock value, low stock alerts |

### Quotations
| Feature | Description |
|---|---|
| Versioned quotations | Each quotation has numbered versions |
| Material line items | Sourced from inventory catalog |
| VAT calculation | Configurable VAT rate |
| Template system | Excel-based layout templates with canvas editor |
| PDF/Excel export | Puppeteer-rendered PDF, ExcelJS-built Excel |
| Approval workflow | draft → sent → approved/rejected |
| Stock-out integration | Approved quotation creates inventory stock-out |
| Project sync | Sync quotation materials into project material list |

### Tasks
| Feature | Description |
|---|---|
| Task management | Typed, grouped, prioritized tasks |
| Task statuses | Configurable status pipeline |
| Issues | Sub-issues within tasks |
| File attachments | Per-task file uploads |
| Cross-module linking | Tasks linked to projects, quotations, etc. |

### Device Management (`device-local`)
| Feature | Description |
|---|---|
| Network scanning | Discover devices on LAN (IP, MAC, hostname, vendor) |
| Camera management | RTSP/HLS streams with encrypted URL storage |
| Layout configuration | Grid layout for multi-camera views |
| Camera viewer | Embedded stream player in configurable grid |

### Email
| Feature | Description |
|---|---|
| SMTP configuration | Host, port, encryption, credentials (env var or direct) |
| Email templates | Code-based templates with subject/body |
| Queue | BullMQ queue with retry logic |
| Send log | Success/failure/retry tracking |

### Import / Export
| Feature | Description |
|---|---|
| Excel import | Template-based import for employees, inventory, etc. |
| JSON import (advanced) | Raw JSON mode for power users |
| Preview before confirm | Row-level validation with error highlighting |
| Export | Excel/CSV/PDF export for all major data sets |

### Reports & Statistics
| Feature | Description |
|---|---|
| HR stats | Employee counts, department breakdown |
| Payroll stats | Monthly salary totals, averages |
| Project progress | Budget vs actual, task completion |
| Inventory reports | Stock value, low stock, movement history |

### Admin / Settings
| Feature | Description |
|---|---|
| Temples (templates) | Generic template system with variable bindings and layout config |
| Document types | Define categories, allowed modules, descriptions |
| Document permissions | Role × document type × security level × action matrix |
| Quotation company settings | Company name, tax code, bank, signature presets |

---

## 6. Reusable Functions & Components

### Backend Utilities

#### `asyncHandler` — `src/utils/async-handler.ts`
Wraps any async Express handler and forwards errors to `next()`.
```typescript
export const asyncHandler = (fn: RequestHandler) =>
  (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);
```

#### `AppError` — `src/utils/app-error.ts`
```typescript
throw new AppError(404, "Resource not found");
throw new AppError(403, "Permission denied");
```

#### `requireAuth` — `src/middleware/authenticate.ts`
Validates Bearer token, loads user from DB, sets `req.user`.

#### `requirePermission(code)` — `src/middleware/require-permission.ts`
RBAC gate. Admin role bypasses all checks.
```typescript
router.delete("/:id", requireAuth, requirePermission("inventory.manage"), asyncHandler(...))
```

#### Zod schema pattern — `src/modules/<domain>/<domain>.schemas.ts`
```typescript
export const createItemSchema = z.object({
  name: z.string().min(1),
  code: z.string().min(1),
  categoryId: z.string().uuid(),
});
export type CreateItemInput = z.infer<typeof createItemSchema>;
```

### Frontend Utilities

#### `useLanguage()` — `lib/i18n.tsx`
```typescript
const { t, lang, setLang } = useLanguage();
// t("Save") → "Lưu" when lang="vi"
```

#### `lib/auth.ts`
```typescript
getStoredUser()        // → CurrentUser | undefined
getStoredAccessToken() // → string | undefined
clearAuthSession()     // wipe localStorage
hasPermission(user, "inventory.view")
```

#### `lib/api-client.ts`
Centralized fetch wrapper that:
- Injects `Authorization: Bearer <token>`
- Returns typed `ApiResponse<T>`
- Redirects to `/login` on 401

#### `services/<domain>-service.ts` pattern
Each domain has a `services/` file with typed async functions:
```typescript
export async function listInventoryItems(params: QueryParams) {
  return apiClient.get<InventoryItem[]>("/inventory", params);
}
```

### Frontend UI Components (all in `components/ui/`)

| Component | Props / Usage |
|---|---|
| `PageHeader` | `title`, `eyebrow?`, `description?`, `actions?` — standard page top |
| `PageSkeleton` | Loading state placeholder for full pages |
| `FilterBar` | Responsive grid container for filter controls (4→6 cols) |
| `ToolbarButton` | `variant: "primary" \| "secondary" \| "danger"` — action bar buttons |
| `fieldClassName(extra?)` | Returns consistent Tailwind input class string |
| `StatusBadge` | Colored badge for status values |
| `DataTable` | Sortable, paginated table with `columns[]` and `data[]` props |
| `TablePagination` | Page controls: prev/next + page indicator |
| `Tabs` | Tabbed navigation |
| `SmartSearch` | Debounced search input with combo behavior |
| `ComboInput` | Searchable select with keyboard navigation |
| `CurrencyPriceInput` | Number input with currency formatting |
| `Dialog` | Accessible modal dialog with backdrop |
| `Modal` | Simpler modal wrapper |
| `Alert` | Alert box (info / warning / error) |
| `Feedback` (toast) | Toast notification system |
| `PermissionGate` | `<PermissionGate permission="x.y">` — conditional render |

### Layout Component

#### `AppShell` — `components/layout/app-shell.tsx`
- Fixed sidebar (lg+) with gradient background `#1558b8 → #012060`
- Sticky top header with breadcrumb title, user info, language toggle, logout
- Mobile hamburger menu
- Route warmup prefetch on hover/focus
- SSE session-replacement listener
- Auth guard: redirects to `/login` if no token

---

## 7. Web UI Layouts & Design System

### Color Tokens (tailwind.config.ts)

| Token | Value | Usage |
|---|---|---|
| `primary` | `#013B85` | Brand color, links, active states |
| `primaryHover` | `#012d6b` | Primary button hover |
| `surface` | `#f7f8fb` | Page background |
| `border` | `#d9dee7` | All borders |
| `ink` | `#172033` | Main text |
| `muted` | `#64748b` | Secondary text, placeholders |

Sidebar gradient: `linear-gradient(160deg, #1558b8 0%, #0a4aa4 20%, #013B85 55%, #012060 100%)`

### Animations

| Name | Usage |
|---|---|
| `shimmer` | Loading skeleton animation |
| `slide-progress` | Top navigation progress bar |

### Page Layout Pattern

```
┌─────────────────────────────────────────────────────┐
│ SIDEBAR (fixed, 64px = w-64 on lg+)                  │
│  [Logo]                                               │
│  [Nav items]                                          │
│  ─────── SETTINGS ────────                            │
│  [Settings nav items]                                 │
└─────────────────────────────────────────────────────┘

Main area (lg:pl-64):
┌─────────────────────────────────────────────────────┐
│ HEADER (sticky, h-[68px], bg-white, shadow)          │
│  [Menu button (mobile)] [Page title] [User info]     │
│  [Language toggle] [Logout]                           │
├─────────────────────────────────────────────────────┤
│ MAIN (px-4 py-6 md:px-8)                            │
│  <PageHeader title="..." actions={...} />            │
│  <FilterBar>                                         │
│    [filter inputs]                                   │
│  </FilterBar>                                        │
│  <DataTable columns={...} data={...} />              │
│  <TablePagination ... />                             │
└─────────────────────────────────────────────────────┘
```

### Standard Page Structure (feature client)

```tsx
<div className="space-y-6">
  <PageHeader
    title={t("Module Name")}
    description={t("Brief description")}
    actions={<ToolbarButton variant="primary">+ {t("Add")}</ToolbarButton>}
  />

  <FilterBar>
    <input className={fieldClassName()} placeholder={t("Search...")} />
    <select className={fieldClassName()}>...</select>
  </FilterBar>

  <DataTable columns={columns} data={rows} />
  <TablePagination page={page} total={total} pageSize={PAGE_SIZE} onChange={setPage} />

  {/* Detail modal / slide-over */}
  {selected && (
    <Dialog open onClose={() => setSelected(null)}>
      ...
    </Dialog>
  )}
</div>
```

### Navigation Structure

**Main nav** (visible to all permitted users):
Dashboard → HR → Attendance → Payroll → Projects → Tasks → Inventory → Purchase Requests → Device Local → Quotations → Templates → Document Types

**Settings nav** (grouped separately at bottom of sidebar):
Departments → Positions → Payroll Settings → Task Settings → Inventory Settings → Quotation Settings → Documents → Permissions → Import/Export → Email

### Sidebar Navigation Rules
- `adminOnlyNavigationHrefs`: Employees, Departments, Positions, Import/Export, Permissions, Email — admin-only
- Permission-gated items: Tasks, Inventory, Purchase Requests, Device Local, Quotations, Payroll Settings, etc.
- Active state: `bg-white/20 text-white font-semibold`
- Inactive state: `text-white/75 hover:bg-white/10`

---

## 8. Database Structure

> All models use `cuid()` as default ID unless noted otherwise.  
> All models include `createdAt` and `updatedAt` (DateTime) unless noted.

### Auth & RBAC

```
User
  id, email, passwordHash, fullName
  employeeId? → Employee
  isActive, lastLoginAt, activeSessionId?
  ↳ userRoles[] → UserRole
  ↳ passwordResetTokens[] → PasswordResetToken

Role
  id, code, name, description
  status: RecordStatus
  ↳ rolePermissions[] → RolePermission
  ↳ userRoles[] → UserRole

Permission
  id, code, name, description
  ↳ rolePermissions[] → RolePermission

UserRole         — userId + roleId (junction)
RolePermission   — roleId + permissionId (junction)

PasswordResetToken
  userId, tokenHash, expiresAt, usedAt?

AuditLog
  actorId, action, module
  targetType, targetId
  oldValue?, newValue? (JSON)
  metadata? (JSON)
  ipAddress?, userAgent?
```

### HR

```
Department
  id, code, name, description?
  status: RecordStatus

Position
  id, code, name
  baseSalary: Decimal
  salaryStepAmount: Decimal
  status: RecordStatus

Employee
  id, employeeCode (unique)
  fullName, phone?, email?
  dateOfBirth?, gender?, address?
  avatarUrl?
  citizenIdNumber?, bankName?, bankAccountNumber?
  departmentId → Department
  positionId → Position
  salaryLevel: Int (step multiplier)
  startDate, employmentType: EmploymentType
  status: EmployeeStatus
  emergencyContactName?, dependentCount: Int
  metadata? (JSON — ID card images, issue date, etc.)
  ↳ user? → User
  ↳ attendanceRecords[]
  ↳ payrolls[]
```

### Attendance

```
AttendanceRecord
  id
  employeeId → Employee
  projectId? → Project
  date: DateTime
  status: AttendanceStatus
  note?, otHours: Float, holidayOtHours: Float, travelKm: Float
  reviewStatus: AttendanceReviewStatus

AttendanceLock
  id
  month: String ("YYYY-MM")
  isLocked: Bool
  lockedById? → User
  lockedAt?
```

### Payroll

```
Payroll
  id
  employeeId → Employee
  month: String ("YYYY-MM")
  positionSalary: Decimal
  workingDays: Float, presentDays: Float
  allowances, bonuses, deductions, insurances, taxes, netSalary: Decimal
  status: PayrollStatus
  ↳ items[] → PayrollItem

PayrollItem
  id
  payrollId → Payroll
  allowanceTypeId? → AllowanceType
  name, code, type
  amount: Decimal
  isTaxable: Bool, isInsuranceBased: Bool

PayrollMonthlyInput
  id
  employeeId → Employee
  month: String
  attendanceAllowance?, mealAllowanceRate?, travelAllowanceRate?
  responsibilityAllowance?, kpiAllowance?, projectBonus?
  salaryAdvance?: Decimal

AllowanceType
  id, code (unique), name
  calculationType: AllowanceCalculationType
  amount: Decimal, unit?
  isTaxable: Bool, isInsuranceBased: Bool
  applyScope: ApplyScope
  status: RecordStatus

TaxSetting
  id
  personalDeduction, dependentDeduction: Decimal
  socialInsuranceRate, healthInsuranceRate, unemploymentInsuranceRate: Float
  effectiveFrom: DateTime
  status: RecordStatus
  ↳ taxBrackets[] → TaxBracket

TaxBracket
  id
  taxSettingId → TaxSetting
  level: Int
  incomeFrom, incomeTo?: Decimal
  taxRate: Float
```

### Projects

```
Project
  id, projectCode (unique)
  name, customerName?, customerContactName?, customerPhone?
  customerEmail?, customerAddress?, customerTaxCode?
  description?
  managerId? → Employee
  startDate?, endDate?, actualCompletedDate?
  location?, status: ProjectStatus
  progressPercent: Float
  budgetEstimated?, budgetActual?: Decimal
  ↳ members[] → ProjectMember
  ↳ plans[] → ProjectPlan
  ↳ tasks[] → ProjectTask
  ↳ issues[] → ProjectIssue
  ↳ materials[] → ProjectMaterial
  ↳ costs[] → ProjectCost
  ↳ documents[] → ProjectDocument

ProjectMember
  projectId → Project, employeeId → Employee
  projectRole, joinedDate, leftDate?, status: RecordStatus

ProjectPlan
  projectId, name, description?
  startDate?, endDate?
  ownerId? → Employee
  progressPercent: Float
  status: RecordStatus, sortOrder: Int

ProjectTask
  projectId, planId? → ProjectPlan
  title, description?
  assigneeId? → Employee, assignedById? → Employee
  priority, status: ProjectTaskStatus
  progressPercent: Float
  startDate?, deadline?, confirmedAt?, submittedAt?, completedAt?

ProjectIssue
  projectId, taskId? → ProjectTask
  title, description?
  severity, reportedById → Employee
  assignedToId? → Employee
  status, rootCause?, solution?
  deadline?, closedAt?

ProjectMaterial
  projectId, quotationId?, quotationVersionId?
  materialCode, materialName, unit
  plannedQuantity, usedQuantity, remainingQuantity: Float
  estimatedUnitPrice, actualUnitPrice?: Decimal
  supplierName?, neededDate?
  status: MaterialStatus

ProjectCost
  projectId, costType, name
  amount: Decimal, costDate, note?

PurchaseRequest
  projectId, materialId? → ProjectMaterial
  requestedById → Employee, approvedById? → Employee
  quantity: Float, reason?, neededDate?
  status
```

### Task Management

```
Task
  id, taskCode (unique)
  title, description?
  assigneeEmployeeId? → Employee
  managerEmployeeId? → Employee
  taskTypeId? → TaskType
  taskStatusId? → TaskStatus
  projectId? → Project
  taskGroupId? → TaskGroup
  priority, progressPercent: Float
  deadline?, completedAt?
  relatedModuleType?, relatedRecordId?, relatedRecordLabel?, relatedRecordUrl?
  ↳ issues[] → TaskIssue
  ↳ files[] → TaskFile

TaskType     — code, name, description?, sortOrder, status
TaskStatus   — code, name, color, sortOrder, isCompleted, status
TaskGroup    — projectId?, code, name, description?

TaskIssue
  taskId, content
  status, progressPercent: Float
  completedAt?
  reportedById, closedById?

TaskFile
  taskId, issueId?
  fileName, fileUrl, fileSize, mimeType
  relativePath?, uploadedById
```

### Inventory

```
InventoryCategory
  id, code, name, department?, description?
  status: RecordStatus

InventoryItemType
  id, code, name, description?
  categoryId → InventoryCategory
  status: RecordStatus

InventorySupplier
  id, code, name, contactName?, phone?, email?
  address?, taxCode?, description?
  status: RecordStatus

InventoryWarehouse
  id, code, name, location?, description?
  status: RecordStatus

InventoryCustomer
  id, code, name, contactName?, phone?, email?
  address?, taxCode?, description?
  status: RecordStatus

InventoryItem
  id, materialCode (unique), materialName
  model?, categoryId, supplierId?, typeId?, warehouseId?, customerId?
  purchasePrice, sellingPrice, markupPercentage: Decimal
  stockQuantity, minimumStockQuantity: Float
  unit, status: RecordStatus
  imageUrl?, description?
  ↳ stockMovements[] → InventoryStockMovement

InventoryStockMovement
  id
  itemId → InventoryItem
  movementType: InventoryMovementType
  quantity: Float, unitCost: Decimal
  previousStock, resultingStock: Float
  referenceType?, referenceId?
  materialOperationId? → MaterialOperation
  projectId? → Project
  note?

InventoryPurchaseRequest
  id, requestCode
  inventoryItemId → InventoryItem
  materialCode, materialName, unit
  quantity: Float
  estimatedUnitPrice?, actualUnitPrice?: Decimal
  supplierName?, reason?, neededDate?
  status

InventoryImportAgentDraft
  id
  sourceType, fileName?, fileUrl?
  supplierId?, projectId?, categoryId?, typeId?
  applyMode, status
  rawText?, pages?, layoutBlocks?, blockAssignments? (JSON)
  summaryData?, columnMapping?, mappingConfidence?, rowExtractionConfidence? (JSON)
  ↳ items[] → InventoryImportAgentDraftItem

InventoryImportAgentDraftItem
  id, draftId → Draft
  lineIndex: Int
  materialCode?, materialName?, model?, unit?
  quantity?, unitPrice?, amount?: Decimal
  currency?, matchStatus, approvalStatus, applyAction?

SupplierImportMapping
  id
  supplierId → InventorySupplier
  supplierName, sourceType
  headerSignature?, columnMapping (JSON)
  confidence: Float
  versionNumber: Int, versionName?, isDefault: Bool

MaterialOperation
  id, operationCode, operationType, status
  purchaseStatus, operationDate
  sourceType?, sourceId?, quotationId?
  projectId?, note?
  ↳ lines[] → MaterialOperationLine

MaterialOperationLine
  id, operationId, lineIndex
  inventoryItemId? → InventoryItem
  quotationVersionItemId?
  materialCodeSnapshot, materialNameSnapshot, modelSnapshot?, unitSnapshot
  requestedQuantity, processedQuantity: Float
  unitCost: Decimal

FileType
  id, name, code, allowedExtensions (JSON), maxFileSize: Int
  description?, isActive: Bool

MaterialFile
  id
  materialId → InventoryItem
  fileTypeId → FileType
  fileName, filePath, fileSize: Int, mimeType
  uploadedById → User, uploadedAt
```

### Quotations

```
Quotation
  id, quotationCode
  quotationType: QuotationType
  projectId? → Project
  currentVersionId? → QuotationVersion
  templeId? → Temple
  customerName?, recipientName?, customerRequest?
  content? (JSON), quotationDate
  numberOfSets: Int
  vatEnabled: Bool, vatRate: Float
  subtotalOneSet, totalBeforeVat, vatAmount, grandTotal: Decimal
  signatureImageUrl?
  status: QuotationStatus
  ↳ versions[] → QuotationVersion
  ↳ items[] → QuotationItem (legacy)

QuotationVersion
  id, quotationId → Quotation
  versionNumber: Int
  quotationType, projectId?, customerName?, …
  (mirrors Quotation fields + workflow fields:)
  customerPoFileName?, customerPoFileUrl?
  approvedAt?, approvedById?, stockOutCreatedAt?, projectSyncedAt?
  ↳ items[] → QuotationVersionItem

QuotationVersionItem
  id, quotationVersionId
  lineIndex: Int
  materialId? → InventoryItem
  materialCodeSnapshot, materialNameSnapshot, modelSnapshot?
  pictureUrlSnapshot?, unitSnapshot
  quantity, unitPrice, amount: Decimal
  vatRate: Float, vatAmount: Decimal

QuotationTemplate
  id, quotationId → Quotation
  defaultVersionId? → QuotationTemplateVersion
  name, fileName?, fileUrl?, fileSize?, mimeType?
  placeholderConfig? (JSON), detectedPlaceholders? (JSON)
  isDefault: Bool, status: RecordStatus
  ↳ versions[] → QuotationTemplateVersion

QuotationTemplateVersion
  id, templateId
  versionNumber: Int
  originalFileUrl?
  sheetName?, layoutConfig?, placeholderConfig?, tableConfig?, canvasConfig? (JSON)
  status: RecordStatus

QuotationBlockGroupTemplate
  id, name, category?
  blocks (JSON)
  builtIn: Bool, status: RecordStatus

QuotationCompanySettings
  id, companyName?, taxCode?, address?, phone?, email?
  bankAccountNumber?, bankName?, bankBranch?
  (+ signature preset fields)
```

### Templates (Temples)

```
Temple
  id, name, description?
  currentVersionId? → TempleVersion
  status: RecordStatus

TempleVersion
  id, templeId → Temple
  versionNumber: Int
  layoutConfig? (JSON), tableConfig? (JSON)
  placeholderConfig? (JSON), detectedPlaceholders? (JSON)
  status: RecordStatus

TempleVariableBinding
  id, templeId → Temple
  columnKey, variableKey, module, dataType
```

### Document Management

```
DocumentType
  id, code (unique), name, category
  allowedModules (JSON), description?
  status: RecordStatus

ProjectDocument
  id, projectId → Project
  documentTypeId → DocumentType
  title, fileName?, fileUrl?
  fileSize?, mimeType?
  version: Int
  securityLevel: SecurityLevel
  status: DocumentStatus
  uploadedById → User
  approvedById? → User, approvedAt?
  note?
  ↳ accessLogs[] → DocumentAccessLog

DocumentPermission
  id
  documentTypeId → DocumentType
  roleId → Role
  securityLevel: SecurityLevel
  canView, canUpload, canEdit, canDelete, canDownload, canApprove: Bool

DocumentAccessLog
  id, documentId → ProjectDocument
  userId → User
  action: DocumentAction
  ipAddress?, userAgent?, metadata? (JSON)
```

### Device / Camera

```
DeviceLocalDiscoveredDevice
  id, ipAddress, macAddress?
  hostname?, deviceName?, vendor?, deviceType: DeviceLocalDeviceType
  source, lastSeenAt, metadata? (JSON)

DeviceLocalCamera
  id, code (unique), name
  discoveredDeviceId? → DiscoveredDevice
  ipAddress, macAddress?
  streamUrlEncrypted (AES-256), streamType
  localViewerPath?
  location?, description?
  status: RecordStatus

DeviceLocalLayout
  id, name, isDefault: Bool
  rows: Int, columns: Int
  ↳ cells[] → DeviceLocalLayoutCell

DeviceLocalLayoutCell
  id, layoutId → Layout
  cameraId? → Camera
  rowIndex: Int, columnIndex: Int
  displayName?
```

### Email & Logging

```
EmailSetting
  id, host, port: Int
  username, passwordEncrypted?
  fromEmail, fromName
  encryption: EmailEncryption
  isActive: Bool

EmailTemplate
  id, code (unique), name, subject, body
  isActive: Bool

EmailLog
  id, toEmail, subject, templateCode?
  status, errorMessage?
  sentAt?

ImportLog
  id, importType, fileName
  status, totalRows, successRows, failedRows: Int
  errorFileUrl?, importedById → User

ExportLog
  id, exportType, fileName, format
  filtersJson? (JSON)
  exportedById → User
```

---

## 9. Permission & Role System

### Core Roles
```
admin            — full access (bypasses all permission checks)
director         — view-heavy access
hr               — HR, attendance, payroll inputs
accountant       — payroll, finance
project_manager  — projects, tasks
team_leader      — team management
project_employee — project participation
employee         — own attendance, payslip
customer_partner — limited external access
```

### Permission Codes (grouped by module)

```
auth.me
users.manage
roles.manage

employees.view | employees.manage | employees.view_sensitive

attendance.view | attendance.manage | attendance.own.submit | attendance.own.update
attendance.own.view | attendance.ot.review | attendance.travel.review
attendance.admin | attendance.lock | attendance.unlock | attendance.audit.view

payroll.view | payroll.input.manage | payroll.review | payroll.configure
payroll.manage | payroll.publish | payroll.lock | payroll.unlock
payroll.template.manage | payroll.email.send | payroll.bank.view | payroll.audit.view
payslip.own.view | payslip.own.download | payslip.own.acknowledge

projects.view | projects.manage

tasks.view | tasks.create | tasks.update | tasks.assign | tasks.status.update
tasks.progress.update | tasks.complete | tasks.reopen | tasks.cancel | tasks.delete
tasks.files.upload | tasks.files.download | tasks.issues.create | tasks.issues.close
tasks.settings.manage

project_documents.view | project_documents.upload | project_documents.download
project_documents.approve | project_documents.manage

imports.manage | exports.manage

inventory.view | inventory.manage | inventory.categories.manage | inventory.types.manage
inventory.suppliers.manage | inventory.adjust_stock | inventory.stock_in | inventory.stock_out
inventory.history.view | inventory.import | inventory.export | inventory.view_cost
inventory.warehouses.manage | inventory.customers.manage | inventory.file-types.manage
inventory_import_agent.parse | inventory_import_agent.view | inventory_import_agent.review
inventory_import_agent.apply | inventory_import_agent.mapping_manage

material_operations.view | material_operations.create | material_operations.edit
material_operations.delete | material_operations.approve | material_operations.reject
material_operations.print | material_operations.export | material_operations.purchase_update

device_local.view | device_local.manage | device_local.scan | device_local.sensitive.view
device_local.layout.view | device_local.layout.manage | device_local.audit.view

quotations.view | quotations.manage | quotations.preview | quotations.export
quotations.delete | quotations.approve | quotations.version.view | quotations.version.create
quotations.stock_out | quotations.customer_po.upload
quotation_templates.view | quotation_templates.manage | quotation_templates.layout_edit
quotation_templates.version_manage | quotation_settings.manage

email.manage
audit_logs.view
```

---

## 10. Shared Types & API Contract

### From `packages/shared/src/types.ts`

```typescript
// All API responses use this wrapper
interface ApiResponse<T> {
  status: "ok" | "error";
  data: T;
}

interface ApiErrorResponse {
  status: "error";
  message: string;
  details?: unknown;
}

// Used for list endpoints with pagination
interface PaginationQuery {
  page?: number;
  pageSize?: number;
  search?: string;
}

// JWT payload / req.user shape
interface AuthUser {
  id: string;
  email: string;
  fullName: string;
  roles: string[];
  permissions: string[];
}
```

### Standard List Response
```json
{
  "status": "ok",
  "data": {
    "items": [...],
    "total": 123,
    "page": 1,
    "pageSize": 20
  }
}
```

### Standard Error Response (from `error-handler.ts`)
```json
{
  "status": "error",
  "message": "Resource not found"
}
```

---

## 11. Linked Variables & Enums

### Prisma Enums

```prisma
enum RecordStatus       { active, inactive }

enum EmployeeStatus     { probation, active, temporarily_inactive, resigned }
enum EmploymentType     { official, probation, seasonal, part_time }

enum AttendanceStatus   { present, leave_paid, leave_unpaid, rest_day }
enum AttendanceReviewStatus { pending, approved, rejected }

enum AllowanceCalculationType {
  fixed_monthly, per_working_day, attendance_rate,
  manual_bonus, project_bonus, deduction
}
enum ApplyScope         { company, department, position, employee }
enum PayrollStatus      { draft, finalized, published, locked }

enum ProjectStatus      { planning, in_progress, paused, completed, cancelled }
enum ProjectTaskStatus  { todo, confirmed, in_progress, pending_review, completed, cancelled }
enum MaterialStatus     {
  not_ordered, purchase_requested, purchasing,
  received, issued, shortage, cancelled
}

enum SecurityLevel      {
  project_public, internal_company, pm_admin_only,
  accounting, confidential, client_shared
}
enum DocumentStatus     { draft, pending_approval, approved, rejected, archived }
enum DocumentAction     { view, upload, edit, delete, download, approve, reject }

enum EmailEncryption    { none, ssl, starttls }

enum InventoryMovementType {
  receipt, issue, adjustment, return, reservation, release
}

enum QuotationStatus    { draft, sent, approved, rejected, cancelled }
enum QuotationType      { commercial, project }

enum DeviceLocalDeviceType {
  unknown, camera, nvr, router, printer, computer, phone, other
}
```

### Key ENV Variables (from `src/config/env.ts`)
```
DATABASE_URL           postgresql://…
JWT_SECRET             signing secret
JWT_EXPIRES_IN         token lifetime
REDIS_URL              redis://…
PORT                   4000
FRONTEND_URL           http://localhost:3000
SMTP_*                 email configuration
UPLOAD_DIR             file upload path
ENCRYPTION_KEY         AES key for camera URLs
```

---

## 12. IoT Inheritance Recommendations

### What to Keep Exactly As-Is
| Component | Why |
|---|---|
| `AppShell` layout | Complete, responsive, bilingual sidebar — just replace nav items |
| Auth system (JWT + RBAC) | Fully generic; add IoT-specific roles/permissions to `packages/shared` |
| `packages/shared` types | `ApiResponse<T>`, `PaginationQuery`, `AuthUser` are universal |
| Middleware (`requireAuth`, `requirePermission`, `asyncHandler`, `AppError`) | 100% reusable |
| All `components/ui/` | Design-system components have zero domain coupling |
| `lib/i18n.tsx` | Add new translation keys; framework is domain-agnostic |
| `lib/auth.ts` + `lib/api-client.ts` | Universal auth/fetch helpers |
| `AuditLog` model | Essential for IoT operations traceability |
| Email module | BullMQ queue + templates are production-ready |
| Import/Export module | Excel import pipeline reusable for IoT device config bulk import |

### What to Adapt
| Component | Adaptation Needed |
|---|---|
| Sidebar navigation | Replace HR/Inventory items with IoT items (Devices, Sensors, Dashboards, Alerts, etc.) |
| `DeviceLocalCamera` module | Expand into full IoT device management (MQTT topics, telemetry streams) |
| `InventoryItem` model | Can become an IoT "Asset" model (device catalog) |
| `Task` module | Reuse for maintenance workflows / work orders |
| `ProjectDocument` module | Reuse for device manuals, firmware files |
| `Temple` / `QuotationTemplate` system | Reuse as a report/alert template engine |

### New IoT-Specific Modules to Add
| Module | Suggested Models / Features |
|---|---|
| `IoTDevice` | device EUI, type, firmware version, location, status, lastSeenAt, metadata |
| `IoTGateway` | gateway that proxies device traffic |
| `IoTTelemetry` | time-series readings (deviceId, metric, value, timestamp) |
| `IoTAlert` | threshold rules + triggered alert events |
| `IoTDashboard` | configurable widget panels (inherit Temple canvas editor) |
| `IoTCommand` | downlink command log (device, command, payload, ack) |
| `IoTFirmware` | version catalog, OTA rollout tracking |
| `IoTSite` | physical location / installation site hierarchy |

### Naming Convention for IoT Permissions
Follow the exact same dot-notation pattern already established:
```
iot_devices.view | iot_devices.manage | iot_devices.configure
iot_telemetry.view | iot_telemetry.export
iot_alerts.view | iot_alerts.manage | iot_alerts.acknowledge
iot_gateways.view | iot_gateways.manage
iot_firmware.view | iot_firmware.manage | iot_firmware.deploy
iot_dashboard.view | iot_dashboard.manage
```

### Database Considerations for IoT
- **Telemetry volume**: Use a time-series extension (TimescaleDB) or separate table partitioned by month. Do NOT store in the main PostgreSQL tables without partitioning.
- **Device state**: Keep a separate `IoTDeviceState` table (latest reading only, indexed on `deviceId`) for real-time queries instead of scanning `IoTTelemetry`.
- **MQTT broker**: Add Mosquitto/EMQX as a new Docker service. API connects via `mqtt` npm package.
- **WebSocket/SSE for live data**: The existing SSE pattern in `session-events.ts` can be copied for live telemetry push to the frontend.
