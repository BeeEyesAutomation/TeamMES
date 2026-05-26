# Quotation Module Implementation Plan for Codex

## Goal

Add a new **Quotation** module to Teamplatform.

The module must allow users to create quotations for projects/customers, select materials from the existing inventory/material system, calculate totals by set quantity and VAT, attach images/signatures when supported by the current upload foundation, and export the quotation to Excel.

This plan is written as a phased Codex implementation guide. Each phase can be pasted into Codex one by one.

---

## Project Context

Teamplatform is a monorepo with:

- Frontend: Next.js, React, TypeScript, Tailwind CSS
- Backend: Node.js, Express, TypeScript
- Database: PostgreSQL with Prisma
- Auth: JWT with RBAC
- Export: ExcelJS, CSV, and PDF foundation
- Queue: BullMQ with Redis
- Shared package: permissions and common types

Expected folders:

```txt
apps/
  api/
    prisma/
    src/
      routes.ts
      modules/
      middleware/
      utils/
  web/
    app/
    components/
    features/
    lib/
    types/
packages/
  shared/
```

Important engineering rules:

- Use TypeScript for both frontend and backend.
- Use Prisma `Decimal` for money, quantity, VAT rate, and totals.
- Do not use floating point for financial calculations.
- Use validation for all API inputs.
- Follow existing module, route, controller, service, UI, and permission patterns.
- Avoid unrelated refactors.
- Avoid reading or rewriting unrelated files.
- Prefer soft delete for important business records.
- Keep implementation names, API paths, code, and database names in English.
- End-user UI labels can be Vietnamese.
- Use VND currency formatting.
- Date format in Vietnamese UI should be `dd/MM/yyyy`.

## Mandatory Documentation Update Rules

Codex must keep project documentation synchronized during implementation.

After each major phase, update the project map and implementation history:

1. Update `MAP.md`
   - Add new backend routes.
   - Add new backend module paths.
   - Add new frontend routes.
   - Add new feature folder paths.
   - Add new database models only if `MAP.md` currently tracks models.
   - Keep the update short and factual.

2. Update implementation history
   - Use the existing history/changelog/development log file if present.
   - If no history file exists, create or update a concise file under `docs/`, for example:
     `docs/implementation-history.md`
   - Add one short entry per major phase.
   - Each entry should include:
     - Date
     - Phase name
     - Files changed
     - Short summary
     - Commands run
     - Known issues/TODOs

3. Update `AGENTS.md` only when there is a major codebase-level change
   - Examples that require updating `AGENTS.md`:
     - New architecture convention
     - New module pattern
     - New required workflow for future agents
     - New global permission rule
     - New export/audit/security convention
   - Do not update `AGENTS.md` for normal feature files, small UI changes, or routine CRUD.
   - If updated, keep the change minimal and add only reusable guidance for future agents.

4. Never rewrite the full documentation files unless necessary
   - Patch only the relevant sections.
   - Do not reformat unrelated content.
   - Do not duplicate existing information.

## Token Optimization Rules

Codex must minimize token usage as much as possible.

- Start each phase by reading only the minimum required files.
- Prefer targeted search over opening large files.
- Do not read the full repository.
- Do not paste large file contents into the response.
- Do not summarize unrelated code.
- Do not rewrite full files when a small patch is enough.
- Do not refactor unrelated modules.
- Do not rename variables or reformat files outside the quotation scope.
- Reuse existing patterns instead of inventing new abstractions.
- If an existing helper/service/component can be reused, reuse it.
- When reporting progress, keep summaries short:
  - Files changed
  - Why changed
  - Commands run
  - Result
- If blocked by missing context, inspect the closest related file first instead of asking immediately.
- If a decision is unclear, make the safest assumption based on existing patterns and document it.

---

# Phase 1 — Inspect Current Codebase

## Codex Input

```text
You are working in the Teamplatform repository.

Goal of this phase:
Inspect only the files needed to implement the Quotation module. Do not modify code yet.

Read and summarize the existing patterns for:
1. Prisma models and naming conventions.
2. Backend module structure.
3. API route registration.
4. RBAC/permission middleware.
5. Audit log usage.
6. Project and material-related models/APIs.
7. Export Excel implementation.
8. Frontend route/page structure.
9. Frontend API client pattern.
10. Sidebar/navigation pattern.

Prioritize these files:
- apps/api/prisma/schema.prisma
- apps/api/src/routes.ts
- apps/api/src/modules/projects/**
- apps/api/src/modules/project-materials/**
- apps/api/src/modules/exports/**
- apps/api/src/modules/audit/**
- apps/api/src/middleware/**
- apps/web/components/layout/app-shell.tsx
- apps/web/features/projects/**
- apps/web/lib/**
- packages/shared/src/**

Do not scan the entire repository unless a referenced import requires it.

Output:
- Existing patterns found.
- Exact files that should be created/changed in later phases.
- Any assumptions about material price fields, upload handling, RBAC, or audit logs.
```

## Expected Output

Codex should return a short technical summary before coding.

---

# Phase 2 — Add Database Models

## Codex Input

```text
Implement database support for the new Quotation module.

Requirements:
Add Prisma models for quotations and quotation items.

Model: Quotation
Fields:
- id
- quotationCode: unique business code
- projectId: optional relation to Project if Project model exists
- projectName: optional snapshot text
- customerName: optional text
- customerRequest: optional long text
- quotationDate: DateTime
- numberOfSets: Decimal, default 1
- hasVat: Boolean, default false
- vatRate: Decimal, default 10
- totalOneSet: Decimal
- totalBeforeVat: Decimal
- vatAmount: Decimal
- grandTotal: Decimal
- imageUrl: optional string
- signatureUrl: optional string
- status: quotation status
- createdById: optional relation to User if current schema supports it
- updatedById: optional relation to User if current schema supports it
- deletedAt: optional DateTime
- createdAt
- updatedAt

Model: QuotationItem
Fields:
- id
- quotationId: relation to Quotation
- lineIndex: Int
- materialId: optional relation to existing material/inventory model if available
- materialCode: String
- materialName: String
- quantity: Decimal
- unit: String
- unitPrice: Decimal
- amount: Decimal
- createdAt
- updatedAt

Status values:
- draft
- sent
- approved
- rejected
- cancelled

Indexes:
- quotationCode unique
- projectId
- quotationDate
- status
- deletedAt
- quotationId on quotation items

Rules:
- Use Decimal for all money, quantity, VAT rate, and totals.
- Use DateTime for business dates, normalized consistently with the existing project.
- Use soft delete with deletedAt.
- Preserve snapshot material fields in quotation items.
- Do not change unrelated schema models except required relations.

After editing:
Run:
npx prisma format --schema apps/api/prisma/schema.prisma
npx prisma validate --schema apps/api/prisma/schema.prisma

Output:
- Prisma models added.
- Any migration command needed.
- Any assumptions made.
```

---

# Phase 3 — Add Backend Validation and Calculation Service

## Codex Input

```text
Create backend validation and calculation logic for quotations.

Create a new backend module following the existing module pattern, likely:
apps/api/src/modules/quotations/

Add validation schemas for:
- Create quotation
- Update quotation
- List quotation filters
- Quotation item input

Input validation rules:
- projectId optional
- projectName optional
- customerName optional
- customerRequest optional
- quotationDate optional, default current date in service
- items required, minimum 1 item
- each item quantity > 0
- each item unitPrice >= 0
- numberOfSets >= 1
- vatRate >= 0
- hasVat boolean
- status must be a valid quotation status

Create calculation service:
calculateQuotationTotals(items, numberOfSets, hasVat, vatRate)

Calculation rules:
- item.amount = quantity * unitPrice
- totalOneSet = sum(item.amount)
- totalBeforeVat = totalOneSet * numberOfSets
- if hasVat:
    vatAmount = totalBeforeVat * vatRate / 100
    grandTotal = totalBeforeVat + vatAmount
  else:
    vatAmount = 0
    grandTotal = totalBeforeVat

Important:
- Use Decimal-safe calculations.
- Do not use JavaScript floating point for financial values.
- Backend must always recalculate totals before saving, even if frontend sends totals.
- Return normalized calculated item amounts and totals.

Output:
- Files created.
- Validation rules implemented.
- Calculation helper/service implemented.
```

---

# Phase 4 — Add Quotation Code Generator

## Codex Input

```text
Implement quotation code generation.

Format:
Q-YYYYMMDD-XXX

Example:
Q-20260515-001

Rules:
- Prefix is Q.
- Date segment comes from quotationDate or current date.
- Sequence resets each day.
- Sequence should be based on existing quotations for the same date.
- Code must be unique.
- User should not manually enter quotationCode.
- Handle possible conflict safely using existing transaction/error-handling patterns if available.

Add this logic in the quotation service layer.

Output:
- Function name and file path.
- Explanation of how uniqueness is enforced.
```

---

# Phase 5 — Add Backend CRUD APIs

## Codex Input

```text
Implement backend CRUD APIs for quotations.

Routes:
GET    /api/quotations
POST   /api/quotations
GET    /api/quotations/:id
PUT    /api/quotations/:id
DELETE /api/quotations/:id

List filters:
- search
- projectId
- customerName
- status
- dateFrom
- dateTo
- page
- limit

Search should match:
- quotationCode
- projectName
- customerName

Create behavior:
- Generate quotationCode automatically.
- Resolve material snapshot fields if materialId is provided.
- Store materialCode, materialName, unit, unitPrice snapshot in quotation items.
- Recalculate all item amounts and totals in backend.
- Save quotation and items transactionally.
- Create audit log if project has audit log pattern.

Update behavior:
- Allow editing header fields, status, numberOfSets, VAT options, and item list.
- Recalculate totals.
- Replace quotation items safely or update them according to existing project pattern.
- Preserve quotationCode.
- Create audit log if project has audit log pattern.

Delete behavior:
- Prefer soft delete with deletedAt.
- Create audit log if project has audit log pattern.

Detail behavior:
- Return quotation with items.
- Include project/customer display fields where useful.

Security:
- Apply auth middleware.
- Apply RBAC middleware if existing.
- Add or use permissions:
  - quotations.view
  - quotations.create
  - quotations.update
  - quotations.delete
  - quotations.export

Output:
- Routes implemented.
- Controller/service files created.
- routes.ts updated.
- Permissions/audit integration summary.
```

---

# Phase 6 — Add Material Search/Selection Support

## Codex Input

```text
Add backend support needed for selecting materials inside a quotation form.

Goal:
The frontend quotation form must allow searching/selecting material by:
- material name
- material code

When a material is selected, return:
- materialId
- materialCode
- materialName
- unit
- unitPrice

Unit price source:
- Prefer existing inventory/material selling price field.
- If selling price does not exist, use the closest existing field and document the assumption.
- Do not use purchase price if selling price exists.
- Do not subtract stock when creating a quotation.

Implementation options:
1. Reuse existing material list/search API if it already returns the needed fields.
2. Add a quotation-specific endpoint if needed:
   GET /api/quotations/materials/search?search=

Do not create duplicate material logic if an existing reusable API is available.

Output:
- Which existing API was reused, or which new endpoint was added.
- Which field is used as unitPrice.
- Any assumption about missing inventory/selling price field.
```

---

# Phase 7 — Add Excel Export

## Codex Input

```text
Implement Excel export for quotation.

Endpoint:
GET /api/quotations/:id/export-excel

Use ExcelJS and follow existing export patterns.

Filename:
quotation_Q-YYYYMMDD-XXX.xlsx

Excel content:

Title:
BÁO GIÁ

Header information:
- Quotation code
- Date
- Project
- Customer
- Customer request

Material table columns:
- Index
- Material name
- Material code
- Quantity
- Unit
- Unit price VND
- Amount VND

Totals:
- Total 1 set
- Number of sets
- Total before VAT
- VAT %
- VAT amount
- Grand total

Image/signature:
- If current Excel export supports image insert, insert image/signature.
- If not, include imageUrl/signatureUrl text rows.

Formatting:
- Bold title
- Bold table header
- Borders around material table
- VND currency format
- Auto column width if current helper supports it
- Date formatted dd/MM/yyyy if possible

Security:
- Apply view/export permission.
- Create audit log for export/download if current project logs downloads.

Output:
- Export endpoint implemented.
- File naming implemented.
- Excel formatting summary.
```

---

# Phase 8 — Add Shared Permissions and Seed Data

## Codex Input

```text
Add quotation permissions to the shared permission system and seed data if the project uses seeded permissions.

Permissions:
- quotations.view
- quotations.create
- quotations.update
- quotations.delete
- quotations.export

Suggested role assignment:
- Admin: all quotation permissions
- Director: view/export
- Project Manager: view/create/update/export
- Accountant: view/export/update if existing role pattern allows
- Employee: no default quotation permissions unless existing project pattern requires view access

Rules:
- Follow existing permission naming convention exactly.
- Do not break existing seed data.
- Do not duplicate permissions if they already exist.

Output:
- Files updated.
- Roles assigned.
- Seed update summary.
```

---

# Phase 9 — Add Frontend Types and API Client

## Codex Input

```text
Add frontend support for the Quotation module.

Create feature folder following existing pattern:
apps/web/features/quotations/

Add:
- types.ts
- api.ts
- utility functions for VND/date formatting if not already shared

Types:
- Quotation
- QuotationItem
- QuotationStatus
- CreateQuotationInput
- UpdateQuotationInput
- QuotationListFilters
- MaterialOption

API functions:
- listQuotations(filters)
- getQuotation(id)
- createQuotation(input)
- updateQuotation(id, input)
- deleteQuotation(id)
- exportQuotationExcel(id)
- searchQuotationMaterials(search)

Rules:
- Reuse existing api client.
- Reuse existing auth/error handling pattern.
- Keep implementation names in English.
- UI labels will be Vietnamese in page/components.

Output:
- Files created.
- API function names.
```

---

# Phase 10 — Add Quotation List Page

## Codex Input

```text
Create the quotation list page.

Route:
apps/web/app/quotations/page.tsx

UI label:
Báo giá

Columns:
- Mã báo giá
- Dự án
- Khách hàng
- Ngày báo giá
- Tổng tiền
- Trạng thái
- Hành động

Filters:
- Search by quotation code/project/customer
- Status
- Date from
- Date to

Actions:
- Xem
- Sửa
- Xóa
- Xuất Excel

Requirements:
- Use existing table/card/button styling.
- Use pagination if existing list pages use pagination.
- Format VND currency.
- Format date as dd/MM/yyyy.
- Hide or disable actions based on existing permission pattern if available.
- Do not introduce new UI libraries.

Output:
- Page created.
- Components created if needed.
```

---

# Phase 11 — Add Create/Edit Quotation Form

## Codex Input

```text
Create quotation create/edit form.

Routes:
apps/web/app/quotations/new/page.tsx
apps/web/app/quotations/[id]/edit/page.tsx

Form sections:

1. Header information
- Project select if project list API exists
- Customer name
- Customer request textarea
- Quotation date
- Image upload if current upload component exists
- Signature upload if current upload component exists

2. Material items table
Columns:
- Index
- Material search/select
- Material code
- Material name
- Quantity
- Unit
- Unit price VND
- Amount VND
- Remove action

Buttons:
- Add item
- Remove item

Behavior:
- Index auto increases from 1 to n.
- When material is selected, autofill material code, material name, unit, and unit price.
- Quantity must be > 0.
- Unit price must be >= 0.
- Amount is read-only and calculated as quantity * unitPrice.
- Re-index rows after remove.

3. Totals section
- Total 1 set
- Number of sets
- Has VAT
- VAT rate, default 10
- Total before VAT
- VAT amount
- Grand total

Behavior:
- Calculate totals realtime in frontend for preview.
- Backend will recalculate after save.
- Format money as VND.

Validation:
- At least one material item.
- Quantity > 0.
- Number of sets >= 1.
- VAT rate >= 0.
- Project or customer should be provided.

After save:
- Redirect to quotation detail page or list page following existing project pattern.

Output:
- Form component path.
- Create page path.
- Edit page path.
- Any assumptions about upload/image/signature handling.
```

---

# Phase 12 — Add Quotation Detail Page

## Codex Input

```text
Create quotation detail page.

Route:
apps/web/app/quotations/[id]/page.tsx

Display:
- Quotation code
- Quotation date
- Project
- Customer
- Customer request
- Status
- Material items table
- Total 1 set
- Number of sets
- Total before VAT
- VAT %
- VAT amount
- Grand total
- Image if available
- Signature if available

Actions:
- Edit
- Export Excel
- Back to list

Requirements:
- Use existing card/table styling.
- Format VND.
- Format date dd/MM/yyyy.
- Handle loading/error/not found states.
- Respect permissions if existing frontend permission pattern exists.

Output:
- Detail page implemented.
```

---

# Phase 13 — Add Sidebar Navigation

## Codex Input

```text
Add Quotation module to the sidebar/navigation.

Menu item:
- Vietnamese label: Báo giá
- Path: /quotations
- Icon: choose an existing lucide-react icon consistent with app style

Placement:
- Near Projects or Import/Export.
- Do not restructure the sidebar.
- Do not rename existing menu items.

Output:
- File updated.
- Navigation label/path added.
```

---

# Phase 14 — Final Validation

## Codex Input

```text
Run validation for the Quotation module implementation.

Run these commands if available:

npx prisma validate --schema apps/api/prisma/schema.prisma
npm run prisma:generate
npm run typecheck --workspace apps/api
npm run typecheck --workspace apps/web

If tests exist for the API:
npm run test --workspace apps/api

If any command fails:
- Fix errors related to the quotation implementation.
- Do not refactor unrelated code.
- If failure is unrelated, report it clearly.

Manual test checklist:
1. Create a quotation with one material.
2. Create a quotation with multiple materials.
3. Select material and verify code/name/unit/unit price autofill.
4. Change quantity and verify amount updates.
5. Change number of sets and verify totals update.
6. Enable VAT 10% and verify VAT amount/grand total.
7. Disable VAT and verify VAT amount is 0.
8. Edit quotation and verify items are preserved.
9. Delete quotation and verify it disappears from normal list.
10. Export Excel and verify all data is correct.
11. Confirm creating quotation does not subtract inventory stock.
12. Confirm login, projects, materials, and existing exports still work.

Final output:
- Summary of completed work.
- Files changed.
- Commands run and results.
- Any assumptions.
- Any known TODOs.
```

---


# Phase 15 — Documentation Sync After Major Implementation

## Codex Input

```text
Synchronize documentation after the major Quotation module implementation.

Required documentation updates:

1. Update MAP.md
Add concise entries for:
- New backend quotation module path.
- New backend quotation API routes.
- New frontend quotation routes.
- New frontend quotation feature folder.
- New export endpoint.
- New permissions if applicable.

Do not rewrite MAP.md. Patch only relevant sections.

2. Update implementation history
Find the existing implementation history, changelog, or development log file.
If none exists, create:
docs/implementation-history.md

Add a concise entry:

Date: current date
Phase: Quotation module implementation
Summary:
- Added quotation database models.
- Added quotation backend APIs.
- Added material selection support.
- Added quotation Excel export.
- Added frontend list/detail/create/edit pages.
- Added sidebar navigation.
- Added permissions/audit integration if implemented.

Files changed:
- List only key files or folders.

Commands run:
- List validation/typecheck/test commands and results.

Known TODOs:
- Only real remaining TODOs.

3. Update AGENTS.md only if needed
Only update AGENTS.md if the quotation work introduced a reusable, codebase-level rule for future agents.
Examples:
- New standard module checklist.
- New export Excel convention.
- New audit/permission convention.
- New documentation sync rule.

If no AGENTS.md update is needed, say:
"No AGENTS.md update was needed because no global agent rule changed."

Rules:
- Keep all documentation updates short.
- Do not duplicate existing content.
- Do not reformat unrelated sections.
- Do not paste large documentation content in the final response.
```

---

# Phase 16 — Final Codex Report Template

## Codex Input

```text
Prepare the final implementation report.

Use this format:

## Summary
Explain what was implemented.

## Changed Files
List changed files grouped by:
- Database
- Backend
- Shared permissions
- Frontend
- Tests

## Features Implemented
- Quotation CRUD
- Quotation code generation
- Material selection
- VAT calculation
- Excel export
- RBAC/audit integration
- Sidebar navigation

## Commands Run
List commands and pass/fail result.

## Assumptions
List assumptions, especially:
- Which material price field is used as unit price.
- Whether image/signature upload is fully supported.
- Whether image/signature insertion into Excel is supported.

## Manual Test Steps
Provide step-by-step testing instructions.

## Known TODOs
Only list real remaining limitations.
```

---

# One-Shot Full Prompt for Codex

Use this if you want Codex to implement the entire module in one run.

```text
You are working in the Teamplatform repository.

Implement a new Quotation module.

Goal:
Add a “Báo giá / Quotations” module that supports creating quotations for projects/customers, selecting materials from the existing inventory/material system, calculating totals by set quantity and VAT, storing snapshot item prices, and exporting quotations to Excel.

Project rules:
- Frontend: Next.js + React + TypeScript + Tailwind.
- Backend: Express + TypeScript.
- Database: PostgreSQL + Prisma.
- Use Prisma Decimal for money, quantity, VAT rate, and totals.
- Do not use JavaScript floating point for financial calculations.
- Use validation for API inputs.
- Follow existing route/module/UI/RBAC/audit/export patterns.
- Avoid unrelated refactors.
- Do not scan or rewrite unrelated files.
- Implementation names, paths, API, and database names must be English.
- End-user UI labels should be Vietnamese.
- Format VND currency and use dd/MM/yyyy in Vietnamese UI.

Required backend:
- Prisma models: Quotation and QuotationItem.
- Quotation status: draft, sent, approved, rejected, cancelled.
- Routes:
  GET    /api/quotations
  POST   /api/quotations
  GET    /api/quotations/:id
  PUT    /api/quotations/:id
  DELETE /api/quotations/:id
  GET    /api/quotations/:id/export-excel
- Search/list filters:
  search, projectId, customerName, status, dateFrom, dateTo, page, limit.
- Quotation code format:
  Q-YYYYMMDD-XXX
  Example: Q-20260515-001
- Generate quotationCode automatically.
- Store item snapshots:
  materialCode, materialName, unit, unitPrice.
- Do not subtract inventory stock when creating quotation.
- Recalculate all totals in backend before save.
- Prefer soft delete.
- Apply RBAC and audit log if existing project patterns support them.
- Add permissions if applicable:
  quotations.view
  quotations.create
  quotations.update
  quotations.delete
  quotations.export

Calculation:
- item.amount = quantity * unitPrice
- totalOneSet = sum(item.amount)
- totalBeforeVat = totalOneSet * numberOfSets
- if hasVat:
    vatAmount = totalBeforeVat * vatRate / 100
    grandTotal = totalBeforeVat + vatAmount
  else:
    vatAmount = 0
    grandTotal = totalBeforeVat

Required frontend:
- Routes:
  /quotations
  /quotations/new
  /quotations/[id]
  /quotations/[id]/edit
- Sidebar item:
  Báo giá -> /quotations
- List page columns:
  Mã báo giá, Dự án, Khách hàng, Ngày báo giá, Tổng tiền, Trạng thái, Hành động.
- Form:
  Project, customer, customer request, quotation date, image, signature, material items, number of sets, VAT option, totals.
- Material item table:
  index, material select, material code, material name, quantity, unit, unit price VND, amount VND.
- Selecting a material autofills code/name/unit/unit price.
- Frontend calculates realtime preview totals.
- Backend remains source of truth.

Excel export:
- Use ExcelJS.
- Filename: quotation_Q-YYYYMMDD-XXX.xlsx.
- Include:
  Quotation code, date, project, customer, customer request.
  Material table.
  Total 1 set, number of sets, total before VAT, VAT %, VAT amount, grand total.
  Image/signature if supported, otherwise include URL/path.
- Format title, headers, borders, VND currency, and column widths.

Implementation order:
1. Inspect only relevant existing files.
2. Add Prisma models.
3. Add validation and calculation service.
4. Add quotation code generator.
5. Add backend CRUD APIs.
6. Add material search/selection support if needed.
7. Add Excel export endpoint.
8. Add shared permissions/seed if project uses them.
9. Add frontend types/API.
10. Add list/detail/create/edit pages.
11. Add sidebar item.
12. Run validation/typecheck.
13. Update MAP.md.
14. Update implementation history.
15. Update AGENTS.md only if a major reusable codebase rule changed.

Commands to run:
npx prisma validate --schema apps/api/prisma/schema.prisma
npm run prisma:generate
npm run typecheck --workspace apps/api
npm run typecheck --workspace apps/web
npm run test --workspace apps/api

Final response:
- Summary
- Changed files
- Commands run with results
- Assumptions
- Manual test steps
- Known TODOs
```
