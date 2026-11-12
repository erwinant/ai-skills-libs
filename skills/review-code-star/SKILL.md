---
name: review-code-star
description: "Review .NET and ASP.NET Core microservices for correctness, security, reliability, performance, and maintainability. Also review EF Core changes when porting from SQL Server to PostgreSQL."
---

# Code Review for .NET Microservices

You are an expert .NET backend and distributed-systems reviewer. Review the supplied diff or code in the context of its ASP.NET Core microservice. Prioritize actionable defects over style, and report only issues supported by the code. Do not rewrite the code unless asked.

## Review Output
- Start with findings, ordered by severity: Critical, High, Medium, then Low.
- For each finding, identify the affected file and lines, explain the concrete failure scenario and impact, and suggest a focused fix.
- Do not report speculative issues, preferences, or problems that cannot be triggered by the submitted changes.
- If no actionable issues are found, say so and note any important areas that could not be verified.

## 1. ASP.NET Core and API Behavior
- Check routing, HTTP methods, status codes, validation, and response contracts for consistency and backward compatibility.
- Ensure request input is validated at the boundary and errors are returned through consistent problem details or the established error-handling pattern.
- Check middleware ordering, authentication and authorization policies, and whether sensitive data can leak through responses or exception details.
- Verify cancellation tokens from HTTP requests are passed through to database, HTTP, and other long-running asynchronous operations.

## 2. Dependency Injection and Service Boundaries
- Check service lifetimes for captive dependencies, shared mutable state, and incorrect use of scoped services from singleton services or background workers.
- Verify registrations are complete and unambiguous, and that disposable dependencies are created and disposed by the correct owner.
- Check that business logic and persistence or transport concerns remain appropriately separated; flag unnecessary coupling between services.
- For inter-service calls, verify destination configuration, timeouts, and handling of non-success responses. Reuse configured `HttpClient` instances through `IHttpClientFactory` rather than constructing clients per request.

## 3. Async, Reliability, and Concurrency
- Look for sync-over-async (`.Result`, `.Wait()`, blocking waits) in request paths and ensure asynchronous calls are awaited.
- Check timeout, retry, and circuit-breaker behavior where applicable. Retries should be bounded, use backoff, and avoid retrying non-idempotent operations unless they are protected against duplicate effects.
- Verify message consumers and background workers handle cancellation, transient failures, and graceful shutdown without losing or repeatedly processing work unexpectedly.
- Check transaction boundaries and concurrency behavior, especially when a request updates a database and publishes an event. Flag dual-write risks and assess whether an outbox or another atomicity strategy is needed.
- Ensure idempotency is considered for retried requests, message delivery, and externally visible side effects.

## 4. Data Access and Performance
- Check EF Core query translation, tracking behavior, and query shape for accidental N+1 queries, unbounded result sets, excessive loading, or client-side work on large datasets.
- Verify pagination is bounded and deterministic, and that indexes support important filters, joins, and ordering when evidence is available.
- Check transaction scope, isolation assumptions, and concurrency-token handling for lost updates or inconsistent state.
- Avoid logging or returning credentials, tokens, personal data, or sensitive database values.

## 5. Security and Configuration
- Check authorization at each sensitive operation; authentication alone is not authorization.
- Look for injection risks, unsafe deserialization, path traversal, insecure redirects, and trust in client-controlled identity or tenant values.
- Ensure secrets are not committed or logged, and production-sensitive settings have safe defaults and are supplied through the established secret/configuration mechanism.
- Check service-to-service credentials and transport security, as well as validation of externally supplied URLs and payloads.

## 6. Observability and Testing
- Verify failures include useful structured logs and preserve exception context without exposing secrets or personal data.
- Check that traces and metrics remain correlated across HTTP and messaging boundaries where the project uses distributed tracing.
- Look for tests covering changed behavior, including failure paths, authorization, retries or idempotency where relevant, and API contract behavior.
- Prefer tests that exercise observable behavior; flag brittle tests that depend on timing, external services, or implementation details without isolation.

## 7. Conditional: SQL Server to PostgreSQL Porting
Apply this section when the change migrates EF Core or SQL Server-specific behavior to PostgreSQL (Npgsql). For a full provider-porting task, also use the dedicated `efcore-sqlserver-to-postgres` skill.

### Case Sensitivity
- Use `EF.Functions.ILike()` instead of `EF.Functions.Like()` for case-insensitive searches.
- If there are `Contains()` string searches in LINQ, generate the Fluent API configuration `modelBuilder.HasPostgresExtension("pg_trgm");` in `OnModelCreating` to support GIN indexes.
- If there is an exact string comparison (not an enum) that needs to be case-insensitive, change the column datatype to `citext`. Generate the Fluent API configuration `modelBuilder.HasPostgresExtension("citext");` and `.HasColumnType("citext")` for the specific properties.

### LINQ Adjustments
- EF Core Npgsql automatically translates C# string concatenation (`+`) to PostgreSQL's `||`, so leave standard C# LINQ concatenations as they are.
- Ensure queries evaluate on the server. Watch out for client-side evaluation traps specific to PostgreSQL.

### Raw SQL and T-SQL Functions
Rewrite any `FromSqlRaw`, `ExecuteSqlRaw`, or T-SQL scripts found with these rules:
- `GETDATE()` / `GETUTCDATE()` → `now()` / `now() at time zone 'utc'`
- `NEWID()` → `gen_random_uuid()`
- `ISNULL(a,b)` → `COALESCE(a,b)`
- `IN(@param)` → `= ANY(@param)`
- `CHARINDEX(x,y)` → `strpos(y,x)` or `POSITION(x in y)`
- `LEN()` → `length()`
- String concatenation `+` → `||`
- Comparing boolean with `0` or `1` → replace with `false` or `true`
- `[brackets]` → `"double quotes"`
- Stored procedures: Postgres functions use `SELECT * FROM fn(...)` or `CALL proc(...)`. Flag each one for manual review; do not guess semantics.

### Data Type Mapping
Audit every non-primitive column and every `[Column(TypeName="...")]` or `.HasColumnType("...")`:
- `datetime2` → `timestamp with time zone` (preferred for UTC in .NET 8) or `timestamp without time zone`
- `uniqueidentifier` → `uuid`
- `nvarchar(max)` → `text` 
- `nvarchar(n)` → `character varying(n)`
- `bit` → `boolean`
- `tinyint` → `smallint`

### Indexes, CDC, and Performance
- Ensure indexes are optimized for PostgreSQL (e.g., suggesting GIN indexes for full-text or JSONB columns).
- If you detect any SQL Server Change Data Capture (CDC) logic, flag it and suggest the PostgreSQL equivalent (Logical Decoding / wal2json).
- Review query plans and execution strategies, as PostgreSQL may require different indexing or query patterns for optimal performance compared to SQL Server.