---
name: countwerk
description: Durable counters and distinct counts for agents over plain HTTP. Use when an agent needs a spend cap, usage budget, step limit, hard deadline, blast-radius stop, a shared counter across parallel workers, a counter that survives restarts and context compaction, distinct counting, or coverage tracking. Counters are increment-only with caps set once at creation, and live outside the agent, so limits cannot be unwound or raised by the agent itself.
---

# countwerk: counters your agent can't reset

A number on the internet. Your agents bump it for 3 millionths of a
dollar. Only you can turn it back.

## When to use it (and when not)

Use countwerk when the number is SHARED across workers, must OUTLIVE
the run, or must sit OUTSIDE the agent's reach. If none of those hold,
use a local variable. This service earns nothing being a worse
variable.

Counters are increment-only. Nothing below the admin token moves a
counter down, so a looping or prompt-injected agent cannot unwind its
own limits with any token it holds. Need a gauge? Use two counters
(`x.opened` / `x.closed`) and subtract; one `read` returns both.

## Setup

Expected in the environment:

- `COUNTWERK_TOKEN`: the op token (`cw_o_...`)
- `COUNTWERK_NS`: the namespace name
- `COUNTWERK_URL`: optional, default `https://countwerk.ai`

If these are absent, tell the operator to fund a namespace. Funding IS
signup, there is no account flow:

```bash
curl -X POST $COUNTWERK_URL/v1/<ns>/fund          # -> 402 with x402 accepts
# sign with any x402 client (e.g. @x402/fetch), retry with X-PAYMENT:
curl -X POST $COUNTWERK_URL/v1/<ns>/fund -H "X-PAYMENT: <signed>"
# -> {balance_micro, token, admin_token}   tokens returned exactly once
```

Dev/testing: a server running with `DEV_MODE=1` accepts
`-H 'x-payment: dev:<micro_usd>'`. Comp keys credit without payment
(`x-comp-key` + `x-comp-amount`), ask the operator.

NEVER handle the admin token (`cw_a_...`). It stays in the operator's
deploy secrets. This skill uses the op token only.

## The five key shapes

Names allow `[a-z0-9._-]`, join parts with dots. All routes need
`-H "Authorization: Bearer $COUNTWERK_TOKEN"` (omitted below for
brevity) and are rooted at `$COUNTWERK_URL/v1/$COUNTWERK_NS`.

1. Spend cap, booked per model call, with a bound the run can never raise:

```bash
curl -sX POST .../count/spend-usd-cents/incr -d '{"by":142,"limit":10000}'
# -> {"name":"spend-usd-cents","value":9986,"limit":10000}
# limit applies ONLY on the counter's first write; ignored afterward.
# past the cap the SERVER answers 409 frozen: your loop cannot argue.
```

2. Step limit with a deadline the run can never move:

```bash
curl -sX POST .../count/steps.run-42/incr -d '{"ttl_seconds":3600}'
# first write sets the deadline; when the window is up the counter
# seals itself read-only (writes 409 frozen, reads still work).
```

3. Fleet budget, 50 workers share one number with zero coordination:

```bash
curl -sX POST .../count/fetches.example.com/incr
# each worker increments before fetching and backs off past the budget.
# honesty: check-then-incr can briefly overshoot under concurrency;
# this is a budget signal, not a hard rate limiter.
```

4. Blast radius, N outward actions per day, hard stop after:

```bash
curl -sX POST .../count/emails.2026-07-04/incr -d '{"limit":25}'
# seal-on-reach: the increment that crosses the limit still applies;
# the NEXT write gets 409 frozen. only the operator's admin token
# moves a bound.
```

5. Coverage, distinct counting across shards, merged at read:

```bash
curl -sX POST .../dcount/seen.shard-3/add -d '{"items":["url1","url2"]}'
curl -sX POST .../dcounts/merge -d '{"dest":"seen.all","sources":["seen.shard-3"]}'
# answers "how many distinct", never "have I seen this one"
```

## The guard loop

Increment first, then compare, then halt. The response carries the new
value in the same round trip, so the guard is one call plus one
compare:

```bash
v=$(curl -sX POST .../count/spend-usd-cents/incr -d "{\"by\":$COST}" | jq -r .value)
[ "$v" -ge "$CAP" ] && exit 42   # cap reached: stop the loop
```

The compare is advisory; for a cap the loop cannot ignore, set `limit`
on the counter's first write and the server enforces it (409 frozen).

## Discipline

- Retries are at-least-once: countwerk does NOT deduplicate, and an
  `Idempotency-Key` header is ignored. A failed (non-2xx) call did not
  apply, so retrying on `retryable: true` is safe. A TIMEOUT is
  ambiguous: read the counter before re-sending. Duplicates round the
  count up, never past a cap, so do not invoice from these numbers;
  bill from your ledger, cap from countwerk.
- The `retryable` field is authoritative, not the status class. Some
  5xx are permanent (`storage_degraded`, `dcount_corrupt`,
  `snapshot_corrupt`) and retrying only re-bills the failed call.
- Batch-first: prefer `counts/incr` (max 1000), `dcounts/add`
  (max 100 / 10k items), and `read` (both kinds, one call, max 1000).
- Prices: never hardcode. `GET $COUNTWERK_URL/pricing` is
  machine-readable and authoritative; quote it, budget from it.

## Errors you will actually see

| status | error | what to do |
|---|---|---|
| 402 | `insufficient_balance` | tell the operator to fund; body has `balance_micro` and `required_micro` |
| 400 | `value_out_of_range` | counters are increment-only, `by` is 1..2^53-1 |
| 400 | `bucket_full` | too many counters share one name-hash bucket; use another name |
| 409 | `frozen` | the counter hit its `limit` or `ttl_seconds`; only the operator's admin token moves a bound. reads still work and carry `frozen: true` |
| 409 | `namespace_archived` | the namespace is frozen read-only; the operator can hydrate |
| 503 | `namespace_busy` | a restore is running; retry |
| 503 | `storage_degraded` | permanent, `retryable: false`; operator-level cure |

Full contract: `$COUNTWERK_URL/llms.txt` serves the complete API as
markdown, with live prices.
