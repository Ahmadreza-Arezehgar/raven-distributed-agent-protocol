# RDAP revocation (address deny-list)

> **Status:** Sprint 0 / G5–G6 alignment note with Identity AuthZ (Role #15).
> Describes the **current** RDAP revoke unit plus the **decided** signed-mode
> policy. Runtime “require a revocations file on start” is **not** claimed
> already enforced.

This is the RDAP-side sibling of the RAVEN G5 cross-stack playbook. It does
**not** live in the RAVEN tree. Cancel, transport, and delegation surfaces that
*consult* this deny-list are specified in
[`rdap-task-lifecycle.md`](rdap-task-lifecycle.md).

**RAVEN playbook (other repo):**
[`docs/engineering/G5_CROSS_STACK_REVOKE_POLICY.md`](https://github.com/Raven-ASHCO/RAVEN/blob/cursor/sprint0-identity-threat-model-d3a8/docs/engineering/G5_CROSS_STACK_REVOKE_POLICY.md)
in [Raven-ASHCO/RAVEN](https://github.com/Raven-ASHCO/RAVEN) —
see [RAVEN PR #5](https://github.com/Raven-ASHCO/RAVEN/pull/5). That path is
**not** in this repository.

Sources checked for current code: `team_agents/raven_identity.py`
(`load_revocations`), `team_agents/config.py`, `team_agents/executor.py`,
`team_agents/server.py`, `team_agents/relay.py`, `rdap.py` (`cmd_init`),
`team_agents/selftest.py`.

---

## 1. Revocation unit = RVN1 address

RDAP revokes **peer addresses**, not device lineages, not public keys as a
separate object, and not A2A task IDs.

`load_revocations(path)` accepts either:

```json
["rvn1q…", "rvn1q…"]
```

or

```json
{"revoked": ["rvn1q…"]}
```

Each entry MUST decode as a version-1 RVN address whose public hash is 20
bytes. Invalid spelling or version fails closed (`ValueError`).

### 1.1 Where the list is enforced (current code)

When a path **is** configured (`NodeConfig.revocations_file` /
`TEAM_REVOCATIONS`), the live set is hot-reloaded and applied at:

| Surface | Code |
|---|---|
| Transport HTTP verify | `verify_http_request(..., revoked=…)` via `RavenRequestAuthenticator` |
| Delegation verify | `verify_delegation(..., revoked=…)` in `TeamAgentExecutor.execute` |
| Cancel authorization | `RavenRequestHandler._authorize_cancellation` (`caller in revoked` → deny; MUST NOT mutate) |
| Git relay / experimental mailbox | `GitRelay._revoked()` / `server._revocations` passed into `verify_delegation` |

A configured file that is **missing, unreadable, or invalid JSON / addresses**
MUST fail closed. Selftest names that exist today:
`revoked sender rejected`, `broken revocation policy fails closed`,
`configured revocation read failure is fail-closed`.

### 1.2 Path unset — current footgun

If `revocations_file` is empty (the default: `TEAM_REVOCATIONS` unset),
`TeamAgentExecutor.current_revocations`, `server._revocations`, and
`GitRelay._revoked` return a **silent empty set**. Signed-mode start does
**not** refuse to boot. `rdap init` does **not** write a default
revocations file.

That empty-by-omission behavior is **current code** and a documented
footgun: operators can believe they have a deny-list when they only have
“no file configured.”

---

## 2. Not device lineage

RAVEN revokes **device lineages** (RVDR1 records and related store rules).
RDAP’s file is an **address deny-list** only.

- There is **no implicit RDAP ↔ lineage bridge**.
- An RVDR1 revocation MUST NOT auto-populate this address deny-list
  (RAVEN playbook A). The reverse is also false: listing an RVN1 address
  here does not revoke a RAVEN device lineage.
- Dual-stack operators MUST run **both** checklists until a real bridge
  exists.
- Data-plane ATSAM fail-closed when a lineage is revoked is a **RAVEN /
  raven-node** behavior. It is separate from this layer-A file deny. RDAP
  does not currently speak ATSAM (see lifecycle §9.5).

---

## 3. Open-Q decision (YES) — signed-mode require-file

**Decided policy (Sprint 0), Identity AuthZ ACK.** This is **open to
implement**. It is **not** already enforced by `rdap init` or `rdap start`.

When `require_signed_tasks` is true (default; not `--open`):

| Rule | Decision |
|---|---|
| Explicit path | The node MUST have a configured revocations file path (`TEAM_REVOCATIONS` / `revocations_file`). |
| Intentional empty list | `[]` or `{"revoked":[]}` is OK — an explicit empty deny-list. |
| Missing / unreadable / invalid | Fail closed (already true **once a path is set**). |
| `--open` | Exempt. Open mode remains the explicit dangerous override. |
| Bootstrap | Prefer `rdap init` writing a default empty revocations file so signed-mode start has a real path. |

Until Role #14 / runtime lands that start-time require-file check and init
write, signed mode with an unset path still means “no addresses revoked,”
not “operator affirmed an empty deny-list.”

---

## 4. Cross-links

- Lifecycle (transport, cancel I2, delegation): [`rdap-task-lifecycle.md`](rdap-task-lifecycle.md)
- RAVEN G5 playbook (other repo): [PR #5](https://github.com/Raven-ASHCO/RAVEN/pull/5) /
  [`docs/engineering/G5_CROSS_STACK_REVOKE_POLICY.md`](https://github.com/Raven-ASHCO/RAVEN/blob/cursor/sprint0-identity-threat-model-d3a8/docs/engineering/G5_CROSS_STACK_REVOKE_POLICY.md)
  on branch `cursor/sprint0-identity-threat-model-d3a8` (not yet on `main` at
  the time of this note)
