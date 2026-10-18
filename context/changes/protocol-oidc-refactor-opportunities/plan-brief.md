# C3: One Post-Authentication Client Check — Plan Brief

> Full plan: `context/changes/protocol-oidc-refactor-opportunities/plan.md`
> Research: `context/changes/protocol-oidc-refactor-opportunities/research.md`

## What & Why

Three endpoints in `protocol/oidc` carry copy-pasted `checkClient()` bodies: authenticate the client, re-check allowed origins, reject bearer-only clients. They have already had to be edited in lockstep (`4692aeee5d`), and one later addition reached only one copy (`06e82d6976`). This change folds them into one helper. It is ranked #2 in the research, and it is the cheapest and most reversible of the opportunities.

## Starting Point

- The copies live in `TokenEndpoint.java:216-235`, `OAuth2GrantTypeBase.java:283-294` (used only by the UMA grant) and `TokenRevocationEndpoint.java:187-199`.
- `AuthorizeClientUtil.authorizeClient` already checks enabled, origins and protocol, so the origin re-check in each copy is redundant.
- No new-framework (`tests/base`) test covers bearer-only or revocation client authentication, and the legacy suite is frozen by CI.

## Desired End State

One helper in `AuthorizeClientUtil` authenticates the client and rejects bearer-only clients with today's `invalid_client`/400. The three sites call it and keep only their own extras. No copy re-checks origins. New `N/oauth/` tests pin the behaviour, and they pass unchanged before and after.

## Key Decisions Made

| Decision | Choice | Why (1 sentence) | Source |
| --- | --- | --- | --- |
| Opportunity order / candidates | C3, C4, C2 ranked; C1, C5–C7 rejected | Accidental debt with an existing abstraction and bounded blast radius | Research |
| Scope of this plan | **C3 only**; C4 and C2 not planned | User choice; C2 also turned out larger (builder type lives in `services`) | Plan |
| Full-scope warning | Stays in `TokenEndpoint` only | Keeps C3 behaviour-neutral; intent of #52923 is a maintainer question | Plan |
| Redundant origin re-check | Dropped, as the last, separate step | Idempotent with `authorizeClient`'s own check; isolated for an easy revert | Plan |
| Other 7 `authorizeClient` callers (incl. Device 403) | Untouched | Each enforces a different subset; aligning them is a policy change | Plan (research scope note) |
| Characterization tests | All 3 entry points in `N/oauth/`, no unit test | Every migrated site gets a guard; `services` has no Mockito | Plan |
| `OAuth2GrantTypeBase.checkClient()` | Kept `protected`, delegating | Out-of-tree subclasses may call or override it | Research |

## Scope

**In scope:**
- a new helper in `AuthorizeClientUtil`
- delegation from the 3 copies
- removing the repeated origin check
- 2 new `N/oauth/` test classes

**Out of scope:**
- C4 (incl. the dead PreAuth KEK mapping and the custom-claims hook), C2, C1, C5–C7
- the other 7 `authorizeClient` callers
- extending the full-scope warning
- any error, status or event change
- the legacy `testsuite/`

## Architecture / Approach

Characterize first. Then add the helper next to the copies; it initially keeps the origin repeat, so each migration is a literal move. Migrate one site per phase (base → revocation → token endpoint), and finally delete the repeat in its one remaining place. Every phase is one revertible commit, and the Phase 1 tests must pass unchanged throughout.

## Phases at a Glance

| Phase | What it delivers | Key risk |
| --- | --- | --- |
| 1. Characterization tests | `TokenRevocationClientAuthTest`, `ClientAuthBearerOnlyTest` passing on today's code | Asserting desired instead of actual values; UMA realm setup |
| 2. Helper + `OAuth2GrantTypeBase` | Helper added; base `checkClient` delegates | Breaking the `protected` extension point |
| 3. `TokenRevocationEndpoint` | Revocation delegates, keeps `event.client` | `event.client` now runs after bearer-only; safe only if no failure event (checked in Phase 1) |
| 4. `TokenEndpoint` | Token endpoint delegates, keeps warning and fields | Warning text or condition drifting |
| 5. Drop origin re-check | Repeat removed from the helper | A caller passing a different or null `cors` (none today) |

**Prerequisites:** a server distribution built from current `services` (`./mvnw install -DskipTests -pl quarkus/dist -am`, inferred); ports 8080, 8443 and 9001 free.
**Estimated effort:** ~2 sessions. Phase 1 is the bulk; Phases 2–5 are small diffs (about 4 main files total).

## Open Risks & Assumptions

- The wrong-secret status and the bearer-only revocation event outcome were not established by the research. Phase 1 records them, and the event outcome decides whether Phase 3 needs its fallback.
- The `tests/base` error parsing needs an exact `application/json` content type. `error` and `error_description` are then available through `parseError()`. A non-JSON body (likely the 403 for a disallowed origin) leaves both `null`, so assert only its status.
- The build command for the distribution is inferred, not documented.

## Success Criteria (Summary)

- A future change to post-auth client checks touches one method, not three.
- No client sees a different status, error or event at the token endpoint, the revocation endpoint or the UMA grant (proven by the Phase 1 tests).
- Bearer-only and revocation client authentication are guarded in the non-frozen test suite.
