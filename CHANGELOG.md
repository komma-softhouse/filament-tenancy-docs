# Changelog

All notable changes to `komma-softhouse/filament-tenancy` are documented here.

## 1.1.0 — 2026-09-10

- Database pool: `->databasePool([...])` spreads tenants across several servers. Placement happens once, at birth, and is written on the tenant, so provisioning, migrations, backups and deletion follow it; nothing already placed is ever moved, and a tenant pinned by hand is never rebalanced. Strategies: least-tenants, round-robin, weighted.
- `tenancy:pool` reports the distribution, the next placement and whether each server answers; `tenancy:doctor` fails on a pool name no connection backs; health reports a tenant placed on a server that no longer exists.
- Backups now dump from the server the tenant actually lives on, rather than from the template connection.
- Pool widget on the Tenants list: how full each server is and which one takes the next tenant. Shown only when a pool is configured.
- **Move to another server** action: dump, restore on the target, drop the original, with the tenant suspended in between and the dump kept either way. Changing the connection alone would only change where the tenant looks, not where its data is.
- The health widget reports tenants still being built or failed, and the list gains a toggleable count of pending invitations.

- Path identification: `->identification('path')` puts every tenant under the central host as `/{slug}/…` instead of on its own subdomain. Livewire requests, which carry no tenant in the URL, are resolved from the session with the referer as a fallback; custom domains and `PreventAccessFromCentralDomains` are skipped, since both contradict the mode.
- `Tenancy::tenantUrl()`: everything that hands somebody over to a tenant now asks how tenants are reached instead of assuming a subdomain.
- Octane support: the tenant, the session cookie and the resolved connections are put back between requests. Registered only when Octane is installed; `tenancy:doctor` fails if it is installed and the reset is off.
- `tenancy:doctor` reports the identification mode, and under path identification checks that central and tenant session cookies do not share a name.

- Resource syncing for users: a change to a person's name or email travels between central and every workspace they belong to, in either direction. The password never travels — it lives in central — and neither do roles. `->resourceSyncing()` turns it off or changes what is carried.
- `IsCentralIdentity` now implements everything stancl needs of a `SyncMaster`, so a host only adds the interface to its User model. Without it, the tenant-to-central direction still works rather than throwing.

- The queue's `database` driver is pinned to the central connection: a job pushed from inside a tenant was landing in that tenant's database, where there is no `jobs` table and no worker looking.
- The cache bootstrapper is only applied when the store can tag. On `file` or `database` it threw the moment tenancy initialized — in every queued job — so the plugin now leaves the cache shared and `tenancy:doctor` says so.

- Workspace picker in the topbar of every tenant panel: an identity that belongs to several tenants crosses between them through the same signed single-use handoff, without signing out. Unfinished tenants are left out.
- **New workspace** page, reached from that picker: creating another tenant owned by the identity already signed in — no second account, no second password. Restrictable with `onboarding.roles`.
- A taken subdomain is reported as soon as the field is left, with four free alternatives offered as a choice: the name without separators, with a word, or with the first free number. `SubdomainSuggestions` checks each one before showing it.
- `TenantRegistrationService::createFor()` splits the birth of a tenant from the birth of an identity; registration is now written in terms of it.
- Creating a tenant while async provisioning is on says so: a notification in the panel, a status instead of a URL in `tenancy:create`.
- `tenancy:doctor` reports tenants that have been waiting more than five minutes on a queue nobody is working, and the ones whose provisioning failed.
- New fluent methods `switcher()` and `onboarding()` on `TenantPanelPlugin`.

- Async provisioning: `->asyncProvisioning()` builds the tenant database on a queue instead of inside the signup request. The founder waits on a signed page on the central domain and is handed into the panel the moment it is ready.
- Tenants now carry a provisioning state (`pending`, `provisioning`, `ready`, `failed`) with the failure reason and the attempt count. Both modes run the same steps, so a queued tenant ends identical to one built in the request.
- `EnsureTenantIsReady` keeps half-built tenants unreachable, answering with a self-refreshing page instead of a database error.
- Health reports an unfinished tenant as *provisioning* rather than broken; the list gains a provisioning column and a *Finish setup* / *Retry setup* action.
- `tenancy:provision {tenant?} --failed --sync` re-runs provisioning for tenants a queue never finished.
- New `TenantReady` event.

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
- Translations: es, gl, pt.