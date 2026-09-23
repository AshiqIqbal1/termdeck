# Roadmap — agent-cockpit work

Backlog from a review of termdeck's agent story. Items 1 and 3 of the original
list shipped in [#13](https://github.com/kiril6/termdeck/pull/13) (agent worktrees + read-only diff
review); what follows is the remainder, re-ordered by dependency rather than by
the order they were raised.

Confidence tags are the reviewer's original ones. The **Assessment** and
**Approach** sections are grounded in the current code and mark anything that
still needs verifying against upstream docs — nothing here should be treated as
a confirmed API until checked.

---

## 1. Replace agent-state heuristics with real hooks

**Original:** *[Likely]* The waiting state uses an ~8s idle timer and approval
detection is a cross-CLI regex. Claude Code hooks (`Notification`, `Stop`) and
Codex's `notify` setting can POST to a local `/api/agent-event`, giving exact
agent state with zero false positives. Keep the regex only as a fallback.

**Assessment — agreed, and this is the keystone item.** The current detection is
honest about being a heuristic (`AGENT_IDLE_MS = 8000`, `APPROVAL_RE`), but it has
two failure modes we already know about:

- A ring-buffer replay on reconnect re-scans history, so an *already-answered*
  prompt can re-fire a false red (documented in `FEATURES.md`).
- Idle detection cannot distinguish "agent is thinking for 10s" from "agent is
  waiting for input". It guesses.

Hooks would replace guessing with fact for the CLIs that support them.

**The crux is correlation, not transport.** A hook fires in a *separate process*
and has no idea which termdeck tab it belongs to. Solving that first makes the
rest straightforward:

1. When spawning an agent tab, inject an identifying env var (e.g.
   `TERMDECK_TAB=<sessionId>:<tabId>`) into the PTY environment.
2. The user's hook command posts that value back with the event.
3. `POST /api/agent-event { tab, type }` → broadcast over the existing WS to the
   owning tab → call `setWaiting()` directly instead of inferring.

**Approach**
- Backend: `POST /api/agent-event` behind the existing `apiGuard`. Note that a
  hook is a *local process*, not a browser — it sends no `Origin`, which
  `apiGuard` already tolerates. Confirm that before relying on it.
- Frontend: new precedence — an explicit hook event always wins over the
  heuristic; the idle timer and regex stay armed only for tabs that have never
  sent a hook event (so Gemini/Copilot/unknown CLIs keep working unchanged).
- Onboarding is the real UX problem: this needs the user to add a hook to their
  own CLI config. Ship a palette action that prints/copies the exact snippet for
  the detected CLI rather than documenting it in the README and hoping.

**Verify before building:** exact hook names, payload shape and config location
for both Claude Code and Codex. Do not hardcode from memory.

**Risk:** low. Purely additive — if no hook ever fires, today's behaviour is
unchanged.

---

## 2. Resume agent context across a reboot

**Original:** *[Likely]* We state nothing survives a machine reboot. Store each
agent tab's session ID and relaunch with `claude --resume <id>` so agent context
survives even when the shell does not.

**Assessment — agreed, and it is much cheaper *after* item 1.** Worth being
precise about what "survives" means here, because `FEATURES.md` currently makes a
narrower claim that stays true:

| Layer | Survives browser refresh | Survives server restart | Survives reboot |
|---|---|---|---|
| PTY (grace timer) | yes | no | no |
| tmux session | yes | yes | **no** |
| Agent *conversation* | n/a | n/a | **this item** |

The shell is genuinely gone after a reboot — that is a process-memory fact we
cannot engineer around. But the agent's *conversation* lives in the CLI's own
session store, so it can be re-entered. This does not contradict the existing
ceiling; it adds a layer above it.

**Why it depends on item 1:** we need the CLI's session id. Scraping it from
terminal output would be another brittle regex — exactly what item 1 exists to
remove. Hook payloads are the clean source (Claude Code's include a session id —
*verify the field name*). Build item 1 first and this becomes: persist the id
with the tab, and on restore offer a **Resume** action.

**Approach**
- Persist `agentSessionId` per tab alongside the `agent` flag (that flag is
  already persisted as of the fix in #11).
- On restore, do **not** auto-resume. Show a *Resume agent* action on the tab.
  Auto-running a resume command on every page load is the same class of bug as
  re-firing a startup `cmd` — which the code deliberately avoids today.
- Resume command is per-CLI and must come from the preset, not be hardcoded.

**Risk:** medium — entirely in getting the id honestly. Do not ship the
output-scraping version.

---

## 3. An MCP / control API for termdeck itself

**Original:** *[Guessing]* on demand. Would let an orchestrator agent spawn
terminals, read output and send input — making termdeck infrastructure rather
than just a UI. Changes the "no token by design" security argument, so think it
through before building.

**Assessment — the reviewer's own caveat is the important part, and it is
correct.** This is the only item on the list that cannot be built incrementally,
because it invalidates a security argument the codebase currently relies on.

Today's model, stated in `FEATURES.md` and `server.js`:

> Loopback bind + WS Origin/Host validation. **No token by design.**

That holds because every endpoint is reachable only by a *browser page* the user
has open, and grants nothing a shell in that folder does not already grant. A
control API breaks both halves: it is designed to be driven by *another local
process*, and `Origin`/`Host` checks are meaningless against a non-browser client
— any local process can set any header, or send none.

So the honest framing: **this is not "add an MCP server", it is "add an
authentication model to termdeck".** Any local process — or any page in any
browser — gaining the ability to spawn shells and send them input is a genuine
escalation, not a UI convenience.

**Prerequisite decision (not a coding task):** does termdeck want an auth token?
That means a generated secret, somewhere to store it, a way to hand it to
clients, and rotation. That is a real product change and it should be decided
deliberately, not arrived at as a side effect of wanting an MCP server.

**If the answer is yes**, the shape is roughly: token generated at first boot,
printed in the startup banner, required on a *separate* `/control` surface (never
bolted onto the existing browser endpoints), with spawn/input capabilities
gated separately from read-only ones.

**Recommendation: do not start this until items 1 and 2 have shipped.** They make
termdeck better at what it already is. This one changes what it *is* — worth
doing only if orchestration is a direction you actually want.

**Risk:** high, and mostly not technical.

---

## 4. Per-tab context / cost meter

**Original:** *[Guessing]* on feasibility. Would require parsing each CLI's
status output, which is brittle.

**Assessment — agree with the reviewer's own scepticism; lowest value, highest
maintenance.** Parsing status output means re-implementing a fragile reader per
CLI, and re-fixing it every time any of them changes its output format. That is
exactly the kind of per-CLI parsing the current design deliberately avoids
("Heuristic by design (idle-timer, CLI-agnostic); **no per-CLI parsing**").

Building this would be the first thing in termdeck that breaks when an upstream
tool changes its cosmetics — and it would break silently, showing a stale or
wrong number rather than nothing.

**If it is ever wanted**, the only defensible route is a structured source, not
screen scraping: telemetry/OTEL output where a CLI offers it, or a hook that
reports usage. Both need checking per CLI, and may not exist.

**Recommendation: do not build.** A wrong cost number is worse than no cost
number. Revisit only if a CLI exposes usage through a stable, structured
interface.

---

## Suggested order

1. **Hooks** (item 1) — highest value, low risk, purely additive, and unblocks the next one.
2. **Resume across reboot** (item 2) — cheap once hooks land, expensive and brittle before.
3. **Control API** (item 3) — blocked on an explicit decision about authentication. Not a coding task yet.
4. **Cost meter** (item 4) — recommend not building.

Items 1 and 2 keep termdeck on its current trajectory: better at coordinating
agents, with no new trust surface. Item 3 is a different product decision.
