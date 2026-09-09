# Changelog

All notable changes to `komma-softhouse/filament-tenancy` are documented here.

## 1.0.0 — 2026-09-09

First release. Database-per-tenant multi-tenancy on stancl/tenancy 3 for Filament 5 and Livewire 4, extracted from the Komma starter kit and closed behind a fluent API: the host registers two plugins and keeps credentials in `.env`.

### The two plugins

- `TenancyPlugin` on the central panel: the Tenants resource and the whole configuration of the tenancy — domains, connections, models, roles, doors, handoff, impersonation, operations.
- `TenantPanelPlugin` on every tenant panel: the tenant guard, the middleware stack that keeps the session on the subdomain, the central-identity login, the profile page, the Members page and the banners. The host declares no middleware.
- Everything is fluent, and every fluent method has a key in `config/filament-tenancy.php`; fluent wins. Closures for redirects and hooks exist only fluently.

### Provisioned at runtime

- The `central` and `tenant_template` connections, the tenant guard, its provider and password broker, the whole `tenancy.*` array, the `universal` middleware group and the Livewire update route — the last one being what makes tenant panel logins work at all.

### Identity, doors and membership

- One credential per person in central; tenant-local users provisioned just-in-time; membership on a central pivot.
- Central door (`/login`) that finds the workspace and hands the session over with a signed, single-use ticket; a picker when somebody belongs to several.
- Public self-registration (`/register`) with honeypot, rate limit, reserved subdomains and optional captcha; embeddable as a Livewire section.
- Staff impersonation as the tenant owner, through the same ticket, recorded in `impersonations`.
- Invitations: a **Members** page on every tenant panel and a relation manager on the central one. Hashed single-use tokens, acceptance on the central domain where a password is always required — an existing identity confirms, a new one is created — and the membership, the local user and the roles written in one transaction. The last owner can be neither removed nor demoted.

### Operations

- Health checks per tenant (database, pending migrations, domain, owner, tenant users, roles, storage), cached, with a column, an infolist section, a filter and a widget.
- One repair per failing check: create the missing database and migrate it, run pending migrations, re-seed roles, assign an owner, provision missing tenant users.
- Leftovers: databases *and* storage directories named like a tenant that no row claims, listed on request and dropped one by one.
- Suspension with a custom message and an optional expiry, enforced by `EnsureTenantIsActive`; support keeps access while impersonating.
- Notices: a banner published into a tenant's panel, optionally emailed to its members.
- Destruction with typed confirmation, and an optional backup first that aborts the deletion if it fails.
- Stats and health widgets; toggleable columns for id, health, database, storage, members, owner and kind; filters by status, kind and health.

### Storage, backups and restore

- Tenant files are deleted along with the tenant (`DeleteTenantStorage` in the `TenantDeleted` pipeline) and reported as a size in the list and the infolist.
- `tenancy:backup` writes the database dump, the files and a `manifest.json` with the tenant row, its domains and its members. The dump goes through `pg_dump` / `mysqldump`, or a file copy after a WAL checkpoint on SQLite; a missing tool is reported before anything is written.
- `tenancy:restore` rebuilds the tenant from the manifest, loads the dump, copies the files back and re-attaches the memberships whose identity still exists. It refuses to write over a live tenant.

### Custom domains

- A tenant can be reached at a domain of its own, on top of the subdomain it is born with, which is never removed.
- Ownership is proved by DNS per domain — a TXT record (ownership only) or a CNAME (ownership and routing). Until then `EnsureDomainIsVerified` serves the pending record instead of the panel.
- `Domain` model with `type`, `is_primary`, `verified_at` and a verification token; the primary domain is what every generated URL uses. `tenancy:domains:verify` re-checks the pending ones hourly.
- The tenant session cookie also works on a custom domain, scoped to that host instead of the base wildcard.

### Tooling

- `tenancy:create`, `tenancy:destroy`, `tenancy:backup`, `tenancy:restore`, `tenancy:domains:verify`, `filament-tenancy:install`.
- `tenancy:doctor`: connections and the CREATE DATABASE privilege, domains and wildcard DNS, the route and config caches, the universal group and the Livewire update route, every panel and its guard, the central schema, spatie, the Redis client and queue prefix, `asset_helper_tenancy`, and the backup tooling. Exits non-zero on failure.
- `Komma\Tenancy\Testing\InteractsWithTenancy` for hosts: `createTenant()`, `actingAsTenantUser()`, `actingAsCentralUser()`, `runInTenant()`, `tenantUrl()`.

### Presentation

- Every in-panel view is built from Filament components, so the panel's own theme decides how the plugin looks.
- A "How does it work?" modal in the header of the Tenants list and the Members page.
- The central login, the registration page and the suspension page can be re-skinned with `authLayout()`, `centralLoginView()` and `centralLoginController()`, without publishing a view.

### Under it

- `TenantRegistrationService`: one birth path for the command, the panel and public registration — atomic on central, with compensation for the physical database.
- Lifecycle observer: a deleted tenant sweeps its memberships and the identities left without another workspace.
- Central migrations for `tenants`, `domains`, `tenant_user`, `tenant_invitations`, `handoff_tickets` and `impersonations`, plus the identity columns on `users`; tenant migrations for its users, sessions and the spatie permission tables, with the permission cache scoped per tenant.
- 34 tests (100 assertions) on SQLite, and CI on PHP 8.3/8.4 against Laravel 12 and 13 with Pint and PHPStan.
- Translations: en, es, gl, pt.