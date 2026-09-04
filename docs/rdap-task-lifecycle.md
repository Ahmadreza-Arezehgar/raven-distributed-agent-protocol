# RDAP task / auth / cancellation lifecycle

> **Status:** Sprint 0 freeze draft (O5) — Role #13 RDAP Protocol. Describes
> intended/current semantics from code; open conformance gaps are NOT claimed
> correct.

This document freezes the **current** RDAP task, transport-auth, delegation,
replay, ownership, and cancellation semantics as implemented in
`team_agents/{task_store,server,executor,client,raven_identity,config}.py` and
summarized in the repository README. It is a protocol-facing note for Sprint 0
Objective O5. It is **not** a claim that every described path is
interoperable-correct, and it is **not** last-message-wins: a later updater
event or signed answer artifact MUST NOT silently overwrite an independently
persisted Cancel terminal without an explicit, owned transition.

Runtime patches are **out of scope** for this document. Open gaps, including
cancel status skew, are recorded so Role #14 (Python Runtime) can close them
later.

Sources checked for this freeze: `team_agents/task_store.py`,
`team_agents/server.py`, `team_agents/executor.py`, `team_agents/client.py`,
`team_agents/raven_identity.py`, `team_agents/config.py`, `team_agents/llm.py`,
`team_agents/relay.py`, `team_agents/__main__.py`, `team_agents/selftest.py`,
and `README.md`.

---

## 1. Normative definitions

The keywords MUST, MUST NOT, SHOULD, and MAY describe intended current
behavior as read from the code. Where the implementation and the intended
end-to-end contract disagree, the disagreement is listed under
[Open conformance gaps](#9-open-conformance-gaps) and is **not** normative
“already correct.”

| Term | Meaning in this freeze |
|---|---|
| **Transport owner / `raven_owner`** | Raven address (or `open:<ip>`) written onto the ASGI request by `RpcIngressLimitMiddleware` after `RavenRequestAuthenticator`, then copied onto `ServerCallContext.user` / `.tenant` by `RavenServerCallContextBuilder`. |
| **Authenticated principal** | A `RavenPeerUser` whose `user_name` does **not** start with `open:`. Open-mode unsigned callers are unauthenticated. |
| **Delegation metadata** | The `raven.*` fields on an A2A message (`sender`, `recipient`, `task_id`, `kind`, `issued_at`, `expires_at`, `signature`, `algorithm`, `context`, `nonce`). |
| **Owner-scoped store key** | The pair `(owner, task_id)` in `BoundedTaskStore`. The same A2A `task_id` MAY exist independently for two owners. |
| **Terminal states** | `COMPLETED`, `CANCELED`, `FAILED`, `REJECTED` (`TaskState.TASK_STATE_*`). |
| **Active (non-terminal)** | Any stored task whose status is not in the terminal set (typically `SUBMITTED` or `WORKING`). |
| **Force-save CANCELED** | `RavenRequestHandler.on_cancel_task` overwriting the owner-scoped store row to `TASK_STATE_CANCELED` when the SDK-returned task is not already canceled. See [§5](#5-cancel-fsm) and [§9.1](#91-critical--cancel-status-skew-open-o5-conformance-gap). |
| **`CANCELLED` string** | Literal brain return value (`OpenAIBrain` / `EchoBrain`) when `cancel_event` is set. This is **not** an A2A `TaskState`. |
| **Peek vs consume** | `verify_delegation(..., consume_replay=False)` verifies the signature without inserting it into the replay cache; `consume_replay=True` (default) calls `ReplayCache.first_time`. |

A2A enum names in this document use the `a2a.types.TaskState` constants
(`TASK_STATE_CANCELED`, and so on). Wire/status text may appear as `canceled`.

---

## 2. Layering — two independent auth planes

RDAP authenticates **twice**, on two independent planes. Passing one plane
MUST NOT be treated as passing the other.

```
HTTP ingress
  → RpcIngressLimitMiddleware (body/concurrency bounds)
  → RavenRequestAuthenticator          # Plane A: transport HTTP signature + HTTP replay
  → raven_owner on ServerCallContext
  → A2A JSON-RPC (a2a-sdk)
  → TeamAgentExecutor                  # Plane B: delegation signature + delegation replay
  → BoundedTaskStore (owner, task_id)
  → OwnerScopedActiveTaskRegistry
```

**Plane A — transport (HTTP request signature).**
`RavenRequestAuthenticator` verifies a Raven HTTP request signature over one
exact method / request-target / body, then records that signature in a
**durable HTTP replay cache**. On success the verified RVN1 address becomes
`raven_owner`. This happens **before** the A2A SDK parses JSON-RPC or looks
up a task.

**Plane B — delegation (task / answer signature).**
`TeamAgentExecutor.execute` (direct A2A) and the Git/mailbox relay paths
verify a recipient-bound, expiring Ed25519 delegation over sender, recipient,
task id, kind (`task` or `answer`), times, payload digest, and nonce. Accepted
delegations are recorded in a **separate durable delegation replay cache**.

The two caches MUST remain separate files. A consumed HTTP signature MUST NOT
satisfy a delegation replay check, and vice versa.

### 2.1 Trust pin and signed-only default

- A trusted peer entry pins an **exact RVN1 address** to an **exact Ed25519
  public key**. `validate_address_public_key` derives the address from the key
  and rejects any mismatch (`raven_identity.py`).
- `NodeConfig.require_signed_tasks` defaults to `True`
  (`TEAM_REQUIRE_SIGNED` ≠ `0`). `build_app` then sets
  `require_raven_auth=config.require_signed_tasks` and
  `TeamAgentExecutor.require_signed` to the same flag.
- `--open` (`team_agents/__main__.py`) is the explicit dangerous override:
  `require_signed_tasks = not args.open`. It disables **both** the required
  HTTP plane and the required delegation plane. It MUST remain loud on the
  public Agent Card (`⚠ OPEN MODE`).

---

## 3. Transport plane (HTTP signature)

### 3.1 Covered fields

`http_request_signing_bytes` / `RavenRequestAuthenticator._FIELDS` bind:

| Binding | Notes |
|---|---|
| HTTP method | Canonicalized `method.upper()` |
| Request target | Exact ASCII path plus query string if present |
| Body digest | SHA-256 of the raw HTTP body |
| Address / sender | Header `raven-request-address` |
| Recipient | Must equal this node's RVN1 address |
| Nonce | 16 raw bytes, hex-encoded |
| Issued-at / expires-at | Unix seconds; default TTL **600s**, compiled max **24h**; ±60s future-skew |
| Algorithm | `ed25519` |
| Context | `raven.a2a.http-request.v1` |
| Signature | Detached Ed25519 over the canonical bytes |

Duplicate `raven-request-*` headers MUST be rejected (401) before
authentication.

### 3.2 Outcomes

Evaluated in `RavenRequestAuthenticator.authenticate` and
`RpcIngressLimitMiddleware`:

| Ingress | `require_raven_auth` | Result |
|---|---|---|
| No `raven-request-*` headers | `True` (default) | Reject **before** the SDK (HTTP 401). |
| Signature present and valid | either | `raven_owner` = verified RVN1 address; `RavenPeerUser(..., authenticated=True)`. |
| `--open` / `require_raven_auth=False` and no signature | `False` | `raven_owner = open:<client-ip>`; **unauthenticated**. |
| Signature **present but invalid / incomplete / replayed / revoked / unknown** | either | Fail **even in open mode** (HTTP 401). A bad signature MUST NOT fall through to `open:<ip>`. |

Trust-file and revocation-file failures on this plane fail closed (no owner,
401 when required or when a signature was presented).

### 3.3 HTTP replay cache

- Path: `{keys_dir}/http-request-replay-cache.sqlite3`
  (`RavenRequestAuthenticator`).
- Replay key: `SHA-256(signature_b64)` stored as hex
  (`ReplayCache.first_time`).
- Once-only: a second presentation of the same signature before expiry MUST
  fail as `HTTP authorization replay`.
- Fail-closed: SQLite error, **8192** active entries, or compiled **8 MiB**
  database-byte ceiling MUST deny the request. A cache already over the byte
  ceiling MUST refuse to open (`rotate it while the node is stopped`).
- Rejected replays MUST be mutation-free (rollback; no housekeeping write).

---

## 4. Task FSM

Stance: **not last-message-wins.** Store retention, owner isolation, and the
Cancel force-save are independent of whichever updater event happens to be
enqueued last.

### 4.1 States

```
                    delegation auth FAIL
                         │
                         ▼
                    REJECTED ──────────┐
                         │             │ never retained in BoundedTaskStore
                         │             ▼
                         │          (dropped; MUST NOT overwrite
                         │           a prior valid row for that id)
                         │
                    delegation OK
                         │
                         ▼
                    SUBMITTED
                         │
                         ▼
                     WORKING
                    /    |    \
                   /     |     \
                  ▼      ▼      ▼
           COMPLETED  FAILED  CANCELED
```

- After delegation succeeds, the executor emits `SUBMITTED` (if no current
  task), then `updater.submit()` / `updater.start_work()`, then
  `WORKING`, then a terminal `complete()` or `failed()`.
- **REJECTED is never retained.** `BoundedTaskStore.save` drops a rejected
  task and MUST NOT replace an existing valid row when an invalid request
  reuses that `task_id`.
- Terminal set used by the store: `COMPLETED`, `CANCELED`, `FAILED`,
  `REJECTED`.

### 4.2 Auth-fail path (mutation-free)

Authentication is the first request-dependent operation in
`TeamAgentExecutor.execute`. On failure the executor MUST:

1. Emit **one** terminal `TASK_STATE_REJECTED` `Task` so SDK dispatchers
   finish cleanly.
2. Return **before** appending team memory, touching Git, registering
   `_cancel_events`, or calling the brain.

A configured trust/revocation-policy exception is treated as the same
auth failure (`delegation authorization unavailable`). A
`transport_owner != meta.sender` mismatch MUST be rejected **before**
`verify_delegation` so a forwarded envelope cannot consume the once-only
delegation replay slot.

Whitespace is part of the signed wire payload. Normalization (`strip`)
happens only **after** successful verification.

RDAP does **not** own an illegal-transition table beyond a2a-sdk
`TaskUpdater`. See [§9.2](#92-no-rdap-owned-illegal-transition-table).

---

## 5. Cancel FSM

`CancelTask` is owner-scoped and re-authorized immediately before the SDK
is allowed to cancel a producer.

```
CancelTask(id)
  │
  ├─ task_store.get(id, owner-context) is None  → TaskNotFound
  │
  ├─ _authorize_cancellation(context)
  │     require_signed ⇒ authenticated ∧ caller ∈ peers ∧ caller ∉ revoked
  │     FAIL → PermissionError; MUST NOT mutate store or cancel_event
  │
  ├─ DefaultRequestHandler.on_cancel_task  (SDK cancels/closes producer)
  │     TeamAgentExecutor.cancel → set owner-keyed asyncio.Event (if registered)
  │
  └─ if returned task is not TASK_STATE_CANCELED
        force-save TASK_STATE_CANCELED into the owner-scoped store
        (Cancel RPC caller never observes a still-working task)
```

`TeamAgentExecutor.cancel` looks up `_cancel_events[(caller, candidate)]`
for `context.task_id`, `task.id`, and the signed `message_id`. It only
`Event.set()`s. It MUST NOT enqueue a terminal status from this hook: the
SDK has already closed the producer queue.

### 5.1 Invariants (I1–I4)

These are the **implemented Cancel-path invariants** as of this freeze.
I4 is a **caller-visible store invariant only**. It does **not** make
end-to-end cancel correct; see [§9.1](#91-critical--cancel-status-skew-open-o5-conformance-gap).

| ID | Invariant | Code |
|---|---|---|
| **I1** | Cancel (and Get / List / Subscribe) resolve only under the transport owner. A miss is `TaskNotFound`. A different trusted peer MUST receive task-not-found and MUST NOT observe or cancel another owner's producer. | `RavenRequestHandler.on_cancel_task` / `on_subscribe_to_task`; `BoundedTaskStore.get`; `OwnerScopedActiveTaskRegistry` |
| **I2** | `_authorize_cancellation` runs after the owner-scoped get and **before** SDK cancel. When `require_signed`, the caller MUST be authenticated, present in the live peer pin file, and not revoked. Authorization failure MUST NOT mutate task state or set a cancel event. | `RavenRequestHandler._authorize_cancellation` |
| **I3** | A successful authorized cancel sets the owner-keyed cooperative `asyncio.Event` only. Same A2A `task_id` values for two owners remain independent (separate store rows, registries, and cancel maps). | `TeamAgentExecutor.cancel`; `_cancel_events[(owner, task_id)]` |
| **I4** | If the SDK-returned task is not `TASK_STATE_CANCELED`, the handler force-saves `TASK_STATE_CANCELED` so the **Cancel RPC caller** never sees a still-working task. I4 is **not** a claim that the in-flight executor/brain path terminates as `CANCELED`. | `RavenRequestHandler.on_cancel_task` |

---

## 6. Delegation plane

### 6.1 Bound fields

`sign_delegation` / `verify_delegation` bind:

- `sender` (RVN1)
- `recipient` (RVN1; must equal `expected_recipient`)
- `task_id` (must equal `expected_task_id`, typically the A2A `message_id`)
- `kind` ∈ {`task`, `answer`}
- `issued_at` / `expires_at` (default TTL 600s, max 24h, ±60s future-skew)
- payload digest `SHA-256(task_text UTF-8)`
- nonce (16 bytes, hex)
- context `raven.a2a.delegation.v2`, algorithm `ed25519`

### 6.2 Transport/delegation sender bind (before replay consume)

In `TeamAgentExecutor.execute`, if metadata is present:

```
transport_owner = context.call_context.user.user_name
if transport_owner != meta.sender:
    reject as 'transport/delegation sender mismatch'
```

This check MUST run **before** `verify_delegation`, so a forwarded signed
body presented under a different HTTP principal cannot consume the
once-only delegation replay entry.

Relay / mailbox paths perform the analogous outer-sender / outer-recipient
check before `verify_delegation`.

### 6.3 Delegation replay cache

- Path: `{keys_dir}/replay-cache.sqlite3` (`NodeConfig.replay_cache_path`).
- Same `ReplayCache` implementation and ceilings as the HTTP cache
  (SHA-256 of signature b64, 8192 / 8 MiB, fail-closed, mutation-free
  replay reject).
- Direct A2A executor uses `consume_replay=True` (default).
- Git relay and experimental mailbox verify with `consume_replay=False`
  (peek). See [§9.3](#93-relay-peek-vs-consume).

---

## 7. Ownership and `BoundedTaskStore`

### 7.1 Isolation

- Store key: `(owner, task_id)` where `owner` is
  `OwnerResolver(context)` (user scope / tenant = `raven_owner`).
- `OwnerScopedActiveTaskRegistry` keeps one a2a-sdk `ActiveTaskRegistry`
  per verified owner so the SDK's task-id-only live map cannot join
  another owner's producer. Owner capacity is
  `max(1, task_store.max_count)`.
- Get / List / Subscribe / Cancel on a task the caller does not own MUST
  behave as not-found, not as a cross-owner status leak.

### 7.2 Bounds and eviction

| Limit | Default | Compiled hard ceiling |
|---|---|---|
| Count | 256 | 4096 |
| Serialized bytes | 8 MiB | 64 MiB |
| Idle TTL (terminal only) | 1 hour | 24 hours |

Environment knobs `TEAM_TASK_STORE_MAX_COUNT`,
`TEAM_TASK_STORE_MAX_BYTES`, and `TEAM_TASK_STORE_TTL_SECONDS` MAY lower
or tune the defaults; they MUST NOT exceed the hard ceilings
(`NodeConfig.__post_init__` / `from_env`).

Invariants:

- **REJECTED is never retained** and MUST NOT clobber a prior valid row.
- When space is required, evict the **oldest terminal** history only.
- **Never evict an active (non-terminal) task** to admit another task.
  Capacity exhaustion raises `TaskStoreCapacityError` (fail closed).
- TTL expiry applies to **terminal** entries only. Active work does not
  idle-expire.
- Copy-on-read / copy-on-write: a caller-mutated `Task` MUST NOT change
  the stored row.

This is race-safe in-process locking (`asyncio.Lock`), not a distributed
store.

---

## 8. Conformance / selftest mapping

High-level mapping from this freeze to checks that exist in
`team_agents/selftest.py` (names copied from that file; this section does
not invent additional test names). The suite is necessary but **not
sufficient** for end-to-end cancel correctness — see §9.1.

| Area | Existing `check(...)` names (representative) |
|---|---|
| Peer pin | `address/public-key binding validates`; `mismatched identity binding rejected` |
| Delegation bind / tamper | `recipient/task-bound delegation verifies`; `task-id substitution rejected`; `recipient forwarding rejected`; `payload tamper rejected`; `missing nonce rejected`; `expired task rejected` |
| Delegation replay | `replay rejection after cache restart is mutation-free` |
| HTTP plane | `HTTP request auth binds Raven owner, method/target/body and replay`; `secure JSON-RPC routes reject unsigned transport requests` |
| Replay ceilings | `replay cache enforces active-row and database-byte ceilings` |
| Trust policy | `revoked sender rejected`; `signed tasks required by default`; `broken revocation policy fails closed` |
| Auth-fail isolation | `unsigned executor rejection has zero durable/team side effects`; `authorization-policy exception rejects without durable mutation`; `transport/delegation mismatch rejects before replay insertion` |
| Bounded store | `bounded task store is race-safe, copy-safe and fail-closed`; `task-store environment tuning cannot exceed compiled hard max`; `unique invalid RPC tasks cannot grow the live task store unbounded` |
| Owner isolation | `real routes isolate List/Get/Subscribe/Cancel by signed Raven owner`; `simultaneous same-ID tasks and cancellation stay owner local` |
| Cancel RPC (caller-visible) | `authorized cancellation publishes a terminal canceled Task` — asserts the Cancel RPC return is `TASK_STATE_CANCELED` (I4 / force-save). It does **not** assert that the executor/brain path terminalizes as `CANCELED`. |
| Relay cache privacy | `Git relay excludes its private replay cache` |

CI runs the same functional suite via `.github/workflows/selftest.yml`.

---

## 9. Open conformance gaps

The following are **gaps**, not fixed behavior. They MUST NOT be cited as
normative “already correct.” This PR does not patch runtime code.

### 9.1 CRITICAL — Cancel status skew (open O5 conformance gap)

**Owner:** Role #14 (Python Runtime).
**Status:** Sprint 0 triage / notes only. **Not** claimed correct. **Not**
fixed in this docs PR.

**Known skew (current code):**

1. `RavenRequestHandler.on_cancel_task` force-saves
   `TaskState.TASK_STATE_CANCELED` into the owner-scoped store when the
   cancel RPC succeeds, so the **Cancel caller** never sees a still-working
   task (I4).
2. Separately, `TeamAgentExecutor` and the brain still treat cancel as the
   string `CANCELLED`: `TeamAgentExecutor.cancel` only sets an
   `asyncio.Event`; `brain.run(..., cancel_event=...)` may still complete
   and return `'CANCELLED'`; the executor path then still calls
   `updater.add_artifact(...)` (including signed `kind=answer` metadata)
   and `updater.complete()`, i.e. the **COMPLETED artifact path**.

Observed terminal store state after a successful Cancel MAY therefore
disagree with in-flight executor completion artifacts / last updater
events. A later `complete()` MUST NOT be read as making Cancel
last-message-wins-correct.

**Do not describe the force-save as making end-to-end cancel already
correct.** I4 only constrains the Cancel RPC response and the
owner-scoped snapshot at that instant.

**Intended semantics (manager decision; recorded here, not implemented
in this PR):**

- A brain/executor `CANCELLED` path SHOULD terminalize as A2A
  `TASK_STATE_CANCELED`.
- `FAILED` SHOULD be reserved for a real failure race, not for a
  successful cooperative cancel.

**Sprint scheduling (not a fix commitment in this PR):**

- Runtime patches are **held** until the Sprint 1 batch.
- Cancel terminalization is **ticket #1** of that batch.
- Sprint 0 = triage and notes only. **The fix lands in Sprint 1, not in
  this docs PR.**

### 9.2 No RDAP-owned illegal-transition table

RDAP does not define its own illegal-transition matrix. Direct A2A status
moves go through a2a-sdk `TaskUpdater` (`submit`, `start_work`,
`update_status`, `complete`, `failed`) plus the handler force-save on
Cancel. Until Role #14 closes §9.1, a `WORKING` → `CANCELED` (store)
transition can race a subsequent `complete()` / `COMPLETED` artifact.
An RDAP-owned table is **not** claimed to exist.

### 9.3 Client answer verify uses in-memory `ReplayCache`

`team_agents/client.py` verifies signed `kind=answer` artifacts with
`reply_replay = ReplayCache()` — no durable path. Restarting the client
process drops that once-only set. Direct A2A server delegation replay
(`replay-cache.sqlite3`) is durable; client-side answer replay is not.

### 9.4 Relay peek vs consume (`consume_replay=False`)

Git relay (`relay.py`) and the experimental mailbox path (`server.py`)
call `verify_delegation(..., consume_replay=False)`: cryptographic
verification without inserting the signature. Relay answers later consume
via `first_time` on acknowledge; tasks use a separate durable
`outcomes.claim(signature, ...)` table. Peek-then-claim vs consume-on-verify
needs an explicit RDAP rule (who may peek, when consume is mandatory, what
happens if peek succeeds and claim/ack fails). That rule is **not** frozen
as correct here.

### 9.5 Unification with raven-node ATSAM

RDAP currently creates its own key under `.team/keys` and does not submit
or receive application payloads through the production `raven-node` ATSAM
session actor. Unifying RDAP with the node identity / protected store and
encrypted Raven carrier is a **cross-repo integration** gap (README
“Important integration gap”). It is out of scope for this PR.

---

## 10. Document history

| Rev | Notes |
|---|---|
| Sprint 0 freeze draft (O5) | Role #13 protocol note from current code. Cancel skew recorded as an open O5 gap owned by Role #14; runtime fix held for Sprint 1. |
