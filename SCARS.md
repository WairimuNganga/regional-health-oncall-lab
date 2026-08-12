# 🩹 Scar Log — Regional Health On-Call Lab

One entry per incident. Each is meant to be read in under a minute — this is what the next on-call engineer should see at 2am, not the full investigation.

---

## OPS-2201 — Missing index + unbounded search payload

- **S — Symptom:** `GET /api/patients/search?lastName=Smith` p95 hit **20.92s** under 200 concurrent shift-change searches (SLO: <300ms). Throughput collapsed from a 49.19 req/s baseline to **11.04 req/s**.
- **C — Cause:** Two stacked bugs, not one. First, no index on `last_name` forced a full table scan of all 100,000 rows per search. Second — and this is the one that actually mattered under load — even after indexing, the endpoint still returned every matching row (up to ~10,000) with every column, including a large `notes TEXT` field, as a single ~3.57MB JSON response. Serializing that much JSON 200 times concurrently saturated Node's single-threaded event loop.
- **A — Action:** `CREATE INDEX idx_patients_last_name ON patients(last_name);` **and** changed the query to an explicit column list with `LIMIT 100`.
- **R — Result:** p95 **20.92s → 277ms** (~76x). Throughput **11.04 → ~1000.9 req/s** (~91x). The index alone made p95 *worse* (29.71s) — only the combination fixed it.
- **Scar / lesson:** A cheaper query plan doesn't guarantee a fast endpoint — always check response payload size too, not just `EXPLAIN` output. The alert that would've caught this: a p95-latency alert scoped to this route, plus a pre-deploy `EXPLAIN` check on any new `WHERE` clause against an unindexed column.
- **Evidence:** `LAB_JOURNAL.md` § OPS-2201 (full k6 + `EXPLAIN ANALYZE` output, both fix attempts); index migration + `server.js` diff in commit history.

---

## OPS-2202 — Connection pool exhaustion, then an unbounded queue

- **S — Symptom:** `GET /api/patients/recent` p95 hit **3.86s** (vs. 67.51ms baseline, ~57x worse) under a 0→2,000-VU surge — with **0% errors** and MySQL `Threads_connected` flat at 3 the entire time, matching the ticket's "DB looks idle" report exactly.
- **C — Cause:** `connectionLimit: 2` in `database.js` meant nearly every request queued inside the Node process waiting for one of only 2 connections — the database was never actually under load. After raising the pool, a second bug appeared: `queueLimit: 0` meant that queue was unbounded, so excess requests hung silently for up to 30s instead of failing.
- **A — Action:** `connectionLimit` 2 → 20 (sized via Little's Law: L = λ×W ≈ 11-15, doubled for headroom), then `queueLimit` 0 → 50.
- **R — Result:** Median response time **2.32s → 99ms**. Failures now return in ~100ms with a clear error instead of hanging. Caveat, documented not hidden: at true extreme overload, successful throughput actually *dropped* (858/s → 243/s) because the test script retries in a tight loop with no backoff, turning fast rejections into a self-inflicted retry storm.
- **Scar / lesson:** "Connections in use pinned at max while latency climbs" is the early-warning signal for this entire class of incident — worth a permanent alert, not just an incident-response check. Also: bounding a queue makes failures honest, it doesn't manufacture capacity — client-side backoff is still needed to get the full benefit.
- **Evidence:** `LAB_JOURNAL.md` § OPS-2202 (k6 output for both attempts, `Threads_connected` and `docker stats` traces, `db_errors_total` cross-check); `database.js` diff.

---

## OPS-2203 — Row lock held across a slow external call

- **S — Symptom:** `POST /api/hospitals/1/admit` failed for **99.91%** of requests under 500 concurrent admissions to the same hospital. Successful requests took a median of **20.35s** (max 46.07s) — one-at-a-time worked fine.
- **C — Cause:** The admit handler ran `UPDATE`, then a simulated 500ms external `notifyBedRegistry()` call, then `COMMIT` — holding the row's exclusive lock for the full 500ms instead of just the write itself. Directly confirmed via `performance_schema.data_locks`: one transaction `GRANTED`, 17 others `WAITING` on the identical row. After fixing that, the pooled connection was found to be held the same way. After fixing *that* too, throughput still plateaued — because a single row can only ever be written by one transaction at a time, a structural limit no code change removes.
- **A — Action:** Moved `notifyBedRegistry()` to run after `conn.commit()`. Then also added `conn.release()` immediately after commit, before the notify call.
- **R — Result:** Successful throughput **~1.6/s → ~8.6/s → ~8.1/s**. `Innodb_row_lock_waits: 821` (avg 2,336ms) confirmed real contention persists even after both fixes — this is the row's honest ceiling, not a leftover bug.
- **Scar / lesson:** Shortening a transaction has a hard limit once you hit single-row write contention under extreme concurrency — the real fix for sustained high-volume writes to one row lives *above* the transaction (a queue, sharded counters, eventual consistency), not inside it. Alert: `Innodb_row_lock_time_avg`/`Innodb_row_lock_waits` exceeding baseline, plus a long-running-transaction detector (>200ms open) — a good permanent dashboard panel, not just an incident check.
- **Evidence:** `LAB_JOURNAL.md` § OPS-2203 (direct `data_locks` + `SHOW ENGINE INNODB STATUS` output, `Innodb_row_lock%` aggregate stats, k6 output for both fix attempts); lock-queue diagram; `server.js` diff.

---

## OPS-2204 — Unbounded export memory, then missing backpressure

- **S — Symptom:** `capacity-api` container crashed and restarted **11 times in a single 2-minute test**, memory hitting the 160MiB limit within ~12-13 seconds of load starting every time. `checks_succeeded: 0%` — nothing ever completed.
- **C — Cause:** `/api/patients/export` loaded the entire ~100,000-row table into one array and one `res.json()` call — O(N) memory with no cap, guaranteed to exceed the container's 160MB limit under concurrent callers. After switching to streaming, a second bug surfaced: `res.write()`'s return value was never checked, so unsent data could still pile up in Node's internal write buffer for any slow client, just relocated from "one giant array" to "the write buffer."
- **A — Action:** Rewrote the handler to stream rows to the response as they arrive from MySQL. Then added backpressure handling: `queryStream.pause()` whenever `res.write()` returns `false`, `resume()` on the response's `'drain'` event.
- **R — Result:** `RestartCount` **11 → 0**, `OOMKilled: false`, `checks_succeeded` **0% → 100%** (117/117), peak memory **160MiB (crash) → ~110MB peak, settling ~90MB**. Cost: median export time is now 51.6s (up from crashing) — the correct trade-off for a bulk export/ETL endpoint.
- **Scar / lesson:** Streaming alone isn't enough to bound memory — it has to respect backpressure, or the same unbounded-growth problem just moves to a different buffer. Alert: any `RestartCount` increment on this container is inherently abnormal and actionable — faster and more reliable than trying to catch a ~12-second memory ramp before it OOMs.
- **Evidence:** `LAB_JOURNAL.md` § OPS-2204 (memory traces for all three stages, `docker inspect` `RestartCount`/`OOMKilled` confirmations, k6 output); `server.js` diff.