# API, SQL & UI — Interview Q&A

A reference covering API design, gateways and traffic control, distributed rate-limiting with Redis, SQL queries and patterns, frontend/UI, and system-design scenarios.

## Table of Contents

- [API Design Fundamentals](#api-design-fundamentals)
- [API Gateway / Traffic Controller](#api-gateway--traffic-controller)
- [SQL Queries](#sql-queries)
- [UI / Frontend](#ui--frontend)
- [Distributed Rate-Limiting with Redis (Deep Dive)](#distributed-rate-limiting-with-redis-deep-dive)
- [API — Most-Asked / Harder Questions](#api--most-asked--harder-questions)
- [SQL — Most-Asked Query Patterns](#sql--most-asked-query-patterns)
- [UI / JS — Most-Asked / Harder Questions](#ui--js--most-asked--harder-questions)
- [Scenario / System-Design Block](#scenario--system-design-block)

---

## API Design Fundamentals

### REST principles & resource modeling.

Resources as nouns (`/orders/123`), HTTP verbs as actions: GET (safe, read), POST (create, non-idempotent), PUT (full replace, idempotent), PATCH (partial, idempotent if designed so), DELETE (idempotent). Statelessness — each request self-contained (auth + context). Use plural nouns, nest only one level, return proper status codes, HATEOAS optional. Filtering/sorting via query params, not new endpoints.

### HTTP status codes that matter.

- **2xx:** 200 ok, 201 created, 202 accepted (async), 204 no content.
- **4xx client:** 400 bad request, 401 unauthenticated, 403 forbidden, 404 not found, 409 conflict, 422 unprocessable, 429 too many requests.
- **5xx server:** 500 internal, 502 bad gateway, 503 unavailable, 504 timeout.

Distinguish 401 (who are you) vs 403 (you can't), 400 (malformed) vs 422 (semantically invalid).

### Idempotency & idempotency keys.

Idempotent = repeating the call yields the same state (GET/PUT/DELETE). POST isn't naturally — use an `Idempotency-Key` header: server stores the key + response, returns the cached result on retry instead of double-charging. Essential for payments and any at-least-once delivery (retries, flaky networks).

### API versioning.

URI (`/v1/orders`) — explicit, cache-friendly, most common. Header (`Accept: application/vnd.api.v2+json`) — cleaner URLs, harder to test. Query param — simple but messy. Rule: version on breaking changes only; additive changes (new optional field) stay backward-compatible. Deprecate with sunset headers + timelines.

### Pagination strategies.

Offset/limit: simple, but slow on deep pages and unstable under inserts. Cursor/keyset (`WHERE id > last_seen ORDER BY id LIMIT n`): stable, performant at scale — the production default. Return `next_cursor` + `has_more`. Avoid `COUNT(*)` on huge tables for total pages.

### REST vs GraphQL vs gRPC.

- **REST:** resource-oriented, cacheable, ubiquitous; over/under-fetching pain.
- **GraphQL:** client picks exact fields, one round-trip for nested data; caching + rate-limiting harder, N+1 on resolvers.
- **gRPC:** binary protobuf over HTTP/2, fast, streaming, strong contracts — best for internal service-to-service, weak browser support.

Pick REST for public APIs, gRPC internal, GraphQL for varied client data needs.

### Auth: API keys vs OAuth2 vs JWT.

- **API keys:** simple service identity, no user context, easy to leak — scope + rotate.
- **OAuth2:** delegated authorization (access + refresh tokens, scopes) for third-party/user consent.
- **JWT:** signed self-contained token (claims, exp) — stateless validation, but can't easily revoke before expiry (use short TTL + refresh, or a denylist).

Always HTTPS; validate signature, exp, aud, iss.

---

## API Gateway / Traffic Controller

### What an API gateway does.

Single entry point fronting backend services: routing, auth/authN offload, rate limiting/throttling, load balancing, request/response transformation, caching, TLS termination, observability (logs/metrics/tracing), and quota/billing enforcement. Decouples clients from internal topology; centralizes cross-cutting concerns so services stay thin.

### Rate limiting algorithms.

- **Token bucket:** bucket refills at rate r, each request takes a token — allows bursts up to bucket size, smooth average.
- **Leaky bucket:** fixed-rate outflow queue — smooths bursts, no spikes.
- **Fixed window:** count per interval — simple but boundary spikes (2x at window edge).
- **Sliding window log/counter:** accurate, more memory.

Token bucket is the common gateway default. Return 429 + `Retry-After` + `X-RateLimit-*` headers.

### Throttling vs rate limiting vs quotas.

Rate limit: requests per short window (100/min) — protect from bursts. Throttling: deliberately slow/queue/reject over capacity (backpressure). Quota: longer-horizon cap (10k/day, or token budget) — billing/fair-use. Tier by API key/user/plan. Enforce at the gateway, return clear headers so clients self-regulate.

### Token cost / metering (LLM-style or billable APIs).

Meter usage per consumer (requests, tokens, compute), aggregate to a quota counter (often Redis), reject/charge when exceeded, and expose usage headers. Pattern: pre-deduct estimate → reconcile actual after response. Store counters with TTL aligned to billing window; emit usage events to a metering pipeline for invoicing/analytics.

### Load balancing strategies.

Round-robin: even rotation. Least-connections: send to least-busy — good for uneven request cost. Weighted: route by capacity. IP/consistent hash: sticky sessions or cache affinity. Layer 4 (TCP, fast) vs Layer 7 (HTTP-aware, content routing). Pair with health checks to eject unhealthy nodes; add active/passive failover.

### Circuit breaker & resilience.

Circuit breaker: after N failures, trip OPEN → fail fast (skip the dying service); after a cooldown go HALF-OPEN to test, then CLOSED on success. Prevents cascading failure + thread exhaustion. Combine with timeouts, bounded retries + exponential backoff + jitter, bulkheads (isolate pools), and fallbacks/graceful degradation.

### Caching at the API layer.

Client/CDN cache via `Cache-Control`, `ETag` (conditional GET → 304), `Last-Modified`. Gateway/edge cache for hot GETs. Server-side: Redis for computed/aggregated responses. Invalidation is the hard part — TTL for simple cases, event-based bust on writes. Never cache user-specific data on shared caches without keying by identity.

### Webhooks design.

Server-to-client push on events; deliver POST to a registered URL. Reliability: retry with backoff, signature (HMAC) so receiver verifies authenticity, idempotency (event id) since delivery is at-least-once, and a dead-letter after max retries. Provide replay + delivery logs. Receiver should ack fast (2xx) and process async.

---

## SQL Queries

### Joins: which and when.

INNER: only matching rows both sides. LEFT: all left + matched right (NULLs where none) — "find customers with no orders" via `LEFT JOIN ... WHERE o.id IS NULL`. RIGHT: mirror. FULL OUTER: all rows both sides. CROSS: cartesian product. Self-join: row vs row in same table (manager/employee). Join on indexed keys; watch row multiplication from one-to-many joins inflating aggregates.

### Indexes & when they help/hurt.

B-tree index speeds WHERE, JOIN, ORDER BY, range scans by avoiding full scans. Composite index column order matters — usable left-to-right (`idx(a,b)` helps `WHERE a`, `WHERE a AND b`, not `WHERE b` alone). Covering index includes all selected cols → index-only scan. Costs: slower writes, storage. Avoid functions on indexed columns (`WHERE YEAR(date)=...` kills index — use range instead).

### Query optimization workflow.

Read `EXPLAIN` / `EXPLAIN ANALYZE`: look for seq/full scans on big tables, bad join order, high row estimates vs actual (stale stats), filesort, temp tables. Fixes: add/adjust indexes, rewrite to be SARGable (no functions on columns), select only needed columns, filter early, avoid `SELECT *`, replace correlated subqueries with joins, update statistics. Measure before/after.

### WHERE vs HAVING, GROUP BY.

WHERE filters rows before aggregation (uses indexes); HAVING filters after, on aggregates (`HAVING COUNT(*) > 5`). Push filters into WHERE whenever possible for performance. GROUP BY collapses rows per key; every non-aggregated SELECT column must be grouped (or functionally dependent).

### Window functions.

Aggregate without collapsing rows: `ROW_NUMBER()`/`RANK()`/`DENSE_RANK() OVER (PARTITION BY x ORDER BY y)`, running totals `SUM() OVER (... ROWS BETWEEN...)`, `LAG`/`LEAD` for prev/next row, `NTILE` for buckets. Classic uses: top-N per group (filter `ROW_NUMBER()=1`), dedup, period-over-period deltas. Runs after WHERE/GROUP BY, before ORDER BY/LIMIT.

### CTEs & recursive CTEs.

`WITH x AS (...)` — name a subquery for readability and reuse; chain multiple. Recursive CTE walks hierarchies/graphs (org chart, BOM, category tree): anchor member `UNION ALL` recursive member referencing itself until no rows. Cap with depth guard to avoid infinite loops. Prefer over deeply nested subqueries for clarity.

### Transactions, ACID & isolation levels.

ACID: Atomicity, Consistency, Isolation, Durability. Isolation levels trade consistency vs concurrency: Read Uncommitted (dirty reads) → Read Committed (default many DBs) → Repeatable Read (no non-repeatable reads) → Serializable (no phantoms, strictest, most locking). Higher = fewer anomalies, more contention. Pick per use case; payments lean stricter.

### Deadlocks & how to avoid.

Two transactions each hold a lock the other needs → DB kills one (victim). Avoid: acquire locks in a consistent order across the app, keep transactions short, lower isolation where safe, add indexes (reduce locked row ranges), and retry the victim with backoff. Monitor deadlock logs to find the offending query pair.

### N+1 query problem.

ORM lazily loads related rows in a loop → 1 parent query + N child queries. Fix: eager-load with a JOIN or batched IN query (e.g., `JOIN FETCH`, `select_related`/`prefetch`, DataLoader in GraphQL). Detect via query logs/APM showing repeated identical queries. Huge latency win at scale.

---

## UI / Frontend

### State management approach.

Local component state for isolated UI; lift state up for shared parent/child; context for low-frequency global (theme, auth); a store (Redux/Zustand) for complex cross-cutting app state. Server state (API data) belongs in a data layer (React Query/SWR) with caching, dedup, background refetch — don't hand-roll it in global state.

### Rendering & performance optimization.

Avoid unnecessary re-renders: memoize (`React.memo`, `useMemo`, `useCallback`), stable keys in lists, split state so updates are local. Code-split + lazy-load routes/components, virtualize long lists (windowing), debounce/throttle inputs and scroll, defer non-critical work. Measure with the profiler before optimizing; premature memoization adds noise.

### Consuming APIs in the UI.

Handle the full lifecycle: loading, success, error, empty states. Cancel stale requests (`AbortController`) to prevent race conditions on fast input. Cache + dedupe with React Query/SWR; retry on transient failure; show optimistic updates then reconcile/rollback on error. Centralize auth headers + error handling in an API client/interceptor.

### CORS — what and why.

Browser same-origin policy blocks cross-origin requests unless the server opts in via `Access-Control-Allow-Origin` (+ methods/headers/credentials). Preflight OPTIONS for non-simple requests. It's a browser-enforced, server-declared policy — not auth. Fix on the server/gateway, not the client; avoid wildcard with credentials.

### Optimistic UI & error handling.

Optimistic: update UI immediately assuming success, send request, roll back + toast on failure — snappier UX for likes/edits. Pair with idempotent APIs so retries are safe. Always surface actionable error states; never leave a spinner hanging on failure. Guard against double-submit (disable button / idempotency key).

### Accessibility & UX baseline.

Semantic HTML (buttons not divs), keyboard navigation + focus management, ARIA only where semantics fall short, sufficient color contrast, alt text, labeled form fields, and visible focus states. Respect reduced-motion. Test with keyboard-only and a screen reader. Accessibility overlaps with SEO and overall robustness.

### Debounce vs throttle.

Debounce: run only after activity stops for X ms (search-as-you-type — wait until user pauses). Throttle: run at most once per X ms during continuous activity (scroll/resize handlers). Debounce = trailing quiet period; throttle = steady rate cap. Choosing wrong one causes laggy or dropped updates.

---

## Distributed Rate-Limiting with Redis (Deep Dive)

### Why Redis / why distributed.

Multiple gateway nodes behind a load balancer each see only their slice of traffic — local counters let a user exceed the global limit N× (N = node count). You need a single shared, atomic counter. Redis fits: in-memory (sub-ms), single-threaded command execution (atomic per command), TTL support, and Lua for multi-step atomicity. Key shape: `rl:{user_or_apikey}:{window}`.

### The atomicity trap.

Naive `INCR key; EXPIRE key 60` is a race: if requests interleave or the process dies between the two commands, the key can lose its TTL and live forever (counter never resets → user permanently blocked). Fixes: do both in one Lua script, or `SET key 1 EX 60 NX` on first hit then `INCR`. Single-command atomicity isn't enough once the algorithm needs read-modify-write — wrap it in Lua so it runs as one atomic unit on the Redis thread.

### Fixed window (simplest).

```lua
-- KEYS[1]=key, ARGV[1]=limit, ARGV[2]=window_sec
local c = redis.call('INCR', KEYS[1])
if c == 1 then redis.call('EXPIRE', KEYS[1], ARGV[2]) end
if c > tonumber(ARGV[1]) then return 0 end  -- blocked
return 1                                      -- allowed
```

O(1), tiny memory. Con: boundary burst — up to 2×limit across a window edge.

### Sliding window log (accurate) — sorted set.

```lua
-- KEYS[1]=key, ARGV[1]=now_ms, ARGV[2]=window_ms, ARGV[3]=limit, ARGV[4]=unique_id
redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1]-ARGV[2])  -- drop old
local n = redis.call('ZCARD', KEYS[1])
if n >= tonumber(ARGV[3]) then return 0 end
redis.call('ZADD', KEYS[1], ARGV[1], ARGV[4])
redis.call('PEXPIRE', KEYS[1], ARGV[2])
return 1
```

Exact, no boundary spike. Cost: O(log n) + memory grows with requests-per-window per key — expensive for hot keys.

### Sliding window counter (practical middle).

Two fixed-window counters (current + previous); estimate by weighting previous by overlap: `count ≈ current + previous × (1 − elapsed_fraction_of_window)`. ~0.003% error, O(1) memory, no boundary burst. Usually the right default.

### Token bucket via Lua (controlled bursts).

Store tokens + last_refill in a hash; refill lazily per request: `tokens = min(capacity, tokens + (now−last)×rate)`; if `tokens≥1` consume one. Bucket size controls burst, rate controls steady-state. No background job needed (lazy refill).

### redis-cell module (GCRA).

`CL.THROTTLE key max_burst count period quantity` → one atomic command returning allowed/remaining/retry-after. Implements GCRA (token-bucket equivalent storing a single "theoretical arrival time" instead of a counter). Use it if you can install the module — less custom Lua, returns the headers you need.

### Clock skew.

Don't trust each node's wall clock for `now` — nodes drift. Use Redis's `TIME` command inside the Lua script so all nodes share one time source. Critical for sliding-window and token-bucket correctness.

### Performance & hot keys.

Each decision = one Redis round-trip (one `EVAL`) — keep scripts O(1). A single very hot key (global limit, or one whale tenant) serializes on one shard → bottleneck. Mitigate: shard the counter (`key:{0..K}`, random shard, limit = total/K), or two-tier below. In Redis Cluster, keep all keys a script touches in one hash slot via `{hashtag}`.

### Two-tier (local + global).

Each node keeps a small local token bucket and periodically reserves a batch of tokens from Redis (e.g., grab 50 at a time) instead of calling Redis per request. Drastically cuts round-trips at high RPS, at the cost of slight over/under-limit precision.

### Failure handling: fail-open vs fail-closed.

If Redis is unreachable, decide deliberately: fail-open (allow — availability over protection, non-critical limits) or fail-closed (reject — protect a fragile backend, e.g., payments). Add a short timeout on the Redis call, a local fallback limiter, and a circuit breaker so a Redis outage doesn't add latency to every request. Make it explicit/configurable per route.

### Response contract.

Return headers so clients self-regulate: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, and on 429 a `Retry-After`. Compute remaining/reset inside the same Lua call so they're consistent with the decision.

### Algorithm pick.

Simple + tolerant of edge bursts → fixed window. Accuracy + cheap → sliding window counter. Exactness for low-volume/high-value → sliding window log. Bursts with steady refill (API/token budgets) → token bucket / redis-cell GCRA.

### Multi-dimensional limits.

Production limits stack several dimensions evaluated together in one decision: per-API-key (plan tier), per-user, per-IP (abuse/DDoS), per-endpoint (protect expensive routes), and a global system cap. Evaluate all relevant buckets atomically (one Lua script touching multiple keys) — request allowed only if every dimension passes; the first to deny returns 429 with which limit was hit. Order checks cheapest/most-likely-to-deny first. Keep keys co-located (same hash slot) in Cluster so the multi-key script is valid. Compose limits hierarchically: a tenant quota that sub-divides across its users, IP limits as a coarse abuse shield under finer per-user limits.

### Token-budget metering (LLM-style cost limits).

Layer a cost limit on top of request limits: meter actual consumption (tokens, compute units, $) not just call count. Pattern: pre-deduct an estimate before the call (reserve max tokens) → run → reconcile the difference after (refund unused, or charge overage) so concurrent requests can't overspend the budget. Store a rolling counter in Redis keyed to the billing window (`INCRBY` by token count, TTL = window). Enforce both a rate (tokens/min, smooths spikes) and a quota (tokens/day or /month, billing cap). On exceed → 429 with `Retry-After` (rate) or 402/403 (quota exhausted). Emit a usage event per request to a durable metering pipeline (Kafka → warehouse) for invoicing/analytics — Redis is the fast gate, the event stream is the source of truth. Reconcile Redis counters against the ledger periodically to correct drift from crashes/partial failures.

---

## API — Most-Asked / Harder Questions

### Idempotency in distributed systems (exactly-once myth).

Networks give you at-least-once, never true exactly-once delivery — so design for at-least-once + idempotent processing = effectively-once. Client sends a unique `Idempotency-Key`; server stores key → result with a status (in-progress/done) in a dedupe store. On retry: if done, return stored result; if in-progress, block or 409. Make handlers idempotent at the data layer too (unique constraints, upserts, conditional writes). Critical for payments, order creation, webhooks.

### Optimistic vs pessimistic locking (concurrency).

- **Pessimistic:** lock the row (`SELECT ... FOR UPDATE`) so others wait — safe under high contention, but blocks/risks deadlocks, hurts throughput.
- **Optimistic:** no lock; read a version/timestamp, on write check it's unchanged (`UPDATE ... WHERE version=v`), retry on mismatch — great for low contention, no blocking.

Web APIs usually use optimistic (version column or ETag + `If-Match` → 412 on conflict). Pick by contention level and conflict cost.

### Async / long-running request patterns.

Don't hold a connection for a slow job. `202 Accepted` + job id, client polls `GET /jobs/{id}` (status → result). Or push: webhooks (server→client), SSE (one-way stream), WebSocket (bidirectional). Queue the work (Kafka/SQS) for durability + backpressure. Return a result URL on completion. Choose polling for simplicity, webhooks for efficiency, SSE/WS for live updates.

### Microservice communication: sync vs async.

Sync (REST/gRPC): simple, immediate response, but tight coupling + cascading failure + latency adds up across hops. Async (message queue/event bus): decoupled, resilient, absorbs spikes, enables event-driven — but eventual consistency + harder debugging/ordering. Rule: sync for read/request-response needing an answer now; async for writes, fan-out, and cross-service workflows. Add timeouts, retries, circuit breakers on sync calls.

### Distributed transactions / Saga pattern.

Two-phase commit doesn't scale across services (locks, coordinator SPOF). Saga: break a transaction into local steps, each with a compensating action; on failure, run compensations in reverse to undo. Orchestration (central coordinator drives steps) vs choreography (services react to events). Trade ACID for eventual consistency + business-level rollback. Pair with idempotency + an outbox pattern for reliable event publishing.

### Outbox pattern (reliable events).

Problem: writing to DB and publishing an event aren't atomic — one can fail. Solution: in the same DB transaction, insert the event into an outbox table; a separate relay polls/CDC-streams the outbox and publishes to the broker, marking sent. Guarantees the event fires iff the data committed. Consumers dedupe via event id (at-least-once). Standard fix for the dual-write problem.

### API security — OWASP API top risks.

Broken object-level authorization (BOLA/IDOR): always check the caller owns the requested object, never trust IDs in the URL. Broken auth: strong tokens, short TTL, rotation. Excessive data exposure: return only needed fields, don't rely on client to filter. Lack of rate limiting. Mass assignment: allowlist bindable fields. Injection: parameterized queries. SSRF on URL inputs. Security misconfig + verbose errors. Validate, authorize per-object, least privilege.

### Designing a scalable REST API (system-design favorite).

Stateless services behind a load balancer (horizontal scale). Cache aggressively (CDN/edge for GETs, Redis for computed). DB: read replicas for read-heavy, sharding/partitioning for write-heavy, connection pooling. Async + queues for heavy work. Rate limiting + circuit breakers + bulkheads for resilience. Pagination (cursor) on list endpoints. Idempotency on writes. Observability (metrics/logs/traces). Versioning for evolution. Design for failure at every hop.

### Preventing retry storms / thundering herd.

Naive retries amplify an outage (everyone retries at once → kill the recovering service). Fix: exponential backoff + jitter (randomize delays so retries spread out), cap retry count, circuit breaker to stop hammering a dead dependency, and request coalescing/single-flight (collapse duplicate concurrent requests into one). For caches, add jittered TTLs to avoid synchronized expiry.

---

## SQL — Most-Asked Query Patterns

### Nth highest value (e.g., 2nd highest salary).

Robust + portable: `DENSE_RANK` to handle ties.

```sql
SELECT salary FROM (
  SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) rnk
  FROM employees) t
WHERE rnk = 2;
```

Or LIMIT/OFFSET (no ties): `ORDER BY salary DESC LIMIT 1 OFFSET 1`. Use `DENSE_RANK` when "2nd highest distinct value" matters.

### Top-N per group.

`ROW_NUMBER` partitioned per group, filter outside.

```sql
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) rn
  FROM employees) t
WHERE rn <= 3;
```

`ROW_NUMBER` (arbitrary tie-break) vs `RANK` (ties share rank, gaps) vs `DENSE_RANK` (ties share, no gaps) — know the difference; it's the follow-up.

### Find & delete duplicates (keep one).

Find: `GROUP BY` the dup columns `HAVING COUNT(*) > 1`. Delete keeping lowest id:

```sql
DELETE FROM t WHERE id NOT IN (
  SELECT MIN(id) FROM t GROUP BY email);
```

Or `ROW_NUMBER() OVER (PARTITION BY email ORDER BY id)` and delete where `rn > 1`. Add a unique constraint after to prevent recurrence.

### Employees earning more than their manager (self-join).

```sql
SELECT e.name FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

Classic self-join test — same table aliased twice.

### Running total / cumulative sum.

```sql
SELECT date, amount,
  SUM(amount) OVER (ORDER BY date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM transactions;
```

Partition (`PARTITION BY account`) for per-group running totals. Window frame clause is the key detail.

### Period-over-period (LAG/LEAD).

```sql
SELECT month, revenue,
  revenue - LAG(revenue) OVER (ORDER BY month) AS mom_change
FROM monthly;
```

`LAG` = previous row, `LEAD` = next; great for growth/delta questions.

### EXISTS vs IN vs JOIN.

IN: fine for small static lists; with a subquery returning NULLs, `NOT IN` breaks (returns no rows). EXISTS: short-circuits on first match, handles NULLs safely, often better for correlated existence checks. JOIN: when you need columns from the other table (but can multiply rows). Rule: existence check → EXISTS; need other table's data → JOIN; small literal set → IN.

### UNION vs UNION ALL.

UNION removes duplicates (extra sort/dedupe cost). UNION ALL keeps all rows, faster — use it when you know there are no dups or don't care. Interviewers check you default to UNION ALL for performance when dedup isn't needed.

### DELETE vs TRUNCATE vs DROP.

DELETE: row-by-row, WHERE-able, logged, fires triggers, transactional/rollback-able, keeps table. TRUNCATE: deallocates pages, fast, resets identity, minimal logging, no WHERE, usually can't be rolled back per-row. DROP: removes the table structure entirely. Speed: TRUNCATE > DELETE for full wipes.

### Clustered vs non-clustered index.

Clustered: defines the physical row order; one per table (often the PK); range scans + ordered reads are fast since data is the leaf. Non-clustered: separate structure with pointers/row-locators to the data; many per table; add a covering index (INCLUDE columns) to satisfy a query from the index alone (index-only scan). Too many indexes slow writes.

### Optimistic concurrency in SQL.

Add a version/rowversion column; update conditionally:

```sql
UPDATE accounts SET balance=?, version=version+1
WHERE id=? AND version=?;
```

If 0 rows affected → someone else updated it → reload and retry. Avoids locks; standard for web apps. Pessimistic alternative: `SELECT ... FOR UPDATE` inside a transaction.

### Scaling reads/writes: replicas, partitioning, sharding.

Read replicas: offload reads, async replication → read-your-writes lag. Partitioning: split one table by range/list/hash within a DB (prune scans, easier maintenance). Sharding: spread data across DBs by a shard key — scales writes, but cross-shard joins/transactions are hard; pick the shard key carefully (even distribution, avoid hotspots). Add caching before sharding.

---

## UI / JS — Most-Asked / Harder Questions

### React reconciliation & keys.

On state change React builds a new virtual DOM tree and diffs it against the old (reconciliation), patching only changed real DOM nodes. Keys tell it which list items are stable across renders — wrong/index keys cause remounts, lost state, and bugs on reorder/insert. Use stable unique ids as keys, never array index for dynamic lists.

### useEffect pitfalls (stale closures, deps, cleanup).

Effect captures the variables from its render (closure) — stale if deps are wrong. Include every value used in the dependency array (or the linter warns). Return a cleanup function to cancel subscriptions/timers/requests and avoid leaks/race conditions. Empty deps = run once on mount. Don't put objects/functions in deps without memoizing them (they're new each render → infinite loops).

### Controlled vs uncontrolled components.

Controlled: React state is the source of truth (value + onChange) — predictable, validate-on-change, but re-renders per keystroke. Uncontrolled: DOM holds the value, read via ref — less code, fewer renders, harder to validate live. Default to controlled for forms needing validation/conditional logic; uncontrolled for simple/perf-sensitive inputs.

### Event loop, microtasks vs macrotasks.

Single-threaded JS runs the call stack, then drains all microtasks (Promise callbacks, `queueMicrotask`), then one macrotask (`setTimeout`, I/O), repeat. Microtasks always run before the next macrotask/render. This is why a resolved Promise's `.then` runs before a `setTimeout(0)`. Common trap question with ordering of console logs.

### Closures (the #1 JS question).

A function retains access to its lexical scope even after the outer function returns. Powers private state, currying, memoization, event handlers. Classic bug: `var` in a loop shares one binding (all callbacks see the final value) — fix with `let` (per-iteration binding) or an IIFE. Closures also cause leaks if they capture large objects that never release.

### Promise.all vs race vs allSettled vs any.

- **all:** resolves when all resolve, rejects fast on first rejection (lose other results).
- **allSettled:** waits for all, returns every status (never short-circuits) — use when you want all outcomes.
- **race:** settles on the first to settle (resolve or reject) — timeouts.
- **any:** first to fulfill, ignores rejections until all fail.

Pick by whether one failure should abort the batch.

### Debounce vs throttle — and implementing debounce.

(See basics above for when.) Implementation: store a timer in a closure; on each call clear the previous timer and set a new one for delay ms — only the last call in a quiet period fires.

```js
function debounce(fn, ms){let t; return (...a)=>{clearTimeout(t); t=setTimeout(()=>fn(...a), ms);};}
```

Throttle instead tracks last-run time and skips calls within the interval. Writing these from scratch is a frequent live task.

### Web security: XSS vs CSRF.

- **XSS:** attacker injects script that runs in the victim's browser (steals tokens/DOM) — defend with output encoding, sanitization, CSP, framework auto-escaping; avoid `dangerouslySetInnerHTML`.
- **CSRF:** tricks an authenticated browser into making an unwanted request using its cookies — defend with anti-CSRF tokens, SameSite cookies, and checking Origin/Referer.

XSS = injected code; CSRF = forged request. Know both defenses.

### Storage: localStorage vs sessionStorage vs cookies.

localStorage: ~5MB, persists, same-origin, JS-accessible, not sent to server — app state/prefs. sessionStorage: same but cleared on tab close. Cookies: tiny (~4KB), auto-sent with every request, server-readable — use `HttpOnly` + `Secure` + `SameSite` for auth tokens (HttpOnly blocks JS/XSS theft). Don't store sensitive tokens in localStorage (XSS-readable).

### CSR vs SSR vs SSG vs hydration.

CSR: browser renders from JS — fast nav, slow first paint, weak SEO. SSR: server renders HTML per request — fast first paint + SEO, more server load. SSG: pre-render at build — fastest, but static/needs rebuild. Hydration: client attaches JS event handlers to server-rendered HTML to make it interactive. Modern stacks mix (per-route choice, streaming SSR, partial hydration). Choose by SEO/freshness/interactivity needs.

### Core Web Vitals / frontend performance.

LCP (largest contentful paint — load speed), CLS (cumulative layout shift — visual stability), INP/FID (interactivity). Improve: code-split + lazy-load, optimize/lazy images, preconnect/preload critical assets, minimize render-blocking CSS/JS, cache (CDN, service worker), reserve space to avoid shifts, virtualize long lists, debounce expensive handlers. Measure with Lighthouse/RUM before optimizing.

---

## Scenario / System-Design Block

### How to drive any API system-design round.

1. **Clarify:** functional requirements, scale (RPS, read:write ratio, data size), latency/consistency SLAs.
2. **Define the API contract** (endpoints, payloads, status codes).
3. **Data model + storage choice** (SQL vs NoSQL, indexes, keys).
4. **High-level architecture** (LB → stateless services → cache → DB → queue).
5. **Scale the bottleneck** (caching, replicas, sharding, async).
6. **Address cross-cutting:** auth, rate limiting, idempotency, observability, failure modes.

Always state tradeoffs out loud — interviewers grade reasoning, not the "right" answer.

### Design a URL shortener.

Contract: `POST /urls {long_url}` → `{short_code}`; `GET /{code}` → 301 redirect. ID generation: base62-encode an auto-increment/snowflake id (short, collision-free) or hash + collision check. Storage: key-value (code → long_url), read-heavy → cache hot codes in Redis, read replicas. Redirect: 301 (permanent, cacheable) vs 302 (lets you count clicks). Scale: codes are immutable → CDN/edge-cacheable; analytics via async event stream (don't block the redirect). Custom aliases need a uniqueness check. ~7 base62 chars ≈ 3.5T URLs.

### Design a rate-limited payment endpoint.

Contract: `POST /payments` with `Idempotency-Key` header. Flow: validate → check idempotency store (return prior result if key seen) → rate-limit per user/card (Redis token bucket) → reserve funds → call processor → persist result keyed by idempotency key → emit event (outbox). Correctness: idempotency prevents double-charge on retry; optimistic locking / unique constraint on `(idempotency_key)` at the DB; fail-closed rate limiting (reject if Redis down — protect money path). Async: settlement/webhooks via queue; saga + compensation if a downstream step fails. Strong consistency, audit log everything.

### Design a notification / fan-out service.

Contract: `POST /notify {users, channel, payload}`. Architecture: API enqueues to a broker (Kafka/SQS) → workers consume and deliver per channel (push/email/SMS) → provider adapters. Fan-out: for one event to many users, write to a queue and let workers parallelize; dedupe by notification id (at-least-once). Reliability: retries + backoff, dead-letter queue after max attempts, idempotent delivery. Scale: partition by user id, autoscale workers on queue depth. Add user preferences + rate limits to avoid spamming. This tests async + outbox + idempotency together.

### Design a file upload service.

Contract: client requests `POST /uploads` → server returns a pre-signed S3/blob URL → client uploads directly to storage (bypass app servers, no proxying large bytes) → client calls `POST /uploads/{id}/complete` → server records metadata + triggers async processing (virus scan, thumbnail, transcode) via queue. Large files: multipart/chunked upload + resumable. Serve via CDN. Store metadata in SQL, blobs in object storage. Key insight interviewers want: don't stream big files through your API — pre-signed URLs + async post-processing.

### Design a paginated, filterable list/search API.

Contract: `GET /items?cursor=&limit=&filter=&sort=`. Pagination: cursor/keyset (`WHERE (sort_key,id) > (last) ORDER BY sort_key,id LIMIT n`) — stable + fast at depth, avoid OFFSET on big tables. Indexing: composite index matching the sort+filter columns; covering index for hot queries. Heavy/free-text search → offload to a search engine (Elasticsearch/OpenSearch) kept in sync via CDC/events, not `LIKE '%x%'` on the primary DB. Cache common filter combos. Return `next_cursor` + `has_more`, never a total COUNT on huge tables.

### Design an idempotent checkout / order API.

Contract: `POST /orders` with `Idempotency-Key`. Steps span services (inventory, payment, shipping) → use a Saga: reserve inventory → charge payment → create shipment; each step has a compensating undo (release inventory, refund) run in reverse on failure. Consistency: optimistic locking on stock (`UPDATE ... WHERE qty>=n`), outbox for reliable events, idempotency to survive client retries. Expose order status via `GET /orders/{id}` (async result). This single scenario exercises idempotency + locking + saga + outbox + eventual consistency — a favorite capstone.

### Real-time updates: polling vs SSE vs WebSocket (design choice).

Short polling: simplest, wasteful at scale. Long polling: fewer empty responses, still HTTP. SSE: server→client stream over one HTTP connection, auto-reconnect — perfect for feeds/notifications/progress (one-way). WebSocket: full-duplex, low overhead — chat, collaborative editing, live trading. Choose by direction + frequency: one-way updates → SSE; bidirectional/interactive → WebSocket; infrequent → polling. Mention connection limits, scaling sticky connections (pub/sub layer like Redis), and fallbacks.
