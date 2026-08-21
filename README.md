# BizOS — Case Study

A multi-tenant business operating system covering the full lifecycle: lead to invoice, hire to payslip, purchase to warehouse.

**The source code for BizOS is private.** This page exists so the work can be reviewed without access to the repository — what was built, what it runs on, and which decisions are worth discussing.

## The problem

Small and mid-sized businesses in East Africa either stitch together several tools or run Odoo/QuickBooks, neither of which handles local compliance natively. BizOS was built as a direct competitor to both, with Kenyan statutory rules treated as a first-class requirement rather than a plugin: M-Pesa payments, KRA PAYE bands, NSSF, SHIF and Housing Levy, KRA PIN fields, KES by default.

## What it does

18 modules in one platform:

- **Revenue** — CRM with a drag-and-drop pipeline, product catalog, quotes with one-click conversion to invoices, invoicing with partial payments and credit notes.
- **Payments** — recorded via Stripe, Paystack, M-Pesa, bank transfer or manual, with gateway reference tracking per invoice.
- **Supply** — multi-warehouse inventory with reorder alerts and full movement history, purchase orders with a receive-goods flow that writes stock movements automatically, light manufacturing (BOM + work orders).
- **People** — employee records with statutory fields, departments, contracts, leave with balance accounting, attendance with overtime, and Kenyan payroll (PAYE / NSSF / SHIF / Housing Levy) through a Draft → Approved → Disbursed workflow.
- **Operations** — projects with a kanban task board, milestones and billable time entries; a helpdesk with threaded conversations and internal notes; expense submission and approval.
- **Finance & admin** — fixed assets with straight-line and declining-balance depreciation, P&L, aged receivables, cash flow and tax summary reports (all CSV-exportable), dashboard KPIs, org settings, team roles and invitations, subscription and billing.

Cross-cutting: print-to-PDF invoices, quotes and batch payslips; CSV exports for statutory and bank disbursement files; dark mode.

## Stack

- **Language / runtime:** TypeScript 5.7 on Bun 1.3
- **Framework:** Next.js 16 (App Router)
- **API:** tRPC v11, TanStack Query for server state
- **Data:** Prisma 7 against Neon PostgreSQL (`@prisma/adapter-pg`)
- **Auth:** Better Auth
- **UI:** HeroUI, TailwindCSS v4 (CSS-first), Framer Motion, Lucide, react-hot-toast
- **Forms / validation:** React Hook Form + Zod
- **Services:** UploadThing (files), Resend (email), Inngest (background jobs), PostHog (analytics), Anthropic SDK (AI), Stripe / Paystack / M-Pesa
- **Tooling:** Biome

## Engineering decisions worth defending

**1. Tenancy enforced at the API boundary, not in the UI.**
Every record carries an `organizationId`, and access runs through a shared `orgProcedure` in tRPC that resolves the caller's org and role (`owner` / `admin` / `member`) before any handler executes. Isolation and role checks are one thing to audit rather than a rule repeated across eighteen modules and hundreds of screens.

```ts
// every module builds on this, so scoping is not a per-query decision
export const orgProcedure = protectedProcedure.use(({ ctx, next }) =>
  next({ ctx: { ...ctx, organizationId: ctx.session.activeOrganizationId } })
);
```

**2. Notifications are computed, not stored.**
The five alert types — overdue invoices, low stock, unresponded tickets, pending leave, pending payroll — are derived live from existing tables on a 60-second refresh, with no event queue, no notification table and no write path to keep in sync. It trades some query cost for the elimination of an entire class of stale-state bugs; the boundary where that trade stops paying is a reasonable interview conversation.

**3. Financial batch jobs are idempotent by construction.**
The monthly depreciation run skips periods it has already processed and caps each charge at the remaining depreciable value, so an asset can never fall below its salvage value regardless of how many times the job is triggered. Payroll and leave follow the same instinct: leave balances deduct on approval and restore on rejection or cancellation, so a retried or reversed action leaves the ledger intact.

## Live link

None. The application is login-gated and multi-tenant, so there is no public demo to link to. Happy to walk through it live in an interview.