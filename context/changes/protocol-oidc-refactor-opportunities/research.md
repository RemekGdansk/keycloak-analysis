---
date: 2026-10-18T12:18:04+02:00
researcher: Claude (Opus 5.5) for RemekGdansk
git_commit: eac37c680040cc42e3920e542df4d93afed789e1
branch: context-map
repository: RemekGdansk/keycloak-analysis
topic: "protocol/oidc: which recorded debt items are worth refactoring, in what target shape and in what order"
tags: [research, refactor-opportunities, oidc, token-manager, grants, spi, client-auth, token-response, verified]
status: complete
last_updated: 2026-10-18
last_updated_by: Claude (Opus 5.5)
verified_commit: eac37c680040cc42e3920e542df4d93afed789e1
last_updated_note: "38 structural claims checked with ast-grep 0.51.0, every ast-grep zero backed by grep (see 'Claim verification (ast-grep)'). 0 refuted; corrections in place as 'X (report: Y)'. The ranking and the intent verdicts are unchanged."
---

# Research: refactor opportunities in `protocol/oidc`

**Date**: 2026-10-18T12:18:04+02:00
**Researcher**: Claude (Opus 5.5) for RemekGdansk
**Git Commit**: eac37c680040cc42e3920e542df4d93afed789e1
**Branch**: context-map
**Repository**: RemekGdansk/keycloak-analysis

## Research Question

`context/changes/analyze-protocol-oidc/research.md` (below: **the analysis**) records the technical debt and structural risks of token issuance and sessions in `protocol/oidc`. It deliberately did not say which of them to fix. This report answers that:

- **Which** recorded problems change code structure (candidates)?
- For each candidate: the current shape, why the code has that shape, and whether an incremental, reversible migration is feasible.
- A ranking of the 2–3 strongest refactor opportunities, plus the rejected ones and why.

This is exploration only. No code was changed, and no decision is made here; the ranking is input for a separate planning session.

**How to read this document.** The tags, abbreviations and conventions are the analysis's own:

- **[E]** evidence (read at the cited `file:line`, a commit, or a count from a stated command)
- **[I]** inference
- **[U]** unknown, with the reason

Path abbreviations:

| Abbreviation | Path |
|---|---|
| `S/` | `services/src/main/java/org/keycloak/` |
| `P/` | `S/protocol/oidc/` |
| `SPI/` | `server-spi-private/src/main/java/org/keycloak/` |
| `N/` | `tests/base/src/test/java/org/keycloak/tests/` (new framework) |
| `L/` | `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/` (legacy, frozen) |
| `U/` | `services/src/test/java/org/keycloak/` |

Commit SHAs are local short hashes. Upstream PR and issue numbers (`#NNNNN`) refer to `keycloak/keycloak` on GitHub, read with `gh`.

**Method.**

- The analysis's findings were taken as given evidence and were not derived again. The repo map (`context/map/repo-map.md`) was used as a prior.
- Three read-only sub-agents each investigated one dimension across all candidates: current shape, history and intent, migration feasibility.
- I re-checked their decisive claims myself:
  - the KEK message mismatch
  - the CI legacy-suite freeze
  - the `TODO: resolve deps issue` comment in `3b3a61dfba`
  - the dates and subjects of the key commits
  - that `TokenManager` has no instance state

---

## 0. Problem inventory and classification (audit this first)

**A CANDIDATE** is a problem whose fix would change code structure: where logic lives, types, dependencies or abstractions. Everything else is a **non-candidate**. Non-candidates are kept as inputs to the feasibility and cost assessment.

| # | Source | Problem as recorded | Class | Why |
|---|---|---|---|---|
| TD-1 | analysis §2 | `TokenManager` is a 1769-line god class with no interface, created with `new` at 11 sites | **CANDIDATE → C1** | The fix moves responsibilities and introduces a seam |
| TD-2 | analysis §2 | The SPI contexts type `tokenManager` as `Object`; 2 downcasts | **CANDIDATE → C2** | The fix changes a type contract across modules |
| TD-3a | analysis §2 | `checkClient()` is duplicated 3×; 10 `authorizeClient` callers each do their own post-auth checks | **CANDIDATE → C3** | The fix consolidates logic into one place |
| TD-3b | analysis §2 | Response finalization is split: `createTokenResponse` serves 5 grants, while ROPC, PreAuth, Refresh, the exchange providers and implicit each finish on their own | **CANDIDATE → C4** | The fix moves finalization into one shared step |
| TD-3c | analysis §2 | The `addCustomTokenResponseClaims` hook has 0 overrides | folded into **C4** | Its fate depends on the C4 target shape |
| TD-4 | analysis §2 | Almost no unit tests; code-exchange error paths, introspection, lightweight tokens and HoK are tested in legacy only | non-candidate (test gap) | **Input:** the safety net every candidate needs |
| TD-5 | analysis §2 | 8 branches with no test found by message search | non-candidate (test gap) | **Input:** characterization prerequisite for C1/C3/C4 |
| TD-6 | analysis §2 | Refresh failure tests can't tell branches apart | non-candidate (test assertion gap) | **Input:** the C4 refresh step needs sharper assertions |
| TD-7 | analysis §2 | Probably unreachable `"Unmatching clients"` check (`P/TokenManager.java:259-260`) | non-candidate | Removing or keeping one guard is a maintainer decision about behaviour, not structure |
| TD-8 | analysis §2 | The code exchange checks user-session validity only, not client-session validity | non-candidate (behaviour/security question) | Adding a check is a behaviour change, open with maintainers |
| TD-9 | analysis §2 | Transient DPoP/MTLS mappers rely on an implicit "implements all 5 interfaces" contract | **CANDIDATE → C5** | The fix changes how the mapper pipeline is typed or filtered |
| TD-10a | analysis §2 | Hidden runtime coupling: SPI selection by id or `order()`/`supports()` | **CANDIDATE → C6** | A fix would change the selection mechanism |
| TD-10b | analysis §2 | Dependency cycle `OrganizationScope` ↔ `TokenManager` | **CANDIDATE → C7** | The fix changes the dependency direction |
| TD-10c | analysis §1.6/§2 | The JS admin client and admin-ui depend on the JSON shape of the token response | non-candidate (wire contract) | A constraint every candidate must keep, not something to restructure |
| TD-11 | analysis §2 | Three unrelated `TokenManager` types | non-candidate (naming) | A rename changes names, not structure. **Input:** it constrains how a C1/C2 interface could be named |
| RM-1 | repo map §3/§4 | 198-file runtime knot in `services` (`AuthenticationManager` hub) | out of scope (context) | It reaches well beyond `protocol/oidc` and needs its own analysis. Kept as context for C1 |
| RM-2 | repo map §6 vs analysis TD-4 | The reading list's "tested without mocks" claim is only partly supported | non-candidate (documentation gap) | — |
| Unk | analysis "Unknowns" | Runtime provider selection, storage behaviour, true coverage (no JaCoCo) | non-candidate (unknowns) | **Input:** the "unknown" cells below |

**7 candidates: C1–C7.** The sub-agents also turned up new findings. These are not separate candidates; each is recorded under the candidate it belongs to:

- **PreAuthorizedCode KEK mapping is dead** (under C4).
  - `P/grants/PreAuthorizedCodeGrantType.java:272-273` compares the exception message with `"Cannot get encryption KEK"`.
  - Both throwers use `"can not get encryption KEK"` (`S/jose/jws/DefaultTokenManager.java:288`, `P/endpoints/UserInfoEndpoint.java:399`).
  - So the 400 mapping never fires [E, re-read]. This is a **defect**, not a refactor, and fixing it is out of scope for this exploration.
- **The origin check is redundant** (under C3). `AuthorizeClientUtil.authorizeClient` already calls `cors.checkAllowedOrigins` when `cors != null` (`P/utils/AuthorizeClientUtil.java:70-72`), and all 3 `checkClient` copies call it again [E].
- **`OAuth2GrantType.Context` has a second `Object` field**, `clientConfig` (under C2): `SPI/protocol/oidc/grants/OAuth2GrantType.java:102` (report: `:101`), cast at `P/grants/OAuth2GrantTypeBase.java:107` [E]. The analysis said 1.
- **The refresh provider is looked up twice**, with the same order/supports pipeline (under C6): `P/TokenManager.java:1347-1356` and `P/grants/RefreshTokenGrantType.java:84-91` [E, re-read].
- **The Organization coupling in `TokenManager` is wider than `valueOfScope`** (under C7). It also covers `OrganizationMembershipMapper.PROVIDER_ID` (`:629,672` (report: `:628,672`)), `Organizations.isEnabled`/`getProvider` (`:671,709,798,1743,1759` (report: `:670,709,797,1759`)) and `validateSelectedOrganization` (`:1742`) [E].

## Summary

- **Ranking (proposal):**
  1. **C4: one token-response finalization step.** The drift is already visible and has a concrete cost: missing RAR sanitizing, a dead KEK mapping, uneven MTLS/policy-event handling.
  2. **C3: one post-authentication client check.** Three copy-pasted bodies are already edited in lockstep (`4692aeee5d`), and warning placement has drifted. It is the cheapest and most reversible change.
  3. **C2: a typed issuance contract in `server-spi-private`.** `Object` is a documented stopgap ("TODO: resolve deps issue"), and the same workaround was already fixed for `cors`. It is also the prerequisite for any future C1 work.
- **Rejected:**
  - **C1** (the whole god-class split): accidental, but the blast radius is 63 files, and the maintainers' own precedent is narrow, feature-driven SPI extractions.
  - **C5** and **C6**: deliberate, load-bearing designs.
  - **C7**: the cheap cut is cosmetic. The real fix is a redesign of how organization scopes fit the scope model, which belongs to a separate analysis.
- **Cross-cutting constraint [E]:**
  - CI rejects PRs to main that add any file under `testsuite/` or add ≥100 lines to one file there (`.github/actions/testsuite-deprecation-check/deprecation-check.sh:13-26`, job `ci.yml:99-106`).
  - No API-compatibility, architecture or cycle guard runs in the build. Pom grep found no revapi, japicmp, archunit, jdepend, checkstyle, errorprone or spotbugs; spotless covers imports only (`pom.xml:1613-1626`).
  - So every refactor's safety net is tests, and **new characterization tests must go in `N/` or `U/`**. Several behaviours each candidate touches are guarded only in `L/` today (TD-4).

---

## 1. Candidates in detail

### C1. `TokenManager` god class (TD-1)

**Current shape**

- **Fields** [E, ast-grep `field_declaration` directly inside the `TokenManager` class body = 1]. Its only field is `private static final Logger logger` (`P/TokenManager.java:164`). [I] Instances carry no state, so the 11 `new TokenManager()` sites are interchangeable.
- **Responsibility clusters** [E, line ranges from the shape sub-agent]:

  | Lines | Cluster | Kind |
  |---|---|---|
  | 166-321 | token and session validation | `validateToken` is instance; `isUserValid`, `validateUserNotBefore`, `lookupUserFromStatelessToken` are static |
  | 323-470 | refresh reuse and confirmation binding | |
  | 473-490 | ID-token verification | |
  | 501-605 | client-session attach and access | static |
  | 610-852 | scope parsing and validation | static |
  | 854-905 | consent | static |
  | 908-1103 | mapper orchestration | instance |
  | 1105-1204 | token init and expiry | |
  | 1206-1563 | response building | nested static `AccessTokenResponseBuilder` at `:1211` |
  | 1585, 1629 | `NotBeforeCheck` / `TokenRevocationCheck` predicates | |
  | 1643-1736 | logout-token verification | |
  | 1738-1769 | organization helpers | |

- **Internal coupling** [E]. The builder holds a `TokenManager` field (`:1218`) and calls back into `createClientAccessToken` (`:1297`), `transformIDToken` (`:1435`), `transformAccessTokenResponse` (`:1535`), `getReuseIdKey` (`:1333`) and `formatTokenType` (`:1244,1276,1298`). Scope validation uses scope parsing (`:706,736`).
- **Existing seams** [E]:
  - `RefreshTokenProvider` (`P/refresh/`), `TokenPostProcessor` (`P/token/TokenInterceptorSpi.java:21`) and `TokenContextEncoderProvider` (`P/encode/`).
  - All three live in `services`.
  - `RefreshTokenContext` carries the concrete `TokenManager` (`P/refresh/RefreshTokenContext.java:8`), so the newest seam still depends on the god class (`P/refresh/DefaultRefreshTokenProvider.java:86,114,135`).
  - `server-spi/.../models/TokenManager.java:27` is an *interface*, but it is the JOSE codec, which constrains naming (TD-11).
- **No subclasses** [E, ast-grep `class $C extends TokenManager` = 0, grep agrees].

**Intent verdict: accidental complexity (high confidence)** [E/I]

- **Growth timeline** (first-parent `main`): 424 lines (2015) → 884 (2020) → 1211 (2021) → 1446 (2023) → 1576 (2026-01) → 1769 (HEAD).
- No single feature dominates. The largest additions are backchannel logout (`ddc2c25951`, +116), duplicate-sub (`a4c4c00d00`, +66) and session-lifespan work.
- **Precedents for shrinking it** are narrow and feature-driven:
  - signature SPI, `24e60747b6` (2018, −41)
  - `SessionExpirationUtils`, `ecf52285bc` (2023, −96)
  - **RefreshTokenProvider**, `53af45a3ab` (2026-06-19, −165 in this file). Its reason (#50161): OID4VCI refresh, where "the user session may not exist". PR #50326 adds: "The DefaultRefreshToken provider should behave exactly same way as before."
- Making the class an SPI or interface appears in no commit message (`git log --grep=TokenManager`). The only proposal is external issue #43052 (2025-09-30), which has no maintainer response [E].

**Feasibility**

- **New seam required** [I]. No abstraction exists. An expand/contract path:
  1. Extract a cluster into a new class.
  2. Leave the `TokenManager` methods as delegating facades, so all 63 referencing files keep compiling.
  3. Later, centralize construction.
- **Out-of-tree contract** [I]. Third-party grants and exchange providers reach `TokenManager` through the C2 casts, so its public signatures are an effective contract.
- **Guards** [E]:
  - `U/protocol/oidc/TokenManagerTest` (1 test, `isValidScope`, boots a real factory; there is no Mockito in `services`).
  - `N/oauth/RefreshToken*`, `OfflineToken*`, `oauth/tokenexchange/*`, `admin/client/ClientScopeEvaluateTest`.
  - Code-exchange errors, introspection and lightweight tokens are tested only in `L/` (TD-4).
- **First prerequisite** [I]: port or characterize the TD-4/TD-5 code-exchange branches in `N/oauth/`.
- **Cost** [I]:
  - minimal step: 1–2 test files
  - one cluster extraction: about 2–3 files
  - interface plus construction change: at least 9 (report: 11) construction files (the 11 sites sit in 9 files), up to 63 if callers retype

### C2. `Object` in the grant and exchange SPI contexts (TD-2)

**Current shape** [E]

- **`Object` fields:**
  - `OAuth2GrantType.Context`: `Object clientConfig` (`SPI/protocol/oidc/grants/OAuth2GrantType.java:102` (report: `:101`)) and `Object tokenManager` (`:111,116`, getter `:205`).
  - `TokenExchangeContext`: `Object tokenManager` (`SPI/protocol/oidc/TokenExchangeContext.java:46,70`, getter `:116`).
  - All other fields in both classes are concretely typed, including `Cors`, an SPI type.
- **Module direction:** `server-spi-private/pom.xml` depends only on `server-spi`, `keycloak-core` and libraries; `services/pom.xml:77` depends on `server-spi-private`. So the SPI module cannot name `services` types.
- **Construction:** 1 site each, `P/endpoints/TokenEndpoint.java:176` and `P/grants/TokenExchangeGrantType.java:61` (ast-grep, grep agrees).
- **Readers:** exactly 2 `(TokenManager)` casts (analysis TD-2), plus the `clientConfig` cast at `P/grants/OAuth2GrantTypeBase.java:107`.
- **No issuance interface** exists in `server-spi(-private)`.

**Intent verdict: a real constraint (module layering) handled with an accidental stopgap (high confidence)** [E]

- `3b3a61dfba` (2021-04-21, "KEYCLOAK-18639 Token Exchange SPI Milestone 1") introduced the fields under `// TODO: resolve deps issue and use correct types` [E, re-read].
- `983680ce0e` (2023-12-13, PR #26251, "OAuth 2.0 Grant Type SPI") repeated the pattern for `clientConfig`, `cors` and `tokenManager`.
- **Precedent:** `cors` was typed properly once `Cors` became an SPI (`b0403e2268`, 2024-01-27, #25446; `b81bf85a06`, 2024-02-20).
- `73cfd9cc80` (2025-02-28) removed the TODO comment but left the `Object` type.
- **Verdict breakdown:** the layering is load-bearing; `Object` is not.

**Feasibility**

- **New narrow interface in `server-spi-private`** [I], implemented by `TokenManager` (the `cors` precedent shows the route). A parallel change:
  1. Add the interface; no callers.
  2. Add a typed getter next to `Object getTokenManager()`.
  3. Switch the 2 casts.
  4. Deprecate the `Object` getter and constructor argument.
- Changing the field or return type in place would be binary-incompatible for compiled third-party providers [I].
- **Policy cover** [E]:
  - `changes.adoc:928-930`: `server-spi-private` "may quite likely change … between minor releases".
  - `OAuth2GrantTypeSpi:34` and `TokenExchangeSpi:34` return `isInternal() = true`, which logs a startup WARN for non-`org.keycloak` factories (`ServicesLogger.java:223`).
  - So an upgrade note is the expected form. No compatibility check covers this module [E, pom grep].
- **Open shape question** [U]:
  - What the interface must expose is bounded by what in-tree grants actually call on `TokenManager`. That set comes from the analysis §1.6 call-site table: `responseBuilder`, `createClientAccessToken`, `attachAuthenticationSession` (static), `verifyConsentStillAvailable` (static) and others.
  - Static methods can't sit behind an instance interface, so the minimal surface is unknown until planning lists the instance methods grants use.
  - The name has to avoid the `models.TokenManager` collision (TD-11).
- **Guards** [E]: the same `N/` grant and exchange tests as C1. Nothing checks binary compatibility.
- **First prerequisite** [I]: list the instance methods of `TokenManager` that in-tree grant and exchange code calls through `context.getTokenManager()`. Planning needs that list to size the interface. Then add the interface with no callers (2 files).
- **Cost** [I]: about 4–6 files plus an upgrade note. `clientConfig` → `OIDCAdvancedConfigWrapper` is a second, independent instance of the same problem. Whether that wrapper type can move into the SPI module is [U].

### C3. Post-authentication client checks duplicated (TD-3a)

**Current shape** [E]

- **The three bodies:**
  - `TokenEndpoint.checkClient` (`P/endpoints/TokenEndpoint.java:217-234`): `authorizeClient`, `cors.checkAllowedOrigins`, bearer-only rejection, plus `clientAuthAttributes`/`clientConfig` and the full-scope deprecation warning.
  - `OAuth2GrantTypeBase.checkClient` (`P/grants/OAuth2GrantTypeBase.java:283-294`): the same without the warning. Its 1 caller is `PermissionGrantType.java:104`.
  - `TokenRevocationEndpoint.checkClient` (`P/endpoints/TokenRevocationEndpoint.java:187-199`): drops `clientAuthAttributes`/`clientConfig`, adds `event.client(client)`.
- **The shared helper:** `AuthorizeClientUtil.authorizeClient` (`P/utils/AuthorizeClientUtil.java:48-88`) already checks enabled (`:65`), allowed origins (`:70-72`) and OIDC protocol (`:74-83`). The origin re-check in all three copies is therefore redundant [I].
- **The other 7 `authorizeClient` callers** each add a different post-auth subset [E, ast-grep 10 calls = grep]:

  | Caller | Origins (repeat) | Bearer-only | Public-client |
  |---|---|---|---|
  | `LogoutEndpoint:742-750` | yes | yes | – |
  | `TokenIntrospectionEndpoint:161-178` | – | – | rejects |
  | `AbstractCibaEndpoint:51-65` | – | yes | – |
  | `AbstractParEndpoint:65-82` | yes | – | – |
  | `IdentityBrokerService:523-532` | yes | – | rejects |
  | `ClientsManagementService:194-203` | – | – | rejects |
  | `DeviceEndpoint:394-403` | – | yes (report: –), via its own `checkClient(String)` (`:402` → `:433`), with `unauthorized_client`/403 instead of `invalid_client`/400 | – |

  Nothing shared covers the bearer-only or public-client checks.

**Intent verdict: accidental complexity, copy-paste (high confidence)** [E]

- `git blame -w -C -C` attributes the `OAuth2GrantTypeBase` body lines to `TokenEndpoint` commits from 2015 onward (`e2b02d414f`, `d8d6348f67`, `99c1ee7f5a`, …). Only the signature belongs to `983680ce0e`, so it was copied during the grant-SPI extraction.
- The revocation copy is an independent copy from `b40c12c712` (2020-01-30, KEYCLOAK-5325).
- All three were edited together in `4692aeee5d` (2026-05-07, #45957, CORS origin check), so keeping them in sync is already a manual chore.
- The full-scope warning landed only in `TokenEndpoint` (`06e82d6976`, 2026-09-18, PR #52924). Issue #52923 asks to warn "whenever any client with full-scope-allowed is authenticating". Whether leaving out revocation and the other 8 callers was intended is [U].

**Feasibility**

- **Existing abstraction:** `AuthorizeClientUtil` is the natural home [I]. Path:
  1. Add a helper that does authorize plus the bearer-only check.
  2. Route the 3 copies to it one at a time.
  3. Keep `OAuth2GrantTypeBase.checkClient()` as a protected delegating method, since out-of-tree subclasses may call or override it.
- **Behaviour to keep** [E]: the warning (`TokenEndpoint:229-233`), `event.client` in revocation, and the `clientConfig`/`clientAuthAttributes` difference.
- **Guards** [E]:
  - Revocation client auth is tested only in `L/oauth/TokenRevocationTest` (11 tests, none for bearer-only or invalid auth) and `TokenRevocationCorsTest` (2).
  - Bearer-only rejection is tested only for ROPC (`L/oauth/AccessTokenTest.java:728-740`, status only) and device (`L/oauth/OAuth2DeviceAuthorizationGrantTest`). The literal `"Bearer-only"` appears in 0 `N/` files.
  - The permission grant (base copy) has `N/authz/services/uma/UmaGrantTypeTest` with 2 tests plus `L/authz/UmaGrantTypeTest` (21).
- **First prerequisite** [I]: an `N/oauth/` characterization test for revocation client auth (bearer-only client, wrong secret, disallowed origin), plus a bearer-only uma-ticket case.
- **Cost** [I]: 1–2 test files, then about 4 main files.
- **Scope note** [I]: routing the other 7 callers onto one policy would be a behaviour change, because each enforces a different subset. That makes it a protocol-policy question, not part of the de-duplication.

### C4. Token-response finalization split across grants (TD-3b/c)

**Current shape** [E, per-site table from the shape sub-agent]

| Site | MTLS bind | ID token | Response client-policy event | KEK → 400 | RAR sanitize | Custom-claims hook | `event.success` | `cors.add` |
|---|---|---|---|---|---|---|---|---|
| `OAuth2GrantTypeBase` `:119-200` (5 grants: AuthorizationCode `:277`, ClientCredentials `:156`, CIBA `:213`, Device `:372`, JWTAuthorization `:201`) | `:150` | `:152` | `:156` | `:174-182`, only when `code=true` (Device passes `false`) | `:193` | `:189` (report: `:190`) | `:195` | `:197` |
| ROPC `:150-183` | `:166` (after the ID token) | `:163` | `:169` | no | **no** | no | `:180` | `:183` |
| PreAuthorizedCode `:231-282` | no | no | **no** | `:272`, **dead** (message mismatch, §0) | no | no | `:281` | `:282` (allow all) |
| RefreshTokenGrantType `:84-125` | `:94` | provider | `:96` | no (maps CERT_VERIFY `:102`) | `:121` | no | `:123` | `:125` |
| StandardTokenExchangeProvider `:348-397` | pre-check `:150`, RT copy `:224` | `:377/384` | `:365` | no | **no** | no | `:395` | `:397` |
| V1TokenExchangeProvider `:369-410` | **none** | `:399` | **none** in the provider | no | **no** | no | `:408` | `:410` |
| Implicit `OIDCLoginProtocol:305-361` | transient mapper only | `:312` | `:329` | no | no | no | n/a | n/a (redirect) |

- The `addCustomTokenResponseClaims` hook (`P/grants/OAuth2GrantTypeBase.java:300`, called `:189`) has 0 overrides (analysis TD-3).
- **Differences that look justified by protocol** [I]:
  - implicit is a redirect with no CORS
  - refresh delegates token building to the provider
  - exchange sets `issued_token_type`
  - PreAuth sets the OID4VCI audience/scope
- **Differences that look incidental** [I]:
  - RAR sanitize missing in ROPC and both exchange paths
  - the MTLS bind order in ROPC
  - V1 with no MTLS bind and no response event
  - the KEK mapping keyed on a flag named `code`
  - the dead PreAuth comparison

**Intent verdict: accidental (high confidence)** [E]

- `createTokenResponse` was factored out of code-to-token in `65c48a4183` (2021-04-29, CIBA).
- Refresh, ROPC, client credentials and exchange were already separate `TokenEndpoint` methods. `983680ce0e` moved them as they were: at that commit only Base, AuthorizationCode, CIBA and Device reference `createTokenResponse`.
- PR #26251's review says "We can have a follow-up". No reason against unifying was found [U on intent, E on absence in the inspected commits and PR].
- **The hook:**
  - `64e0b450aa` (2025-09-15, PR #40751) added it with one override in `AuthorizationCodeGrantType`.
  - `416a6017c2` (2026-01-16, #44961) replaced that override with `AuthorizationDetailsProcessorManager` and removed it, leaving the hook unused. No out-of-tree consumer is mentioned [E].

**Feasibility**

- **Existing abstraction:** `createTokenResponse` [E]. Path [I]:
  1. Extract a finalize step (response policy event, build with KEK mapping, RAR sanitize, success/CORS) that `createTokenResponse` calls.
  2. Move ROPC, Refresh and PreAuth onto it one per commit.
  3. The exchange providers don't extend the base class, so they need the step as a shared helper.
- **Each step changes runtime behaviour** for its grant, e.g. refresh starts running the hook and the KEK mapping. So every step is a separately revertible commit with its own characterization test [I].
- **Guards** [E]:

  | Path | Tests |
  |---|---|
  | Refresh | `N/oauth/RefreshTokenTest`, `RefreshTokenRevokeTest` |
  | ROPC | `L/oauth/ResourceOwnerPasswordCredentialsGrantTest` (27); `N/` has no dedicated class, only `doPasswordGrantRequest` usage |
  | PreAuth | `U/protocol/oidc/grants/PreAuthorizedCodeGrantTypeTest` (lifespan only), 9 `N/oid4vc` files |
  | Exchange | `N/oauth/tokenexchange/*` |
  | Implicit | [U] |
  | Per-grant `*_RESPONSE` event assertions | [U], not grepped exhaustively |

- **First prerequisite** [I]: an `N/oauth/` characterization test that records the token-response JSON field set (including `authorization_details` and `token_type`) for code, refresh, ROPC and both exchange versions. It must assert today's differences as they are, so each later step shows exactly what changed. It also needs TD-6's sharper refresh assertions.
- **Cost** [I]: 1 test file minimal; about 8 main files for the full migration.

### C5. Transient DPoP/MTLS mappers outside the type filter (TD-9)

**Current shape** [E]

- `ProtocolMapperUtils.getSortedProtocolMappers` filters with a `Predicate` (`S/protocol/ProtocolMapperUtils.java:152,164`), then appends the DPoP and MTLS transients for OIDC clients (`:166-172`), so the predicate never sees them.
- `MtlsHoKTokenUtil.MtlsHoKProtocolMapper` (`S/services/util/MtlsHoKTokenUtil.java:171-172`) also implements all 5 interfaces, and overrides only `transformAccessToken` (`:202-209` (report: `:203-211`)).
- The `TokenManager` callers (`:910,929,946,958,1097`) filter with `instanceof` and then cast unconditionally (e.g. `:914` (report: `:915`)) [E]. [I] A transient mapper missing one interface would throw `ClassCastException`.
- The MTLS binding also happens explicitly in grants (`P/grants/OAuth2GrantTypeBase.java:207-225`), so the access token is bound twice [I].

**Intent verdict: deliberate constraint, load-bearing (high confidence)** [E]

- Introduced in `94784182df` (2024-07-29, #29967, for RFC 9449 §5: "applicable for all access token requests regardless of grant type").
- Javadoc at `S/services/util/DPoPUtil.java:116-121`: "a protocol mapper that cannot be modified by administration users … generic solution for DPoP on all implemented grantTypes, even custom-implemented grantTypes".
- MTLS followed in `81af44c0e9` (2026-03-19, #47314).
- On #21921, a maintainer calls DPoP "very hard to fully decouple".

**Feasibility.** The hazard can be closed in about 1–2 files by also applying the predicate to the transient stream, or by asserting fail-fast [I]. That is hardening inside a deliberate design, not a structural refactor.
**Guards** [E]: `N/oauth/DPoPTest` (46 tests); mTLS HoK only in `L/oauth/hok/HoKTest` (16).

### C6. Runtime provider selection (TD-10a)

**Current shape** [E]

| Provider | Selection | Recorded |
|---|---|---|
| Grant (`TokenEndpoint:241`) | by id | event detail `GRANT_TYPE` (`:247`) |
| Refresh, issue (`TokenManager:1347-1356`) | order + `supports` | `REFRESH_TOKEN_PROVIDER_ID` (`:1360-1361`) |
| Refresh, renewal (`RefreshTokenGrantType:84-91`) | the same pipeline, duplicated | nothing |
| Exchange (`TokenExchangeGrantType:73-87`) | order + `supports` | nothing beyond the unsupported reason |
| Encoder (`TokenManager:1108`) | plain default provider | — |
| Post-processors (`:1462-1469`) | all of them, in order | — |

The comparators are inconsistent: `f2.order() - f1.order()` vs `Integer.compare`.

**Intent verdict: deliberate, load-bearing for exchange, refresh and authorization details (high confidence)** [E]

- keycloak-community `design/token-exchange-spi.md` weighed "provider-side selection … `supports()` … deterministic ordering" against "policy-based selection … configurable by end users (and thus would require a UI)", and chose the provider side.
- It was implemented in `3b3a61dfba` and reused in `983680ce0e`, `64e0b450aa`, `53af45a3ab` and `cf072dd872`. PR #50326 states `supports` picks the OID4VCI refresh provider.
- Post-processor ordering is real but was fixed late (`3f9f2f6122`, 2026-06-29, #50269).

**Feasibility.** Only the duplicated lookup and the missing debug logging are accidental: about 3 files, behaviour-neutral [I]. This is housekeeping, not a refactor opportunity.

### C7. `OrganizationScope` ↔ `TokenManager` cycle (TD-10b)

**Current shape** [E]

- `TokenManager.parseScopeParameter` (`:850-852`) is a pure, static `split(" ").distinct()`. It has 4 external callers (`OrganizationScope:377`, `ClientScopeAuthorizationRequestParser:95`, `DownscopeAssertionGrantEnforcerExecutor:95,100`, `RejectMayActClaimExecutor:180`; ast-grep = grep).
- In the other direction, `TokenManager` needs from Organizations:
  - the scope kind (`valueOfScope`, `:644,712,798`)
  - membership resolution (`:650` (report: `:648`))
  - dynamic client-scope construction (`toClientScope`, `:683`)
  - the mapper id (`:629,672` (report: `:628,672`))
  - feature checks (`:671,709,798,1743` (report: `:670,709,797`); `:1759` is `getProvider`)
  - selected-org validation (`:1742`)
- Pluggable scope abstractions exist and Organizations does not use them: `ParameterizedScopeTypeProvider` (`SPI/protocol/oidc/scope/`, 5 implementations, used at `TokenManager:833`) and `AuthorizationRequestParserProvider.parseScopes` (`SPI/protocol/oidc/rar/AuthorizationRequestParserProvider.java:33-49`). Grep for organization code in `rar/` and `scope/` returned nothing [E].

**Intent verdict: accidental (medium-high confidence)** [E/I]

- Both directions were created in one commit, `8e0436715c` (2024-08-15, "Support for ALL and ANY organization scope values"), with a one-line body that gives no reason.
- Later edits at this seam include a security fix, `f19e1f2b4e` (2026-06-04, CVE-2026-9791, "org data exposed when the feature is disabled"), and `5a8cf83282`/`de81c0a421` (2026, #45900).
- Open #49021 criticises the hard-coded mapper id.

**Feasibility**

- **The cheap cut** [I]: move `parseScopeParameter` into a neutral utility (3–6 files, reversible). This removes only the Organization → `TokenManager` edge; every reverse-direction use listed under "Current shape" above stays.
- **The real fix** [I]: make organization scopes resolve through a scope extension point instead of being special-cased in `TokenManager`. That is a redesign of how the organization concept fits the client-scope model (ANY/ALL/SPECIFIC semantics, membership, feature gating). **Stopping here, as instructed:** it belongs to a separate, later analysis.
- **Guards** [E]: `N/organization/mapper/OrganizationOIDCProtocolMapperTest` (53 tests). No cycle guard exists to stop the cycle returning.

---

## 2. Refactor opportunities (ranked proposal)

**Ranking criteria**, all from evidence above:

- (a) Is the debt accidental, not load-bearing?
- (b) Is there evidence that it is already costing something (drift, lockstep edits, defects)?
- (c) Does an existing abstraction allow an incremental, reversible path?
- (d) Can a safety net be built in `N/`/`U/` despite the legacy freeze?
- (e) Blast radius.

### #1. C4: one token-response finalization step

- **Current → target.** Today 7 finishing sites each assemble their own subset of {response policy event, build + KEK mapping, RAR sanitize, custom-claims hook, success, CORS} (table in §1 C4). The target is **one shared finalize step** that every grant and exchange path calls. Protocol-specific differences become explicit parameters or overrides, not missing lines.
- **Why it earns its place:**
  - **Cost of the debt [E]:**
    - The drift is already producing visible inconsistencies: RAR sanitize missing in ROPC and both exchange paths, V1 without the MTLS bind or a response policy event, and a dead KEK mapping in PreAuth (`PreAuthorizedCodeGrantType.java:272-273` vs `DefaultTokenManager.java:288`).
    - The response policy events are a security-policy extension point (client policies). Uneven coverage means a policy can silently not apply to a grant [I].
  - **Cost of the change [I]:** about 8 main files, 1 per step. Each step is a behaviour change for one grant, so the test cost is real but bounded.
- **Blast radius** [E, analysis §1.6]:
  - the 10 grants extending `OAuth2GrantTypeBase`
  - 3 concrete exchange providers
  - 9 builder-carrying `*_RESPONSE` events reaching `triggerOnEvent` at 5 sites
  - `OIDCAccessTokenResponseMapper` (5 implementations)
  - the JSON token-response shape consumed by the JS admin client (TD-10c)
  - out-of-tree `OAuth2GrantTypeBase` subclasses (internal SPI)
- **Incremental path** [I]:
  1. Add characterization tests.
  2. Extract the finalize step out of `createTokenResponse`, with no behaviour change for the 5 current users.
  3. Move ROPC → PreAuth → Refresh → Standard exchange → V1, one per commit, each with a test showing the intended difference.
  4. Decide the hook's fate (keep, or deprecate) last.
- **First prerequisite step:** an `N/oauth/` test recording today's token-response field set, `authorization_details` and `token_type` for authorization_code, refresh, ROPC and both token-exchange versions, plus TD-6-style error-description assertions on refresh. It must not go in `L/` (CI freeze).

### #2. C3: one post-authentication client check

- **Current → target.** Today there are 3 copy-pasted `checkClient` bodies (`TokenEndpoint:217-234`, `OAuth2GrantTypeBase:283-294`, `TokenRevocationEndpoint:187-199`), each repeating an origin check the helper already performs. The target is **one helper in `AuthorizeClientUtil`** that returns the authenticated client after the bearer-only check. The 3 sites delegate to it, keeping their documented differences explicitly.
- **Why it earns its place:**
  - **Cost of the debt [E]:** a security-relevant fix had to be applied to all three copies at once (`4692aeee5d`, #45957). The full-scope warning reached only one copy (`06e82d6976`) [U whether intended].
  - **Cost of the change [I]:** about 4 main files and 1–2 test files. It is the cheapest and most reversible candidate, and it is designed to be behaviour-neutral: the documented per-site differences are kept, and the characterization tests are there to prove it.
- **Blast radius** [E/I]:
  - token endpoint, revocation endpoint, and the permission (UMA) grant
  - `OAuth2GrantTypeBase.checkClient()` is protected, so out-of-tree subclasses keep a delegating method
  - the other 7 `authorizeClient` callers are deliberately left out (their check sets differ; aligning them is a policy question)
- **Incremental path** [I]:
  1. Add characterization tests.
  2. Add the helper with no callers.
  3. Switch `OAuth2GrantTypeBase` (1 caller), then `TokenRevocationEndpoint`, then `TokenEndpoint`.
  4. Drop the redundant origin call last, as its own revertible step.
- **First prerequisite step:** an `N/oauth/` test for revocation client authentication (bearer-only client, wrong secret, disallowed origin), asserting whatever status and `error` the code returns today [U: the exact values were not run], plus a bearer-only case for `urn:ietf:params:oauth:grant-type:uma-ticket`. Today these are guarded only in `L/` or not at all.

### #3. C2: a typed token-issuance contract in `server-spi-private`

- **Current → target.** Today `OAuth2GrantType.Context` and `TokenExchangeContext` carry `Object tokenManager` (and `Object clientConfig`), downcast in 2 (+1) places. The target is a **narrow issuance interface in `server-spi-private`** that the `services` `TokenManager` implements, exposed through typed getters (the shape the `Cors` SPI already took).
- **Why it earns its place:**
  - **Cost of the debt [E/I]:**
    - The authors marked the `Object` type as a stopgap (`3b3a61dfba` TODO).
    - A signature change in `TokenManager` breaks out-of-tree grant and exchange providers at runtime, not at compile time (TD-2).
    - It is also the seam any future C1 extraction needs: the in-tree grant and exchange bases get their `TokenManager` *instance* from the context (`OAuth2GrantTypeBase:116`, `AbstractTokenExchangeProvider:116`). They also call its static methods directly, which this interface would not cover [E/I].
  - **Cost of the change [I]:** about 4–6 files plus an upgrade note. It is purely additive until the deprecation step.
- **Blast radius** [E]:
  - 2 context classes, 1 construction site each, 2 `TokenManager` casts and 1 `clientConfig` cast
  - every out-of-tree `OAuth2GrantType` or `TokenExchangeProvider` (internal SPIs: `isInternal() = true`, documented as changeable between minors)
  - no compatibility check would catch a mistake
- **Incremental path** [I]:
  1. Inventory the instance methods called through the context.
  2. Add the interface with no callers.
  3. `TokenManager implements` it.
  4. Add the typed getters next to the `Object` ones.
  5. Switch the 2 casts.
  6. Deprecate the `Object` accessors (removal is a later release).
  7. Handle `clientConfig` separately.
- **First prerequisite step:** produce the list of `TokenManager` instance methods that in-tree grant and exchange code calls via `context.getTokenManager()` (ast-grep over `P/grants/**` and `P/tokenexchange/**`). It sizes the interface and settles the name (it can't be `TokenManager`, TD-11). The interface's surface is [U] until this exists.

**Suggested order** [I]: C3 → C4 → C2.

- C3 is the smallest and builds the `N/oauth/` client-auth tests.
- C4 reuses that test harness and delivers the most consistency value.
- C2 is independent of both. It can run in parallel, but it pays off most before any C1 work.

### Considered and rejected

| Candidate | Verdict | Why rejected (evidence) |
|---|---|---|
| **C1** splitting or interface-ifying `TokenManager` as a whole | accidental | The blast radius is 63 files and 11 construction sites (analysis §1.6). The safety net is weakest exactly here: code-exchange and introspection behaviour are guarded only in `L/`, which CI freezes. The maintainers' precedent is narrow, feature-driven extraction (`53af45a3ab` RefreshTokenProvider, behaviour-identical), and nobody upstream has engaged with "make it pluggable" (#43052). Better treated as a by-product: C2 creates the seam, and later narrow extractions follow feature needs. |
| **C5** transient-mapper typing | deliberate | Load-bearing by design (`DPoPUtil.java:116-121`, RFC 9449 §5). The real hazard can be closed in 1–2 files (filter or assert), which is hardening, not restructuring. |
| **C6** order/supports provider selection | deliberate | A documented design choice (keycloak-community `token-exchange-spi.md`). Only housekeeping is left: de-duplicating the refresh lookup (`TokenManager:1347-1356` / `RefreshTokenGrantType:84-91`), a consistent comparator, debug logging. |
| **C7** Organization cycle | accidental | The cheap cut (moving `parseScopeParameter`) removes one edge and leaves all the real coupling. The real fix is a redesign of how organization scopes plug into the client-scope model, which is a business-concept question for a separate analysis. The CVE fix at this seam (`f19e1f2b4e`) makes that analysis worth having. |

---

## Claim verification (ast-grep)

**Scope.** These are the structural claims (counts, "only here", "overrides X but not Y", mirrored pairs) that the candidates' current shape, cost and blast radius rest on, and so the ranking with them. Run with ast-grep 0.51.0 over the whole repo at `eac37c6800`. Every ast-grep zero was repeated with `grep -rn --include='*.java'` (excluding `target/`). Patterns marked *rule* are YAML rules (`ast-grep scan -r`). Abbreviations as above. Verdicts: **C** confirmed, **R** refined (substance holds, count/line/qualifier corrected), **X** refuted.

**Method notes.**

- The pattern form `class $C extends X { $$$ }` returns 0 even for `OAuth2GrantTypeBase`, which has 10 subclasses, so it is not a reliable test for "no subclasses". Every hierarchy claim below uses the *rule* `class_declaration` + `has: {field: superclass | interfaces, …}`, and grep confirms each result.
- `OrganizationMembershipMapper.PROVIDER_ID` as a pattern returns 0. grep finds `:629,672`, and those lines are used.
- A single-line grep for `implements … OIDCAccessTokenResponseMapper` finds 1 file because the `implements` lists wrap. The ast-grep rule (5) is authoritative there, and it is not a zero.

| # | Claim (where) | Verdict | Evidence (`file:line`) | Method (pattern / rule) |
|---|---|---|---|---|
| 1 | `TokenManager` is 1769 lines (TD-1) | **C** | `P/TokenManager.java` = 1769 lines (`wc -l`) | `wc -l` |
| 2 | Its only field is `logger` (C1) | **C** | `P/TokenManager.java:164` | *rule* `field_declaration` inside the `class_body` of `class_declaration` named `^TokenManager$` (`stopBy: neighbor`) = 1 |
| 3 | `new TokenManager()` at 11 sites (TD-1, C1) | **R** | 11 in `services/src/main`, in **9 files**: 10 × `new TokenManager()` in 8 files (`ClientScopeEvaluateResource:213,245,283`, `AuthorizationTokenService:158`, `KeycloakIdentity:161`, `PolicyEvaluationService:361`, `AdminRoot:81`, `P/AccessTokenIntrospectionProvider:83`, `P/OIDCLoginProtocolService:84`, `P/refresh/AbstractRefreshTokenProvider:328`) + 1 fully qualified (`P/OIDCLoginProtocol.java:306`). Plus 1 test site, `N/oauth/RefreshTokenRevokeTest.java:663`. `admin-client-core/.../Keycloak.java:95` is the unrelated admin-client `TokenManager` (TD-11) | `new TokenManager()`; `new org.keycloak.protocol.oidc.TokenManager()`; grep `new TokenManager(` |
| 4 | `TokenManager` has no subclasses (C1) | **C** | 0 | *rule* `class_declaration` + `superclass` `^(org\.keycloak\.protocol\.oidc\.)?TokenManager$` = 0; grep `extends +…TokenManager\b` = 0 |
| 5 | 63 files reference `TokenManager` (C1, rejected-table) | **C** | 61 files import it or name it fully qualified (59 in `services/src`, 1 `N/`, 1 `L/`) + 2 same-package files without an import (`P/AccessTokenIntrospectionProvider`, `P/OIDCLoginProtocolService`) = 63 | grep `import org.keycloak.protocol.oidc.TokenManager;` / FQN, plus `grep -lw` in `P/*.java` |
| 6 | `TokenManager` implements no interface; no issuance interface in `server-spi(-private)` (C1, C2) | **C** | `P/TokenManager.java:163` `public class TokenManager {` | `class TokenManager implements $$$I { $$$ }` = 0; grep of the declaration line |
| 7 | `Object` fields: `OAuth2GrantType.Context` `clientConfig` + `tokenManager`; `TokenExchangeContext` `tokenManager` (TD-2, C2, §0) | **R** | `SPI/protocol/oidc/grants/OAuth2GrantType.java:102` (report: `:101`) `clientConfig`, `:111` `tokenManager`, constructor `:115-116`, getters `:165` and `:205`; `SPI/protocol/oidc/TokenExchangeContext.java:46`, constructor `:70`, getter `:116`. These are the only `Object` fields in `SPI/protocol/oidc/**` | *rule* `field_declaration` with `type` `^Object$`; `public Object $M() { $$$ }` |
| 8 | 1 construction site per context (C2) | **C** | `P/endpoints/TokenEndpoint.java:176`; `P/grants/TokenExchangeGrantType.java:61` | `new OAuth2GrantType.Context($$$)`, `new TokenExchangeContext($$$)`; grep agrees |
| 9 | Exactly 2 `(TokenManager)` casts + 1 `clientConfig` cast (TD-2, C2, #3) | **C** | `P/grants/OAuth2GrantTypeBase.java:116`, `P/tokenexchange/AbstractTokenExchangeProvider.java:116`; `(OIDCAdvancedConfigWrapper)` at `P/grants/OAuth2GrantTypeBase.java:107`. No other reader of `getTokenManager()`/`getClientConfig()` (`AuthorizationTokenService:367` is an unrelated typed getter) | `(TokenManager) $E`; `(OIDCAdvancedConfigWrapper) $E`; `$X.getTokenManager()`; grep agrees |
| 10 | `isInternal() = true` for both SPIs (C2) | **C** | `SPI/protocol/oidc/grants/OAuth2GrantTypeSpi.java:34-35`, `SPI/protocol/oidc/TokenExchangeSpi.java:34-35` | grep |
| 11 | `checkClient()` duplicated 3× (TD-3a, C3, #2) | **C**, qualified | The 3 authorize-and-check copies: `P/endpoints/TokenEndpoint.java:217-235`, `P/grants/OAuth2GrantTypeBase.java:283-294`, `P/endpoints/TokenRevocationEndpoint.java:187-199`. 4 more methods named `checkClient` exist (`AuthorizationEndpoint:284`, `DeviceEndpoint:407`, `IdentityBrokerService:189`, `ssf/.../StreamService:378`), but they look up a client by id and don't call `authorizeClient`, so they are not copies | `$R checkClient($$$) { $$$ }`; grep `\bcheckClient\s*\(` |
| 12 | The 3 copies' bodies differ as described (warning only in `TokenEndpoint`; revocation drops `clientAuthAttributes`/`clientConfig`, adds `event.client`) | **C** | `TokenEndpoint:229-233`; `TokenRevocationEndpoint:191` | read after locating with #11 |
| 13 | Base `checkClient` has 1 caller, `PermissionGrantType:104`, and no overrides (C3) | **C** | `P/grants/PermissionGrantType.java:104`; the only `checkClient` declaration in `P/grants/**` is the base one | `checkClient()` in `P/grants`; #11 |
| 14 | 10 `AuthorizeClientUtil.authorizeClient` calls (TD-3a, C3) | **C** | `TokenEndpoint:218`, `OAuth2GrantTypeBase:284`, `TokenRevocationEndpoint:188`, `LogoutEndpoint:743`, `TokenIntrospectionEndpoint:163`, `AbstractCibaEndpoint:51`, `AbstractParEndpoint:67`, `DeviceEndpoint:394`, `IdentityBrokerService:523`, `ClientsManagementService:195` | `AuthorizeClientUtil.authorizeClient($$$)` = 10; grep agrees |
| 15 | The helper already re-checks origins, so the copies' origin check is redundant (§0, C3) | **C** | `P/utils/AuthorizeClientUtil.java:70-72`; the copies at `TokenEndpoint:223`, `OAuth2GrantTypeBase:289`, `TokenRevocationEndpoint:193` | read |
| 16 | Per-caller post-auth matrix (C3 table) | **R** | Origins: Logout, Par, IdentityBroker (+ the 3 copies). Public-client: Introspection, IdentityBroker, ClientsManagement. Bearer-only: Logout, Ciba (+ the 3 copies) **and `DeviceEndpoint`** (report: –), through its own `checkClient(String)` (`:402` → `:433`), with `unauthorized_client`/403 instead of `invalid_client`/400. The rule misses Device because the check is in a callee; grep found it | *rule* `method_declaration` that `has` (`stopBy: end`) `AuthorizeClientUtil.authorizeClient($$$)` and a `method_invocation` named `^checkAllowedOrigins$` / `^isBearerOnly$` / `^isPublicClient$`; grep over the 7 files |
| 17 | `createTokenResponse` serves 5 grants (TD-3b, C4) | **C** | `AuthorizationCodeGrantType:277`, `ClientCredentialsGrantType:156`, `CibaGrantType:213`, `DeviceGrantType:372`, `JWTAuthorizationGrantType:201` (+ internal overload `OAuth2GrantTypeBase:204`) | `createTokenResponse($$$)`; grep agrees |
| 18 | Base finalize step lines (C4 table, base row) | **R** | MTLS `:150`, ID token `:152`, policy event `:156`/`:158`, KEK `:174-183`, hook `:189` (report: `:190`), RAR `:193`, success `:195`, CORS `:197` | read after the patterns in #20 |
| 19 | `addCustomTokenResponseClaims` has 0 overrides (TD-3c, C4) | **C** | declared `P/grants/OAuth2GrantTypeBase.java:300`, called `:189`; no other declaration | *rule* `method_declaration` named `^addCustomTokenResponseClaims$` = 1 (the base); grep = the same 2 lines |
| 20 | RAR sanitize missing in ROPC and both exchange paths (C4, #1) | **C** | `sanitizeBeforeSendingTokenResponse` has only 2 call sites: `OAuth2GrantTypeBase:193`, `RefreshTokenGrantType:121`. So ROPC, PreAuth, Standard (and Delegation, which inherits it), V1 and implicit all lack it | `$X.sanitizeBeforeSendingTokenResponse($$$)`; grep agrees |
| 21 | V1 exchange: no MTLS bind, no response policy event; Standard: pre-check + RT copy (C4) | **C** | V1: 0 matches for `triggerOnEvent`, `checkAndBindMtlsHoKToken`, `bindTokenWithClientCertificate`; grep `mtls\|hok\|ResponseContext` in V1 = 0. Standard: `:150` → `:213-214`, RT copy `:224`, event `:365` | `session.clientPolicy().triggerOnEvent($$$)`, `checkAndBindMtlsHoKToken($$$)`, `MtlsHoKTokenUtil.bindTokenWithClientCertificate($$$)` |
| 22 | PreAuth: no MTLS bind, no response policy event, `allowAllOrigins` (C4) | **C** | No `triggerOnEvent`/bind in `PreAuthorizedCodeGrantType`; `cors.allowAllOrigins()` `:282` | same patterns; grep agrees |
| 23 | Dead PreAuth KEK mapping: `"Cannot get encryption KEK"` vs throwers' `"can not get encryption KEK"` (§0, C4, #1) | **C** | `P/grants/PreAuthorizedCodeGrantType.java:273-274`; throwers `S/jose/jws/DefaultTokenManager.java:288`, `P/endpoints/UserInfoEndpoint.java:399`. No production code throws the capitalised string | string-literal patterns `"Cannot get encryption KEK"` = 1, `"can not get encryption KEK"` = 4 (2 throwers + 2 in the base mapping); case-insensitive grep agrees |
| 24 | 10 grants extend `OAuth2GrantTypeBase` (#1 blast radius) | **C** | AuthorizationCode, ClientCredentials, Ciba, Device, JWTAuthorization, Permission, PreAuthorizedCode, Refresh, ROPC, TokenExchange; no indirect subclasses | *rule* `class_declaration` + `superclass` has `OAuth2GrantTypeBase` = 10; grep = 10 |
| 25 | 3 concrete exchange providers; none extends the grant base (C4, #1) | **C** | `AbstractTokenExchangeProvider implements TokenExchangeProvider` (`:88`) → `V1TokenExchangeProvider` (`:82`), `StandardTokenExchangeProvider` (`:69`) → `TokenExchangeDelegationProvider` (`:50`) | *rule* `interfaces` has `^TokenExchangeProvider$` or `superclass` in the chain; grep agrees |
| 26 | 10 `*_RESPONSE` events, 9 builder-carrying, 5 trigger sites (#1) | **C** | `server-spi/.../ClientPolicyEvent.java:35-60` (10 values); response-context triggers: `OAuth2GrantTypeBase:158`, ROPC `:169`, Refresh `:96`, Standard `:365`, `OIDCLoginProtocol:329` | grep `_RESPONSE`; `session.clientPolicy().triggerOnEvent($$$)` filtered to `*ResponseContext` |
| 27 | `OIDCAccessTokenResponseMapper` has 5 implementations (#1) | **C** | `HardcodedClaim:40`, `UserSessionNoteMapper:41`, `ScriptBasedOIDCProtocolMapper:50`, `DPoPUtil:574`, `MtlsHoKTokenUtil:171` | *rule* `interfaces` has `^OIDCAccessTokenResponseMapper$` (`stopBy: end`) |
| 28 | `MtlsHoKProtocolMapper` implements all 5 mapper interfaces and overrides only `transformAccessToken` (C5, TD-9) | **R** | `S/services/util/MtlsHoKTokenUtil.java:171-172`; its methods are `getId`, `getDisplayCategory`, `getDisplayType`, `getHelpText`, `getConfigProperties` and `transformAccessToken` `:202-209` (report: `:203-211`). `DpopProtocolMapper` (`DPoPUtil.java:574-576`) has the same shape (`transformAccessToken` `:610-635`) | *rule* `method_declaration` inside `class_declaration` named `^(MtlsHoKProtocolMapper\|DpopProtocolMapper)$` |
| 29 | `TokenManager` filters with `instanceof`, then casts (C5) | **R** | `instanceof` at `:910,929,946,958,1097`; casts at `:914,933,950,962,1100` (report: e.g. `:915`) | `$X instanceof $T`; `($T) $E`; grep agrees |
| 30 | The predicate never sees the transient mappers (C5) | **C** | `S/protocol/ProtocolMapperUtils.java:164` `.filter(filter)` runs before the `Stream.concat` of the transient DPoP/MTLS mappers at `:168-172` | read |
| 31 | The refresh provider is looked up twice with the same order/supports pipeline (C6, §0) | **R** | The order/supports pipeline appears at exactly 2 sites: `P/TokenManager.java:1347-1356`, `P/grants/RefreshTokenGrantType.java:84-91`. There is also a third, **id-based** refresh-provider lookup that is not a copy of that pipeline: `P/endpoints/TokenRevocationEndpoint.java:162-164` | `$S.getProviderFactoriesStream(RefreshTokenProvider.class)` = 2; grep `RefreshTokenProvider.class` |
| 32 | Inconsistent comparators (C6) | **C** | `f2.order() - f1.order()` at 4 sites (`TokenManager:1349`, `RefreshTokenGrantType:86`, `TokenExchangeGrantType:75`, `rar/AuthorizationDetailsProcessorManager:51`) vs `Integer.compare` at 1 (`TokenManager:1466`, post-processors) | `$A.order() - $B.order()`; `Integer.compare($$$)` |
| 33 | `parseScopeParameter` is static, pure, with 4 external callers (C7) | **C** | `P/TokenManager.java:850-852`; 4 files, 5 calls: `OrganizationScope:377`, `ClientScopeAuthorizationRequestParser:95`, `DownscopeAssertionGrantEnforcerExecutor:95,100`, `RejectMayActClaimExecutor:180`; internal `:636,706,858` | `TokenManager.parseScopeParameter($$$)`; grep agrees |
| 34 | Reverse coupling `TokenManager` → Organizations (§0, C7) | **R** | `valueOfScope` `:644,712,798`; membership `:650` (report: `:648`); `toClientScope` `:683`; mapper id `:629,672` (report: `:628,672`); `Organizations.isEnabled` `:671,709,798,1743`, `getProvider` `:1759` (report: `:670,709,797,1759`); `validateSelectedOrganization` declared `:1742`, called `:945,957` | `OrganizationScope.valueOfScope($$$)`, `$S.resolveOrganizations($$$)`, `$X.toClientScope($$$)`, `Organizations.$M($$$)`, `validateSelectedOrganization($$$)`; grep for the mapper id (pattern = 0) |
| 35 | `ParameterizedScopeTypeProvider` has 5 implementations, used at `TokenManager:833` (C7) | **C** | `IntegerScopeType`, `StringScopeType`, `CustomRegexScopeType`, `UsernameScopeType`, `BooleanScopeType`; `P/TokenManager.java:833` | *rule* `interfaces` has `^ParameterizedScopeTypeProvider$` = 5; grep = 5 |
| 36 | No organization code in the `rar/` and `scope/` extension points (C7) | **C** | 0 | `grep -rli organization` over `SPI/protocol/oidc/{rar,scope}` and `P/{rar,scope}` = 0 (text claim, so grep is the right tool) |
| 37 | Grants get the `TokenManager` instance from the context (C2, #3) | **C** | `OAuth2GrantTypeBase:116`, `AbstractTokenExchangeProvider:116` (= #9) | #9 |
| 38 | Construction-cost line "at least 11 construction files" (C1 cost) | **R** | 9 (report: 11) production files, see #3 | #3 |

**Totals:** 38 claims: 29 confirmed, 9 refined, 0 refuted.

**Effect on the ranking.** No result refutes a claim that a ranked or rejected position rests on. Four refinements touch a candidate's inputs; none is applied to "Refactor opportunities (ranked)":

- **C1 (rejected)**, #3/#38: construction is spread over 9 files, not 11. The 63-file blast radius and the `L/`-only safety net, the actual grounds for rejection, are confirmed (#5). *To be decided at the planning stage* whether the slightly smaller construction cost matters.
- **C3 (#2)**, #11/#16: `DeviceEndpoint` also rejects bearer-only clients, with a different error contract (`unauthorized_client`/403). This strengthens the report's scope note (the 7 other callers enforce different subsets) and does not touch the 3-copy de-duplication. *To be decided at the planning stage* whether the planned helper's bearer-only check should be parameterised for status/error, or Device stays out of scope.
- **C6 (rejected)**, #31: besides the 2 duplicated order/supports lookups there is a third, id-based refresh-provider lookup in `TokenRevocationEndpoint:162-164`. This only widens the housekeeping list. *To be decided at the planning stage* whether it belongs to that housekeeping.
- **C4 (#1)**, #20/#21: confirmed as stated. One addition: `TokenExchangeDelegationProvider` inherits Standard's finalization, so "both exchange paths" covers 3 concrete providers.

---

## Code References

- `P/grants/OAuth2GrantTypeBase.java:107,116,119-200,283-294,300`: `clientConfig`/`tokenManager` casts, `createTokenResponse`, the base `checkClient`, the hook
- `P/grants/PreAuthorizedCodeGrantType.java:266-280`: dead KEK mapping (message mismatch)
- `S/jose/jws/DefaultTokenManager.java:288`, `P/endpoints/UserInfoEndpoint.java:399`: `"can not get encryption KEK"` throwers
- `P/grants/ResourceOwnerPasswordCredentialsGrantType.java:150-183`, `P/grants/RefreshTokenGrantType.java:84-125`, `P/tokenexchange/StandardTokenExchangeProvider.java:348-397`, `P/tokenexchange/V1TokenExchangeProvider.java:369-410`: self-finishing response paths
- `P/endpoints/TokenEndpoint.java:176,217-234,241`; `P/endpoints/TokenRevocationEndpoint.java:187-199`; `P/utils/AuthorizeClientUtil.java:48-88`
- `SPI/protocol/oidc/grants/OAuth2GrantType.java:102 (report: 101),111,116,205`; `SPI/protocol/oidc/TokenExchangeContext.java:46,70,116`
- `P/TokenManager.java:164,850-852,1211,1347-1361`; `P/refresh/RefreshTokenContext.java:8`
- `S/protocol/ProtocolMapperUtils.java:152-172`; `S/services/util/DPoPUtil.java:116-121,571-576`; `S/services/util/MtlsHoKTokenUtil.java:171-211`
- `S/organization/protocol/mappers/oidc/OrganizationScope.java:376-379`
- `.github/actions/testsuite-deprecation-check/deprecation-check.sh:13-26`; `.github/workflows/ci.yml:99-170,1205-1229`

## Architecture Insights

- **The maintainers' refactoring style is narrow, feature-driven extraction behind an SPI, keeping behaviour identical.** Examples: the signature SPI, `SessionExpirationUtils`, `RefreshTokenProvider`, `Cors`. Opportunities that follow this style (C3, C4, C2) fit the codebase better than a big-bang split (C1).
- **Layering is real and load-bearing.** `server-spi-private` can't see `services`. Every stopgap in the SPI contexts stems from that, and the `Cors` SPI shows the accepted way out.
- **The safety net is tests only.** Grep of the pom files found no API, architecture or cycle checking (spotless covers imports only). The legacy suite is frozen by CI, so every characterization test has to be written fresh in `tests/base` or `services/src/test`. TD-4 is therefore a cost item on every opportunity, not an independent chore.

## Historical Context (from prior changes)

- `context/changes/analyze-protocol-oidc/research.md`: the source of TD-1…TD-11 and every blast-radius count reused here. Current verdicts on its claims:
  - TD-2 "`tokenManager` typed `Object`": **supported**, but **partial** as a count. The grant context also has `Object clientConfig` (`OAuth2GrantType.java:102` (report: `:101`)).
  - TD-3 "base `checkClient` used only by `PermissionGrantType`": **supported**.
  - TD-3 "refresh path skips … the KEK-error mapping": **supported**. New: PreAuth's own KEK mapping is dead.
  - TD-9 "works only because `DpopProtocolMapper` implements all five": **supported**, and it also applies to `MtlsHoKProtocolMapper` (`MtlsHoKTokenUtil.java:171-172`).
  - TD-10 "the refresh provider (`P/TokenManager.java:1347-1356`)" chosen at runtime: **supported**. New: a second copy of the lookup in `RefreshTokenGrantType:84-91`. The analysis's wording ("through `session.getProvider` or by `order()`/`supports()`") is **supported** for `TokenContextEncoderProvider` as well: it is the plain default provider (`:1108`), not order-based.
  - TD-10 Organizations cycle: **supported**; the coupling is wider than the cited lines (§0).
- No other prior changes; `context/archive/` holds only `README.md`.

## Related Research

- `context/changes/analyze-protocol-oidc/research.md`
- `context/map/repo-map.md` (prior; risk zones 1 and 5)

## Open Questions

1. C2: which `TokenManager` instance methods do grants and exchange providers really need through the context? This sizes the interface and is the first prerequisite.
2. C3: was leaving revocation and the other `authorizeClient` callers out of the full-scope warning intended (#52923 wording vs `06e82d6976`)? Maintainer question.
3. C4: should V1 token exchange gain the MTLS bind and the response policy event, or is V1 frozen as legacy? This is a product decision for planning.
4. C4/C1: do out-of-tree `OAuth2GrantTypeBase` subclasses, `addCustomTokenResponseClaims` overrides or providers that cast `getTokenManager()` exist in practice? [U]: there is no telemetry in the repo.
5. Coverage: JaCoCo for implicit/hybrid finalization and for the per-grant `*_RESPONSE` client-policy events, which were not established.
6. Defect, outside this change's refactor scope: the dead PreAuth KEK mapping (`PreAuthorizedCodeGrantType.java:272-273`). A one-line fix, if wanted, not a refactor.
7. Upstream discussion #26249 (grant SPI) could not be fetched [U].
