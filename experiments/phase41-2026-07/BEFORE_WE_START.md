# Before we start — Phase 4.1 pre-flight

Do these before spending the ~$30–60 full batch. Each one, if skipped, either
**invalidates** the verdict or **pollutes** the token comparison. `analyze.mjs`
already refuses a verdict while any blocker stands, so an un-prepped batch just
burns money for an INSUFFICIENT.

Ordered by how much it hurts to skip.

## 1. Quality scoring — BLOCKER (build before spending)

The decision rule is "pb overhead ≤ ⅓ of fat **at equal quality**
(checklist ≥ 11/12)." Today the harness records only `loc` and `tsc`; **nothing
scores the checklist**. Without it a cheaper condition that ships worse code
reads as a win, and the cost comparison is meaningless — `analyze.mjs` will (by
design) return INSUFFICIENT for every run.

- **Do:** port the checklist from `../token-cost-2026-07/CHECKLIST.md` into a
  scorer, or define a manual rubric applied **during** each session (not
  reconstructed later). Record the score into each session's `meta.json` as
  `checklist` (e.g. `"checklist": 11`) — the field `analyze.mjs` reads.
- **Why during, not after:** scoring 36 transcripts weeks later is unreliable
  and un-blinded; score at run time while the app is in front of you.
- **R4 tie-in:** anchor checklist items to OWASP ASVS where applicable, so the
  quality bar is external, not self-defined.

## 2. Environment stability — HIGH (Docker Postgres now available)

`fat-1-s1`'s own transcript: *"Docker's host port-forwarding is broken… installing
a native Postgres via Homebrew."* That session burned turns fighting infra —
noise, not signal, and it varies per session, inflating variance **unequally**
across conditions and corrupting the token comparison.

- **Do:** stand up ONE stable Postgres before the batch and point every session
  at it via a fixed connection string, so all conditions face identical infra.
  **Docker access is now available** — spin up e.g.
  `docker run -d --name fonderie-bench-pg -e POSTGRES_PASSWORD=postgres -p 5432:5432 postgres:16`
  and export the URL the skeletons expect. (Ask the operator to start it; the
  harness should assume it is already up, never provision per session.)
- **Alternative:** stub verification to a fixed harness so DB effort is constant
  across conditions. Either way the goal is: **equal, quiet infra for all runs.**

## 3. Confirm `pb` actually triggers the brain — HIGH (fold into the pilot)

In `pb`, discovery of not-yet-installed capabilities (sessions 2–4) depends on
the model calling `brain_query`. If it improvises or reads files instead, the pb
overhead is polluted *and* quality drops — a silently broken pb.

- **Do:** the 2-session pilot must be `pb` **sessions 1 → 2** (session 1 = auth is
  already installed → no discovery; session 2 = billing is the first discovery
  moment). After it, assert `results/pb-<seq>-s2.brain.log` is **non-empty**.
- Pairs with a `fat` 1 → 2 pilot for the first real pb/fat ratio via `analyze.mjs`.

## 4. Verify `pb` installs like a real user — MEDIUM

Now that the packages are published with co-located fragments, confirm the `pb`
condition installs from **npm** (real fragments/versions) rather than the local
monorepo workspace — otherwise the benchmarked brain may not match what ships.
Check `run-sequence.sh`'s install step before the batch.

## Readiness gate

Do **not** launch the full 36-session batch until:

- [ ] #1 quality scorer/rubric in place, writing `checklist` to meta
- [ ] #2 one stable Postgres up, all sessions pointed at it (Docker ready)
- [ ] #3 pilot run `pb` 1→2 + `fat` 1→2, `brain.log` non-empty, `analyze.mjs` clean
- [ ] #4 install path confirmed (npm vs workspace)

Then: `./stage41.sh` (resumable) → `node analyze.mjs` → decision per the locked
pre-registration in `BRAIN_PLAN.md § Phase 4.1`. A gate that fails twice ends
the phase; the rule is not renegotiated.
