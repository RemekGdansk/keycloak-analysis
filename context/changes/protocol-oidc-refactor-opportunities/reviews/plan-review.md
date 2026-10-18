<!-- PLAN-REVIEW-REPORT -->
# Plan Review: C3: One Post-Authentication Client Check

- **Plan**: context/changes/protocol-oidc-refactor-opportunities/plan.md
- **Mode**: Deep (claims checked inline)
- **Date**: 2026-10-18
- **Verdict**: REVISE → SOUND after triage
- **Findings**: 1 critical, 1 warning, 2 observations

## Verdicts

| Dimension | Verdict |
|-----------|---------|
| End-State Alignment | WARNING |
| Lean Execution | PASS |
| Architectural Fitness | WARNING |
| Blind Spots | FAIL |
| Plan Completeness | WARNING |

## Grounding
8/8 paths ✓, 9/9 symbols ✓ (line ranges match), 8/8 regression test classes exist, Progress↔Phases ✓, brief↔plan ✓

Confirmed in code: the 3 copies match the plan's table; `authorizeClient` already calls `cors.checkAllowedOrigins(session, client)` on the same `Cors` (`AuthorizeClientUtil.java:70-72`); the UMA grant reaches `checkClient()` before any ticket handling when no Bearer header is sent (`PermissionGrantType.java:102-104`); revocation has no catch that calls `event.error` on the `checkClient` path.

## Findings

### F1 — Phase 2–5 guard tests run against a stale server

- **Severity**: ❌ CRITICAL
- **Impact**: 🏃 LOW — quick decision; fix is obvious and narrowly scoped
- **Dimension**: Blind Spots
- **Location**: Phases 2–5, Success Criteria
- **Detail**: Each phase builds only with `./mvnw -pl services compile`. The test server is `quarkus/dist/target/keycloak-<ver>.zip` (`DistributionKeycloakServer.java:347`), which is reinstalled only when the zip's date changes. Without a rebuild, the "Phase 1 tests pass unchanged" step tests the Phase 1 build and passes no matter what.
- **Fix**: Add "Server distribution rebuilt from current services" (`./mvnw install -DskipTests -pl quarkus/dist -am`) to Phases 2–5 before the tests, with a zip-timestamp check in Phase 2.
- **Decision**: FIXED

### F2 — End state promises more tests than Phase 1 writes

- **Severity**: ⚠️ WARNING
- **Impact**: 🏃 LOW — quick decision; fix is obvious and narrowly scoped
- **Dimension**: End-State Alignment
- **Location**: Desired End State bullet 4 + Verify line
- **Detail**: The end state promised the full case matrix (wrong secret, origin, valid) at all 3 entry points, but Phase 1 covers less. The Verify grep would also hit `LogoutEndpoint.java:747` and `AbstractCibaEndpoint.java:55`.
- **Fix**: Narrow bullet 4 to the Phase 1 matrix and list the exact expected grep hits.
- **Decision**: FIXED

### F3 — New public helper doesn't say what happens when cors is null

- **Severity**: OBSERVATION
- **Impact**: 🏃 LOW — quick decision; fix is obvious and narrowly scoped
- **Dimension**: Architectural Fitness
- **Location**: Phase 2 helper contract; Phase 5
- **Detail**: `authorizeClient` accepts a null `cors`, but the new helper would throw an NPE (Phases 2–4) or build a `CorsErrorResponseException` with a null cors (after Phase 5).
- **Fix**: Phase 2 contract requires non-null `cors` (Javadoc + `Objects.requireNonNull`).
- **Decision**: FIXED

### F4 — "setErrorDescription sets nothing" is a red herring

- **Severity**: OBSERVATION
- **Impact**: 🏃 LOW — quick decision; fix is obvious and narrowly scoped
- **Dimension**: Plan Completeness
- **Location**: Key Discoveries bullet 3; brief Open Risks
- **Detail**: `parseError()` fills `errorDescription` directly from the JSON, so `getErrorDescription()` works for JSON error bodies. The note could have led the implementer to drop the description assertions.
- **Fix**: Reworded in plan.md and plan-brief.md.
- **Decision**: FIXED
