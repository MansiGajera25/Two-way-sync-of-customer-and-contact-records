# Two-way sync: Salesforce ⇄ Portal (Postgres)

Customers and contacts. Fields in scope:
- **Customer:** name, status, owner
- **Contact:** full name, phone, email, city, owner, customer

Scale: 5 M records. Target: a change is visible on the other side within 30 s while both systems are up.

---

## 1. Assumptions and open questions

The questions marked ★ change a core rule. Ask these first.

| # | Question | Our choice | If the answer is the opposite |
|---|---|---|---|
| 1 | Peak / quiet edit rate? | Peak is 20 edits/s across both sides, of which 10/s go out to Salesforce. A quiet hour is about 1/s. | At 200/s or more, outbound traffic exceeds the cap. We would need composite batching, and 30 s would depend on queue depth. |
| 2 | Is the 100 rps cap ours alone? | Yes. We run at **80 rps**. That budget covers updates, retries and catch-up queries, and leaves 20 rps headroom. | A shared cap shrinks our budget, drain time grows in proportion, and we need a central limiter. |
| 3 | Does a composite call count as 1 request? | **Not assumed.** All math uses one record per call. Composite is an optional speed-up only. | If confirmed, the drain falls from about 51 min to under 1 min (§10). |
| 4 | Webhook carries changed fields only, or the full record? | Changed fields only (as in the sample). | Full record: we diff each field against `sf_value`. Only fields that differ count as changed. |
| 5 | Is the `eventId` the same on redelivery? | Yes. We dedupe on it. | We dedupe on `(recordId, occurredAt, hash(changedFields))` instead. |
| 6 ★ | Does the webhook carry a record version (`SystemModstamp`)? | No. We order by `occurredAt` and break ties as in §7. We **ask to add `sfModstamp`** so webhooks and catch-up rows share one order source. | If it exists, we order Salesforce events by it. That is immune to clock skew. |
| 7 | Does Salesforce retry when we return a non-2xx? | Yes, for a limited window. We return 2xx only after commit. | With fire-and-forget delivery, the periodic catch-up poll (§10) becomes our only safety net for lost webhooks. |
| 8 | How is the webhook authenticated? | HMAC-SHA256 over the body, plus a timestamp header. Reject if the timestamp is more than 5 min old. | Without signing, we need mTLS or a Salesforce IP allowlist. |
| 9 | Time and actor on a portal edit? | Time is the Postgres commit time (`now()` in the transaction). Actor comes from the authenticated session, never from the payload. | If we trusted the client clock, we would need skew bounds, and spoofing would be possible. |
| 10 | Can Salesforce time and our time be compared? | Yes. Both use NTP, with skew under 1 s. | If not, cross-side ordering uses arrival order at our database. Edits within the skew window are flagged for review. |
| 11 | Unit of merge? | **Field**, not record. | Record-level merging would lose concurrent edits to different fields. That breaks the brief. |
| 12 | Two portal users edit the same field? | The later commit wins. The loser's value goes to history and the loser gets a notice. Only the winning value goes to Salesforce. | If saves must be blocked instead, we need optimistic locking on the form and a 409 flow. |
| 13 ★ | Does our own API write trigger a webhook? Do Salesforce triggers or flows edit other fields in the same save? | Yes to both. Echo detection is **per field** (§9). We never drop a webhook just because it came from the integration user. | If Salesforce has no automation, the per-field check stays anyway because it is cheap. |
| 14 ★ | Can we query Salesforce for records changed since time T? | Yes, with SOQL on `SystemModstamp > T`, paged at 2,000 rows. **Those calls count against the cap.** (Assumed: this API is not in the brief.) This query returns only the **latest** full record, so it is used for verification only. History comes from #27. | If not, we need a full 5 M-record sweep (2,500 calls plus a diff of every row) or Change Data Capture replay (72 h retention). |
| 27 ★ | Can we read every intermediate Salesforce change, with the field and the user who made it? | Yes. The client enables **Field History Tracking** on our 9 fields (Account: name, status, owner. Contact: name, phone, email, mailing city, owner, AccountId). We query `ContactHistory` and `AccountHistory` (`ParentId, Field, OldValue, NewValue, CreatedById, CreatedDate`). Assumed: the API is not in the brief, and tracking must be switched on **before** go-live because it is not retroactive. | **Change Data Capture replay** (72 h retention, gives each event with its changed fields and user) does the same job. If neither is allowed, intermediate edits during an outage are truly lost. We would record a `gap` history row so the loss is visible, not silent, and tell the client this breaks "dropping history counts as losing a change". |
| 15 | During the outage, are webhooks down too? | Yes. They may never arrive. | If webhooks still flow, inbound needs no catch-up and only outbound drains. |
| 16 | Where do pending calls live? | pg-boss queue in our Postgres. The job is enqueued in the same transaction as the edit (§6). | With a separate broker, we need a transactional bridge, or we risk lost or doubled sends. |
| 17 ★ | Can we add an external-ID field `Portal_Id__c` in Salesforce? | Yes. Creates are upserts on it, so a retry never creates a duplicate. | Creates are not idempotent. We must look up by email before retrying, and duplicates are still possible. |
| 18 | Same person created on both sides? | No automatic merge. We flag likely duplicates by email. | Automatic merging needs matching rules plus a merge flow, which brings deletes into scope. |
| 19 | A contact arrives before its customer? | Store the contact with an unresolved `sf_customer_id`. Resolve it when the customer arrives. | Strict foreign keys mean parking the event and retrying it, which risks the 30 s target. |
| 20 | Salesforce owner Id → portal user? | Static mapping table. Store the raw Salesforce Id when there is no mapping. | If the owner must be a real portal user, the record blocks until an admin fixes the mapping. |
| 21 | Salesforce rejects an update (permanent 4xx)? | Revert to the last value Salesforce accepted, **only if no newer change exists** (version compare-and-set). History keeps the rejected value. The editor is notified. Details in §6. | If the client says to keep the portal value, the field is flagged `diverged` and goes on a nightly ops report for manual fixing. |
| 22 | How long is history kept? | Forever. The table is partitioned by month. | With bounded retention, "lost change" only holds within the window. |
| 23 | Initial 5 M backfill? | Out of scope. Assumed done through the Bulk API. | Otherwise it is a separate subproblem: 2,500 query pages plus an insert job. |
| 24 | Formatting differences (phone, email case)? | Normalise before any compare: phone to E.164, email to lowercase. | Without normalising, spurious conflicts appear and echoes go unmatched. |
| 25 | Does the webhook carry `Portal_Id__c`? | Yes. We add it to the payload, so the webhook for a record we just created matches our row even before we have processed our 2xx. | Without it, a create race can make a duplicate portal record. We would have to look up by `sf_id` after a short delay. |
| 26 | Does pg-boss 10 support the `stately` policy with `singletonKey`, `deadLetter`, and `send` inside our transaction (`db` option)? | Assumed. Confirm in the day-1 spike. | If `send` can't join our transaction, we keep a small outbox table plus a relay that moves rows into pg-boss. |

---

## 2. Systems, boundary, interfaces

```
 Sales user                              Finance / CS user
     |                                          |
     v                                          v
 Salesforce CRM                         Portal app (ours)
 (their DB)                                     |  savePortalEdit()
     |                                          v
     |                         +----------- Postgres (ours) ------------+
     |                         |  record, field_state, field_history   |
     |                         |  inbox_event, sf_send_log, sync_state |
     |                         |  pgboss.* (job queue)                 |
     |                         +---------------------------------------+
     |                              ^                       |
     |  (A) HTTPS webhook           | write                 | fetch pg-boss jobs
     |  POST /webhooks/salesforce   |                       v
     +----------------------> [ webhook handler ]    [ sync worker ]
     |                                                      |
     |  (B) HTTPS record API: upsert / update / query       |
     <------------------------------------------------------+
           (token bucket: 80 rps)
   ======================= public internet =======================
```

**Choice: one service with two process types.** Both processes come from the same codebase and share the same Postgres.
- **The webhook handler** must answer fast and must never wait on Salesforce. It scales with inbound traffic.
- **The sync worker** owns every outbound call and the rate limiter. It can crash and restart without losing work.
- **What crosses between them is Postgres rows only:** `field_state`, `sync_state`, and pg-boss jobs (`sf-sync`, data `{ portalId }`). They make no direct calls to each other. pg-boss lives in the same Postgres, so there is no second store to keep consistent.
- We run **one worker** for now, so the token bucket lives in that process. Several workers would need a shared limiter (out of scope, §12).

**Interfaces**

| Side | Interface |
|---|---|
| Salesforce → us | `POST /webhooks/salesforce`. Headers: `X-Signature` (HMAC-SHA256 of the body) and `X-Timestamp`. Body: as in the sample, plus an optional `sfModstamp` (question #6). Responses: `200` means applied or duplicate. `401` means a bad signature. `400` means a malformed body (Salesforce should not retry). `500` means a transient failure (Salesforce retries). |
| Portal UI → sync | `savePortalEdit(session, recordId, objectType, changedFields)`. Actor comes from `session`; time comes from the database. |
| Us → Salesforce (assumed standard REST) | **Create:** `PATCH /sobjects/{type}/Portal_Id__c/{portalId}` (upsert). **Update:** `PATCH /sobjects/{type}/{sfId}` with the changed fields of one record. **Catch-up:** `GET /query?q=SELECT … WHERE SystemModstamp > :t ORDER BY SystemModstamp`, paged at 2,000 rows. |

---

## 3. Subproblems in build order (riskiest first)

| # | Subproblem | Why this position |
|---|---|---|
| 1 | **Field-level state and the ordering rule** (`field_state`, `field_history`, the compare in §7) | Every other path calls it. It is schema, so it is the most expensive thing to change later. If it is wrong, data is lost silently. |
| 2 | **Spike: confirm the catch-up query** (question #14) the external-ID upsert (question #17), and the pg-boss features (#26) | These are external assumptions. If either is false, the outage and create designs change. Find out on day 1, not day 5. |
| 3 | **pg-boss queue, rate-limited worker, restart recovery, dead letter** | Needed for the 6-hour outage and the 100 rps cap. pg-boss supplies leases, retries and dead letter. We still own the token bucket, the circuit breaker, and the read of `value ≠ sf_value`. |
| 4 | **Inbound webhook:** verify, dedupe, apply | Straightforward once #1 exists. |
| 5 | **Echo matching** (loop rule, §9) | Needs both #3 (the send log) and #4 (the inbound path). |
| 6 | **Rejection handling** (4xx compare-and-set revert, §6) | Needs `field_state.version` and `sf_send_log`. |
| 7 | **Outage detection, catch-up, drain** | Builds on #2, #3 and #4. |
| 8 | **Creates and identity** (Salesforce Id ↔ portal Id, contacts before their customer) | Mostly mapping work, low risk. |
| 9 | **Metrics and alerts** (§11) | Last, but before going live. |

---

## 4. Data we store

```sql
-- Enum types. Adding a value later: ALTER TYPE … ADD VALUE (cannot remove one).
CREATE TYPE object_type     AS ENUM ('Customer', 'Contact');
CREATE TYPE change_source   AS ENUM ('portal', 'salesforce', 'sf_automation', 'catchup');
CREATE TYPE history_status  AS ENUM ('applied', 'superseded', 'echo', 'rejected', 'rejected_superseded', 'gap');
CREATE TYPE send_status     AS ENUM ('in_flight', 'done', 'rejected', 'failed');
CREATE TYPE sf_status       AS ENUM ('up', 'down');

-- Identity. Portal-created records have no sf_id until Salesforce confirms the create.
-- Every transaction that writes a record's state locks this row first (§7).
record (
  portal_id        uuid PK,
  sf_id            text UNIQUE NULL,
  object_type      object_type,
  sf_customer_id   text NULL        -- set on a Contact whose customer we have not seen yet (#19)
)

-- One row per (record, field): our current value, and what we believe Salesforce holds.
field_state (
  portal_id        uuid,
  field            text,
  value            text,           -- current value, normalised (#24)
  version          bigint,         -- +1 on every change to `value`; used for compare-and-set (§6)
  order_ts         timestamptz,    -- time of the edit that set `value`; used by the ordering rule (§7)
  order_tiebreak   text,           -- see §7
  actor            text,
  source           change_source,
  sf_value         text,           -- the value Salesforce holds, as far as we know
  sf_value_ts      timestamptz,    -- time of the Salesforce event (or our 2xx) that set sf_value
  diverged         boolean DEFAULT false,   -- set when the push guard trips (§9)
  PRIMARY KEY (portal_id, field)
)
-- A field needs sending when value IS DISTINCT FROM sf_value.

-- Every change ever seen, including ones that lost. Never updated except `status`. Partitioned by month.
field_history (
  id, portal_id, field, value, actor,
  source   change_source,
  order_ts, received_at, event_id NULL,
  status   history_status,
  error    text NULL
)

-- Dedupe ledger for webhooks and catch-up rows.
inbox_event (
  event_id     text PK,     -- webhook eventId, or 'cu:' || sfId || ':' || SystemModstamp for catch-up rows
  received_at  timestamptz,
  payload      jsonb
)

-- Every value we sent to Salesforce.
-- Used for echo matching (§9), the rejection compare-and-set (§6) and the push guard (§9).
sf_send_log (
  id             bigserial PK,
  portal_id      uuid,
  field          text,
  value          text,
  field_version  bigint,     -- the field_state.version that was sent
  sent_at        timestamptz,
  status         send_status,
  sf_response    jsonb NULL
)
INDEX sf_send_log (portal_id, field, sent_at)

-- Single-row table: typed columns instead of key/value text.
sync_state (
  id                 boolean PK DEFAULT true CHECK (id),
  sf_status          sf_status DEFAULT 'up',
  outage_start       timestamptz NULL,
  catchup_watermark  timestamptz
)
owner_map (sf_user_id PK, portal_user_id NULL)
user_notice (id, portal_user_id, portal_id, field, message, created_at, read_at)

-- pg-boss creates and owns its own schema (pgboss.*) in the same database.
-- We read it for metrics only and never write to it directly.
```

Size estimate: 5 M records × about 5 fields gives about 25 M `field_state` rows. That is fine for Postgres with the primary-key index.

---

## 5. Inbound path: webhook → database

The whole path runs in **one transaction**, and we return 200 only after commit.

1. **Verify** the HMAC and the timestamp (no more than 5 min old). On failure, return `401`.
2. **Validate** the body: `objectType`, fields within scope, a parseable `occurredAt`. On failure, return `400` and log it.
3. **Dedupe:** `INSERT INTO inbox_event … ON CONFLICT (event_id) DO NOTHING`. If no row was inserted, it is a duplicate, so return `200` and stop (§8).
4. **Resolve identity and lock:**
   - Find `record` by `sf_id`.
   - If there is none, find it by `Portal_Id__c` from the payload (#25) and store the `sf_id`. This covers the webhook for our own create arriving before we have processed our 2xx.
   - If there is still none, `INSERT … ON CONFLICT (sf_id) DO NOTHING`. This is a Salesforce-side create.
   - Then `SELECT … FROM record WHERE portal_id = $1 FOR UPDATE`.
5. **For each field** in `changedFields`:
   1. Normalise the value (#24).
   2. **Update the Salesforce value:** if `occurredAt > sf_value_ts`, set `sf_value` and `sf_value_ts`. Do this for every event, echoes included.
   3. If `changedBy` is the integration user, run the **echo check** (§9). If it matches, write history `echo`, set the send log row to `done`, and skip to the next field.
   4. Apply the **ordering rule** (§7) to `value`. If the incoming change is newer, update `value`, `version + 1`, `order_ts = occurredAt`, `source` and `actor`, then write history `applied`. If it is older, write history `superseded` and leave `value` alone.
6. **Enqueue only if needed:** if any field of the record now has `value ≠ sf_value`, call `boss.send('sf-sync', { portalId }, { singletonKey: portalId, db: tx })`. Normally the applied change makes the two equal and nothing is enqueued. This step only fires in the race in §7, case C.
7. Commit, then return `200`. On any exception, roll back and return `500`, so Salesforce retries. The retry is safe because the dedupe insert was rolled back too.

Latency: one round trip to the database. That is well inside 30 s.

---

## 6. Outbound path: portal edit → Salesforce

### Queue: pg-boss instead of our own outbox

| Concern | Own outbox table | pg-boss 10 |
|---|---|---|
| Durability | Postgres | Postgres (same database) |
| Enqueue in the edit's transaction | Yes | Yes, with `send(…, { db })` (#26) |
| Leases and crash recovery | Custom code | Built in (`expireInSeconds`) |
| Retries with backoff | Custom code | Built in (`retryLimit`, `retryDelay`, `retryBackoff`) |
| Dead letter | Custom code | Built in (`deadLetter` queue option) |
| One call per record at a time | Custom claim query | `singletonKey` with the `stately` policy: at most 1 active and 1 queued job per key |
| Cron for catch-up | Separate scheduler | Built in (`schedule`) |
| Global rate limit (rps) | Custom code | **Not provided.** We keep our token bucket. |
| Delay set by the response (429 `Retry-After`) | Easy | Not direct. We re-send with `startAfter` (below). |
| Merging repeated edits | Update the pending row | Not needed: the job carries only `portalId` |

**Decision: pg-boss.** It replaces the most error-prone custom code: leases, claim queries, backoff and dead letter. We still own three things: the token bucket, the circuit breaker, and deciding what to send.

**The job is a pointer, not a payload.** Job data is `{ portalId }`. When the handler runs, it reads `field_state` and sends every field where `value ≠ sf_value`. This is what makes pg-boss fit:
- When a job for the record is already queued, `send` returns `null` and nothing is lost, because the queued job reads the latest values.
- There is no stale check and no merging, because nothing in the job can go stale.

**Queues**

| Queue | Policy | Settings |
|---|---|---|
| `sf-sync` | `stately` | `singletonKey = portalId`, `retryLimit 10`, `retryDelay 2`, `retryBackoff true`, `expireInSeconds 60`, `deadLetter 'sf-sync-dead'`, retention 7 days |
| `sf-catchup` | `singleton` | cron every 15 min, plus one job on recovery. `retryLimit 5`, `deadLetter 'sf-catchup-dead'` |

Retry budget for `sf-sync`: 2 + 4 + … + 1,024 = 2,046 s, about 34 min, before a job goes to dead letter.

**Portal save** (one transaction):
1. Actor comes from the session. `ts = now()`.
2. Lock the `record` row (`FOR UPDATE`).
3. For each field: set `value`, `version + 1`, `order_ts = ts`, `source = 'portal'`, then write history `applied`.
4. `boss.send('sf-sync', { portalId }, { singletonKey: portalId, db: tx })`. A `null` return means a job is already queued, which is fine.
5. Commit.

**Job handler `sf-sync({ portalId })`:**
1. **Short transaction.** Lock the record row. Read the fields where `value IS DISTINCT FROM sf_value AND NOT diverged`. If there are none, complete the job. Otherwise insert one `sf_send_log` row per field (`in_flight`, `field_version`, `sent_at = now()`) and commit. **No lock is held during the HTTP call.**
2. Take one token from the bucket (80 rps).
3. Send **one** call: an upsert on `Portal_Id__c` if `sf_id` is null, otherwise an update with all those fields.
4. Handle the result:

| Result | Action |
|---|---|
| `2xx` | In a transaction, lock the record. For each sent field, mark the send log `done`, and set `sf_value` to the sent value and `sf_value_ts = now()` if that is newer than the current `sf_value_ts`. On a create, store `sf_id`. If any field still has `value ≠ sf_value` (it was edited during the call, or see §7 case C), send a new job. Complete. |
| `429` | Pause the bucket until `Retry-After`. Mark the send log `failed`. Send a new job with `startAfter` set to the `Retry-After` time and the same `singletonKey`. Complete. **This does not use up a retry.** |
| `5xx`, timeout, connection error | Mark the send log `failed` and feed the breaker. Throw, so pg-boss retries with backoff. |
| Permanent `4xx` | Apply the rejection rule below, then complete. A rejection is a handled outcome, not a job failure. |
| Handler bug or process crash | The handler throws or the process dies. pg-boss retries, and on a crash the job expires after 60 s first. After 10 failures the job moves to `sf-sync-dead`, and that pages someone (§11). |

**Circuit breaker:** while it is open (§10), call `offWork('sf-sync')`. Queued jobs stay in `created` or `retry`, no attempts are used, and nothing dead-letters during an outage.

**Restart:** pg-boss expires active jobs after 60 s and retries them. A resend is safe because an update just sets the same values again and a create is an upsert on `Portal_Id__c`.

**Rejection rule** (decision #21). Take `field_version` from the rejected `sf_send_log` row. For each rejected field, in one transaction:
```sql
UPDATE field_state
   SET value = sf_value, version = version + 1,
       source = 'salesforce', order_ts = now()
 WHERE portal_id = $1 AND field = $2 AND version = $sent_field_version;
```

| Rows updated | Meaning | Action |
|---|---|---|
| 1 | Nothing newer exists. | Revert done; now `value = sf_value`, so nothing is sent. History marks the edit `rejected`, with the Salesforce error. Notice to the editor: "Your change to City (Noida) was rejected by Salesforce: <reason>". |
| 0, newer portal edit | The user edited again. | Keep the newer value; the next job sends it. History marks the old edit `rejected_superseded`. Notice: "Earlier value X rejected: <reason>. Newer value Y is still being sent." If Y is rejected too, revert to `sf_value`, never to X. |
| 0, newer Salesforce change | Salesforce already wrote a newer value. | Nothing to revert. History marks the edit `rejected_superseded`, and the editor gets a notice. |

We compare versions, not values. A user can re-enter the exact rejected value as a new edit, and a value compare would wrongly revert it.

---

## 7. Rule when both sides change the same record

**Different fields:** both edits survive. State is per field, so they never touch the same row.

**Same field:** last writer wins on the order key. The loser is still written to history with status `superseded`.

```
order key = (order_ts, tiebreak)

order_ts:  webhook    → occurredAt  (or sfModstamp if #6 is answered yes)
           catch-up   → SystemModstamp
           portal     → Postgres now() at commit

tiebreak:  when order_ts is equal, the Salesforce source wins, then the higher eventId wins.
```

The incoming change applies only if its key is greater than the stored `(order_ts, order_tiebreak)`. This check runs under `FOR UPDATE` on the `field_state` row, so concurrent writers are serialised.

Why Salesforce wins a tie: it is the client's system of record for sales, and any deterministic rule is enough to give one current value.

**Two portal users:** both saves lock the same row. The later commit has the later `now()` and wins. The earlier user gets a `user_notice` (#12). Only the winning value goes out, because the job reads the current value when it runs (§6).

**Full-record webhook (if #4 flips):** only fields whose normalised value differs from `sf_value` go through this rule. The rest are ignored.

### Concurrency walkthrough: pg-boss plus the row lock

**Lock rule.** Every transaction that writes a record's state starts with `SELECT … FROM record WHERE portal_id = $1 FOR UPDATE`. That covers the portal save, the webhook, a catch-up row, handler steps 1 and 4, and a rejection. There is one lock per record and it is always taken first. Writers to the same record therefore run one at a time and can't deadlock on field order. The lock is held only for the transaction, a few milliseconds.

**Case A: different fields at the same moment.** A portal user edits city while a sales user edits phone.
1. The portal transaction locks the record.
2. The webhook transaction waits a few ms, then applies phone.
3. Both fields survive. The job sends city. Phone already equals `sf_value`, so it is not sent.

**Case B: same field, and both edits reach our database before we call Salesforce.**
1. The portal sets city to Noida at 10:00:02.
2. A webhook sets city to Pune with `occurredAt` 10:00:03.
3. The lock serialises the two transactions. In either commit order, §7 picks Pune, because 03 > 02.
4. Now `value = sf_value = Pune`, so the job finds nothing to send. Both sides hold Pune.

**Case C: race across the HTTP call, where the lock can't help.**
1. 10:00:02: the portal sets Noida. The job starts and sends Noida. No lock is held.
2. 10:00:03: a sales user sets Pune in Salesforce. Its webhook arrives. Pune (03) beats Noida (02), so `value = Pune` and `sf_value = Pune` (03).
3. 10:00:04: our Noida lands in Salesforce and overwrites Pune. The 2xx returns. We set `sf_value = Noida` (ts 04, newer than 03). Now `value` (Pune) ≠ `sf_value` (Noida), so we send a new job.
4. The next job sends Pune. Its echo matches the send log. Both sides now hold Pune, which is the edit the rule says should win.

If the webhook from step 2 arrives *after* our 2xx, we end in the same place. The 2xx sets `sf_value = Noida` (04). The late webhook is older (03), so `sf_value` stays Noida, but `value` becomes Pune, and we push Pune. If our clock was off, the echo of Noida carries Salesforce's own `occurredAt` and corrects `sf_value_ts`.

**Is the row lock enough? Yes. We don't need an advisory lock.**
- The row lock covers every write to our database, and every update targets an existing `record` row.
- Creates are covered by constraints, not locks. The unique `record.sf_id` with `INSERT … ON CONFLICT` handles two webhooks for the same new record. `Portal_Id__c` (#25) handles the webhook for our own create arriving before our 2xx.
- The only gap is across the HTTP call. Holding any lock across that call, advisory or row, would tie up a database connection until the timeout and block portal saves while Salesforce is slow. That is exactly wrong during an outage. Two other mechanisms close the gap: `singletonKey`, which allows one call per record at a time, and the `value ≠ sf_value` check (case C).


---

## 8. Rule for duplicate and out-of-order webhooks

**Duplicate:** `inbox_event.event_id` is the primary key. A second delivery hits `ON CONFLICT DO NOTHING` and gets `200`, and nothing else happens. Because the insert sits in the same transaction as the apply, a crash midway leaves neither, and Salesforce's retry processes it cleanly. Catch-up rows use the synthetic id `cu:{sfId}:{SystemModstamp}`, so a webhook and a catch-up row for the same save are both processed. That is safe, because the second one loses the ordering rule (its key is equal, not greater) and is recorded only as history.

**Older webhook after a newer one:** the per-field compare in §7 rejects it, because `occurredAt` is not greater than the stored `order_ts`. It is still written to `field_history` as `superseded`, with its real `occurredAt`, so history sorted by `order_ts` shows the true order of edits. It never overwrites the newer value.

Example: the phone is edited at 11:30:04 (event A) and at 11:30:09 (event B). B arrives first and is applied. A arrives afterwards, 04 < 09, so A is written to history as `superseded` and the phone stays at B.

---

## 9. Rule that stops the loop

**Rule 1 (the guarantee): send only differences.** The worker sends a field only when `value ≠ sf_value`. Every event from Salesforce updates `sf_value`: a normal webhook, an echo, or a catch-up row. Therefore:
- When we apply a Salesforce change, `value = sf_value` afterwards, so nothing is sent back.
- Our own echo makes `sf_value` equal to what we sent, so nothing more is sent.
- Only a real difference produces a call: a portal edit, or the race in §7 case C. That call removes the difference.

*This replaces the earlier rule "inbound changes never enqueue".* That rule could not repair case C, where Salesforce ends up holding our older value.

**Rule 2 (correct history): echo check per field.** This runs only when `changedBy` is the integration user. The webhook may be our own echo, or it may contain changes Salesforce automation (triggers or flows) made in the same save.

```
match = an sf_send_log row with status 'done' or 'in_flight',
        same portal_id and field, same normalised value,
        sent_at within the last 10 min
```

- **Match:** it is an echo. Write history `echo` and leave `value` alone. The portal actor stays the author of record. (`sf_value` was already updated in §5, step 5.2.)
- **No match:** it is a real change made by automation. Apply it through §7 with `source = 'sf_automation'` and actor `salesforce-automation (via integration user)`.
- **Salesforce reformatted our value** (for example the phone format): there is no match, so the value is applied as an automation change. Its `occurredAt` is newer, so `value = sf_value` afterwards and nothing is sent. The loop ends after one step.

Webhooks from any other user skip the echo check and go straight to §7.

**Rule 3 (safety guard).** If `sf_send_log` has more than 3 sends for the same `(portal_id, field)` within 10 min:
- set `diverged = true`,
- stop sending that field (the handler skips it),
- raise an alert (§11).

This catches bugs, such as a mismatch between our normaliser and Salesforce's, before they burn the rate limit forever.

---

## 10. The 6-hour Salesforce outage

**Detection:** a circuit breaker in the worker. After 10 consecutive connection errors, timeouts or 5xx responses, set `sync_state.sf_status = 'down'` and set `outage_start` to the time of the first failure, and call `offWork('sf-sync')`. While Salesforce is down, the worker sends one probe call every 30 s, using one token. The first `2xx` sets the status to `up`, sends one `sf-catchup` job, and resumes `sf-sync` work after that job completes.

**What is on disk during the outage:**
- pg-boss holds at most one queued `sf-sync` job per edited record (`stately` policy). The values themselves are in `field_state`, where `value ≠ sf_value`. Both are in Postgres and survive a restart.
- The worker is not fetching jobs, so no job uses up its retries and nothing dead-letters during the outage.
- `field_state` and `field_history` keep accepting portal edits as normal.
- Webhooks: Salesforce cannot deliver them. Anything lost there is recovered by catch-up.

**Drain on recovery, in this order:**
1. **Catch-up first, in three steps** (#27). The window starts at `outage_start − 5 min`, an overlap margin.
   1. **Field history.** Query `ContactHistory` and `AccountHistory` with `CreatedDate > window`, `ORDER BY CreatedDate, Id`, paged at 2,000 rows. **Each history row is one field change, with its user and time.** Turn each row into a single-field event and run it through the inbound path (§5):
      - `event_id = 'h:' || history Id`. This is stable, so a re-run of catch-up dedupes on it.
      - `changedFields = { field: NewValue }`. The Salesforce field name is mapped to ours (for example `MailingCity` → `city`).
      - `occurredAt = CreatedDate`, `changedBy` = the `CreatedById` user.
      - The echo check (§9) applies as normal, so our own writes found in history are recorded as `echo`.

      Three changes to phone become three history rows in order. Two end as `superseded` and the last as `applied`. Nothing is lost.
   2. **Users.** Collect the distinct `CreatedById` values and resolve them to emails with one `SELECT Id, Email FROM User WHERE Id IN (…)` query. Expect a few hundred users at most, so 1 call.
   3. **Verify with the full record.** Query `SystemModstamp > window`. This step does not create history. For each field, compare Salesforce's value with our `sf_value`:
      - Equal: nothing to do.
      - Different: history missed something, such as a field that isn't tracked or a create. Apply the value with `source = 'catchup'` and actor `unknown (from LastModifiedById)`, and write a `gap` history row so the hole is visible. Count these rows as a metric. Normally the count is 0.
   - **Duplicates across sources.** A change can arrive both as a webhook and as a history row, with different ids. History `CreatedDate` has only second precision. Before writing a history-row event, skip it if `field_history` already holds the same `(portal_id, field, value, actor)` with `order_ts` in the same second.
   - **Order within one second.** Use history `Id` order. Assumed close enough, because ties within one second on one field are rare.

   All of this puts Salesforce's values in place **before** we push portal edits, so §7 decides each conflict correctly.
2. **Resume `sf-sync` work.** Each job sends only fields where `value ≠ sf_value`. A portal edit that lost to a newer Salesforce value in step 1 is therefore never sent. Everything else is sent at the bucket rate.
3. Every call in steps 1 and 2 uses the same 80 rps bucket, and catch-up has priority.

**Arithmetic (one record per call, worst case: peak traffic for all 6 h, no merging of repeated edits):**

```
Outage length       6 h × 3,600 s            = 21,600 s

Outbound backlog    10 edits/s × 21,600 s     = 216,000 calls
Catch-up: history   10 SF field changes/s × 21,600 s = 216,000 rows
                    216,000 ÷ 2,000 rows/page        = 108 calls
Catch-up: users     1 IN-query                       = 1 call
Catch-up: verify    ≤ 216,000 changed records ÷ 2,000 = 108 calls
Total calls         216,000 + 108 + 1 + 108          = 216,217

Budget              80 rps (our run rate)
  minus live load   − 10 rps (new edits keep coming)
  net for backlog   = 70 rps

Drain time          216,217 ÷ 70              = 3,088.8 s
                    3,088.8 ÷ 60              ≈ 51.5 min
```

Two things make this smaller in practice: one call per record carries all of its changed fields, and fields superseded by Salesforce are skipped. 51.5 min is the upper bound.

*Optional, only if Salesforce confirms a composite call of 200 records counts as one request:* 216,000 ÷ 200 = 1,080 calls. (1,080 + 217) ÷ 70 ≈ 19 s.

**Lost webhooks while Salesforce is up:** the same catch-up query also runs every **15 min** (pg-boss `schedule` on `sf-catchup`) from `catchup_watermark`. It runs the same three steps. At 10 edits/s that is 9,000 rows ÷ 2,000 = 5 history calls, plus 1 user call and 5 verify calls, so 11 calls per run, which is small. It covers webhooks Salesforce dropped outside an outage.

---

## 11. Numbers to watch

The three core numbers:

| Metric | Why | Page someone when |
|---|---|---|
| **Age of the oldest queued `sf-sync` job** (`pgboss.job` in state `created` or `retry`, by `created_on`) | The direct measure of the 30 s promise on the outbound side. It also fires if the worker process is dead, because then no job runs. | Over **30 s for 5 min** while `sf_status = up`. During an outage, alert when the projected drain (queued jobs ÷ 70) goes over **60 min**. |
| **Inbound apply lag, p99** (`commit time − occurredAt`) | Shows slow webhooks, missing webhooks, or a backed-up database. | p99 over **30 s for 5 min**, or **no webhooks for 15 min** during business hours. |
| **Salesforce rejections (permanent 4xx) per hour** | Each one is a reverted user edit. A spike means a new validation rule or a mapping bug. | Over **50 per hour**, or any 4xx on a create. |

Failure alerts, which catch jobs that fail for good and handlers that crash:

| Metric | Why | Page someone when |
|---|---|---|
| **Dead-letter jobs** (size of `sf-sync-dead` and `sf-catchup-dead`) | A job failed all its retries, or the handler kept crashing on it. Its record stays out of sync until someone acts. | **Any job (> 0).** Runbook: read the job's last error, fix the cause, then re-send `{ portalId }` to `sf-sync`. |
| **Webhook 5xx rate** | The inbound handler is failing. Salesforce retries, but it gives up eventually. | Over **1% for 5 min**. |
| **Diverged fields** (`field_state.diverged = true`) | The loop guard tripped (§9). The field is no longer being synced. | **Any (> 0).** |

---

## 12. What we are leaving out

- Deletes, and merging duplicate records (#18).
- Salesforce login and OAuth flows. Credentials are assumed to be configured.
- Any visual UI. `user_notice` is the data only.
- A second region, and a load test against real Salesforce.
- The initial 5 M backfill (#23).
- Composite or Bulk API batching. It is designed for, but not built (#3).
- More than one sync worker. That needs a shared rate limiter.
- Archiving `field_history`. It is kept forever and partitioned (#22).
- An admin UI for `owner_map` and for resolving `diverged` fields.

---

## 13. Tests

**Harness:**
- **Fake Salesforce:** an in-process HTTP server. It can:
  - store records,
  - switch between up and down,
  - return 429, 4xx or 5xx on demand,
  - hold a response until the test releases it,
  - count and timestamp every call,
  - post webhooks to our endpoint, including echoes from the integration user.
- **Database:** a real test Postgres with pg-boss.
- **Cap:** each test sets its own rate cap.

### Duplicate webhook

| # | Steps | Expect |
|---|---|---|
| D1 | Post the same `eventId` twice. | Both return 200. One `inbox_event` row, one history row `applied`. |
| D2 | Post the same `eventId` twice, in parallel. | Same as D1. No error. |
| D3 | Force an exception after the dedupe insert, then redeliver. | The first attempt returns 500 and leaves no rows. The redelivery applies once. |

### Out of order

| # | Steps | Expect |
|---|---|---|
| O1 | Phone event B (11:30:09), then event A (11:30:04). | Value is B. A is in history as `superseded`. History sorted by `order_ts` shows A before B. |
| O2 | Two events with equal `occurredAt`. | The winner follows the tiebreak and is the same on every run. |
| O3 | An old event on a field that no newer event has touched. | It is applied. Ordering is per field, not per record. |

### Echo not sent back

| # | Steps | Expect |
|---|---|---|
| E1 | Portal edits city. Fake Salesforce posts the echo from the integration user. | Fake Salesforce received exactly 1 call. History `echo`. The actor is still the portal user. No new job. |
| E2 | The echo also contains a field changed by automation. | The automation field is applied with `source = 'sf_automation'` and is not sent back. Call count is still 1. |
| E3 | Fake Salesforce reformats the phone in its echo. | The reformatted value is applied once. No further calls. |
| E4 | Fake Salesforce always rewrites a field to a value our normaliser treats as different. | Sending stops after 3 sends. `diverged = true`. The alert fires. |
| E5 | Sales edits phone (a normal webhook). | Zero calls to fake Salesforce. |

### Restart while Salesforce is down

| # | Steps | Expect |
|---|---|---|
| R1 | Fake Salesforce goes down. Portal edits 100 records. Kill the worker, restart it, keep Salesforce down. | 100 jobs are still queued. The breaker is open. No job is in dead letter. |
| R2 | Continue R1: bring Salesforce up. | The catch-up job runs before the first update call. All 100 records are delivered. Both sides are equal. |
| R3 | Kill the worker after a call is sent but before the 2xx is processed. | The job runs again after 60 s. The resend is harmless. State ends consistent. |
| R4 | During the outage, sales edits the same field a portal user edited later, but the sales edit is newer (it reaches us through catch-up). | The portal value is never sent. Both sides hold the sales value. |
| R5 | During the outage, sales changes phone 3 times. Fake Salesforce holds 3 history rows. Bring Salesforce up. | `field_history` has all 3 changes, each with the sales user and `CreatedDate`. Two are `superseded` and the last is `applied`. |
| R6 | Run catch-up twice over the same window. | No new history rows on the second run (dedupe on `h:` id). |
| R7 | One change was delivered as a webhook and also appears as a history row. | One history row, not two. |
| R8 | Fake Salesforce changes a field with no history row (tracking off). | The verify step applies the value and writes a `gap` row. The gap metric is 1. |

### Concurrent edits to the same record

| # | Steps | Expect |
|---|---|---|
| C1 | In parallel: portal edits city, webhook edits phone. | Both survive. Exactly one call, carrying city. |
| C2 | In parallel: portal sets city (02), webhook sets city (03). | Value is the webhook's. No call. |
| C3 | Case C from §7: hold our Noida call, post the Pune webhook (03), then release. | Fake Salesforce receives 2 calls (Noida, then Pune). Both sides end on Pune. |
| C4 | Two portal users save the same field in parallel. | The later commit wins. The loser gets a notice. Salesforce ends on the winning value. |
| C5 | 50 rapid edits to one record. | Fake Salesforce never sees more than 1 concurrent call for that record. The final values match. |

### Rate limit

| # | Steps | Expect |
|---|---|---|
| L1 | Cap set to 5 rps. Edit 50 records. | Fake Salesforce sees no more than 5 calls in any 1 s window. All 50 records are delivered in about 10 s. |
| L2 | Cap 5 rps, with a catch-up job and retries running at the same time. | Catch-up and retry calls count inside the same 5 rps. |
| L3 | Fake Salesforce returns 429 with `Retry-After: 3`. | No calls for 3 s. The job is delivered afterwards. Its retry count is unchanged and nothing is dead-lettered. |

### Failed job alert

| # | Steps | Expect |
|---|---|---|
| F1 | Fake Salesforce returns 500 for one record only, with `retryLimit` set to 2. | The job lands in `sf-sync-dead` and the alert fires. Other records still sync and the breaker stays closed. |
| F2 | The handler throws a bug on one record. | Retries are used up, the job goes to dead letter, and the alert fires. |
| F3 | The worker process crashes while it is handling a job. | The job expires and is retried. Repeated crashes put it in dead letter and the alert fires. |
| F4 | Fake Salesforce returns a permanent 400. | **No** dead letter. The rejection rule runs (revert and notice), and the rejection count goes up. |
| F5 | The webhook handler returns 5xx for more than 1% of requests over 5 min. | The webhook 5xx alert fires. |
