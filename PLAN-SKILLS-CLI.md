# Plan: the three-layer pattern (CLI + lazy skills + LLM) for Fonderie

_2026-07-22. A robust plan to lower token usage by adopting the CLI+skills
pattern the industry converged on (Cloudflare Wrangler, Vercel, Anthropic
skills, Playwright's MCP-vs-CLI split, skills.sh). Written to reconcile with our
own N=3 measurement, so we pull the lever that actually moves our number._

## What we already proved, and the layer we mismeasured

- **CLI vs MCP transport is a wash for us.** N=3 verdict
  (`experiments/phase41-2026-07/DISCOVERY-CLI-VS-MCP.md`): pb-cli/fat = 0.29 ≈
  pb/fat = 0.28. The MCP schema tax we removed was only 752 tok — a rounding
  error. Do not re-litigate transport.
- **The real cost is the RESIDENT project brain: 6–13K tok/turn**, compiled
  eagerly and re-read every turn. That is our 0.28, and it is what the article's
  lazy-loading pattern attacks. We optimized transport; we should optimize
  *when knowledge loads*.

The article's headline savings (GitHub MCP 55K at idle; Anthropic 150K→2K, 98.7%)
come from **lazy loading + code execution**, not from the shell per se. Two
mechanisms, and we under-use both:
1. **Lazy skill layering** — a tiny router (~30 tok/entry) that pulls a
   per-capability body in only when the task touches it. Load scales with what
   the agent does, not what it might do.
2. **CLI responses go into a pipe, not context** — the agent greps/filters/peeks,
   reads only what matters, instead of a tool return flooding the window.

## Map: the 3-layer pyramid onto assets we already have

| Layer | Article | What we already have | The gap to close |
| --- | --- | --- | --- |
| **CLI** (deterministic) | a binary the shell invokes, 0 schema | `brain-query.mjs` (concept→package+recipe+signatures), proven to trigger 12/12 | package it as `fonderie` bin; add pipe-friendly output (`--json`) so the agent filters before reading |
| **Skills** (lazy guidance) | base `SKILL.md` router (56 lines) → pull `atl-jira/SKILL.md` (368) on demand | **we ship the OPPOSITE**: the fat skill loads all 17 packages' signatures eagerly; the project brain compiles everything installed | **the lever** — restructure into a router + per-package bodies that load lazily |
| **LLM** (reasoning) | reasoning only | — | keep knowledge OUT of resident context; let the model pull it |

## The lever: turn the eager project brain into a lazy skill router

Today `generate-project-brain.mjs` emits one `CLAUDE.md` with the full
signatures of every installed package — resident every turn. Replace with the
article's layering:

- **`SKILL.md` router (~a few hundred tokens, always resident):** the 17 concept
  IDs + one-line descriptions (we already have this — the R2 concept enum is
  literally a routing table) + "for concept X, read `skills/fonderie/<pkg>.md`
  or run `fonderie brain query <concept>`."
- **Per-package bodies (loaded lazily, only when touched):** our existing
  per-package signature + outcomes fragments become `skills/fonderie/auth.md`,
  `billing.md`, … The agent reads `billing.md` only in the billing session — not
  auth's or teams'. This is `pb-scoped` (banked) taken to its logical end:
  scoping at *read time* by the agent, not compile time by us.
- **Security invariants stay in the router** (they are small and always apply).

Net: a session that touches 1–2 packages carries a router + 1–2 bodies, not all
17. Expected resident footprint drops from 6–13K toward ~1–3K for typical
sessions — the win the transport experiment could not deliver.

## Cross-vendor portability (a second, free win)

Adopt the open skill shape (name / description / body) so the same files work in
Claude Code, Codex CLI, Copilot CLI, Cursor, Gemini — the article's "the agent
does not care where it runs." List on skills.sh. One artifact, every harness, no
MCP server to keep alive. This is also our R1/Phase-3 install-matrix goal.

## The two-axis honesty (do not repeat the wrong-layer mistake)

The article is explicit and so are our own numbers: **tokens saved can cost wall
clock.** CLI steps (round trip per call); MCP streams. Our N=3 saw pb-cli make
more calls (45 vs 29). Playwright's own rule: **CLI for coding agents; MCP for
exploratory / self-healing / long-running loops.** So:
- Measure BOTH axes — resident tokens AND wall-clock — every arm.
- Keep MCP `brain-serve` as the supported option for the stateful/long-running
  case; ship lazy-skills+CLI as the default for the build-a-SaaS case (coding
  agent, deterministic).

## Pre-registered experiment (locked before data)

Condition **`pb-lazy`**: router `SKILL.md` + lazy per-package bodies + CLI
discovery; NO eager project brain, NO MCP. Same 4-session growing-app workload,
`claude-opus-4-8`, same completion gate and 39-point rubric.

- **Metrics:** resident tokens/turn (`instrument.mjs`), wall-clock/session,
  completion, quality.
- **Decision (locked):**
  - `pb-lazy` resident **< 0.28 × fat** at equal completion + quality, and
    wall-clock within ~1.5× of pb → **ship lazy-skills+CLI as default.**
  - resident saving but quality/completion drops (the agent doesn't pull the
    body it needs) → the eager brain was buying sufficiency; keep pb, refine the
    router.
  - resident saving but wall-clock blows up (too many lazy reads) → document as
    the low-token option, keep pb default for latency-sensitive users.
- **Cheap pilot first** (`pb-lazy` vs `pb`, sessions 1→3, ~$6); full N=3 only if
  the pilot clears the gate. Same discipline as Phase 4.1: n=1 is directional,
  never a verdict.

## Sequence

1. Extend `instrument.mjs` to also report **wall-clock/session** (the second
   axis) — it currently reports tokens only. [$0]
2. Build the router + lazy-body generator (`generate-skill.mjs`): emit
   `SKILL.md` (router) + `skills/fonderie/<pkg>.md` from the existing fragments.
   Wire a `pb-lazy` harness condition. [$0]
3. Run the pilot; decide per the locked rule.
4. If it wins: package the `fonderie` CLI bin + ship the skill in the open
   cross-vendor format (Phase 3 install matrix), MCP kept as the stateful option.

## What does NOT change

The micro-backend architecture — the deterministic, audited bricks that make an
LLM's non-deterministic output consistent — is the product and is untouched.
This plan only changes *how the brain's knowledge reaches the agent*: lazily,
through skills + a CLI, instead of eagerly resident. Same knowledge, loaded when
needed.
