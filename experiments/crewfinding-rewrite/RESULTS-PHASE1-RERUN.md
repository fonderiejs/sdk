# Phase-1 oracle re-run — against the published, fixed packages

Re-ran the contract oracle against a backend built on the **published** SDK
(post-release: `onResponse` config, the cookie fix) with the "adopter" additions
a real app would write: an `onResponse` that flattens Fonderie's envelope to
crewfinding's flat shapes, and a thin path-alias shim. Measures how far the
shipped fixes closed the Phase-1 gap.

## Result: 25 / 25 — GREEN (was 6 / 25 vanilla)

Two stages, both against the **published** SDK:
1. **SDK config alone (`onResponse` + the cookie fix): 18 / 25.** Every
   path-matching endpoint (register/login/refresh/logout) went fully green —
   flat `{user,accessToken,refreshToken}`, `IUserDTO`, and `access_token`/
   `refresh_token` **HttpOnly cookies now arrive**. Those were the two biggest
   Phase-1 divergences, closed with one config option and zero handler edits.
2. **+ a ~15-line contract shim → 25 / 25.** The residual was pure route-naming;
   a thin express middleware rewriting crewfinding's paths (and one method) to
   Fonderie's closes it completely.

| endpoint | vanilla | final |
|---|---|---|
| register, login, refresh | ✗ | ✓ flat shape + `IUserDTO` + **cookies** |
| logout, forgot | ✗ | ✓ `{ ok: true }` |
| reset, verify (contract shape) | ✗ | ✓ |
| GET /users/me → `GET /users` | ✗ | ✓ (path alias) |
| PATCH /users/me → `PUT /users/profile` | ✗ | ✓ (method+path alias) |
| workspaces | skip | skip |

**The complete adopter integration** (in `backend/`):
- **`onResponse`** — flatten Fonderie's `{reason,explanation,result}` envelope to
  crewfinding's shapes, incl. mapping the user (`result.user` → flat `IUserDTO`,
  `profileImageUrl`→`avatarUrl`). *SDK config, no handler edits.*
- **path/method shim** — ~15 lines of express middleware:
  `/users/me`→`/users`, `PATCH /users/me`→`PUT /users/profile`,
  `/auth/forgot-password`→`/auth/email/forgot`, etc.
- **cookies** — automatic (SDK fix #55/#56).

**Verdict: Outcome A is achievable.** An existing frontend works **unchanged**
against a Fonderie-rebuilt backend with (a) one SDK config option and (b) a thin,
app-owned contract shim — no fork of the SDK, no frontend edits. The Phase-1
"needs a full adapter" gap is now "needs a config callback + ~15 lines."

### Two bugs this closure surfaced
- The 18→25 gap was initially masked by a **`docker cp` nesting mistake** (the
  shim `index.ts` landed in `/app/src/src/`, so the container ran an alias-less
  build) — a harness error, not a code one; the shim logic was always correct.
- `updateProfile`/`me` nest the user under `result.user`; the `onResponse`
  mapper had to read `r.user ?? r` (one-line fix).

## 🚨 Release defect found (more important than the score)

The re-run could not install the published packages cleanly. Root cause:

- **`events@2.0.0` and `customers@2.0.0` shipped wrong peer ranges** —
  `@fonderie/core@^1.0.0` / `store@^1.0.0`, but those are `0.2.0` / `0.1.2` →
  `npm install` fails with `ERESOLVE`.
- **`events@2.0.0`'s tarball is missing its migration SQL** (`dist/migrations/sql`
  absent) → won't boot.

Cause: both were published from an **earlier partial release** built when
`core`/`store` were assumed to be `1.0.0`; they were never rebuilt, and moving
`latest` onto them (the dist-tag repair) exposed the bad builds. The **current
source is correct** (`core@^0.2.0`, `store@^0.1.1`) and a fresh build ships the
SQL. Fix: republish both as **2.0.1** (changeset `republish-events-customers`).

## Honest caveats
- After the events/customers `2.0.1` republish, a clean install *still* needs
  `--legacy-peer-deps` because **`@fonderie/rate-limit@1.0.0` has the same broken
  peer ranges** (`core@^1.0.0`) — a third stale build from the partial release
  that the first fix missed. The republished events `2.0.1` ships its SQL cleanly
  (no hand-copy needed). rate-limit needs the same `1.0.1` republish.
- Ran fully in-docker (host↔container networking is broken in this environment),
  so it's a faithful *contract* run, still a proxy for the live frontend.

---

## Re-run #2 — against the published 3.0.0 SDK (2026-07-24)

Re-ran the oracle against the **freshly published whole-SDK 3.0.0 release**
(the email-templates + localization + theme work; `core@0.3.0`, `store@0.1.2`,
everything else `3.0.0`, `rate-limit@2.0.0`).

**Result: 25/25 assertions pass, 0 failed** (11 requests; workspace GET/PUT
green-skipped without `workspace_id`, same as before).

**The release defect from re-run #1 is gone.** `npm install @fonderie/*@latest`
resolved **cleanly in-container — no `ERESOLVE`, no `--legacy-peer-deps`**:

- `@fonderie/core 0.3.0`, `auth 3.0.0`, `events 3.0.0`, `workspaces 3.0.0`,
  `store 0.1.2`, `adapter-express 3.0.0` — all installed and booted.
- Backend applied every migration (events, auth ×13, workspaces ×3) and reached
  `LISTENING` first try; the `events` migration SQL ships (the missing-SQL bug
  from re-run #1 is fixed).

Peer ranges are internally consistent across the 3.0.0 cascade — every new
`^3.0.0` / `^0.3.0` reference matches the version its target bumped to. Ran fully
in-docker on a private network (`cf-net`/`cf-pg`/`cf-app`) since host↔container
networking is broken in this environment — a faithful contract run.

**Verdict unchanged and now clean: Outcome A holds on the published SDK.** An
existing frontend works unchanged against a Fonderie-rebuilt backend via one
`onResponse` config option + a ~15-line path shim, installed from npm with a
plain `npm install`.

### Broken builds deprecated on npm (2026-07-24)

The three stale artifacts from the partial release (the defect above) are now
**`npm deprecate`d** so anyone who explicitly pins them gets steered to the fix,
while `latest` stays warning-free:

| Deprecated | Reason | Message points to |
| --- | --- | --- |
| `@fonderie/events@2.0.0` | `core@^1.0.0`/`store@^1.0.0` peers → ERESOLVE, **and** missing migration SQL | `2.0.1` |
| `@fonderie/customers@2.0.0` | `core@^1.0.0`/`store@^1.0.0` peers → ERESOLVE | `2.0.1` |
| `@fonderie/rate-limit@1.0.0` | `core@^1.0.0`/`store@^1.0.0` peers → ERESOLVE | `1.0.1` |

Scope verified: only these three versions carry the flag; the replacements
(`2.0.1`/`2.0.1`/`1.0.1`) and the current release (`events`/`customers` `3.0.0`,
`rate-limit` `2.0.0`) are clean, and every `latest` dist-tag is untouched.
Confirmed on install — `npm install @fonderie/events@2.0.0` prints the
`npm warn deprecated …` notice; `@2.0.1` and `@latest` install silently. This is
the version-level cleanup only; the scope-wide `@fonderie`→`@fonderiejs`
deprecation (MIGRATION-FONDERIEJS.md checklist item 8) stays deferred until the
1.0.0 migration.
