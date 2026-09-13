# Example run: reviewing the rate-limiting middleware PR

A worked example of the panel this skill runs, on a sample diff you can rerun yourself. The code
is at [FortunaTerra-Group/review-panel-demo](https://github.com/FortunaTerra-Group/review-panel-demo),
a small Django API: `main` is the service before the change, `feat/per-tenant-rate-limit` is the
branch reviewed below, and `feat/per-tenant-rate-limit-folded` is the result after the fold. The
defects in the branch were seeded to demonstrate the skill; the run also found things that were
not seeded, called out where they appear. The feature is the one in the governed-build-loop
skill's [example contract](https://github.com/FortunaTerra-Group/goal-contract/blob/main/plugin/examples/example-run.md)
("add per-tenant rate limiting to the public API"); the demo repo carries it as `CONTRACT.md`.
Three clauses matter here: wave 1(a) specifies a *sliding-window* counter, and the DO-NOT list
says limits must not apply to tenants on the unlimited-tier list and the limit check must not
add a synchronous database round-trip to the request hot path.

The branch is two commits: a window-key helper, then the middleware below, wired into
`MIDDLEWARE` after the API-key auth middleware, with three tests. The suite is green.

```python
# metrics/middleware/rate_limit.py  (new file; line numbers as in the repo)
import time                                                              # 1

from django.conf import settings                                         # 3
from django.http import JsonResponse

from metrics.models import TenantTier                                    # 6
from metrics.ratelimit.windows import window_key
from metrics.redis_client import get_redis


class TenantRateLimitMiddleware:                                         # 11
    """Per-tenant request limit. Runs after ApiKeyAuthMiddleware, which sets request.tenant_id."""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):                                         # 17
        tenant = request.tenant_id                                       # 18
        limit = get_tenant_limit(tenant)                      # DB read  # 19
        key = window_key(tenant, time.time(), settings.RATE_LIMIT_WINDOW_SECONDS)
        redis = get_redis()
        count = redis.incr(key)                                          # 22
        if count == 1:
            redis.expire(key, settings.RATE_LIMIT_WINDOW_SECONDS)        # 24
        if count > limit:                                                # 25
            return JsonResponse({"error": "rate_limited"}, status=429)
        return self.get_response(request)


def get_tenant_limit(tenant_id):                                         # 30
    try:
        return TenantTier.objects.get(tenant_id=tenant_id).rate_limit    # 32
    except TenantTier.DoesNotExist:
        return settings.DEFAULT_RATE_LIMIT
```

`metrics/ratelimit/windows.py` (the first commit) computes `int(now // window_seconds)` and
names the key `ratelimit:{tenant}:{window}`. Its docstring says "fixed window."

Scope for the panel is the whole PR (`git diff main...HEAD`): both commits, the settings line,
and one more change that turns out to matter, a `@pytest.mark.django_db` mark added to the
existing `test_healthz_needs_no_key` in `tests/test_auth.py`.

## Panel: 4 lenses convened

Condensed from a real run (2026-09-12). Each lens ran as its own agent with the procedure in
this repository, the diff, the files it touches, and the contract; each executed probes against
the code (Django test client, the repo's fake Redis) before returning. Seeded: the hot-path
query, the unread unlimited-tier list, the fixed window, the two-call increment-and-expire, and
the `django_db` mark. Not seeded, found anyway: the NULL limit crash and the proxy test.

**Architecture: BLOCK**
> The unlimited-tier list has no reader. `settings.UNLIMITED_TIER_TENANTS` is defined and,
> per `git grep`, referenced nowhere else; the middleware enforces on every tenant (lines 18 to
> 26). Probe: 101 requests as an enterprise tenant, the 101st is a 429. Second: a live
> `TenantTier.objects.get` on every request (line 19, query at line 32), which the contract's
> DO-NOT list forbids in so many words; `CaptureQueriesContext` shows three queries per request
> against two on main. Third: `rate_limit` NULL is documented as valid in the model and seeded
> in the fixtures, and `count > limit` at line 25 raises `TypeError` for it. Also a boundary
> note: the counter algorithm and the tier lookup both live inside the middleware rather than in
> the `ratelimit` package the first commit created, so the middleware owns policy, counting, and
> HTTP at once. `metrics/middleware/rate_limit.py:18-26,32`

**Security: BLOCK**
> Public paths are now rate-limited under a shared sentinel. Auth sets `tenant_id = None` on
> `/healthz` and passes through; this middleware then counts every anonymous caller under the
> key `ratelimit:None:<window>` at the default limit. Probe: 101 GETs to `/healthz` with no
> credentials, the 101st is a 429. A load balancer that treats non-2xx as unhealthy will pull
> instances. The tell in the diff is `tests/test_auth.py:16`: the `django_db` mark was added to
> `test_healthz_needs_no_key` because the public path now hits the database; the diff silenced
> the signal instead of reading it. Same unlimited-tier and NULL findings as above (a class of
> customers throttled, a class of customers 500ing). `tenant_id` itself comes from the
> `api_keys` row, the ORM parameterizes the query, and the key cannot collide across tenants, so
> no new injection surface. Wave 2 note: the admin endpoint must not be able to write a value
> that throttles an exempt tenant, so the list must dominate the table.

**Quality: BLOCK**
> Zero of the contract's three evidence items has a test, and the implementation fails all
> three. Boundary: the window is fixed; probe with limit 3, six requests accepted inside a
> two-second span across a boundary, twice the limit. Exempt tenant: no test, and it is
> throttled. Zero DB queries after warm-up: no test, and there is one per request. One more: the
> existing `test_tenant_without_tier_row_uses_default_limit` (tests/test_rate_limit.py:20) fires
> one request and asserts a 200, which passes for any limit of one or more; it never observes the
> default. And the green 10 of 10 is itself a proxy: the suite never exercises the enterprise,
> legacy, or `None` tenant, never crosses a window boundary, and never counts queries.

**Performance: BLOCK**
> Measured against main: an authenticated request cost two DB queries and zero Redis calls;
> on this branch it costs three DB queries and one Redis call, and a rejected request still
> pays the tier query because line 19 runs before the check at line 25. The limiter forwards a
> tenant's burst to the database at full rate. `incr` then `expire` (lines 22 and 24) are two
> commands: `INCR` is atomic, so this is not a counting race, but a failure between them leaves
> a key with no TTL. Because the key is per window that is a slow leak, not a lockout. Two
> smaller items: no `Retry-After` on the 429, so clients tight-retry into the DB cost; and the
> clock is read straight from `time.time()`, so the boundary test cannot be written until it is
> injectable.

## Synthesis and fold

Four lenses, four BLOCKs. Three root causes found by all four independently: the live tier
query on the hot path, the unlimited-tier list nothing reads, and the fixed window where the
contract said sliding. Two more found by three of four: the `None` bucket on public paths, and
the NULL limit. That convergence is the panel's strongest signal; it says the defects are in the
code, not in one reviewer's taste.

The fold was one change set: a cached tier lookup with an invalidation hook for the wave-2 admin
path; the unlimited-tier check before any I/O; a sliding window with the trim, the count, the
admit, and the TTL in one Lua script; an injectable clock; public paths skipped by deriving the
decision from auth's output rather than a second path list; NULL limits falling back to the
default; an explicit fail-closed 503 when Redis is unavailable; `Retry-After` on both; the
`django_db` mark removed so the healthz test guards the regression again; and the contract's
three evidence tests plus a real default-limit test.

## What the second and third runs showed

The author of the fold believed it was clean. It was not, and the second run said so: three
SIGN-WITH-CHANGE and one BLOCK. The block was on a test, not on code. The boundary test's fake
clock started twenty seconds into a fixed-window bucket, so its "burst, then two seconds later"
never crossed a bucket edge, and the Quality lens proved it by swapping the sliding window for a
fixed one: all eleven tests stayed green. The implementation was right; the proof was not. The
other lenses found a `getattr` default that turned a mis-ordered middleware chain into "everyone
is unlimited" with no signal, a process-local cache whose `invalidate()` and test over-claimed,
a cache sized at Django's default of 300 entries so the tier query came back onto the hot path
above 300 active tenants (measured: 100 percent miss at 400), and a Redis client with no socket
timeouts, so "fails closed" was true for a refused connection and false for a stalled one.

Those were folded. The third run returned four SIGN-WITH-CHANGE, converged on one item: the new
timeouts were correct (three lenses measured 0.50 s against a real silent socket) but no test
could see them, because the test fixture replaced the client factory with a lambda that discarded
keyword arguments. Two lenses independently expected redis-py's retry policy to multiply the
stall, measured it, and dropped the finding before reporting. That fold added a test that records
what the factory receives and a test that opens a real accept-and-never-reply socket and asserts
a 503 inside the bound; both fail with the timeouts removed. No fourth run was made.

**Verdict after three runs: mergeable, with the remaining notes recorded for wave 2.** The runs
converge; they do not reach four bare SIGNs, and a reader should not expect their own to either.
The full output of every lens in all three runs, the briefs each was given, and a synthesis per
run are in the demo repository under
[`panel/`](https://github.com/FortunaTerra-Group/review-panel-demo/tree/main/panel). Everything
there is Claude Opus 5 output from 2026-09-12 and is labelled as such; another model, or another
day, will word it differently and may weigh a finding differently. What should reproduce is the
shape: independent lenses, probes against the code, refutation before reporting, and findings
that name a file and a line.
