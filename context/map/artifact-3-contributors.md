# Artifact 3: Contributors (key people per area)

Window: since 2025-10-18 (last 12 months to 2026-10-17). Areas are the top 5 from [artifact-1-territory.md](artifact-1-territory.md) and [artifact-2-structure.md](artifact-2-structure.md). Method: non-merge commits touching the area's paths. Excluded bots and automations (Dependabot, Weblate, github-actions) and agent-authored commits (Copilot, 9). Kept human-authored commits with an agent as co-author. Mass commits (>150 files) and noise paths (lockfiles, `.properties`, yaml/json/xml/md/adoc, docs, test resources) were skipped. Author names only. Identities are merged across spellings and addresses (for example Marek Posolda / `mposolda`, Vlasta Ramik / `vramik`, Steven Hawkins / Steve Hawkins). "Awambeng" and "Awambeng Rodrick" were merged by hand.

Team size: the project has 80+ active authors and a core of about 15, so concentration of knowledge in one person is a real risk signal here (unlike a 2-3 person team). Counts are commits touching the area, not lines.

## 1. OIDC/token (`TokenManager`, `protocol.oidc`, JOSE)

| Person | Commits | Last active | Themes |
|---|---|---|---|
| Ricardo Martin | 53 | Sep 2026 | Grants and scope handling, client policies (`may_act` rejection), organization scopes. Also moves tests to the new CI. |
| Marek Posolda | 43 | Oct 2026 | Token and session core, JWE and JOSE validation, deprecating client switches. Also security fixes. |
| Giuseppe Graziano | 42 | Oct 2026 | Token exchange (certificate checks, audience), CIMD, refactors. |
| Thomas Diesler | 35 | Jun 2026 | OID4VCI and FAPI2 conformance, security fixes and tests. |
| Martin Bartoš | 32 | Oct 2026 | Token-exchange delegation (`act`/`may_act`) and introspection. |

Also relevant:
- Takashi Norimatsu (12): resource indicators, CIMD.
- jimmychakkalakal / Jimmy Chakkalakal (14): redirect and logout validation, refresh-replay vulnerability.
- Alexander Schwartz (16): offline sessions, HTTP resource leaks.

Who to ask: Marek Posolda for token/session internals and JOSE; Giuseppe Graziano and Martin Bartoš for token exchange; Ricardo Martin for client policies and scopes.

## 2. Authentication flows and brokering

| Person | Commits | Last active | Themes |
|---|---|---|---|
| Ricardo Martin | 40 | Oct 2026 | Login and flow behaviour, certificate validation (CRL/OCSP), brute-force and session limits. |
| Giuseppe Graziano | 31 | Aug 2026 | IdP side: Google hosted-domain checks, signed-JWT client-auth bypass fix. |
| Stefan Guilhen | 29 | Oct 2026 | Broker token endpoint v2, prompt/organization redirects, tests. |
| Pedro Igor | 21 | Oct 2026 | Broker cache invalidation, validations, flaky tests. |
| Alexander Schwartz | 19 | Oct 2026 | Event deduplication, HTTP response handling on IdP callbacks. |
| Vlasta Ramik | 19 | Oct 2026 | Organization invitations, domain-based IdP routing, consent screen. |
| Martin Kanis | 16 | Oct 2026 | Organization and email-verification security issues. |

Also relevant:
- Pedro Ruivo (14): root authentication session hardening, cacheless feature.
- Martin Bartoš (11): passkeys/WebAuthn, username enumeration.

Who to ask: Stefan Guilhen and Giuseppe Graziano for brokering; Ricardo Martin for login flows; Vlasta Ramik and Martin Kanis for the organization overlap; Pedro Ruivo for session handling.

## 3. FGAP / authorization

| Person | Commits | Last active | Themes |
|---|---|---|---|
| Pedro Igor | 43 | Oct 2026 | Authorization services and policy evaluation, permission filtering of groups and roles. |
| Vlasta Ramik | 40 | Oct 2026 | FGAP v2 hardening (9 security fixes), admin-permissions client, bypass fixes. |
| Martin Kanis | 13 | Oct 2026 | FGAP v2 information-disclosure fixes (stats, composite roles, organizations). |
| Stefan Guilhen | 10 | Oct 2026 | Ownership checks, enforcing view permission during policy evaluation. |
| Ryan Emerson | 8 | Jul 2026 | AuthZEN (evaluations API, subject lookup). |

Also relevant:
- Steven Hawkins (5): admin authentication, type-provider refactor (`DefaultClientService` removal, #52304).
- Lukas Hanusovsky (5): FGAP tests.
- Stan Silvert (3): view-permission filtering in the Admin UI.

Who to ask: Pedro Igor and Vlasta Ramik are the two main knowledge holders. Ryan Emerson owns AuthZEN but has been quiet since July.

## 4. OID4VC / SSF / SCIM

| Person | Commits | Last active | Area and themes |
|---|---|---|---|
| Thomas Diesler | 60 | Jul 2026 | OID4VC: issuance flows, FAPI2 conformance, security fixes. |
| Stefan Guilhen | 41 | Sep 2026 | SCIM (39 of 41), including promotion to supported; SSF discovery fixes. |
| Awambeng (merged) | 37 | Sep 2026 | OID4VC: metadata (`issuer_info`), SD-JWT, JSON-LD hardening. |
| Stan Silvert | 37 | Oct 2026 | SSF/CAEP: receiver-driven subjects, account-purged events. |
| forkimenjeckayang | 34 | Oct 2026 | OID4VC tests, key attestation, Admin UI integration. |
| Pedro Igor | 33 | Sep 2026 | SCIM: user and group lookups, permission checks. |
| Marek Posolda | 32 | Sep 2026 | OID4VC: token scope, proofs and attestations, access-token restrictions. |
| Thomas Darimont | 15 | Oct 2026 | SSF/CAEP emitters (session revoked). |

Also relevant:
- Giuseppe Graziano (13): OID4VC token audience, bearer ordering.
- Martin Kanis (9): SCIM security (`maxOperations`, organization groups).

Who to ask: SCIM: Stefan Guilhen and Pedro Igor. SSF: Stan Silvert and Thomas Darimont. OID4VC: Marek Posolda, Awambeng and forkimenjeckayang. Thomas Diesler wrote the most OID4VC code but has not committed since July.

## 5. Test migration and SPI boundary (`tests/`, `testsuite/`, `server-spi-private`)

The broadest area: 1,359 commits and 185 people. Most appear because they test their own changes, so only people whose themes are the framework or SPI itself are listed.

| Person | Commits | Themes |
|---|---|---|
| Stefan Guilhen | 101 | Cross-cutting: tests plus `server-spi-private` features and fixes. |
| Vlasta Ramik | 91 | Cross-cutting: SPI changes, 40 legacy arquillian touches. |
| Pedro Igor | 88 | Legacy arquillian (43), flaky tests, SPI. |
| Ricardo Martin | 87 | New CI and test moves. |
| Stian Thorgersen | 50 | Test framework: `Base4TestSuite`, migration of old tests, FIPS and dependency upgrades. |
| Lukas Hanusovsky | about 45 (author-list count, not area-specific) | New-framework tests and utilities (`ManagedRealm`, `EventAssertion`). |

Who to ask: Stian Thorgersen for the migration plan and end state. For the SPI boundary, the cross-cutting people (Stefan Guilhen, Vlasta Ramik, Pedro Igor) touch `server-spi-private` most.

## People who recur across areas (best first contacts)

- Stefan Guilhen: brokering, SCIM, FGAP, SPI (4 areas).
- Vlasta Ramik: organizations, FGAP, SPI (3).
- Pedro Igor: authorization, SCIM, legacy tests (3).
- Ricardo Martin: tokens, login flows, tests.
- Giuseppe Graziano: tokens, brokering.
- Marek Posolda: tokens, OID4VC.

## Caveats

- Area membership comes from path regexes and is approximate. Stian Thorgersen's counts in areas 1-3 include broad refactors and dependency commits. Stefan Guilhen's "core JOSE" count in area 1 looks inflated by organization representations.
- Theme labels (security fix, test, feature, refactor) come from commit-subject keywords and are rough.
- Departures were not checked. Thomas Diesler and Ryan Emerson were last active in these areas in July 2026; confirm availability before relying on them.
- Peter Zaoral, Michal Vavřík and Václav Muzikář have 21 to 67 commits overall but did not rank in any area table; they may work on tests or the Quarkus side, which was not checked.
- Agent detection is by author name; the script is not kept in the repo.
