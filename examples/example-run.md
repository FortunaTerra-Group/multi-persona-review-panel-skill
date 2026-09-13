# Example run: reviewing the rate-limiting middleware PR

A worked example of the panel this skill runs, on a sample diff you can rerun yourself. The code
is at [FortunaTerra-Group/review-panel-demo](https://github.com/FortunaTerra-Group/review-panel-demo):
`main` is the service before the change, `feat/per-tenant-rate-limit` is the branch reviewed
below, and `feat/per-tenant-rate-limit-folded` is the result after the fold. The defects in the
branch were seeded to demonstrate the skill; the run also found things that were not seeded,
called out where they appear. The feature is the one in the governed-build-loop skill's
[example contract](https://github.com/FortunaTerra-Group/goal-contract/blob/main/plugin/examples/example-run.md)
("add per-tenant rate limiting to the public API", a fictional service called Acme Metrics API).
Two clauses of that contract matter here: wave 1(a) specifies a *sliding-window* counter, and
the DO-NOT list says limits must not apply to tenants on the unlimited-tier list and the limit
check must not add a synchronous database round-trip to the request hot path.

This is the spine PR under review (line numbers as shown, counting the comment as line 1):

```python
# middleware/rate_limit.py  (new file)
def tenant_rate_limit_middleware(request, get_response):
    tenant = request.tenant_id
    limit = get_tenant_limit(tenant)          # DB read, see below
    key = f"ratelimit:{tenant}:{current_window()}"
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, WINDOW_SECONDS)
    if count > limit:
        return HttpResponse(status=429)
    return get_response(request)

def get_tenant_limit(tenant_id):
    row = db.query("SELECT rate_limit FROM tenant_tiers WHERE tenant_id = %s", tenant_id)
    return row.rate_limit if row else DEFAULT_LIMIT
```

Scope for the panel is the whole PR (`git diff main...HEAD`), which also includes the wave-1
Redis counter helper and a config module, omitted here for brevity.

## Panel: 4 lenses convened

This is a condensed transcript of a real run of the skill on the branch above (2026-09-12). Each
lens ran as its own agent with the diff, the files it touches, and the contract; each executed
probes against the code before returning. Seeded: the hot-path query, the unread unlimited-tier
list, the fixed window, and the two-call increment-and-expire. Not seeded, found anyway: the
missing-tenant bucket, the NULL limit, the proxy default-limit test, and committed bytecode.

**Architecture: BLOCK**
> `get_tenant_limit` issues a synchronous DB query on every request in the hot path (called at
> line 4, query at line 14). The contract's DO-NOT list names this exact case: "config lookup
> must be cached, not queried live." Probe: five requests, `db.query_count == 5`. A direct
> contract violation is a blocking defect. Also: `UNLIMITED_TIER_TENANTS` is defined in config
> and imported by nothing, so the effective unlimited set silently becomes "whatever the tier
> table says," a second owner for a list the contract gave one owner. Cache the lookup with an
> invalidation hook for the wave-2 admin path. `middleware/rate_limit.py:4,14`

**Security: BLOCK**
> Unlimited-tier tenants are throttled: probe, 101 requests as an enterprise tenant, the 101st
> returns 429. That is a denial of service against the customer class the contract says must
> see no change. Second: a request with no `tenant_id` (auth skipped or misordered) lands in a
> shared key `ratelimit:None:...` instead of failing closed; one anonymous caller can 429 every
> other pre-auth request. `tenant_id` itself comes from the auth middleware, the SQL is
> parameterized, and the key cannot collide across tenants, so no new injection surface. Fix:
> check the unlimited list before any I/O; reject a missing tenant with 401.
> `middleware/rate_limit.py:3,9`

**Quality: BLOCK**
> Zero of the contract's three evidence items has a test. (1) Boundary: the key is a fixed window
> named by `current_window()`; probe with limit 3, six requests accepted inside a two-second span
> across a boundary, twice the limit. The contract specified a sliding window. (2) Exempt tenant:
> no test, and it fails. (3) Zero DB queries after warm-up: no test, and `Database.query_count`
> exists on main for exactly this assertion. One more: `test_unknown_tenant_uses_default_limit`
> passes for any limit of one or more; it never observes the default. And a tier row whose
> `rate_limit` is NULL raises `TypeError` at `count > limit`, which the test double cannot reach.
> `middleware/rate_limit.py:5,9,15`; `tests/test_rate_limit.py:39`

**Performance: BLOCK**
> Same hot-path finding, measured: baseline on main is zero DB and zero Redis calls per
> authenticated request; this branch adds one DB round trip and one Redis round trip to every
> request, and the DB query runs before the limit check, so rejected requests pay it too. The
> limiter forwards a tenant's burst to the shared database at full rate. Separately, `incr` and
> `expire` are two commands (lines 6 and 8): `INCR` is atomic, so this is not a counting race,
> but a failure between the two leaves a key with no TTL. Because the key is per window it is a
> slow memory leak, not a lockout. Do the increment and the TTL in one Lua script. The clock is
> read directly from `time.time()`, so the boundary test cannot be written until it is injectable.

## Synthesis and fold

Four lenses, four BLOCKs, and three root causes, each found by all four independently: the
live tier query on the hot path, the unlimited-tier list that nothing reads, and the fixed window
where the contract said sliding. That convergence is the panel's strongest signal; it means the
defects are in the code, not in one reviewer's taste. The fold was one change set: a cached tier
lookup with an invalidation hook, the unlimited-tier check before any I/O, a sliding window with
the increment and the TTL in one Lua script, an injectable clock, a 401 on a missing tenant, NULL
limits falling back to the default, and the contract's three evidence tests plus a real
default-limit test. Second run: four SIGNs.

**Verdict after fold: mergeable.** Every finding was concrete and named, and the green suite on
the first branch had asked none of the questions that mattered. That is the point of running the
panel before the push rather than after a human reviewer's first pass, or after production.
