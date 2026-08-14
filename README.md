# Book Reviews Platform

An ASP.NET MVC (.NET 10) application for managing books, authors, and reviews, built on raw ADO.NET (no ORM, no `DataSet`/`DataTable`/`DataView`), with Redis caching and Docker-based local/production setup.

## What it does

- **Books** — create books with publication year and up to 5 linked authors (existing and/or newly created authors in the same workflow); browse a paginated book list; filter/report views (by publication year, by minimum review count with average score).
- **Authors** — create and browse authors; listing shows real book count per author; filter books by exact publication year (pre-fillable via URL query string).
- **Reviews** — create a review from a book's page (`Reviews/Create?bookId=X`); browse a paginated review list showing book + author context and score.
- **Pagination** — every list view supports both a classic numbered pager (offset-based) and, where appropriate, infinite-scroll-style fetching (keyset-based), with soft (AJAX) page changes, smooth fade transitions, a selectable page size (10/25/50/100), and a jump-to-page input.
- **Caching** — Redis-backed, versioned-invalidation caching sits above the DAL, used from the Service layer to avoid repeat round trips for list and lookup queries.

## Architecture

Strict top-down layering; each layer only talks to the one below it.

```
Controller → Service (+ Cache) → Repository (DAL) → ADO.NET → SQL Server
```

- **Controllers** — HTTP concerns only: routing, model binding, calling a single service method. No SQL, no business rules, no direct cache access.
- **Services** — business rules, validation, orchestration across repositories, transaction boundaries via `IUnitOfWork`. The only layer allowed to call `ICacheService` — checks cache before a repository call on reads, invalidates on writes. Constructor-injected, and each service logs key operations and failures (see **Logging** below).
- **Repositories (DAL)** — one per aggregate (`BooksRepository`, `AuthorsRepository`, `ReviewsRepository`), all extending `BaseRepository`. Own all SQL access; never leak ADO.NET types (`SqlConnection`, `SqlDataReader`, etc.) outside the DAL, and have no knowledge of caching. Queries are written to fill DTOs directly — no intermediate entity mapping step.
- **Caching** — `Caching/` is a peer to the DAL, not part of it. `ICacheService`/`RedisCacheService` wrap `StackExchange.Redis`; nothing outside `RedisCacheService` talks to `IConnectionMultiplexer`/`IDatabase` directly.
- **Common** — cross-layer, shape-only types with no persistence or UI logic: pagination contracts, the generic SQL query-building engine.

### Data model boundaries

- **DTOs** — used directly by the DAL as well as between Controller ↔ Service ↔ DAL. No separate Entity layer — repositories map `SqlDataReader` rows straight to DTOs. Plain application data, no persistence logic.
- **ViewModels** — used by Views only, UI-shaped (e.g. carry display-only fields, validation attributes).

### Unit of Work

`IUnitOfWork` holds a single `SqlConnection` for its lifetime and hands each repository the live transaction/connection via delegates, rather than repositories owning either directly. `ExecuteTransactionAsync` wraps a delegate in a transaction (begin → run → commit, or rollback + rethrow on failure) — used whenever a single logical action spans multiple repository calls that must succeed or fail together (e.g. inserting a book and linking its authors).

### SQL query construction

Two small, purpose-built builders live in `Common/QueryBuilding/` (Builder + Director pattern):

- **`QueryBuilder`** — fluent builder for `SELECT` queries: joins, filtering, grouping, ordering, and both pagination styles. Supports composing a query as a subquery inside another's `FROM` clause.
- **`InsertQueryBuilder`** — fluent builder for `INSERT` queries, including multi-row inserts (batched `VALUES (...), (...), ...` in a single round trip) and `OUTPUT Inserted.<column>` for retrieving generated IDs.
- **Query Directors** (`BookQueryDirector`, `AuthorQueryDirector`, `ReviewQueryDirector`) — one per entity, living alongside that entity's repository in the DAL. Each holds a `Columns` constants class (used consistently in `SELECT` and in `reader.GetOrdinal(...)`) and one static method per query shape the repository needs. Repository methods stay thin: ask the Director for a `BuiltQuery`, turn it into a `SqlCommand`, execute, map the reader.

All parameters are passed through `SqlCommand` parameters — never string-concatenated into SQL.

### Pagination

- **Offset pagination** (`PagedRequest`/`PagedResult<T>`) — supports "jump to page N" and total page count; used for shallow, page-numbered browsing. One-to-many joins (e.g. reviews joined to authors) paginate on the base table via a subquery to avoid fanned-out join rows corrupting page size.
- **Keyset (seek) pagination** (`KeysetRequest`/`KeysetResult<T>`) — supports only "is there more," no total count; used for deep/infinite lists and high-churn data.
- Both are driven through `QueryBuilder.Paginate(...)` / `.PaginateKeyset(...)`, with page-change requests served as an AJAX partial swap (`_PaginatedContent`) via `soft-pagination.js`, including a fade transition on swap and a direct page-number jump input alongside the numbered pager.

### Caching

Redis-backed, versioned-invalidation cache sitting above the DAL, called only from the Service layer:

- `ICacheService`/`RedisCacheService` expose `GetAsync<T>`/`SetAsync<T>`/`RemoveAsync`/`InvalidateAsync`, keyed via `CacheKey` (never raw strings).
- List-shaped keys (`Paged`/`Keyset`/`PagedScoped`) are version-tagged per resource (`{resource}:version`, bumped via atomic `INCR` on writes) rather than deleted individually. Single-entity keys (`ById`) are removed directly on update/delete instead.
- Settings (`RedisCacheSettings`: host, port, password, expiration) are bound via the options pattern and validated on startup — the password is never interpolated into a raw connection string, since it may contain delimiter characters.

### Validation

Custom `ValidationAttribute` subclasses (e.g. `PublicationYearAttribute`, `ScoreRangeAttribute`) express single-property domain rules and are used on ViewModels. Non-numeric-typed inputs pass through as valid (the binder is assumed to have already coerced the type). `IValidatableObject` is reserved for genuine cross-field rules that a single attribute can't express.

### Database scripts & seeding

DAL includes a **`Database queries`** folder holding the raw SQL scripts for the database itself (outside of application query-building code):

- Schema creation scripts (tables, constraints, indexes).
- Seed data scripts for local development/testing.

Script/resource paths and other cross-cutting literal values (e.g. embedded resource names, `InitialCatalog`) are centralized per assembly rather than hardcoded at each call site — one `internal static class {Assembly}ConstantsLocator` per assembly (e.g. `DALConstantsLocator` in the DAL), with nested static classes grouping related constants (e.g. `ScriptsFiles`) so a path change happens in one place, not scattered across the codebase. Pattern for a new assembly-scoped constant group: add a nested static class inside that assembly's `{Assembly}ConstantsLocator`, not a new top-level locator or inline literals.

Seeding runs automatically on container startup, gated by the `ENVIRONMENT` value in `.env` (mapped to `ASPNETCORE_ENVIRONMENT`): outside `Production`, schema creation and seed data both run; in `Production`, only schema creation runs and seed data is skipped. {CONFIRM this matches the actual check in your seeding script — e.g. whether it's an exact `!= "Production"` comparison or something else.}

### Logging

Services take an injected logger (`ILogger<TService>`) and log around key operations for observability:

- Entry-level info logs for significant business operations (e.g. creating a book, submitting a review).
- Warning logs for expected-but-notable conditions (e.g. an out-of-range page request being clamped).
- Error logs (with exception detail) around failures before rethrowing or returning an error result, so failures are traceable without leaking internals to the client.
- `RedisCacheService` logs cache hit/miss at debug level and version-bump/invalidation events at info level.

Repositories and Controllers do not log directly — logging is a Service-layer responsibility, keeping the DAL and Controllers free of cross-cutting concerns.

## Running the project

### Prerequisites

- Docker and Docker Compose
- A `.env` file in the project root (see below) — never committed

### Environment variables (`.env`)

All secrets and environment-specific values are supplied via `.env`, never hardcoded in `appsettings.json` or the compose file.

| Variable | Purpose |
|---|---|
| `COMPOSE_PROJECT_NAME` | Prefix for container/network/volume names |
| `ENVIRONMENT` | `Development` / `Production` — sets `ASPNETCORE_ENVIRONMENT`, also gates DB seeding |
| `APP_PORT` | Host port mapped to the app's HTTP port (container 8080) |
| `ENABLE_HTTPS` | `true`/`false`. Controls whether the container's entrypoint script exports HTTPS binding + cert env vars to Kestrel at all — see note below on why this can't be done with a plain `${VAR:-default}` in compose |
| `APP_HTTPS_PORT` | Host port mapped to the app's HTTPS port (container 8081) — only relevant when `ENABLE_HTTPS=true` |
| `CERT_NAME` | Filename (no extension) of the `.pfx` dev cert, mounted from `CERT_PATH` |
| `CERT_PATH` | Host folder containing the `.pfx` cert, mounted read-only into the container at `/https`. Defaults to an empty placeholder folder when HTTPS is off |
| `CERT_PASSWORD` | Password for the `.pfx` cert |
| `DB_USER` / `DB_PASSWORD` | SQL Server `sa` credentials |
| `DB_PORT` | Host port mapped to SQL Server (container 1433) |
| `DB_NAME` | Database name created/used by the app |
| `REDIS_HOST` / `REDIS_PORT` | Redis connection details used by the app container (service name `redis`, container port 6379) |
| `REDIS_PASSWORD` | Redis auth password — also passed to the Redis container via `--requirepass` |

Copy `.env` from the template committed in the repo (values above use placeholders/dots for secrets — replace with real values) before first run. Never commit real secret values.

### Start the stack

```bash
docker compose up --build
```

This starts the web app, SQL Server, and Redis as defined in `docker-compose.yml`. On first run against a fresh SQL Server volume, schema creation (and seed data, outside Production) runs automatically.

### HTTPS

HTTPS is toggled through a single `.env` flag: `ENABLE_HTTPS`. No separate compose file, no extra flags.

**Why this needs an entrypoint script, not just a compose env default:** Kestrel treats `Kestrel__Certificates__Default__Path`/`Password` as configured — and eagerly tries to load the certificate at startup — the moment those environment variable *keys exist at all*, regardless of their value and regardless of whether `ASPNETCORE_URLS` even includes an `https://` binding. A plain `${CERT_NAME:-cert}` fallback in `docker-compose.yml`'s `environment:` block still produces a non-empty key with a path to a file that doesn't exist, so the app crashes with `FileNotFoundException` even in HTTP-only mode. Compose has no syntax to omit an env var entirely based on another var's value — `${VAR:-default}` only substitutes a value.

The fix: the image's entrypoint (`docker-entrypoint.sh`) reads `ENABLE_HTTPS` at container start and only `export`s the Kestrel URL/cert variables when it's `true`; otherwise it explicitly `unset`s them so Kestrel never sees the keys and never attempts to load a cert.

- **HTTP only (default):** `ENABLE_HTTPS=false` — port 8081 is still mapped but nothing binds to it, no cert vars are set, no cert file is required.
- **HTTPS enabled:** `ENABLE_HTTPS=true`, plus a valid `CERT_PATH`/`CERT_NAME`/`CERT_PASSWORD` pointing at a real dev certificate.

One-time local cert setup before enabling HTTPS (dev only — see note below):

```bash
dotnet dev-certs https -ep <CERT_PATH>/<CERT_NAME>.pfx -p <CERT_PASSWORD>
dotnet dev-certs https --trust
```

`--trust` only trusts the cert on the machine you ran it on — this whole flow is for local development. It is **not** a production TLS setup. In a real production deployment, terminate TLS at a reverse proxy (nginx, Caddy, a cloud load balancer) with a certificate from a real CA (e.g. Let's Encrypt) in front of the app container, rather than handing Kestrel a self-signed dev cert.

Then just:

```bash
docker compose up --build
```

— no `-f` flags, no code changes; the toggle lives entirely in `.env`.

## Project scope / constraints

- ASP.NET MVC, .NET 10.
- Data access via ADO.NET only — `SqlConnection`, `SqlCommand`, `SqlDataReader`, parameterized queries, manual POCO/DTO mapping.
- `DataSet`, `DataTable`, `DataView` are never used.
- SQL Server dialect: `TOP (@N)` / `OFFSET ... FETCH NEXT @N ROWS ONLY` (no `LIMIT`).
- Caching via Redis (`StackExchange.Redis`), wrapped behind `ICacheService` only.
- Secrets and environment-specific config live in `.env`, not in `appsettings.json` or the compose file.

## Not yet built (deferred, kept compatible with)

Specification Pattern, `UPDATE`/`DELETE` query builders.

**Partially built, not fully wired:** Keyset (seek) pagination infrastructure (`KeysetRequest`/`KeysetResult<T>`, repository/service-level query support) exists and is used for at least one entity's repository method, but isn't yet exposed through a live view/controller action. Likely future target: selector/autocomplete-style lists, per the original use-case guidance for when keyset is preferred over offset.
