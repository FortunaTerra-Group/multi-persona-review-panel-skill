# Example run: reviewing the rate-limiting middleware PR

A worked example of the panel this skill runs, on a sample diff. The feature is the one in the
governed-build-loop skill's
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

**Architecture: BLOCK**
> `get_tenant_limit` issues a synchronous DB query on every request in the hot path (called at
> line 4, query at line 14). The contract's DO-NOT list names this exact case: "config lookup
> must be cached, not queried live." A direct contract violation is a blocking defect, not a
> style note. Cache the tier-to-limit mapping (even a 60s in-process TTL) and invalidate it from
> the admin write path planned for wave 2. `middleware/rate_limit.py:4,14`

**Security: SIGN**
> `tenant_id` is read from `request.tenant_id` (line 3), which the existing auth middleware sets
> upstream of this one. No new trust boundary is introduced and no client-supplied header is
> trusted directly. No blocking issue. One non-blocking note: confirm the admin endpoint planned
> for wave 2 requires the same auth level as other tenant-config mutations, since it is not in
> this diff.

**Quality: BLOCK**
> Two acceptance criteria from the contract are not met and neither has a test.
> (1) The unlimited-tier list is never consulted: `count > limit` (line 9) is enforced for every
> tenant, and a tenant with no tier row falls through to `DEFAULT_LIMIT` (line 15), so the
> customers the contract says must see no change are the ones most likely to be throttled. This
> is the default-scenario question from step 3, and the answer is no.
> (2) The contract specifies a sliding window; this key (line 5) is a fixed window named by
> `current_window()`, which lets a tenant send `limit` requests in the last second of one window
> and `limit` more in the first second of the next. Nothing pins the window semantics.
> Needs: the unlimited-tier check before enforcement, with a test; and either the sliding window
> the contract asked for or a contract amendment, plus a boundary test asserting the accepted
> rate over any span of `WINDOW_SECONDS`. `middleware/rate_limit.py:5,9,15`

**Performance: SIGN-WITH-CHANGE**
> Same root cause as the Architecture finding, from the other direction: a DB round-trip plus a
> Redis round-trip per request roughly doubles hot-path latency against the pre-change baseline.
> Once the lookup is cached this resolves; flagging it separately because a cache with too long
> a TTL brings the profile back under a miss storm. One more item: a crash between `incr` (line
> 6) and `expire` (line 8) leaves a key with no TTL. Because the key is per window it is a slow
> memory leak rather than a lockout, but it is a leak. Set the TTL in the same operation as the
> increment.

## Synthesis and fold

Architecture and Performance found the same hot-path cost independently, so they fold into one
change: replace the live query with a cached lookup (60s TTL, explicit invalidation hook left for
the wave-2 admin API). Quality's BLOCK became two changes: check the unlimited-tier list (from the
same cached config) before any counting, with a test for a listed tenant; and implement the
sliding window the contract specified, with a boundary test. Performance's leak was closed by
doing the increment and the TTL in a single Lua script. Security's SIGN stands; its note was
carried into the wave-2 contract instead of blocking this PR.

**Verdict after fold: mergeable.** Two BLOCKs and one required change, each concrete and named.
Finding them before the push is the point; the alternative was a human reviewer's first pass, or
production.
