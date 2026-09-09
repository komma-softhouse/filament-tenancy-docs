# Architecture

## The shape

```
                central domain (demo.test)                    tenant hosts (*.demo.test)
 ┌──────────────────────────────────────────────┐   ┌──────────────────────────────────────┐
 │ /admin   Filament central panel (staff)      │   │ /app   Filament tenant panel          │
 │          TenancyPlugin → Tenants resource    │   │        TenantPanelPlugin → stack,     │
 │ /login   EntryController (the door)          │   │        Login, EditProfile, banners    │
 │ /register RegisterTenant (Livewire)          │   │ /whoami                               │
 │ /impersonation/leave                         │   │                                       │
 └──────────────┬───────────────────────────────┘   └──────────────┬───────────────────────┘
                │ central connection                                │ tenant connection
   ┌────────────▼─────────────┐                        ┌────────────▼───────────────┐
   │ tenants · domains         │   handoff_tickets      │ users (global_user_id)      │
   │ users (identities)        │◄──────────────────────►│ sessions · password tokens  │
   │ tenant_user (membership)  │   signed URL + ticket  │ roles · permissions         │
   │ impersonations            │                        │ …the tenant's own tables    │
   └──────────────────────────┘                        └────────────────────────────┘
```

Two contexts, two connections, one identity.

- **Central** owns the registry: tenants, domains, identities, memberships, handoff tickets, impersonation records. The default connection is switched to central at boot so every host model that is not tenant-specific lands there without a `$connection` property.
- **Tenant** is one database per tenant, created from the template connection by stancl. Anything the tenant panel touches lives here, including a local `users` table.

## Boot sequence

1. **Package `register()`** — `Tenancy` singleton is seeded from `config/filament-tenancy.php` and `apply()` writes: `database.connections.{central,tenant_template}`, `database.default`, `auth.guards.tenant`, `auth.providers.tenant_users`, `auth.passwords.tenant_users`, the central broker connection, and the full `tenancy.*` array. This happens before any provider boots, so stancl and Laravel read finished values.
2. **Panel providers `register()`** — Filament registers each panel; plugins receive `register($panel)` immediately. `TenancyPlugin` records the central panel id and registers the Tenants resource. `TenantPanelPlugin` records `kind ⇒ panel` and applies guard, login, profile, middleware, banners. Fluent calls write into the singleton.
3. **Package `boot()`** — `apply()` runs again (fluent wins), central migrations are loaded, the `universal` middleware group is declared, tenancy middleware gets highest priority, the event pipeline and listeners are bound, the tenant model is observed, the prune schedule is added.
4. **`app->booted`** — after every provider: the Livewire update route is redefined with `['web', 'universal']`, and the plugin's `routes/central.php` and `routes/tenant.php` are mapped.

## Identity model

A person is **one** row in central `users` (the global identity: name, email, password). Each tenant they belong to has a row in central `tenant_user` (membership, coarse role) and a row in that tenant's `users` (local user, `global_user_id`, display fields cached, no credential). The local row is what Filament authenticates on the tenant guard and what spatie roles attach to.

`Login::authenticate()`:

1. rate limit;
2. find the identity in central by email, `Hash::check`;
3. membership in the current tenant (`Membership::isMember`);
4. local user by `global_user_id`, created on the spot if missing;
5. `Filament::auth()->login()` on the tenant guard, session regenerated.

`EditProfile` saves name, email and password on the central identity and mirrors name and email to the local row.

## The doors and the handoff

Tenants live on subdomains nobody remembers. Every entry point that has just proven who the visitor is ends the same way: `Handoff` mints a `handoff_tickets` row (central, 60 s, single use) and a temporary signed URL to `filament.{panel}.auth.login?handoff={ticket}` built on the tenant's own host.

`Login::mount()` claims the ticket: signature valid, ticket unspent and unexpired, same tenant. Then it runs the same membership + provisioning + login path as the password form. `intent` rides along (`impersonation` marks the session).

Doors: `EntryController` (`/login`: credentials → one workspace straight in, several → picker), `Impersonate` (staff → owner's seat, recorded), `RegisterTenant` (founder → panel URL; with `registrationRedirect()` anything else).

Why a ticket and not only a signed URL: a signed URL is replayable for its whole window and lingers in browser history. Why central and not cache: the door runs without tenancy, the panel runs inside one, and the Redis bootstrapper prefixes keys per tenant.

## Session on the subdomain

`SetSubdomainUrlDefault` is the first middleware of a tenant panel. On a `{sub}.base` host it sets `session.cookie` to the tenant cookie name and `session.domain` to `.base`, before `EncryptCookies` and `StartSession` read them. The login POST and the following GET on the subdomain then share one session, and the central panel — which keeps the application cookie name — never shares one with a tenant. It also injects the `{subdomain}` route default for URL generation.

## Livewire

Livewire requests do not pass through the panel middleware; they hit `/livewire/update` with its own stack. The plugin declares a `universal` group holding only `InitializeTenancyByDomain` and, with stancl's `UniversalRoutes` feature enabled, redefines the update route with `['web', 'universal']`: on a tenant host tenancy initializes before the Livewire component runs (so `tenant()` is set during the login submit), on the central host it does nothing.

## Birth of a tenant

`TenantRegistrationService::register()` is the only path — command, panel and public registration call it.

```
guards (reserved, domain, id, email, orphan database)
central transaction {
    identity (central users)          ← password_set_at = now
    tenant row                        → TenantCreated → CreateDatabase → MigrateDatabase → SeedTenantRoles → host jobs
    domain
    membership (owner)
    tenant->run: local user + owner role
    TenantRegistered event, onTenantRegistered()
}
on failure: drop the half-born database (through the stancl manager), rethrow
```

The physical `CREATE DATABASE` cannot join the transaction, hence the explicit compensation. With an explicit id the database name is predictable, so an orphan can be detected up front and dropped when forced.

## Death of a tenant

`TenantLifecycleObserver::deleting` (central transaction): collect member ids, delete memberships, hard-delete the identities that have no membership left anywhere. Then stancl's `TenantDeleted` pipeline drops the database; `domains` cascades. `tenancy:destroy` adds a defensive physical drop.

## Domains

A tenant is born with one domain — `{slug}.base`, marked `subdomain`, primary
and verified by construction, since we issued it under a zone we control — and
may add domains of its own, marked `custom` and unverified.

The distinction is not cosmetic. stancl resolves a tenant from the host of the
request, so a row in `domains` is permission to be served under that name. If
adding a row were enough, anybody could point a hostname at this server, claim
it in the panel, and serve somebody else's workspace under their own name.
`EnsureDomainIsVerified` sits right after tenancy initialization and turns away
any host whose row has no `verified_at`, answering with the DNS record that is
still missing.

Verification is a DNS lookup from the server: a TXT record under
`_tenancy-verify.{domain}` carrying the token (ownership only, so it can be
created before the cutover), or a CNAME pointing here (ownership and routing at
once, unusable on a zone apex). A failed lookup is "not verified", never an
exception.

`is_primary` decides which domain every generated URL uses — the handoff, the
registration redirect, the links in the panel. The subdomain cannot be removed
and takes the primary flag back if the custom domain that held it is deleted:
a tenant with no reachable door is a tenant nobody can fix.

Backups carry the subdomain only. A custom domain still points wherever it
pointed, and restoring its row would claim a hostname the restore has no proof
of.

Routing and TLS stay with the host: the wildcard certificate for `*.base` does
not cover a customer's domain.

## Storage and backups

stancl's filesystem bootstrapper points `storage_path()` at
`storage/{suffix_base}{tenant_id}` while tenancy is initialized, so each tenant
accumulates a directory of its own. `TenantStorage` is the piece that knows
where those directories are from *outside* tenancy, where the bootstrapper is
not in effect: it reports size, deletes on destruction (through
`DeleteTenantStorage` in the `TenantDeleted` pipeline, after `DeleteDatabase`)
and lists the ones no tenant row claims.

`TenantBackup` writes three things, because a dump alone restores the data and
leaves an unreachable tenant: the database dump, the storage directory, and a
`manifest.json` with the tenant row, its domains and its members. Identities
are referenced by email and never copied — a person is central and belongs to
nobody's backup.

The dump is delegated to the vendor's own tool (`pg_dump`, `mysqldump`; a file
copy after a WAL checkpoint on SQLite), because a hand-rolled INSERT generator
is a data-loss bug waiting to happen. When the tool is missing the plugin says
so before writing anything.

Restoring creates the tenant through the normal pipeline — so the database is
created and migrated — and loads the dump on top; that is what lets a backup
restore across schema versions. It refuses to write over a live tenant.

## Roles

Central: spatie roles on the identity decide access to the central panel (`roles.staff`), tenant management (`roles.manage`) and impersonation (`impersonation.roles`).

Tenant: `TenantRolesSeeder` runs inside every new database on the tenant guard; the founder gets the owner role. The permission cache key is scoped per tenant on `TenancyBootstrapped` and reset on `TenancyEnded`.

## Bootstrappers

Database, cache (tagged), filesystem (suffixed disks), queue (jobs carry their tenant), Redis (prefix on `cache` only — never on the queue connection, or jobs dispatched from a tenant panel would land where no worker looks).

## Extension points

- Extend `Komma\Tenancy\Models\Tenant` and point `tenantModel()` at it; extra real columns through `tenantColumns()`.
- `afterTenantCreated([...])` for seeding jobs; `tenantMigrationPaths([...])` for tenant tables.
- `middleware([...])` on `TenantPanelPlugin` for guards that need tenancy (entitlements, billing).
- `afterLoginRedirect()`, `registrationRedirect()`, `onTenantRegistered()`.
- Extend `Login` / `EditProfile` / `RegisterTenant` and pass the subclass.
- `tenantResourceRelationManagers([...])` to hang host relations off the Tenants resource.

## Out of scope in 1.x

Shared-schema tenancy, custom domains per tenant, path-based resolution, tenant-scoped billing. Traffic routing (DNS, wildcard certificates, proxies) belongs to the host.