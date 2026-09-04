# LLM runtime boundary (Role #16, Sprint 0)

Durable review note for Identity AuthZ, Adversarial QA, and other RDAP
roles. Claims below were checked against the Python sources on this
snapshot (`team_agents/`, `rdap.py`). This document does not change
runtime behavior.

## Architecture law

**LLM proposes → deterministic runtime validates → authorization checks → tool executes.**

The model must never receive a direct shell channel. Any shell path goes
only through `ToolBox.dispatch`, and only when the operator has enabled
`NodeConfig.allow_shell`. There is no `os.system`, `Popen`, or
`subprocess` call from `OpenAIBrain` / `EchoBrain`. The sole agent-facing
shell entry is `ToolBox.tool_run_command`, which is reached exclusively
via `dispatch` after the gates below.

Never LLM→shell.

Two authorization planes sit on this path and are **not** the same check:

| Plane | Where | What it answers |
|---|---|---|
| Peer / A2A delegation | `TeamAgentExecutor.execute` → `verify_delegation` | Is this signed, recipient-bound task from a trusted, non-revoked peer? |
| Tool capability | `ToolBox` (`allow_shell`, `_safe_read`, `commit_staged`) | May this already-accepted task invoke this tool with these arguments? |

A verified Raven delegation does **not** grant project writes or a shell.
Those require a separate operator grant (`--allow-shell` /
`TEAM_ALLOW_SHELL=1` on the `from_env` path). Trusting a peer is not a
project-write or shell grant.

---

## 1. Call path: LLM → validate → authz → tool

### Process wiring

`team_agents/server.py` `build_app` constructs one shared stack per
process:

1. `TeamMemory(config.repo_path, …)`
2. `ToolBox(config, memory)`
3. `build_brain(config, toolbox)` → `OpenAIBrain` or `EchoBrain`
   (`team_agents/llm.py`)
4. `TeamAgentExecutor(config, brain, memory, trusted_peers=…,
   require_signed=…, identity=…)`

The same `ToolBox` instance is reused by every concurrent A2A task on
that node. `OwnerScopedActiveTaskRegistry` isolates A2A task IDs per
Raven principal; it does not give each task its own toolbox.

### Ingress (before the executor)

HTTP JSON-RPC POST `/` is bounded and (by default) Raven-request-signed
in `RpcIngressLimitMiddleware` + `RavenRequestAuthenticator`
(`team_agents/server.py`). Optional `BearerAuthMiddleware` runs when
`NodeConfig.auth_token` is set. A verified Raven address becomes
`ServerCallContext.user` (`RavenPeerUser`). This is **transport**
authn/authz. It is not tool authorization.

Git-relay and experimental mailbox workers
(`_start_services` in `server.py`, `GitRelay.process_inbox`) call
`verify_delegation` (or the relay equivalent) and then
`brain.run(text)` — the same brain/toolbox, not a second tool path.

### Executor: delegation authz, then brain

`TeamAgentExecutor.execute` (`team_agents/executor.py`) is explicit:
authentication is the first request-dependent operation. It must not
append task text to team memory, touch Git, or register cancellation
state until the delegation has been verified.

Order after a JSON-RPC `Send` reaches the executor:

1. Read `context.get_user_input()` as `wire_text` (must be `str`).
2. `extract_raven_meta` → Raven delegation fields
   (`sender`, `recipient`, `task_id`, `kind`, `signature`, …).
3. `authorize()` (worker thread):
   - reject if transport owner ≠ delegation `sender` (prevents a
     forwarded envelope from consuming another peer's replay slot);
   - `verify_delegation(..., required=self.require_signed,
     expected_recipient=identity.address, expected_kind='task', …)`.
4. On failure: emit one terminal `TASK_STATE_REJECTED` task and
   **return**. No journal, no Git, no `brain.run`.
5. On success: `text = wire_text.strip()` (whitespace is part of the
   signed payload; normalize only after verify).
6. Register a per-`(owner, task_id)` `cancel_event`.
7. Journal `incoming task: {text[:120]}` and, if meta is present,
   `delegation verified: {sender}`.
8. `answer = await self.brain.run(text, cancel_event=cancel_event)`.
9. Optionally `sign_delegation(..., kind='answer')` and complete the
   A2A task.

`--open` / `TEAM_REQUIRE_SIGNED=0` sets
`NodeConfig.require_signed_tasks=False`, which is passed through as
`require_signed=False`. That weakens **peer** authz only. It does not
enable `allow_shell`.

Cancellation (`TeamAgentExecutor.cancel` +
`RavenRequestHandler._authorize_cancellation`) is a separate Raven
principal check. It sets the task's `cancel_event`; it does not authorize
tools.

### Brain: propose only

#### `OpenAIBrain.run` (`team_agents/llm.py`)

Used when `LLMConfig.provider` is `openai`, `groq`, `openrouter`,
`ollama`, or `custom`.

1. Load `.team/GOAL.md` via `_team_goal` → `TeamChat.get_goal()`
   (bounded reader; see §3).
2. Build `SYSTEM_PROMPT` (role text + optional `TEAM GOAL` appendix).
   Rule 4 is prompt-only: *“Never use an operator-enabled shell to
   bypass `read_file` sensitive-path denials.”* That is not an OS
   sandbox.
3. User message is the **raw delegated `task_text`**. The brain does
   not sanitize it.
4. Loop up to `LLMConfig.max_steps` (default 12):
   - POST `{base_url}/chat/completions` with
     `'tools': self.toolbox.schemas()`.
   - If the model returns no `tool_calls`, accept plain-text
     `content` as the answer (no `final_answer` required).
   - For each call: `fn = call['function']['name']`;
     `args = json.loads(...)` or `{}` on `JSONDecodeError`.
   - **`result = await self.toolbox.dispatch(fn, args)`** — the only
     execution step.
   - Append `role=tool` with `result[:8000]`.
   - If `fn == 'final_answer'` and the handler returns
     `'final answer accepted; you may stop now'`, return
     `args['answer']` from **this** invocation.

Brain-layer validation is intentionally thin: JSON parse, 8000-character
result truncation, and the `final_answer` success-string check.
Argument types, required keys, and name allowlisting are not enforced
here. They are concentrated in `ToolBox`.

`final_answer` is invocation-local. The answer is never stored on the
process-wide `ToolBox` shared by concurrent A2A tasks
(`llm.py` comment at the return site; covered by
`team_agents/selftest.py`). A malformed `final_answer` (`{}`) does not
reuse a previous task's answer.

#### `EchoBrain` (no tool loop)

Used when the provider is `echo` (default). `EchoBrain.run` does **not**
call `ToolBox.dispatch`. It writes
`.team/outputs/{node_name}/{slug}-{uuid}.md` through
`TeamMemory.resolve_in_repo`, then `log_event` / `set_task`. That is
deterministic demo I/O inside shared team memory, not a shell channel
and not the LLM tool path. Node names are re-validated
(`validate_node_name`) before they become path components.

### `ToolBox.dispatch` → individual tools

```text
OpenAIBrain.run
    → toolbox.dispatch(name, args: dict)
        → getattr(self, f'tool_{name}', None)
        → handler(**args)     # TypeError / other exceptions → "ERROR: …"
```

`ToolBox.dispatch` (`team_agents/tools.py`):

- Unknown name → `'ERROR unknown tool: {name}'` (string, not an
  exception).
- `final_answer` runs on the event loop; every other handler is
  `asyncio.to_thread` so filesystem/Git/subprocess work cannot stall
  ASGI.
- There is **no** explicit allowlist of schema names inside `dispatch`.
  Resolution is `getattr(..., f'tool_{name}')`. High-risk handlers
  still exist when they are omitted from `schemas()`; they fail closed
  on `allow_shell` (see §2 and selftest: direct `dispatch('write_file')`
  is denied when `allow_shell=False`).
- Extra or mistyped kwargs become `TypeError` and are returned to the
  model as `ERROR: …`. There is no JSON-Schema validation of types or
  required fields beyond what the handler signature and body enforce.

### Default tools (always in `schemas()`, no `allow_shell`)

| Tool | Handler | Effect |
|---|---|---|
| `list_files` | `tool_list_files` | Repo listing; skips `.git` / `.team` / venv-like trees, `_sensitive_path`, symlink/reparse |
| `read_file` | `tool_read_file` | Bounded text read via `_safe_read` (see §3) |
| `git_status` | `tool_git_status` | `git status --short` through `TeamMemory._git_checked` (argv Git, not a shell) |
| `board_read` | `tool_board_read` | Bounded **delta projection**, not raw `BOARD.md` |
| `board_set_task` | `tool_board_set_task` | Mutates shared board deltas |
| `log_event` | `tool_log_event` | Journal delta, text clipped to 400 chars at write |
| `remember_fact` | `tool_remember_fact` | Fact delta; may `commit_team` if `auto_commit_memory` |
| `read_facts` | `tool_read_facts` | All stored facts back into the model |
| `claim_file` / `release_file` | lock helpers | Advisory `.team/locks` only |
| `final_answer` | `tool_final_answer` | Requires `answer: str`; returns the accept string |

A trusted, signed task is **intended** to read ordinary non-sensitive
project files and mutate shared `.team` memory (board, facts, journal,
locks, outputs). That is not a documentation gap; it is the default
authority model. Filename policy is defense in depth, not content-aware
DLP.

### CLI flags that select this stack

Wizard (`rdap.py` `cmd_start` / `start` subparser):

- `./rdap start --allow-shell` →
  `NodeConfig(..., allow_shell=bool(args.allow_shell),
  require_signed_tasks=not args.open, …)` then `serve(cfg)`.
- `./rdap start --open` → unsigned tasks accepted (peer plane only).
- `--provider` / `--model` / `--base-url` select `LLMConfig` (echo vs
  hosted/custom).

Direct module (`team_agents/__main__.py` `cmd_serve`):

- `python -m team_agents serve --allow-shell` sets
  `cfg.allow_shell = True` after `NodeConfig.from_env()`.
- `NodeConfig.from_env()` already sets
  `allow_shell=(os.environ.get('TEAM_ALLOW_SHELL', '') == '1')`.
- `--open` → `require_signed_tasks = False`.

`NodeConfig.allow_shell` defaults to `False`
(`team_agents/config.py`).

---

## 2. `--allow-shell` / `TEAM_ALLOW_SHELL` risk

### What the grant enables

`allow_shell=True` is a **single node-wide bit**. It does two things:

1. **Schema exposure** — `ToolBox.schemas()` appends `write_file`,
   `run_command`, and `git_commit` so the hosted model can see them.
2. **Runtime checks** — the handlers themselves refuse the action when
   the bit is false:

   - `tool_write_file`: `'ERROR: project file writes disabled on this
     node (allow_shell=false)'`
   - `tool_run_command`: `'ERROR: shell disabled on this node
     (allow_shell=false)'`
   - `tool_git_commit` → `TeamMemory.commit_staged(...,
     explicitly_authorized=self.config.allow_shell)` which raises
     `PermissionError` unless authorized.

The agent-facing shell tool is named **`run_command`**, not `run_shell`.
`run_command` is `subprocess.run(command, shell=True, cwd=repo,
capture_output=True, timeout=timeout)` with stdout+stderr clipped to
`MAX_CMD_OUTPUT` (10_000). The command string is operator-OS-user
authority: no argv allowlist, no path policy, no syscall filter.

`write_file` uses `TeamMemory.resolve_in_repo` only (repo-escape
check). It does **not** call `_safe_read` / `_sensitive_path`. With the
grant on, the model can create or overwrite any in-repo path the OS user
can write, including names that `read_file` would deny.

`git_commit` still refuses staged paths that are local/private `.team`
state (`keys`, mesh, replay, …). It never stages files. Combined with
`run_command` (`git add …`), that refusal is not a containment boundary.

`read_file` denials remain in force when `allow_shell=True` (selftest:
operator-mode seed path still returns `ERROR:`). The bypass is
`run_command`, not `read_file`.

### Default-off posture

| Entry | How the bit becomes true |
|---|---|
| `NodeConfig` dataclass | `allow_shell: bool = False` |
| `NodeConfig.from_env()` | **only** `TEAM_ALLOW_SHELL=1` (any other value, including `true` / `yes`, is false) |
| `python -m team_agents serve` | `from_env()` plus `--allow-shell` (flag can only turn the bit **on**) |
| `./rdap start` | **`--allow-shell` only.** `cmd_start` constructs `NodeConfig` directly and does **not** read `TEAM_ALLOW_SHELL`. |

Operators who export `TEAM_ALLOW_SHELL=1` and then run `./rdap start`
without `--allow-shell` do **not** enable the grant. Operators who run
`python -m team_agents serve` with the env var set **do**. Document and
alert on the actual argv/env of the process that called `serve()`.

Neither CLI currently prints a banner when the grant is on
(`--open` and the experimental mailbox do). The extended Agent Card
description includes `shell=on` / `shell=off`
(`build_extended_card`). The public card does not.

### Why this expands blast radius

Treat `--allow-shell` as **full local code and data access** for the
server OS user, scoped in practice to “whatever that user can do from
`cwd=repo` with `shell=True`.”

Concrete expansions versus default-off:

- **Path-denial bypass.** `read_file` cannot open `.env`,
  `.team/keys/…`, `.git/…`, credential-named files, symlinks, or
  hardlinks. `run_command` can `cat` them (and paths **outside** the
  repo).
- **Credential exfil.** Tool results are returned to the model (clipped)
  and may be copied into the signed A2A answer, journal, facts, or
  `.team/outputs`.
- **Repo mutation.** `write_file` overwrites project and secret-named
  paths; `run_command` can `git add` / rewrite history / install hooks
  if the OS user can; `git_commit` commits whatever is already staged
  except private `.team`.
- **Host mutation.** `shell=True` is not confined to the Git worktree
  (network, home directory, other processes, package installs).
- **Prompt instruction is not a control.** `SYSTEM_PROMPT` forbids using
  the shell as a `read_file` bypass. A prompt-injected or
  goal-injected model is not bound by that sentence.

### Operator guidance

**Acceptable (still high-risk) only when all of the following hold:**

- Dedicated OS account / throwaway workspace; no production credentials
  in the tree or in that user's environment.
- Operator is at the keyboard and can stop the node.
- Peers on the trust list are the same trust domain as “this Unix user
  may run commands I typed.”
- `--open` is **not** set. Unsigned + shell is “anyone who can reach
  the port owns the account.”
- Secrets live outside the delegated project tree even then.

**Not acceptable:**

- Any node that holds `.team/keys`, cloud API keys, or a checkout with
  customer/source secrets, unless that node is already treated as
  equivalent to unsandboxed remote code execution.
- Unattended / CI / shared-host deployments.
- Enabling the grant because “the agent got a PermissionError from
  `read_file`.” That denial is the product.

**Logging / monitoring expectations (today vs. should):**

| Event | Today | Expectation if the grant is on |
|---|---|---|
| Grant enabled | No startup banner; extended card `shell=on` | Operator-visible banner + process inventory |
| `run_command` | **Not journaled.** Output (10k) returns to the model only | Log argv/cwd/exit (and ideally refuse to enable without a sink) |
| `write_file` | `log_event(..., 'wrote {path} ({n} bytes)')` — path only | Same, plus treat journal as attacker-influenced |
| Incoming task | First 120 chars after successful delegation | Keep; do not log rejected payloads (already the case) |
| Remote activity GET `/raven/activity` | 403 unless `TEAM_AUTH_TOKEN` is configured | Do not scrape the journal over the LAN without Bearer |
| Model prompt / tool transcripts | Not persisted by the runtime | If you record them, they contain peer-controlled text |

The team journal is shared, Git-synced, and writable by
`log_event`. It is not a high-integrity audit log. If the grant is on,
use OS-level process accounting / an external SIEM; do not rely on
`.team/journal.md`.

---

## 3. Controls inventory and Sprint 0 gaps

### Controls that exist (verified)

**Peer plane (before any tool):**

- Default `require_signed_tasks=True`; `--open` is an explicit override.
- `verify_delegation` binds sender, recipient, task id, kind, payload;
  replay cache; hot-reloaded trust/revocation files fail closed.
- Transport Raven HTTP signature is separate from delegation metadata.
- Rejected tasks: no TeamMemory/Git writes, not retained in
  `BoundedTaskStore`.

**Prompt / GOAL.md assembly:**

- `TeamChat.get_goal` (`team_agents/chat.py`): regular file only, no
  symlink, `st_nlink == 1`, `O_NOFOLLOW`, identity check, hard cap
  `MAX_GOAL_BYTES` (64 KiB). Oversized or aliased GOAL → empty string
  (`_team_goal` / selftest).
- `set_goal` enforces the same byte limit on write.

**Filesystem reads (default tools):**

- `_safe_read`: relative paths only; rejects `..`, NUL, `:`, `~`,
  trailing space/dot; `_sensitive_path` on logical and resolved parts
  (`.git`, `.ssh`, `.team/keys|mesh-*|replay-cache`, env/key/pem-style
  names, secret-token filenames).
- Open with `O_NOFOLLOW`; require regular file, no reparse, nlink==1,
  matching inode; `MAX_READ_CHARS` (20_000).
- `list_files` applies the same sensitivity skip and drops
  symlink/reparse entries.
- `read_file` denials still apply when `allow_shell=True`.

**Dispatch / schema:**

- High-risk tools omitted from `schemas()` when `allow_shell=False`.
- High-risk handlers fail closed if invoked anyway.
- Unknown `dispatch` names return `ERROR unknown tool`.
- Handler exceptions are stringified to the model, not raised into the
  HTTP stack.

**Shared-state bounding:**

- `board_read` projects validated task deltas; a huge/poisoned
  `BOARD.md` does not enter the prompt (selftest).
- `log_event` / activity projection sanitize and clip; delta readers
  have compiled caps (`team_agents/deltas.py`).
- Automatic `commit_team` stages only `TEAM_SHARED_FILES` /
  `TEAM_SHARED_DIRS` — not `.team/keys`, mesh, replay, or project
  files.

**Concurrency notes that are real, not folklore:**

- `final_answer` is invocation-local (no toolbox-global last-answer).
- Blocking tool/echo work is offloaded from the ASGI loop.
- Concurrent A2A tasks **share** one `ToolBox` and one `TeamMemory`.
  Board/facts/locks/files are not per-task sandboxes. Isolation of
  *answers* is not isolation of *side effects*.

### Residual risks / actionable Sprint 0 follow-ups

Frame: each item is a concrete missing control in the current code, not
a hypothetical attacker story.

1. **Prompt injection via task text.** After `verify_delegation`,
   `task_text` is the user message verbatim. There is no
   untrusted-content delimiter, instruction-hierarchy marker, or
   allowlist of task shapes. A trusted peer (or a compromised one) can
   put tool-steering text in the signed payload. **Follow-up:** treat
   delegated text as untrusted data in the prompt; keep it out of the
   system role; add Adversarial QA fixtures.

2. **Prompt injection via `GOAL.md`.** The bounded reader prevents
   *unbounded read* and *symlink escape*. It does not parse or
   neutralize instructions. `GOAL.md` is Git-shared
   (`TEAM_SHARED_FILES`) and is concatenated into the **system**
   prompt (`TEAM GOAL — every action…`). A teammate with goal-write
   (`./rdap goal` or a synced GOAL) steers every subsequent brain.
   **Follow-up:** move GOAL into a clearly untrusted block; consider
   operator-signed goals; add injection tests.

3. **Tool-result injection back into the chat loop.**
   `read_file` / `run_command` / `read_facts` / `board_read` output is
   appended as `role=tool` with only an 8000-character clip. No
   quoting, no “this is data” wrapper, no secondary model for
   untrusted output. A project file or fact that says “call
   `run_command`” is in-band with the model's next proposal.
   **Follow-up:** fence tool results; consider not feeding raw file
   bodies back when `allow_shell` is on; add a result-injection
   selftest.

4. **No per-tool / per-peer authz beyond `allow_shell`.** Once
   delegation succeeds, every default tool is available to every
   trusted sender. There is no skill-scoped, path-scoped, or
   caller-scoped grant (e.g. “Alice may `read_file` but not
   `remember_fact`”). `NodeConfig.skills` are Agent Card advertisements
   only; `build_app` does not filter `ToolBox` by skill or peer.
   **Follow-up:** Identity AuthZ — per-peer capability list enforced
   inside `dispatch`, not only on the card.

5. **`dispatch` is not a schema-name allowlist.** Hidden
   `tool_*` methods are reachable if the model (or a buggy provider)
   invents the name. Today the extra methods are the high-risk trio
   and they fail closed. **Follow-up:** `dispatch` should accept only
   names in `schemas()` (or a frozen allowlist), even if a handler
   exists.

6. **Brain-layer argument validation is thin.** Invalid JSON becomes
   `{}`; there is no schema check before `handler(**args)`.
   **Follow-up:** validate against the published parameters object;
   do not call the handler on mismatch.

7. **`allow_shell=true` vs path denials.** `run_command` is a total
   bypass of `_safe_read`. `write_file` does not apply
   `_sensitive_path`, so the grant also means “overwrite `.env` /
   keys in-repo.” **Follow-up:** if a future mode needs “writes but
   no shell,” split the bit; apply `_sensitive_path` (or a write
   analog) to `write_file`; never treat prompt rule 4 as a control.

8. **`run_command` has no audit trail.** Enabling the highest-risk
   tool produces no journal line and no startup warning on `./rdap
   start` / `serve()`. **Follow-up:** fail closed or warn loudly;
   log command + exit code to an operator-only sink (not the shared
   journal).

9. **Wizard vs `from_env` mismatch for `TEAM_ALLOW_SHELL`.**
   `./rdap start` ignores the env var. Split-brain operator config is
   a review hazard. **Follow-up:** honor the same env in `cmd_start`,
   or document-only if product intent is “wizard is argv-only”
   (current code is argv-only).

10. **Shared toolbox side effects under concurrency.** Two accepted
    tasks can `board_set_task` / `write_file` / `run_command` on the
    same memory. `final_answer` isolation does not extend to those
    mutations. **Follow-up:** per-task working snapshot or explicit
    “no overlapping high-risk tools” lock.

11. **Default-authority injection surface (not a bug, still a residual
    risk).** `remember_fact`, board notes, and journal lines written
    by one task are readable by the next brain (`read_facts`,
    `board_read`). Content is not treated as untrusted. **Follow-up:**
    same fencing as tool results; cap fact/board text the model sees.

12. **No content-aware secret detection on `read_file`.** Policy is
    filename/path/identity. A secret in `docs/notes.md` is in policy.
    **Follow-up:** keep secrets out of the tree (already README
    guidance); optional scanner is later-sprint, not a missing gate
    in the current design.

### Gaps checklist (copy for Manager / QA)

- [ ] Task text is untrusted data in the prompt (not system-role).
- [ ] GOAL.md is untrusted data (not system-role); injection tests exist.
- [ ] Tool results are fenced; result-injection selftest exists.
- [ ] `dispatch` allowlists schema names.
- [ ] Brain validates args against the tool schema.
- [ ] Per-peer / per-tool grants exist inside `dispatch` (Identity AuthZ).
- [ ] `write_file` sensitive-path policy (if writes remain coupled to shell).
- [ ] Split or keep coupled `allow_shell` documented as RCE-equivalent.
- [ ] Startup banner + audit for `run_command` when the grant is on.
- [ ] `TEAM_ALLOW_SHELL` vs `./rdap start` behavior is one policy.
- [ ] Concurrent high-risk tool overlap is defined (lock or isolate).

---

## Symbol index

| Symbol | File |
|---|---|
| `SYSTEM_PROMPT`, `OpenAIBrain.run`, `EchoBrain`, `build_brain`, `_team_goal` | `team_agents/llm.py` |
| `ToolBox.schemas`, `dispatch`, `tool_read_file`, `tool_write_file`, `tool_run_command`, `tool_git_commit`, `tool_final_answer`, `_safe_read`, `_sensitive_path` | `team_agents/tools.py` |
| `TeamAgentExecutor.execute`, `verify_delegation` (call), `extract_raven_meta` | `team_agents/executor.py` |
| `NodeConfig.allow_shell`, `NodeConfig.from_env`, `TEAM_ALLOW_SHELL` | `team_agents/config.py` |
| `cmd_serve`, `--allow-shell`, `--open` | `team_agents/__main__.py` |
| `cmd_start`, `--allow-shell`, `--open` | `rdap.py` |
| `build_app`, `build_extended_card`, relay `brain.run` | `team_agents/server.py` |
| `TeamChat.get_goal`, `MAX_GOAL_BYTES` | `team_agents/chat.py` |
| `commit_staged`, `resolve_in_repo`, `TEAM_SHARED_*` | `team_agents/memory.py` |
