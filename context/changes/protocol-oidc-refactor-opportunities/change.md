---
change_id: protocol-oidc-refactor-opportunities
title: Rank protocol-oidc technical debt as refactor opportunities
status: plan_reviewed
created: 2026-10-18
updated: 2026-10-18
archived_at: null
---

## Notes

**Intent:** we have an analysis of a module that documents its technical debt and structural risks: `context/changes/analyze-protocol-oidc/research.md`. This change answers the question that analysis deliberately left open: **WHICH** of those problems are worth fixing, in what target shape, and in what order. We explore each recorded problem in the code and its history, then organise them as refactor opportunities.

The change runs in stages: exploration → decision and plan → implementation. During exploration, no refactoring happens and no decision gets made.

**Exploration output:** this change's own `research.md`, ending with a ranking of the options and their trade-offs. I'll read the report first. We decide what to do during planning, and refactoring starts only once the plan has been agreed.
