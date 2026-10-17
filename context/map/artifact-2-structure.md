# Artifact 2: Structure (jdeps dependency, cycle and testability analysis)

Scope: `services/src/main`, `core/src/main`, `server-spi-private/src/main`, the three active modules in [artifact-1-territory.md](artifact-1-territory.md). Built with `./mvnw -pl services -am install -DskipTests` (HEAD 2827b3fa10, version 999.0.0-SNAPSHOT), analysed with JDK 25 `jdeps` on the module jars. Graphviz 16.1.0 for rendering.

## Method

- Package level: `jdeps -verbose:package` on the `keycloak-services` jar.
- Class level: `jdeps -verbose:class` on the three jars (3,257 classes, 16,098 class-to-class edges). Nested classes were merged into their top-level file.
- Runtime vs type-only: Java erases imports, so jdeps cannot tell them apart. Each edge was classified from bytecode with `javap -c`.
  - Runtime: the class appears in an instruction (`invoke*`, `new`, `checkcast`, `instanceof`, `anewarray`, field access, `ldc`).
  - Type-only: the class appears only in signatures, fields, generics or annotations.
- "Dist/build directory" imports do not apply to Java. The only generated source is `ServicesLogger_$logger`.
- Excluded as by design: Provider↔Factory pairs (SPI pattern, 11 pairs) and the `Closing*Stream` classes.

## Key findings

1. No cycles between the three modules. All 6,000+ cross-module edges go `services → server-spi-private → core`, none back. Every cycle is inside one module.
2. `services` has one runtime knot of 198 files (346 files when type references count). It contains `TokenManager`, `AuthenticationManager`, `AuthenticationProcessor`, `IdentityBrokerService`, `LoginActionsService`, `OIDCLoginProtocol`, and the FGAP and organization code. Package split of the 198: `services.resources` 63, `protocol.oidc` 36, `authorization.admin` 9, `authentication.actiontoken` 8.
3. Smallest real loops are almost all two files: 104 runtime two-file cycles (75 `services`, 25 `server-spi-private`, 4 `core`). Including type references there are 250 two-file cycles, 147 of them type-only on at least one side.
4. `OID4VCIssuerEndpoint` (77 commits, top file) is not in a runtime cycle. Its loops with `OID4VCUtil` and `PreAuthorizedCodeGrantType` are runtime one way only.
5. Package-level (`services` jar): 210 packages, 657 internal edges, one cycle group of 21 packages and one of 2. Highest fan-in: `provider` 170, `utils` 56, `broker.provider` 43, `crypto` 39, `authentication` 36, `protocol` 34.

## Cycles by active area

| Area | Smallest runtime loops | Note |
|---|---|---|
| OIDC/token | `TokenManager` ↔ `UserSessionUtil`, `DefaultClientSessionContext`, `InitialRefreshTokenContext`, `AuthenticationManager`, `OrganizationScope` | `OIDCLoginProtocol` ↔ `TokenManager` is runtime one way only |
| Authn flows | `AuthenticationProcessor` ↔ `DefaultAuthenticationFlow`, `FormAuthenticationFlow`, `ClientAuthenticationFlow`, `LoginActionsService`, `AuthenticationFlowURLHelper`; `AuthenticationManager` ↔ `AuthenticationSessionManager`, `UserConsentManager`, `UserSessionUtil`, `IdentityBrokerService` | 12 runtime 2-cycles, all in the 198-file SCC |
| Broker/IdP | `IdentityBrokerService` ↔ `AuthenticationManager`, `TokenManager`, `Urls`, `SerializedBrokeredIdentityContext`; `SAMLEndpoint` ↔ `SAMLIdentityProvider`; OID4VP has its own 4-file SCC | OID4VP loop is self-contained |
| FGAP/authorization | `server-spi-private`: 17-file SCC (`fgap.evaluation.*`, `policy.evaluation.*`), `AuthorizationProvider` ↔ `Evaluators`, `RepresentationToModel`, `AdminPermissionsSchema`; `services`: `MgmtPermissions` ↔ 7 `*Permissions` classes | Cut candidate: `AdminPermissionsSchema` → `RepresentationToModel` |
| Organizations | `OrganizationScope` ↔ `Organizations`, `TokenManager` | Org changes can alter issued tokens |
| OID4VC | `CredentialBuildConfig` ↔ `SupportedCredentialConfiguration`; `UserVerifiableCredentialResource` ↔ `UserResource` | Mostly benign |
| Workflows | `Workflow` ↔ `WorkflowProvider`, `WorkflowStep` ↔ `Workflows`, `WorkflowExecutor` ↔ `WorkflowTask`, `WorkflowRepresentation` ↔ `WorkflowStateRepresentation` | Small, contained, area fading |
| `core` JOSE | `Algorithm` ↔ `RSAProvider` (4-file SCC with `JWSHeader`, `JWSInput`); JWE 5-file SCC; `EdECUtilsImpl` ↔ `JWKBuilder` | Low change rate, wide dependants |

## Testability risks (`services/src/main`)

Per-file counts come from `jdeps` on the `services` jar plus `javap`. The JAX-RS, event and static-state counts are rough: regexes are loose, and the static-mutable count was unreliable (every file showed 1, likely the static initializer), so it was not used.

- `services/src/test` has 129 test files and no Mockito. Tests are pure logic, or build a real `ResteasyKeycloakSession` and factory (as `TokenManagerTest` does).
- Heaviest files by Keycloak classes used: `TokenManager` 120, `SamlService` 111, `RealmAdminResource` 110, `AuthenticationManager` 109, `IdentityBrokerService` 109, `OID4VCIssuerEndpoint` 105, `SAMLEndpoint` 100, `UserResource` 99, `LoginActionsService` 96.
- `KeycloakSession` calls: `AuthenticationManager` 70, `IdentityBrokerService` 54, `UserResource` 43, `RealmAdminResource` 34, `LoginActionsService` 34, `OID4VCIssuerEndpoint` 33.

| Where | Best test level |
|---|---|
| `TokenManager` | Integration (`tests/base` oauth/oidc); one unit test exists |
| `AuthenticationManager` | Integration to e2e (login, logout, consent) |
| `IdentityBrokerService` | E2E with a real or fake IdP (1 `tests/base` file mentions it) |
| `LoginActionsService`, `AuthenticationProcessor` | E2E with a browser (0 `tests/base` refs, 3 arquillian) |
| `OID4VCIssuerEndpoint` | Integration (19 `tests/base` files reference it) |
| `RealmAdminResource`, `UserResource`, `ClientResource` | Integration via Admin API; Admin UI is the only e2e consumer |
| `MgmtPermissions` and `*Permissions` | Integration via authorization tests; no test names the class |
| `SAMLEndpoint`, `SAMLIdentityProvider` | Integration; only `SAMLParsingTest` covers parsing |
| `AbstractOAuth2IdentityProvider`, `OIDCIdentityProvider` | Integration with a mock IdP |
| `Organizations` | Integration (14 `tests/base` files) |
| `FreeMarkerLoginFormsProvider` | E2E or theme rendering test |

Most suspicious modules: `protocol.oidc` (most security fixes, hub class), `services.managers` + `services.resources` (the 198-file knot), `services.resources.admin`, then `protocol.oid4vc` (large file, but covered).

## Subgraph

`knot.svg` (also `knot.dot`, `knot.png`): runtime two-file cycles around the five hubs `TokenManager`, `AuthenticationManager`, `AuthenticationProcessor`, `IdentityBrokerService`, `LoginActionsService` (23 files). Red two-way arrow means a runtime loop, grey means one direction only. Loops between two non-hub files are left out, so it understates density. `AuthenticationManager` has a red loop with each of the other four hubs. `Urls` and `UserSessionUtil` are small shared files with red loops back to the hubs, so they are the cheapest cuts. In the PNG preview the first letter of one label ("IdentityBrokerService") was clipped; the SVG was not checked.

## Relation to artifact 1

- `TokenManager` (53 commits, security fixes) is the biggest hub and the top fix target.
- `server-spi-private` shares 71% of its commits with `services`; `core/src/main` shares 77% of its commits with `services`. No module-level cycle explains this, so it is co-change through shared types.
- `services/src/main` + `tests/base/src` co-change at 63%, consistent with integration tests being the main safety net.

## Open items

- List which of the 198 files `TokenManager`, `AuthenticationProcessor` and `IdentityBrokerService` pull in, and the cheapest edges to cut.
- Check with `git log` whether the hub pairs (for example `TokenManager` and `UserSessionUtil`) change together, and whether fix commits there also touched `tests/base` or arquillian.
- Scan for mutable static fields properly.
- Find which tests exercise `MgmtPermissions`.
- Decide whether the `AuthenticationProcessor` ↔ flow classes loop is intentional.

## Caveats

- Only these three modules were analysed. `model`, `federation`, `quarkus` and the rest of the repo were not.
- The runtime/type-only split is derived from bytecode, so a type-only edge can still matter at compile time and for class loading.
- Reflection and SPI lookups (`session.getProvider(...)`) are not edges in jdeps, so real coupling through providers is higher than shown.
- Intermediate scripts and data (`.py`, `.pkl`) were deleted at the user's request, so these numbers cannot be regenerated from this folder.
