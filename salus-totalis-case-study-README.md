# Salus Totalis — Multi-tenant Healthcare SaaS

**Case study / architecture documentation.** Salus Totalis is a commercial
product in active development — the source code is private. This repository
documents the architecture, technical decisions, and engineering practices
behind it.

## What it is

A multi-tenant SaaS platform for healthcare clinics: patient records,
scheduling, professional management, financial/billing, and subscription
management. Built solo, from scratch.

## Architecture

Two independent services sharing a single database:

| Service | Role | Stack |
|---|---|---|
| **API** | Stateless REST API — all business logic, persistence, tenant isolation | Laravel 12, Sanctum, MySQL 8.4, Redis |
| **App** | Admin/clinic UI — two Filament v5 panels, reads the same database directly | Laravel 12, Filament v5, Livewire 3, Tailwind v4 |

The API panel (`/admin`) serves cross-tenant platform administration. The
clinic panel (`/painel`) serves each clinic's own staff, scoped strictly to
their tenant.

### Multi-tenancy

Single-database multi-tenancy via [stancl/tenancy](https://tenancyforlaravel.com/)
(API side) and a `TenantScope` applied to every domain model (App side),
resolving `tenant_id` from the authenticated user. `super_admin` bypasses the
scope explicitly (`isSuperAdmin()` check), applied as defense-in-depth on
every cross-tenant resource.

### Domain design

```
Core/       — Tenant, Plan, Module, Billing, Reference (always active)
Modules/    — Users, Patients, Professionals, Scheduling, Appointments,
              Records, Finance (always active per tenant)
Addons/     — Advanced Reports, Full Finance, Inventory, Multi-Unit,
              External Integrations, Digital Signature, Public API
              (opt-in, gated by subscription plan)
```

Modules communicate exclusively through Laravel Events — never direct calls
— to keep boundaries enforced as the domain grows. Addons never modify
Core or Modules tables directly.

### RBAC

Role enforcement via `spatie/laravel-permission` (teams disabled — roles are
global, isolation is handled entirely by `tenant_id`, not by Spatie's team
feature). Five hierarchical roles: `super_admin`, `tenant_admin`, `manager`,
`professional`, `receptionist`.

### Billing

Stripe (Laravel Cashier v16), installed on the App side only — the API reads
`subscriptions` directly from the database without depending on Cashier.
Self-service checkout, customer portal, 14-day trial, and per-plan module
gating via `TenantModule`.

## Architecture Decision Records

Documented in `docs/ARCHITECTURE.md` and `docs/adr/`. Highlights:

- **Single-database multi-tenancy** over separate databases per tenant —
  simpler operations at this stage, isolation enforced at the query layer.
- **No route model binding on tenant-scoped controllers** — the tenant scope
  isn't active yet when Laravel resolves route bindings, so binding by ID
  would silently bypass isolation. Controllers resolve records manually
  inside the scoped context instead.
- **Spatie Permission with `teams: false`** — roles are global, tenant
  isolation is handled entirely by the app's own `tenant_id` scoping, not by
  Spatie's team feature, to keep the two concerns separate.
- **Professional data split** — `Professional` holds a `user_id` FK (1:1) to
  `User`; personal data lives on `User`, professional data (council
  registration, specialties) lives on `Professional`.

## Testing

- **API:** 124 Pest v3 tests, 1 skipped (reference-data sync test) — auth,
  tenant isolation, RBAC, scheduling, appointments, records, finance, plans.
- **App:** 36 Laravel Dusk / Selenium E2E tests — login isolation between
  panels, calendar, patient listing, settings, admin panel CRUD,
  tenant-isolation regression tests, records, billing.

## Compliance & security

- i18n in Portuguese (BR) and English.
- LGPD data portability export.
- Security headers: HSTS, CSP with a nonce, Subresource Integrity (SRI).

## Stack

PHP 8.5 · Laravel 12 · MySQL 8.4 · Redis · Sanctum · stancl/tenancy ·
spatie/laravel-permission · Filament v5 · Livewire 3 · Tailwind v4 ·
Stripe (Cashier v16) · Pest v3 · Laravel Dusk · Docker
