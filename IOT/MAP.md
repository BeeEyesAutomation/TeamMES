# MAP.md

## Current Code Map

```txt
team-platform
+-- apps/
|   +-- api/                 Express + TypeScript API
|   |   +-- prisma/          PostgreSQL Prisma schema, migrations, seed data
|   |   +-- src/
|   |       +-- app.ts       Express middleware and /api router mount
|   |       +-- routes.ts    API route registry
|   |       +-- config/      Environment parsing
|   |       +-- jobs/        BullMQ email queue and worker
|   |       +-- middleware/  Auth, RBAC, errors, not-found handling
|   |       +-- modules/     Business modules by domain
|   |       +-- prisma/      Prisma client singleton
|   |       +-- utils/       Shared API utilities
|   +-- web/                 Next.js + React + Tailwind UI
|       +-- app/             App Router pages
|       +-- components/      Layout, brand, and reusable UI components
|       +-- features/        Feature-specific clients and API wrappers
|       +-- lib/             API client and auth helpers
|       +-- public/          Brand assets
|       +-- services/        Frontend service helpers
|       +-- types/           Frontend domain types
+-- packages/
|   +-- shared/              Shared permissions and common types
+-- docs/                    Implementation log, decisions, TODOs, setup notes
+-- Data/                    Local brand/source assets
```

## Backend Modules

All backend routes are mounted under `/api` from `apps/api/src/routes.ts`.

```txt
/api/auth                         Login, logout, current user, password flows
/api/users                        User listing/admin foundation
/api/roles
/api/permissions                  RBAC role and permission data
/api/employees                    Employee profiles and sensitive-field handling
/api/departments                  Department CRUD
/api/positions                    Position, base salary, and salary step CRUD
/api/attendance                   Manual attendance, month summaries, locks
/api/allowance-types
/api/allowance-rules
/api/employee-monthly-allowances  Allowance configuration and monthly values
/api/payroll
/api/payrolls
/api/payslips                     Payroll calculation, publish/finalize, payslips
/api/salary-advances              Salary advance foundation
/api/tax-settings
/api/tax-brackets                 Configurable tax and insurance rules
/api/projects                     Project CRUD, dashboard, profile, timeline
/api/projects/:id/plans
/api/projects/:id/tasks
/api/projects/:id/issues
/api/projects/:id/materials
/api/projects/:id/costs
/api/projects/:id/members         Nested project operations
/api/project-plans
/api/project-tasks
/api/project-issues
/api/project-materials
/api/project-costs
/api/project-members              Direct child-resource update/delete routes
/api/document-types
/api/document-permissions
/api/project-documents
/api/projects/:id/documents       Project document metadata and access logs
/api/email                        SMTP settings, templates, logs, retry/test queue
/api/imports                      Template, preview, confirm, error-file workflow
/api/exports                      Excel, CSV, and PDF export generation
/api/statistics                   HR, payroll, project, issue, material, cost stats
/api/reports                      Report route foundation
/api/audit-logs                   Audit log listing
/api/quotations                   Quotation CRUD, versions, preview, Excel export, approval, customer PO, quotation stock out
/api/quotation-templates          Excel template upload, placeholder mapping, default template management
/api/quotation-template-versions  Canvas layout JSON, material table config, duplicate/restore template versions
/api/quotation-settings           Company quotation header/settings
```

## Frontend Routes

```txt
/                               Dashboard
/login                          Login
/forgot-password                Password reset request
/reset-password                 Password reset completion
/change-password                Authenticated password change
/employees                      Employee list
/employees/new                  Create employee
/employees/:id                  Employee detail
/employees/:id/edit             Edit employee
/departments                    Department list
/departments/new                Create department
/departments/:id/edit           Edit department
/positions                      Position list
/positions/new                  Create position
/positions/:id/edit             Edit position
/attendance                     Manual attendance screen
/payroll                        Payroll runs
/payroll/:id                    Payroll detail
/payroll/payslip                Payslip lookup/view
/payroll/settings               Payroll, allowance, tax settings
/projects                       Project list
/projects/:id                   Project detail tabs
/document-permissions           Document type/permission admin
/import-export                  Import templates, preview/confirm, exports
/reports                        Reporting/statistics dashboard
/email                          SMTP settings, templates, logs
/quotations                     Quotation list
/quotations/new                 Create quotation
/quotations/:id                 Quotation detail, versions, preview, approval, customer PO, stock out, export
/quotations/:id/edit            Edit quotation and create a new latest version
/quotations/templates           Quotation Excel template manager
/quotations/settings            Company quotation settings
```

Navigation labels are Vietnamese in `apps/web/components/layout/app-shell.tsx`, while implementation names, paths, and code stay in English.

## Domain Map

```txt
Auth and RBAC
+-- Users, roles, permissions, role assignments
+-- JWT auth middleware and permission gates
+-- Audit context for sensitive/state-changing operations

HR
+-- Departments
+-- Positions
|   +-- baseSalary
|   +-- salaryStepAmount
|   +-- salaryLevel salary preview
+-- Employees
    +-- profile and employment status
    +-- Citizen ID fields and images
    +-- bank fields
    +-- emergency contacts
    +-- sensitive data masking by permission

Attendance
+-- Manual roll call by date
+-- Active employees default to present
+-- Saved exceptions: paid leave, unpaid leave, absence/status records
+-- OT hours, travel kilometers, optional project linkage
+-- Monthly summaries and attendance locks

Payroll
+-- Position salary
+-- Configurable allowance types
+-- Monthly allowances, bonuses, deductions, advances
+-- Configurable tax settings and brackets
+-- Insurance and personal income tax calculation
+-- Draft, finalize, lock/publish, payslip email trigger

Projects
+-- Project profile, status, customer/contract fields
+-- Plans/phases
+-- Tasks with assign, confirm, progress, submit, approve, return
+-- Issues with create, update, close, reopen
+-- Materials, costs, members
+-- Dashboard summaries and audit-backed timeline

Project Documents
+-- Document types
+-- Security levels
+-- Role/document/security/action permission matrix

Quotations
+-- Quotation master records with commercial/project type
+-- Immutable quotation versions and version item snapshots
+-- Inventory material selection and Decimal-safe quotation calculations
+-- Uploaded Excel templates with placeholder mappings
+-- Built-in preview and Excel export
+-- Approval, customer PO upload, and one stock-out per approved version
+-- Metadata upload, view, edit/archive, approve/reject, download URL
+-- Access logs for sensitive document actions

Email Automation
+-- Configurable SMTP settings
+-- Email templates
+-- Email logs and retry
+-- BullMQ + Redis queue
+-- Nodemailer worker
+-- Triggers from tasks, issues, payroll, attendance/document workflows

Import/Export and Reports
+-- Excel templates
+-- JSON-row preview/confirm import foundation
+-- Import logs and error file download
+-- Excel/CSV/PDF exports
+-- Export logs
+-- Statistics and reporting dashboards
```

## Prisma Model Map

```txt
Auth/RBAC
+-- User
+-- PasswordResetToken
+-- Role
+-- Permission
+-- UserRole
+-- RolePermission
+-- AuditLog

HR and Attendance
+-- Department
+-- Position
+-- Employee
+-- AttendanceRecord
+-- AttendanceLock

Payroll, Tax, Allowances
+-- TaxSetting
+-- TaxBracket
+-- AllowanceType
+-- Payroll
+-- PayrollItem

Projects
+-- Project
+-- ProjectMember
+-- ProjectPlan
+-- ProjectTask
+-- ProjectIssue
+-- ProjectMaterial
+-- ProjectCost
+-- PurchaseRequest

Documents
+-- DocumentType
+-- ProjectDocument
+-- DocumentPermission
+-- DocumentAccessLog

Email and Files
+-- EmailSetting
+-- EmailTemplate
+-- EmailLog
+-- ImportLog
+-- ExportLog
```

## Data Flow Map

```txt
Employee profile
-> Position and salary level
-> Attendance month
-> Allowances, bonuses, deductions, advances
-> Tax and insurance settings
-> Payroll calculation
-> HR review/finalize
-> Payslip publication
-> Optional queued email notification
```

```txt
Project profile
-> Plans and members
-> Tasks, confirmations, progress updates
-> Issues, materials, and costs
-> Project documents and approval workflow
-> Audit timeline
-> Dashboard statistics
-> Export/report generation
```

```txt
Project document upload metadata
-> Route permission check
-> Document permission matrix check
-> Project membership/security-level check
-> Optional approval or rejection
-> Access log write
-> View/download permission check
```

```txt
Import
-> Download template
-> Submit rows
-> Validate and preview
-> Confirm import
-> Save supported records
-> Write import log and optional error file
```

```txt
Export
-> Choose report type and filters
-> Generate Excel, CSV, or PDF
-> Return file response
-> Write export log
```

```txt
Quotation
-> Select project or enter customer directly
-> Search Inventory materials
-> Snapshot material code, name, unit, and selling price
-> Calculate one-set subtotal, total before VAT, VAT amount, and grand total
-> Upload quotation images and signature image
-> Export quotation Excel file
-> Write audit and export logs
```

## Quotation Module Map

Implemented backend:

```txt
/api/quotations                   Quotation list/create/detail/update/deactivate
/api/quotations/:id/images        Quotation image upload
/api/quotations/:id/signature     Signature image upload
/api/quotations/:id/export/excel  Quotation Excel export
/api/quotation-template-versions  Quotation template version layout/table APIs
apps/api/src/modules/quotations/  routes, schemas, service, Excel builder
```

Implemented frontend:

```txt
/quotations                       Quotation list
/quotations/new                   Create quotation
/quotations/:id                   Quotation detail and Excel export
/quotations/:id/edit              Edit quotation
apps/web/features/quotations/     API wrapper, list, form, detail, item selector, totals panel
apps/web/features/quotations/quotation-canvas-editor.tsx  Canvas template layout editor
apps/web/features/quotations/quotation-layout-preview.tsx  Layout-config web preview renderer
apps/web/types/quotations.ts      Quotation frontend types
```

Implemented Prisma models:

```txt
Quotation
QuotationItem
QuotationImage
QuotationTemplateVersion
ExportLog is reused for quotation export history
```

## Shared Infrastructure

```txt
Validation:       Zod schemas in each backend module
Database:         PostgreSQL via Prisma
Money/quantity:   Prisma Decimal fields
Auth:             JWT bearer tokens and RBAC middleware
Sensitive logs:   AuditLog and DocumentAccessLog
Email queue:      BullMQ + Redis, processed by Nodemailer worker
Frontend API:     apps/web/lib/api-client.ts, NEXT_PUBLIC_API_URL fallback http://localhost:4000
Shared package:   packages/shared exports common permissions/types
```

## Current Review Notes

- `MAP.md` now reflects the implemented code paths instead of only the initial target architecture.
- There are duplicate local files with names like `...(1).ts`, `...(1).tsx`, and `...(1).prisma` in the working tree. Treat canonical filenames without `(1)` as the active source unless intentionally reviewing those backups.
- The working tree contains many existing modified/untracked files. This map update only changes `MAP.md`.
