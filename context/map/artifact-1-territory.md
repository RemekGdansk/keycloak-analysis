# Artifact 1: Territory (git-history activity map)

Window: last 12 months to 2026-10-17 (history starts 2013, so the full 12 months were used). Repo: keycloak. Method: non-merge commits; excluded bots (Dependabot, Weblate, github-actions), mass changes (>150 files or >30 renames; >100 files/>12 dirs for coupling), and noise (lockfiles, snapshots, generated, images, `.properties`, yaml/json/xml/md/adoc, `pom.xml`, `.github`, docs, test resources). Renames followed to current path. 3,051 commits counted after filters. AI-assisted commits (Claude `Assisted-by`/`Co-authored-by`, Copilot Autofix/agent, Cursor) are included in the counts and reported separately: 166 (5.4%), a lower bound because it relies on trailers.

## a) Top 10 modules (commits touching folder; AI in brackets)

1. `services/src/main` 1024 (37): server core, OIDC/SAML/OID4VCI endpoints, REST resources
2. `tests/base/src` 932 (33): new-framework integration tests (OID4VC, admin API v2, SCIM)
3. `testsuite/integration-arquillian/tests` 478 (9): legacy test suite, touched during migration
4. `server-spi-private/src/main` 324 (13): internal model/provider interfaces
5. `js/apps/admin-ui` 311 (15): Admin Console
6. `model/jpa/src` 214 (13): persistence, Liquibase
7. `quarkus/runtime/src` 164 (12): startup, CLI, config
8. `model/infinispan/src` 154 (11): caching/clustering
9. `core/src/main` 129 (2): shared representations, token/JWT model
10. `server-spi/src/main` 119 (6): public provider SPI

## b) Top files

`OID4VCIssuerEndpoint.java` 77, `Profile.java` 57 (feature flags), `TokenManager.java` 53, `OID4VCIssuerTestBase.java` 49, SCIM `UserTest.java` 42, `PicocliTest.java` 40, `ClientApiV2Test.java` 38, `DefaultClientService.java` 37 (**deleted 2026-09-18**, see below), `KeycloakProcessor.java` 34, `RealmBuilder.java` / `UserResourceTypeProvider.java` 33.

## Quarterly trend (Q1 2025-10-18..2026-01-16, Q2 ..04-17, Q3 ..07-17, Q4 ..10-17)

Commits per quarter 532 / 810 / 776 / 933; AI-trailer commits 0 / 9 / 42 / 115 (about 12% in Q4).

| Area | Q1/Q2/Q3/Q4 | Label |
|---|---|---|
| SCIM | 0/29/30/52 | growing (promoted to supported in Q4) |
| SSF / CAEP | 0/0/10/43 | growing, new campaign from Q3 |
| FGAP / authorization | 31/49/70/63 | growing then plateau (AuthZen in Q2, then bypass fixes) |
| Organizations | 15/50/44/55 | growing then steady |
| Broker/IdP | 45/50/38/80 | steady, Q4 security spike |
| OID4VC | 46/74/100/56 | seasonal, peak Q3 |
| Admin API v2 | 10/41/33/17 | seasonal, peak early, fading |
| Workflows | 57/32/10/11 | fading (promoted to supported Q2) |
| Admin UI | 66/89/77/72 | steady |
| OIDC/token | 37/58/78/68 | growing then plateau |
| DB/JPA | 26/39/65/49 | seasonal, peak Q3 |
| Quarkus/CLI/config | 92/118/80/85 | steady |
| Cache/cluster, authn flows, login UI, SAML | | steady |
| LDAP federation | 10/8/14/22 | growing (small base) |

Campaigns visible in messages:
- Security hardening wave: CVE/bypass/escalation/DoS-keyword commits 5 / 18 / 29 / 85.
- Migration of tests to the new test framework: 13 / 34 / 18 / 34 (Organization tests in Q2, `Base4TestSuite` in Q4).
- CIMD (Client ID Metadata Documents): 10 commits in Q2.
- Workflows built in Q1 and promoted in Q2; SCIM built in Q2 and promoted in Q4.

## Fix and revert share (top 5 areas; single primary-area attribution per commit)

Fix rule: subject contains fix/hotfix/bug/regression/NPE/flaky/CVE id. Revert rule: subject starts with "Revert".

| Area | Commits | Fix share | Reverts |
|---|---|---|---|
| Quarkus/CLI/config | 274 | 38% (103) | 3 |
| OID4VC | 221 | 15% | 0 |
| Admin UI | 217 | 25% | 0 |
| OIDC/token | 130 | 21% (11 security fixes) | 0 |
| Broker/IdP | 113 | 16% (6 flaky-test fixes) | 0 |

Project baseline 20% (471 of 2,340). Only 4 reverts overall.

Reading:
- Quarkus's 38% is inflated by the `fix:` commit-prefix habit; 90 of 103 are refinements, so it is not a breaking area.
- Admin UI fixes are minor UI issues and E2E flakiness.
- OIDC/token fixes are mostly serious security bugs (it is important and exposed).
- OID4VC work is mostly new feature code plus spec-conformance fixes.

Upstream `keycloak/keycloak` read-only GitHub data: open `kind/bug` issues were 9 (`area/dist/quarkus`), 4 (`area/oid4vc`), 30 (`area/admin/ui`), 28 (`area/oidc`) and 17 (`area/identity-brokering`). PR counts by label are unreliable because few PRs are labelled.

## Co-change couplings (module level, % of the less active module's commits)

2,498 commits analysed, 28 mass commits skipped.

| Pair | Together | % of less active |
|---|---|---|
| `services/src/main` + `tests/base/src` | 576 | 63% |
| `services/src/main` + arquillian tests | 290 | 62% |
| `server-spi-private/src/main` + `services/src/main` | 225 | 71% |
| `server-spi-private/src/main` + `tests/base/src` | 208 | 66% |
| `tests/base/src` + arquillian tests | 188 | 40% |
| `model/jpa/src` + `tests/base/src` | 104 | 50% |
| `js/apps/admin-ui` + `services/src/main` | 100 | 33% |
| `core/src/main` + `services/src/main` | 93 | 77% |
| `services/src/main` + `services/src/test` | 84 | 91% |

Strong smaller couplings:
- `tests/base/src` + `tests/utils-shared/src`: 95%.
- `quarkus/config-api/src` + `quarkus/runtime/src`: 92%.
- `scim/model/src` + `scim/tests/base`: 79%.
- `rest/admin-v2/services` + `rest/admin-v2/tests`: 74%.

Top triple: `server-spi-private` + `services/src/main` + `tests/base/src`, 165 commits (52%).

Conclusions:
- Tests are written with the code.
- Legacy and new suites are changed in lockstep during migration.
- The private SPI is not an independent boundary: 71% of its commits also touch `services`.

## Common-denominator files

All are hand-edited, and none is generated. Modules co-changed are in brackets.
- Root `pom.xml` (71): dependency and platform upgrades, plus wiring new modules.
- `.github/workflows/ci.yml` (47): 15 of 72 commits by Dependabot.
- `messages_en.properties` of Admin UI (44): 153 manual edits, about 6.6 lines each. The translated locales are automated by Weblate (54 commits) and are the cheap, bot-driven cost.
- `Profile.java` (42): feature-flag promotions.
- Per-release upgrade and release-note docs (24 to 39).
- `js/pnpm-lock.yaml`: 29 Dependabot edits and 22 human edits.

## Existence check

- All 12 directories tested in the top pairs still exist.
- 9 of the 10 files in the top-files table exist at HEAD.
- `DefaultClientService.java` was deleted 2026-09-18 (#52304, logic moved into the type provider). Several legacy OID4VC arquillian tests were also deleted or migrated (for example `OID4VCIssuerEndpointTest`, removed 2026-05-29 in #49476). About 6% of non-noise file touches (950 of 15,122) are on paths absent from HEAD.
- Do not cite `DefaultClientService.java` as a current hotspot.

## Caveats

- The "test-framework" bucket captures any commit touching `tests/`; do not read trends from it.
- AI detection is trailer-based, so it is a lower bound.
- "Seasonal" is a mid-year peak across 4 buckets, not a proven cycle.
- Rename tracking depends on Git similarity, and heavy rewrites show as delete plus add.
- Area attribution by path regexes is approximate, and fix and security keyword matching is loose.
