# Atomic budget admission for a public LLM agent

*November 2026 · SYNCRONIX public beta*

## The constraint

The research agent opened to the public with no sign-in, under hard limits: **$0.50 per day** in total,
**$0.20 maximum per run**, **one run at a time**, and **one run per visitor per day**. The cost of a run is known only
when it ends, and requests arrive concurrently.

## Why a simple check fails

A first version checked `spent_today < budget` before starting a run. Two failure modes:

1. **Runs in flight are invisible.** A run started at $0.45 spent can finish at $0.60. Two concurrent runs can both
   pass the check.
2. **In-process counters do not survive restarts** and do not coordinate between processes.

## Design

Every run is admitted in **one PostgreSQL transaction under a transaction-scoped advisory lock**:

```sql
begin;
select pg_advisory_xact_lock(hashtext('syncronix:admission'));
-- close runs stuck past their time limit, charging their full reserve
update runs set status='failed', cost_usd=coalesce(cost_usd, reserved_usd), reserved_usd=0
 where status='running' and created_at < now() - <time limit + margin>;
-- one read of everything the decision needs
select count(*) filter (where status='running')                       as active,
       sum(cost_usd)     filter (where created_at >= today)            as spent,
       sum(reserved_usd) filter (where status='running')               as reserved,
       count(*) filter (where created_at >= today and ip_hash=$1
                        and not is_admin and <counts toward quota>)    as visitor_runs
  from runs where created_at >= today or status='running';
-- refuse: busy | visitor limit | budget used up | budget held by runs in progress
insert into runs (..., status, reserved_usd) values (..., 'running', 0.20);
commit;
```

- **Reservations:** a run holds its $0.20 ceiling while active. A run may start only if
  `spent + reserved + 0.20 ≤ budget`.
- **Settlement:** when a run ends, one `UPDATE` records its real cost and clears its reserve, so unused reserve
  returns to the pool immediately.
- **Crash safety:** a run still marked running after its time limit is closed by the next admission and charged its
  full reserve; at process start, interrupted runs are closed the same way. The budget always errs on the safe side.
- **The per-run ceiling is enforced inside the agent:** a paused run is not resumed once its cost reaches the
  ceiling, and the optional read-more turn is skipped past half of it.
- **Visitor identity without storing IPs:** a keyed HMAC of the IPv4 address or IPv6 /64 prefix. IPv6 is grouped
  by /64 because one device or household controls a whole prefix.
- **Fairness:** a run that fails at ~$0 (e.g. a provider outage) does not use the visitor's daily run; a cancelled
  run that did research does. Free replays of the visitor's own saved results do not count.

## Evidence

Tests against a real PostgreSQL instance, using the production admission code:

| Scenario | Result |
|---|---|
| 25 simultaneous requests, 1 global slot | exactly 1 admitted, 24 refused as busy |
| 12 simultaneous requests, budget room for two $0.20 reserves | exactly 2 admitted; spend + reserves never exceeded the budget |
| 6 simultaneous requests from one visitor | exactly 1 admitted |
| Run settles below its reserve | unused reserve available to the next run immediately |
| Run stuck past its time limit | closed by the next admission and charged its full reserve |
| Admin key | bypasses the per-visitor limit only; still bound by concurrency and budget |
| 10 simultaneous HTTP requests from different visitors | 1 stream started, 9 × `503 busy` with `Retry-After` |

The HTTP tests ran against a separate server instance pointed at an unreachable AI endpoint, so admitted runs
failed immediately and no test spent money.

## Trade-offs

- A serialised admission is a throughput limit by design; with one active run allowed it is not a bottleneck.
- Reserving the full ceiling means the last ~$0.20 of a day can go unused.
- Shared networks share a daily run; an actor with many networks can exhaust the day's budget but never exceed it.
