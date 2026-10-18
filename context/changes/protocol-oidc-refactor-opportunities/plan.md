# C3: One Post-Authentication Client Check — Implementation Plan

## Overview

Consolidate the three copy-pasted `checkClient()` bodies in `protocol/oidc` (token endpoint, revocation endpoint, grant base class) into one helper in `AuthorizeClientUtil` that authenticates the client and rejects bearer-only clients. Each site delegates to it one at a time, keeping its own documented extras explicit. The change is behaviour-neutral and is guarded by new characterization tests in the new test framework (`tests/base`), because the legacy suite is frozen by CI.

This is opportunity **#2 (C3)** from `context/changes/protocol-oidc-refactor-opportunities/research.md`. The planning session narrowed this change to C3 only; C4 and C2 are not planned here.

Path abbreviations (same as the research): `P/` = `services/src/main/java/org/keycloak/protocol/oidc/`, `N/` = `tests/base/src/test/java/org/keycloak/tests/`, `L/` = `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/`.

## Current State Analysis

Three methods do the same post-authentication work:

| Site | Body | Site-specific extras |
|---|---|---|
| `P/endpoints/TokenEndpoint.java:216-235` (`private`, called `:161` for every grant except UMA and PreAuth) | `authorizeClient` → set `client`, `clientAuthAttributes`, `clientConfig` → `cors.checkAllowedOrigins` → bearer-only rejection | full-scope-allowed deprecation warning (`:229-233`) |
| `P/grants/OAuth2GrantTypeBase.java:283-294` (`protected`, 1 caller: `P/grants/PermissionGrantType.java:104`, only when no bearer token is sent) | same, without the warning | — |
| `P/endpoints/TokenRevocationEndpoint.java:187-199` (`private`, called `:99`) | `authorizeClient` → set `client` → `event.client(client)` → `cors.checkAllowedOrigins` → bearer-only rejection | `event.client(client)`; no `clientAuthAttributes`/`clientConfig` |

All three throw the identical bearer-only error: `CorsErrorResponseException(cors, OAuthErrorException.INVALID_CLIENT, "Bearer-only not allowed", Response.Status.BAD_REQUEST)`.

`AuthorizeClientUtil.authorizeClient` (`P/utils/AuthorizeClientUtil.java:48-88`) already checks client enabled (`:65`), allowed origins when `cors != null` (`:70-72`) and OIDC protocol (`:74-83`), and returns `ClientAuthResult` (`:125-141`). The origin re-check in each copy is therefore redundant: `DefaultCors.checkAllowedOrigins(session, client)` (`services/src/main/java/org/keycloak/services/cors/DefaultCors.java:91-97`) only re-resolves the same client's web origins and re-runs `checkOrigin()` (`:199-216`), which throws `ForbiddenException("Invalid origin")` — already thrown by the first call if it would fail.

History (research §1 C3): the copies were edited in lockstep in `4692aeee5d` (#45957) and the warning reached only one copy in `06e82d6976` (#52923) — the maintenance cost this change removes.

Test coverage today: revocation client auth only in `L/oauth/TokenRevocationTest` / `TokenRevocationCorsTest` (no bearer-only or wrong-secret case); bearer-only only in `L/oauth/AccessTokenTest.java:728-740` (ROPC, status only); the string `"Bearer-only"` appears in 0 `N/` files; no `N/` revocation-endpoint test class exists. CI rejects new files under `testsuite/` (`.github/actions/testsuite-deprecation-check/deprecation-check.sh:13-26`).

## Desired End State

- `AuthorizeClientUtil` has one helper that authenticates the client and rejects bearer-only clients with today's `invalid_client` / 400 / `"Bearer-only not allowed"`.
- `TokenEndpoint.checkClient`, `OAuth2GrantTypeBase.checkClient` and `TokenRevocationEndpoint.checkClient` each call that helper and contain only their site-specific extras. No copy re-runs `cors.checkAllowedOrigins`.
- `OAuth2GrantTypeBase.checkClient()` still exists as a `protected void` method with the same signature (out-of-tree subclasses may call or override it).
- New `N/oauth/` tests pin the Phase 1 matrix: bearer-only (status, `error`, `error_description`) at the token endpoint (ROPC, `client_credentials`), the UMA grant and the revocation endpoint; disallowed origin (403) and a valid request at the token and revocation endpoints; wrong secret at the revocation endpoint. They pass before Phase 2 and after every later phase, unchanged.

Verify: `grep -rln "Bearer-only not allowed" services/src/main/java/org/keycloak/protocol/oidc` returns exactly 3 files: `utils/AuthorizeClientUtil.java` (the helper), `endpoints/LogoutEndpoint.java` and `grants/ciba/endpoints/AbstractCibaEndpoint.java` (out-of-scope callers, untouched). The Phase 1 test classes pass at the final commit.

### Key Discoveries:

- `TokenEndpoint.checkClient()` is skipped for `UMA_GRANT_TYPE` and `PRE_AUTH_GRANT_TYPE` (`P/endpoints/TokenEndpoint.java:160-163`); the UMA grant calls the base copy itself, and only when no bearer token is present (`P/grants/PermissionGrantType.java:102-104`). A bearer-only UMA test must therefore send client credentials and no `Authorization: Bearer` header.
- The test helper already supports what the tests need: `oauth.origin(...)` (`tests/utils-shared/src/main/java/org/keycloak/testsuite/util/oauth/AbstractOAuthClient.java:76`, header added at `AbstractHttpPostRequest.java:66-68`), `tokenRevocationRequest`/`doTokenRevoke` (`AbstractOAuthClient.java:209-213`), `passwordGrantRequest` (`:109`), `permissionGrantRequest` (`:125`), and `getStatusCode()`/`getError()`/`getErrorDescription()` on responses (`AbstractHttpResponse.java:47-71`). No test-framework change is needed.
- Error JSON is parsed only for an exact `application/json` content type: `parseError()` (`AbstractHttpResponse.java:~99-104`) then fills `error` and `errorDescription`, so `getErrorDescription()` works for the bearer-only and wrong-secret JSON bodies. (The public `setErrorDescription` at `:75` is a no-op, but nothing depends on it.) A non-JSON body leaves both `null`. A disallowed origin produces a 403 from `ForbiddenException`, probably not as an OAuth JSON body, so assert its status only.
- Test class pattern to follow: `N/oauth/AudienceDisabledClientTest.java` (`@KeycloakIntegrationTest`, `@InjectRealm(config = …)`, `@InjectOAuthClient`, nested `RealmConfig` with `ClientBuilder…bearerOnly(true)`); events via `@InjectEvents` as in `N/oauth/RefreshTokenRevokeTest.java:74,100`.

## What We're NOT Doing

- **C4** (token-response finalization), including the dead PreAuthorizedCode KEK mapping and the `addCustomTokenResponseClaims` hook, is excluded. It stays a candidate for a later change.
- **C2** (typed issuance contract in `server-spi-private`) is excluded. Planning found that `responseBuilder` returns the `services`-only `TokenManager.AccessTokenResponseBuilder`, so C2 needs its own design.
- **C1, C5, C6, C7** stay rejected, as in the research.
- The other 7 `authorizeClient` callers (`LogoutEndpoint`, `TokenIntrospectionEndpoint`, `AbstractCibaEndpoint`, `AbstractParEndpoint`, `DeviceEndpoint`, `IdentityBrokerService`, `ClientsManagementService`) are not migrated. Their check sets differ, e.g. Device returns `unauthorized_client`/403. Aligning them is a protocol-policy question.
- The full-scope-allowed warning is not extended to revocation or UMA. It stays in `TokenEndpoint`, and whether that was intended remains a maintainer question (#52923).
- No error code, status, message or event changes, and no changes to the frozen `L/` suite.

## Implementation Approach

Characterize first, then add the helper next to the existing copies and migrate one site per phase: `OAuth2GrantTypeBase` (smallest, 1 caller), then `TokenRevocationEndpoint`, then `TokenEndpoint`. Until the last phase the helper keeps the repeated origin check, so every migration phase is a literal move of the shared lines. Phase 5 then deletes that repeat in one place. Each phase is one revertible commit, and the Phase 1 tests must pass unchanged after every phase.

## Critical Implementation Details

**Ordering in revocation.** Today `event.client(client)` runs before the origin re-check and the bearer-only rejection (`TokenRevocationEndpoint.java:191-198`). After delegation it runs after them. That is unobservable only if neither failure path emits an event. Neither throw calls `event.error(...)`, but Phase 1 must confirm it with an `@InjectEvents` assertion for the bearer-only revocation case. If an event *is* emitted with the client set, the helper must not move `event.client` behind the bearer-only check: let the revocation site set it from the helper's result before the check, e.g. by keeping the bearer-only check at the call site for revocation only. Re-plan that phase before proceeding.

**Field assignment order in `TokenEndpoint` and the base.** Today `clientAuthAttributes` and `clientConfig` are assigned before the bearer-only check. After the change they are assigned from the helper's result, after it. Both failure paths throw, and nothing reads those fields on the throw path, so the order is safe.

## Phase 1: Characterization tests

### Overview

Pin today's observable client-authentication behaviour of the three entry points in `N/oauth/`, against unchanged production code.

### Changes Required:

#### 1. Revocation endpoint client-auth tests

**File**: `tests/base/src/test/java/org/keycloak/tests/oauth/TokenRevocationClientAuthTest.java` (new)

**Intent**: Guard `TokenRevocationEndpoint.checkClient` before it is rewired. Today this is covered only by frozen legacy tests, and not at all for bearer-only or a wrong secret.

**Contract**: realm with a confidential client (with a web origin), a bearer-only client and a user. Cases:
- valid revoke of a refresh token: 200, and the token is unusable afterwards
- wrong client secret: today's status and `error`
- bearer-only client credentials: 400, `invalid_client`, `"Bearer-only not allowed"`, plus an `@InjectEvents` assertion recording whether any event is emitted
- `oauth.origin(...)` set to a disallowed origin: 403
- allowed origin: success

Assert the values the code returns today. Any value the research could not establish (e.g. the wrong-secret status) is read from the first run and then fixed in the assertion.

#### 2. Token endpoint and UMA bearer-only tests

**File**: `tests/base/src/test/java/org/keycloak/tests/oauth/ClientAuthBearerOnlyTest.java` (new)

**Intent**: Guard the `TokenEndpoint` and `OAuth2GrantTypeBase` copies.

**Contract**:
- Bearer-only client on the password (ROPC) grant and on `client_credentials`: 400, `invalid_client`, `"Bearer-only not allowed"`.
- Bearer-only client on `urn:ietf:params:oauth:grant-type:uma-ticket` via `permissionGrantRequest()`, with client credentials and no bearer token: same expectations. This needs `AUTHORIZATION` enabled on a resource-server client, following `N/authz/services/uma/UmaGrantTypeTest.java`.
- One disallowed-origin case on the token endpoint: 403.
- One valid confidential-client `client_credentials` request: 200.

### Success Criteria:

#### Automated Verification:

- New tests pass on unchanged production code: `./mvnw package -f tests/pom.xml -Dtest=TokenRevocationClientAuthTest,ClientAuthBearerOnlyTest`
- No file added or grown under `testsuite/`: `git diff --stat main -- testsuite/` is empty
- No production file changed in this phase: `git diff --stat HEAD~1 -- services/src/main` is empty

#### Manual Verification:

- Reviewer confirms each assertion records today's value, not a desired one, including the event outcome of the bearer-only revocation case (it decides the revocation ordering note above)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: Shared helper + `OAuth2GrantTypeBase`

### Overview

Add the helper and route the base class copy (1 caller, the UMA grant) through it.

### Changes Required:

#### 1. Helper

**File**: `P/utils/AuthorizeClientUtil.java`

**Intent**: One home for the post-authentication steps all three copies share.

**Contract**: new `public static ClientAuthResult` method taking `(KeycloakSession session, EventBuilder event, Cors cors)`. It calls `authorizeClient(session, event, cors)`, then `cors.checkAllowedOrigins(session, client)` (kept for now, removed in Phase 5), then throws the existing bearer-only `CorsErrorResponseException`, then returns the `ClientAuthResult`. The name must state the bearer-only rejection (e.g. `authorizeClientRejectBearerOnly`). Unlike `authorizeClient`, it requires a non-null `cors` (all 3 callers pass one): say so in its Javadoc and enforce it with `Objects.requireNonNull(cors, "cors")` as the first statement. The existing `authorizeClient` stays unchanged for the other 7 callers.

#### 2. Base class copy

**File**: `P/grants/OAuth2GrantTypeBase.java`

**Intent**: Delegate instead of duplicating, keeping the extension point for subclasses.

**Contract**: `protected void checkClient()` keeps its signature. Its body calls the helper and assigns `client`, `clientAuthAttributes` and `clientConfig` from the result.

### Success Criteria:

#### Automated Verification:

- services compiles: `./mvnw -pl services compile -q`
- Server distribution rebuilt from current services, so the tests below exercise this phase's code: `./mvnw install -DskipTests -pl quarkus/dist -am`, and `quarkus/dist/target/keycloak-*.zip` is newer than the last edit under `services/src/main` (the test framework reinstalls the server only when the zip's modification date changes, `DistributionKeycloakServer.java:347`)
- Phase 1 tests still pass: `./mvnw package -f tests/pom.xml -Dtest=TokenRevocationClientAuthTest,ClientAuthBearerOnlyTest`
- Existing UMA tests pass: `./mvnw package -f tests/pom.xml -Dtest=UmaGrantTypeTest`
- `checkClient` in `OAuth2GrantTypeBase` is still `protected void checkClient()` (ast-grep `protected void checkClient() { $$$ }` over `P/grants/` = 1)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 3: `TokenRevocationEndpoint`

### Overview

Route the revocation copy through the helper.

### Changes Required:

#### 1. Revocation copy

**File**: `P/endpoints/TokenRevocationEndpoint.java`

**Intent**: Delegate the shared steps, keeping revocation's own `event.client(client)`.

**Contract**: `private void checkClient()` calls the helper, assigns `client`, then calls `event.client(client)`. It does not read `clientAuthAttributes`/`clientConfig` (unchanged from today). If Phase 1 showed the bearer-only failure emits an event carrying the client, apply the fallback from "Critical Implementation Details" instead.

### Success Criteria:

#### Automated Verification:

- services compiles: `./mvnw -pl services compile -q`
- Server distribution rebuilt from current services: `./mvnw install -DskipTests -pl quarkus/dist -am`
- Phase 1 tests pass unchanged: `./mvnw package -f tests/pom.xml -Dtest=TokenRevocationClientAuthTest,ClientAuthBearerOnlyTest`
- Existing revocation-adjacent N/ tests pass: `./mvnw package -f tests/pom.xml -Dtest=RefreshTokenRevokeTest,OID4VCRefreshTokenRevocationTest`

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 4: `TokenEndpoint`

### Overview

Route the token endpoint copy through the helper.

### Changes Required:

#### 1. Token endpoint copy

**File**: `P/endpoints/TokenEndpoint.java`

**Intent**: Delegate the shared steps, keeping the site-specific field assignments and the full-scope deprecation warning local.

**Contract**: `private void checkClient()` calls the helper, assigns `client`, `clientAuthAttributes` and `clientConfig` from the result, then runs the existing full-scope-allowed warning block unchanged (same condition, same `LOGGER_FULL_SCOPE_ALLOWED.warnf` text, same admin-client exclusions).

### Success Criteria:

#### Automated Verification:

- services compiles: `./mvnw -pl services compile -q`
- Server distribution rebuilt from current services: `./mvnw install -DskipTests -pl quarkus/dist -am`
- Phase 1 tests pass unchanged: `./mvnw package -f tests/pom.xml -Dtest=TokenRevocationClientAuthTest,ClientAuthBearerOnlyTest`
- Broad token-endpoint regression in N/: `./mvnw package -f tests/pom.xml -Dtest='RefreshTokenTest,AuthorizationCodeTest,ClientAuthSecretSignedJWTTest,AudienceDisabledClientTest,DPoPTest'`
- The literal `"Bearer-only not allowed"` no longer appears in `TokenEndpoint.java`, `OAuth2GrantTypeBase.java` or `TokenRevocationEndpoint.java` (grep = 0 in those 3 files)

#### Manual Verification:

- Starting a local server with a full-scope-allowed confidential client and requesting a token still logs the deprecation warning once per request, with the same text

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 5: Drop redundant origin re-check

### Overview

Remove the now-single repeated `cors.checkAllowedOrigins(session, client)` call from the helper, as its own revertible step.

### Changes Required:

#### 1. Helper

**File**: `P/utils/AuthorizeClientUtil.java`

**Intent**: The repeat duplicates the check `authorizeClient` already performs on the same `Cors` (`:70-72`). `DefaultCors.checkAllowedOrigins` re-resolves the same origins and re-checks them (`DefaultCors.java:91-97`), so it can never fail where the first call passed.

**Contract**: the helper's body is `authorizeClient` → bearer-only rejection → return. Add a one-line comment that `authorizeClient` already enforces allowed origins when `cors` is non-null. The three callers always pass the endpoint's non-null `cors`.

### Success Criteria:

#### Automated Verification:

- services compiles: `./mvnw -pl services compile -q`
- Server distribution rebuilt from current services: `./mvnw install -DskipTests -pl quarkus/dist -am`
- Phase 1 tests pass unchanged, including both disallowed-origin cases: `./mvnw package -f tests/pom.xml -Dtest=TokenRevocationClientAuthTest,ClientAuthBearerOnlyTest`
- No `cors.checkAllowedOrigins(session, client)` remains in the helper or the 3 former copies (ast-grep `cors.checkAllowedOrigins($$$)` over those 4 files: the only match is the one inside `authorizeClient` itself)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Testing Strategy

### Unit Tests:

- None. `services` has no Mockito, and a helper unit test would need a real session/factory, which adds setup without coverage beyond the integration tests (decided in planning).

### Integration Tests:

- `N/oauth/TokenRevocationClientAuthTest` covers bearer-only, wrong secret, disallowed origin, allowed origin and valid revoke.
- `N/oauth/ClientAuthBearerOnlyTest` covers bearer-only on ROPC, `client_credentials` and the UMA ticket, plus a disallowed origin and a valid request.
- Existing `N/` suites named per phase serve as regression.

### Manual Testing Steps:

1. Phase 1: review that the assertions record today's values and note the bearer-only revocation event outcome.
2. Phase 4: confirm the full-scope-allowed warning still logs at the token endpoint.

Prerequisite for every `tests/pom.xml` run: a server distribution built from the current `services` code, e.g. `./mvnw install -DskipTests -pl quarkus/dist -am`. This command is inferred and not in the docs. Ports 8080, 8443 and 9001 must be free. `KC_TEST_SERVER=embedded` speeds up local iteration (`test-framework/docs/RUNNING_TESTS.md:12-43`).

## Performance Considerations

Phase 5 removes one extra `WebOriginsUtils.resolveValidWebOrigins` call per request at the token and revocation endpoints and the UMA grant. That is a minor gain, not a goal.

## Migration Notes

None. No data, configuration or wire-contract change. `OAuth2GrantTypeBase.checkClient()` keeps its `protected` signature for out-of-tree `OAuth2GrantType` subclasses (internal SPI, `isInternal() = true`).

## References

- Research: `context/changes/protocol-oidc-refactor-opportunities/research.md` (§1 C3, §2 #2, claim verification #11–#16)
- Source analysis: `context/changes/analyze-protocol-oidc/research.md` (TD-3a, TD-4)
- `P/utils/AuthorizeClientUtil.java:48-88,125-141`; `P/endpoints/TokenEndpoint.java:160-163,216-235`; `P/grants/OAuth2GrantTypeBase.java:283-294`; `P/grants/PermissionGrantType.java:102-104`; `P/endpoints/TokenRevocationEndpoint.java:99,187-199`
- `services/src/main/java/org/keycloak/services/cors/DefaultCors.java:91-97,199-216`
- Test pattern: `N/oauth/AudienceDisabledClientTest.java`, `N/oauth/RefreshTokenRevokeTest.java:74,100`, `N/authz/services/uma/UmaGrantTypeTest.java`
- CI freeze: `.github/actions/testsuite-deprecation-check/deprecation-check.sh:13-26`

## Progress

> Convention: `- [ ]` pending, `- [x]` done. Append ` — <commit sha>` when a step lands. Do not rename step titles. See `references/progress-format.md`.

### Phase 1: Characterization tests

#### Automated

- [ ] 1.1 New tests pass on unchanged production code
- [ ] 1.2 No file added or grown under testsuite/
- [ ] 1.3 No production file changed in this phase

#### Manual

- [ ] 1.4 Reviewer confirms assertions record today's values, including bearer-only revocation event outcome

### Phase 2: Shared helper + OAuth2GrantTypeBase

#### Automated

- [ ] 2.1 services compiles
- [ ] 2.2 Server distribution rebuilt from current services
- [ ] 2.3 Phase 1 tests still pass
- [ ] 2.4 Existing UMA tests pass
- [ ] 2.5 OAuth2GrantTypeBase.checkClient is still protected void checkClient()

### Phase 3: TokenRevocationEndpoint

#### Automated

- [ ] 3.1 services compiles
- [ ] 3.2 Server distribution rebuilt from current services
- [ ] 3.3 Phase 1 tests pass unchanged
- [ ] 3.4 Existing revocation-adjacent N/ tests pass

### Phase 4: TokenEndpoint

#### Automated

- [ ] 4.1 services compiles
- [ ] 4.2 Server distribution rebuilt from current services
- [ ] 4.3 Phase 1 tests pass unchanged
- [ ] 4.4 Broad token-endpoint regression in N/
- [ ] 4.5 "Bearer-only not allowed" literal gone from the 3 former copies

#### Manual

- [ ] 4.6 Full-scope-allowed warning still logged at the token endpoint

### Phase 5: Drop redundant origin re-check

#### Automated

- [ ] 5.1 services compiles
- [ ] 5.2 Server distribution rebuilt from current services
- [ ] 5.3 Phase 1 tests pass unchanged, including disallowed-origin cases
- [ ] 5.4 No repeated cors.checkAllowedOrigins in helper or former copies
