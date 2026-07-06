# Contribution: [Adapter: Perplexity Sonar Model Provider]

**Contribution Number:** 1  
**Student:** Sai Subhash Manam  
**Issue:** https://github.com/orthogonalhq/nous-core/issues/314  
**Pull Request:** https://github.com/orthogonalhq/nous-core/pull/411  
**Branch:** https://github.com/saisubhash9/nous-core/tree/feat/314-perplexity-adapter  
**Status:** Phase IV [Complete — Merged ✅]

> **Note on PR base branch:** This PR targets the upstream integration branch
> `feat/contributor-friendly-inference-provider-surface` on `orthogonalhq/nous-core`,
> **not** `main`. The maintainer explicitly required Cloud API provider adapters to
> stack against that integration branch (not `dev`/`main`) until the provider-surface
> work is fully integrated. It is a real PR against the upstream repository (not a
> draft against my fork) and has been **merged**.

---

## Why I Chose This Issue

I wanted to tackle an issue that bridges backend tool development with Large Language Model APIs. Because I frequently utilize LLM APIs to build automated tools and optimize complex workflows, building a dedicated model provider adapter is a natural extension of my current skill set.

Working on the `nous-core` framework also gives me a great opportunity to look under the hood of a local-first AI agent architecture. By implementing the Perplexity Sonar provider, I gained hands-on experience with how agentic frameworks standardize communication across diverse, external LLM endpoints using interface patterns — and, importantly, how a well-designed framework lets you add a whole new vendor *without* touching its runtime.

---

## Understanding the Issue

### Problem Description

The `nous-core` project currently lacks native support for Perplexity's API, specifically their Sonar models. For the local agent to communicate with Perplexity, a certified provider leaf that satisfies the framework's `ProviderDefinitionLeaf` / `ProviderFactoryModule` / `ProviderAdapter` contracts needs to be created. Without this, users cannot route agent tasks through Perplexity's infrastructure.

### Expected Behavior

Developers and users of `nous-core` should be able to select Perplexity Sonar as their active model provider. Once the user supplies the necessary Perplexity API key (`PERPLEXITY_API_KEY`), the framework should seamlessly establish a connection and route prompts/responses through the Perplexity API just as it does for other supported models.

### Current Behavior

Attempting to use Perplexity Sonar models natively within `nous-core` is currently impossible. Because there is no dedicated provider leaf, the framework has no vendor identity, endpoint, credential metadata, or factory to instantiate the provider.

### Affected Components

This implementation primarily touches the provider package under `self/subcortex/providers/`. Specifically:
* A new certified provider leaf under `self/subcortex/providers/src/providers/perplexity/`.
* A small, backward-compatible change to the shared `chat-completions` protocol provider (`protocols/openai-api/provider.ts`) — an optional `completionsPath` so OpenAI-compatible vendors that use a different path (Perplexity uses `/chat/completions`, no `/v1`) can override it. The default is unchanged.
* The **generated** provider catalogs (`provider-definitions.ts`, `provider-adapters.ts`, `provider-factories.ts`), which are regenerated — never hand-edited.
* Existing test suites that exhaustively enumerate the vendor roster.

Notably, **no changes to bootstrap, runtime, or `@nous/shared` were required** — the integration is fully definition-driven.

---

## Reproduction Process

### Environment Setup

Setting up the local development environment for this monorepo presented a specific challenge regarding Node.js version compatibility with native C++ modules.

1. **Initial Attempt:** I created a Conda environment (`nous`) and installed Node.js. It defaulted to a too-new major (v26).
2. **The Error:** Running `pnpm install` caused `node-gyp` to fail when compiling the `better-sqlite3` native module. The C++ V8 API in Node 26 introduced breaking changes (`GetPrototype` removed, `Value` deprecated), causing the build to crash.
3. **The Solution:** `CONTRIBUTING.md` specifies Node 22+. I pinned the environment to Node 22 (LTS) via `conda install -c conda-forge nodejs=22`. I cleared `node_modules`, re-enabled Corepack (`corepack enable && corepack prepare pnpm@latest --activate`), and ran `pnpm install` again.
4. **Success:** Native compilation succeeded. The active toolchain is now **Node v22.22.3 / pnpm 10.6.2**, and `pnpm build` / `pnpm test` pass.

### Steps to Reproduce
*(For this specific issue, reproduction means setting up the codebase and verifying Perplexity is absent.)*
1. Fork and clone the `nous-core` repository.
2. Inspect `self/subcortex/providers/src/providers/`.
3. Observe that folders exist for `anthropic`, `openai`, `ollama`, and `codex-cli`, but **`perplexity` is missing**.

### Reproduction Evidence

- The environment is fully stable. Git remotes are configured (`origin` → my fork, `upstream` → `orthogonalhq/nous-core`).
- I synced my working branch `feat/314-perplexity-adapter` to the tip of the active integration branch `feat/contributor-friendly-inference-provider-surface` before starting (the maintainer's required PR target for Cloud API adapters — not `dev`).

---

## Solution Approach

### Analysis

Per the live [provider-adapter docs](https://docs.nue.orthg.nl/docs/development/provider-adapters), model provider adapters are "certified provider leaves" with a fixed file shape. The **Anthropic leaf** is the reference for structure/naming/exports/tests.

Crucially, Perplexity's API is **OpenAI Chat Completions–compatible**. The docs' Protocol Decision Table maps this exactly to *"same protocol, different defaults/credentials → keep vendor-specific metadata in the leaf and reuse shared protocol logic."* This means the leaf should **reuse** `self/subcortex/providers/src/protocols/openai-api/**` rather than implement a custom wire protocol — so no `implementation.ts`, no custom `invoke()`/`stream()`. Perplexity needs only Bearer-token auth and its own endpoint/model/path defaults.

### Proposed Solution

Create a new provider leaf at `self/subcortex/providers/src/providers/perplexity/` that declares Perplexity's identity/endpoint/credentials as metadata and delegates request execution and response parsing to the shared `chat-completions` protocol and adapter.

### Implementation Plan (UMPIRE)

**Understand:** The system needs a strictly-typed, metadata-only leaf that routes agent requests to Perplexity's Sonar models through the existing chat-completions path, with credentials and endpoint flowing in via the framework's definition-driven plumbing.

**Match:** Mirror the file shape of `self/subcortex/providers/src/providers/openai/` (the existing OpenAI-compatible leaf), which itself wraps `protocols/openai-api/provider.ts`. Use Anthropic as the reference for leaf conventions and test coverage.

**Plan:**
1. Scaffold `self/subcortex/providers/src/providers/perplexity/`.
2. `definition.ts` — metadata-only `ProviderDefinitionLeaf`: `vendorKey`, `displayName`, `protocol`/`adapterKey: 'chat-completions'`, `defaultEndpoint`, `defaultModelId: 'sonar'`, `auth`, `capabilities`. **No** hand-authored `wellKnownProviderId` (derived from `vendorKey`).
3. `provider.ts` — `providerFactory` wrapping the shared `ChatCompletionsProvider`.
4. `adapter.ts` — re-export the shared `chat-completions` adapter.
5. `index.ts` — export the leaf's public surface.
6. Regenerate catalogs with `pnpm --filter @nous/subcortex-providers run generate:providers`.
7. Add a dedicated test file plus extend the existing vendor-roster assertions.

**Implement:** ✅ Completed on branch `feat/314-perplexity-adapter`. Final leaf:

| File | Responsibility |
|---|---|
| `definition.ts` | `vendorKey: 'perplexity'`, `protocol`/`adapterKey: 'chat-completions'`, `defaultEndpoint: 'https://api.perplexity.ai'`, `defaultModelId: 'sonar'`, `auth` (env `PERPLEXITY_API_KEY`, vault ns `perplexity`, `Authorization`/`bearer`, `required`, `purpose: 'api_key'`), `capabilities.streaming`. No discovery, no `wellKnownProviderId`. |
| `provider.ts` | `providerFactory` that resolves `PERPLEXITY_API_KEY`, **fails closed** if absent, and constructs `ChatCompletionsProvider` with `completionsPath: '/chat/completions'`. |
| `adapter.ts` | Re-exports `chatCompletionsAdapter` as `providerAdapter`. |
| `index.ts` | Leaf public exports (required by the generator). |

**Review:** Passed `pnpm --filter @nous/subcortex-providers run check:generated`, `... run typecheck`, and the full package test suite. Commit follows Conventional Commits: `feat(providers): add Perplexity (Sonar) certified provider leaf` (review fixes folded in on rebase).

**Evaluate:** Verified end-to-end through the definition-driven path: the catalog hydrates Perplexity with a derived built-in id; the factory constructs a `ChatCompletionsProvider` with the resolved key/endpoint; a `perplexity`-vendor config resolves to the `chat-completions` adapter; and a mocked-fetch test asserts the composed request URL is exactly `https://api.perplexity.ai/chat/completions`. Live API verification with a real key is pending (covered below).

---

## Testing Strategy

### Unit Tests

- [x] `adapter.ts` resolves to the shared `chat-completions` adapter, and a `perplexity`-vendor config resolves to that adapter key.
- [x] `provider.ts` factory constructs a `ChatCompletionsProvider` with the resolved API key and Perplexity endpoint/model.
- [x] `provider.ts` factory **fails closed** — throws when no `PERPLEXITY_API_KEY` is present, even if `OPENAI_API_KEY` is set (no silent OpenAI fallback).
- [x] Composed request URL is `https://api.perplexity.ai/chat/completions` (no `/v1`); the shared provider keeps its `/v1/chat/completions` default and honours the `completionsPath` override.
- [x] `definition.ts` passes the framework's strict Zod validation (`ProviderDefinitionSchema`), declares the correct `auth.header` (Bearer), and intentionally omits dynamic model discovery.

### Integration Tests

- [x] Catalog hydration: Perplexity is present in `PROVIDER_DEFINITIONS` with a stable built-in id derived from `vendorKey` (no hand-authored UUID).
- [x] Codegen freshness + public-export + provider-pipeline integration tests pass with the new vendor (alongside the other vendors that landed on the integration branch).
- [ ] Live `invoke()`/`stream()` against the real Perplexity API — pending a provisioned `PERPLEXITY_API_KEY`. (Streaming behavior is already exercised generically by the shared `ChatCompletionsProvider` suite.)

### Test Evidence (before/after)

**Before** — no Perplexity leaf exists:

```
$ ls self/subcortex/providers/src/providers/
anthropic  codex-cli  github-copilot-cli  groq  llama-cpp  ollama  openai
# → no `perplexity/` directory
```

**After** — leaf added, catalogs regenerated, full suite green:

```
$ pnpm --filter @nous/subcortex-providers run check:generated
> node scripts/generate-provider-aggregates.mjs --check
# (exit 0 — generated catalogs are fresh)

$ pnpm --filter @nous/subcortex-providers run typecheck
# TYPECHECK EXIT: 0  (no TS errors)

$ pnpm --filter @nous/subcortex-providers exec vitest run --config vitest.config.ts
 Test Files  27 passed | 1 skipped (28)
      Tests  377 passed | 2 skipped (379)
```

Targeted proof of the endpoint fix (mocked `fetch`):

```
✓ Perplexity request URL composition > calls Perplexity at /chat/completions (no /v1 prefix)
    → fetch called with 'https://api.perplexity.ai/chat/completions'
✓ Perplexity factory fails closed > throws instead of falling back to OPENAI_API_KEY
```

### Manual Testing

Pending a real Perplexity API key. The shared chat-completions provider (which Perplexity reuses) already has live/mocked coverage for request shape, auth, error mapping (401/403/429/timeout/abort), and streaming.

---

## Challenges Faced

Real obstacles hit during this contribution and how each was resolved:

1. **Node.js 26 broke the native build.** `pnpm install` failed compiling `better-sqlite3` under Node 26 (`node-gyp` errors from removed/deprecated V8 C++ APIs). *Resolved by* pinning the Conda env to Node 22 LTS, clearing `node_modules`, re-enabling Corepack, and reinstalling.

2. **Wrong endpoint path from reusing the shared provider.** The shared `ChatCompletionsProvider` hardcoded `/v1/chat/completions`, but Perplexity's OpenAI-compatible endpoint is `/chat/completions` (no `/v1`) — so the composed URL would have 404'd. *Resolved by* verifying the real path against Perplexity's API reference and Spring AI, then making the path **configurable** (`completionsPath`, OpenAI default preserved) and having the leaf pass `/chat/completions`, with a URL-assertion test.

3. **Credential-leak risk via OpenAI fallback.** The shared provider falls back to `OPENAI_API_KEY` when no key is passed, which could send an OpenAI credential to `api.perplexity.ai`. *Resolved by* making the Perplexity factory resolve `PERPLEXITY_API_KEY` and **fail closed** (throw) if absent, with a test proving it throws even when `OPENAI_API_KEY` is set.

4. **CI type error not caught locally.** After adding the dedicated test file I only ran Vitest (which strips types without checking), so a `tsc` error — reading optional discovery fields off the narrowed `as const` leaf — slipped through and failed CI. *Resolved by* widening to `ProviderDefinitionLeaf` for those assertions, and adopting a habit of running the package `typecheck` (not just tests) before pushing.

5. **PR carried unrelated docs churn.** The branch had picked up `docs/app/**` + `docs/lib` changes from `docs: update NueOS documentation branding` merge commits that weren't part of the integration branch. *Resolved by* rebasing onto the current integration tip to drop them, leaving a provider-leaf-only diff.

6. **Rebase reconciliation with parallel provider work.** While in review, the integration branch gained **groq**, **github-copilot-cli**, and **llama-cpp** leaves, which touched the same generated catalogs and vendor-roster tests. *Resolved by* rebasing onto the new tip, regenerating the catalogs, and reconciling the roster assertions to include both the new vendors and `perplexity`. Residual generated-catalog/roster merge churn from several leaves landing in parallel was resolved maintainer-side during merge (tracked by **#414**).

---

## Implementation Notes

### Week 1 Progress

- Set up the `nous-core` monorepo.
- Diagnosed and fixed a `node-gyp` compilation failure caused by an incompatible Node.js v26 install, pinning to v22 LTS via Conda.
- Connected the fork to upstream and created `feat/314-perplexity-adapter`.
- Read `.architecture` and the provider-adapter docs to understand "certified provider leaves," the generated-catalog workflow, and the testing checklist.
- Established the file-structure blueprint, choosing the OpenAI-compatible (chat-completions) path over a custom protocol.

### Week 3 Progress — Implementation & PR (2026-06-22)

- Synced the working branch to the tip of the integration branch (`feat/contributor-friendly-inference-provider-surface`) — the maintainer's required PR target for Cloud API adapters (explicitly **not** `dev`).
- Built the 4-file Perplexity leaf (`definition.ts`, `provider.ts`, `adapter.ts`, `index.ts`) reusing the `openai-api` protocol; regenerated the three provider catalogs via `pnpm generate:providers` (never hand-edited).
- Extended five existing vendor-roster tests and added a dedicated `perplexity-provider.test.ts` (10 tests at this point).
- Confirmed **zero core/bootstrap/runtime edits**: the runtime resolves `PERPLEXITY_API_KEY` from `auth.envVar` and the endpoint from `defaultEndpoint` automatically.
- Final verification at this stage: `check:generated` ✅, `typecheck` ✅, suite **315 passed | 2 skipped** (the integration branch had fewer vendors then).
- CI initially failed on a type error in the new test (reading optional discovery fields off the narrowed `as const` leaf type); fixed by widening to `ProviderDefinitionLeaf` for those assertions, then re-pushed.
- Opened **PR [#411](https://github.com/orthogonalhq/nous-core/pull/411)** against the integration branch with a detailed writeup.

### Week 4 Progress — Maintainer Review Round 1 (2026-06-28)

The maintainer reviewed the PR as an early-access provider integration and requested three changes before merge. All addressed and force-pushed (branch commit `45c906fd`):

**1. Endpoint/path composition — confirmed wrong, fixed.**
The maintainer suspected the composed URL was off. I verified against [Perplexity's API reference](https://docs.perplexity.ai/api-reference/chat-completions-post) and [Spring AI](https://docs.spring.io/spring-ai/reference/api/chat/perplexity-chat.html): Perplexity's OpenAI-compatible endpoint is **`https://api.perplexity.ai/chat/completions` — no `/v1`**. The shared `ChatCompletionsProvider` hardcoded `/v1/chat/completions`, which would have produced a 404. Rather than hardcode a special case, I made the completions path **configurable** (`completionsPath` option, default `/v1/chat/completions` so OpenAI/groq/llama-cpp are unchanged), and the Perplexity leaf passes `/chat/completions`. Added a mocked-fetch test asserting the exact request URL, plus a default/override regression test on the shared provider. This is the useful signal for the provider-surface cleanup tracked in **#413**.

**2. Prevent fallback to `OPENAI_API_KEY` — fail closed.**
The shared provider falls back to `process.env.OPENAI_API_KEY` when no key is supplied, which could send an OpenAI credential to `api.perplexity.ai`. The Perplexity factory now resolves `PERPLEXITY_API_KEY` explicitly and **throws** if it is absent, so the OpenAI fallback path is never reachable for this vendor. Added a test proving it throws even when `OPENAI_API_KEY` is set.

**3. Removed unrelated docs-app changes.**
The PR was carrying `docs/app/**` and `docs/lib/layout.shared.tsx` changes that came from `docs: update NueOS documentation branding` merge commits not part of the integration branch. I rebased onto the current integration tip to drop them — the diff is now provider-leaf-only (15 files, all under `self/subcortex/providers/`).

**Rebase reconciliation.** The integration branch had advanced and added **groq**, **github-copilot-cli**, and **llama-cpp** providers — touching the same generated catalogs and roster tests. I rebased onto the new tip, regenerated the catalogs, and reconciled the roster assertions so they include both the new vendors and `perplexity`.

**Re-verification:** `check:generated` ✅, `typecheck` ✅, suite **377 passed | 2 skipped** (now includes 13 Perplexity tests). Force-pushed the cleaned branch and replied on the PR mapping each review point to its fix.

### Week 5 Progress — Merged (2026-07-05) ✅

- The maintainer **merged the PR as the initial early-access Perplexity Sonar provider leaf.**
- Confirmed all three requested PR-local changes were accepted: Perplexity uses `/chat/completions`, fails closed instead of falling back to `OPENAI_API_KEY`, and the unrelated docs-app changes were removed.
- The small shared `ChatCompletionsProvider` path override is covered by tests and preserves the OpenAI default `/v1/chat/completions` behavior.
- Remaining generated-catalog and global roster-test conflicts were **maintainer-side merge churn** from several provider leaves landing in parallel (tracked by **#414**); the maintainer resolved those during the merge.
- Contribution #1 is complete. Next: pick up a follow-up issue (candidate: the `#413` provider-surface cleanup, which this work already nudged forward).

### Code Changes

- **New files (leaf):** `providers/perplexity/{definition,provider,adapter,index}.ts` and `__tests__/perplexity-provider.test.ts`.
- **Modified:** `protocols/openai-api/provider.ts` (backward-compatible `completionsPath`); regenerated catalogs `provider-{definitions,adapters,factories}.ts`; roster tests `adapter-resolver.test.ts`, `provider-codegen.test.ts`, `provider-definitions/provider-definition-types.test.ts`, `provider-definitions/provider-definitions.test.ts`, `provider-pipeline-integration.test.ts`; and a default/override path test in `chat-completions-provider.test.ts`.
- **Key commits:** `45c906fd` — `feat(providers): add Perplexity (Sonar) certified provider leaf` (review fixes folded in on rebase). Merged via PR #411 (upstream merge commit: `<fill-from-PR-page>`).
- **Approach decisions:** reuse the shared `chat-completions` protocol instead of a custom implementation; keep the leaf metadata-only with a derived provider id; make the path configurable rather than special-casing Perplexity; fail closed on credentials; keep the diff provider-leaf-only.

---

## Pull Request

**PR Link:** https://github.com/orthogonalhq/nous-core/pull/411
**Base branch:** `feat/contributor-friendly-inference-provider-surface` (upstream integration branch — the maintainer's required target for Cloud API adapters; see the note at the top of this README).

**PR Description (summary):** Adds an OpenAI Chat Completions–compatible Perplexity (Sonar) certified provider leaf reusing the shared `chat-completions` protocol; regenerates the provider catalogs; extends the vendor-roster tests. Definition-driven with no bootstrap/runtime/`@nous/shared` changes. **Closes #314.**

**Acceptance Criteria Checklist:**
- [x] Tests added (`perplexity-provider.test.ts`, 13 tests) + shared-provider path regression test.
- [x] All tests passing (`377 passed | 2 skipped`).
- [x] `typecheck` clean (exit 0) and generated catalogs fresh (`check:generated`).
- [x] Follows project style / provider-leaf docs (certified leaf shape, no hand-authored `wellKnownProviderId`, generated catalogs not hand-edited).
- [x] No breaking changes — the shared `completionsPath` default preserves OpenAI's `/v1/chat/completions` behavior; change is additive/backward-compatible.
- [x] References the issue with `Closes #314`.

**Maintainer Feedback log:**
- **2026-06-28 — Review Round 1 (maintainer):** Requested three changes before merge — (1) verify/correct the endpoint path (`/v1/chat/completions` vs Perplexity's `/chat/completions`), (2) prevent fallback to `OPENAI_API_KEY`, (3) remove unrelated docs-app changes. Non-blocking follow-ups noted: `nativeToolUse` handling (#390) and the broader protocol/adapter capability-source cleanup (#413).
- **2026-06-28 — My response (commit `45c906fd`):** Addressed all three — configurable `completionsPath` + URL test; fail-closed factory + test; rebased onto the integration tip to drop docs churn and reconcile with the newly-landed groq/github-copilot-cli/llama-cpp leaves. Re-verified (`377 passed | 2 skipped`), force-pushed, and commented on the PR mapping each point to its fix.
- **2026-07-05 — Merge (maintainer):** Merged as the initial early-access Perplexity Sonar provider leaf; confirmed the requested changes were addressed and that the shared path override preserves the OpenAI default. Residual generated-catalog/roster conflicts from parallel provider leaves were resolved maintainer-side (tracked by #414). *(Upstream merge commit: `<fill-from-PR-page>`.)*

**Status:** **Merged ✅**

---

## Learnings & Reflections

### Technical Skills Gained

- The "certified provider leaf" pattern: definition-driven vendor integration where identity/endpoint/credentials are declarative metadata and the runtime wires everything from `vendorKey` + `auth.envVar` + `defaultEndpoint` — no runtime branches.
- Reusing a shared wire protocol (`chat-completions`) across vendors, and how to extend it safely (backward-compatible option) instead of forking it.
- Generated-catalog codegen workflows (`generate:providers` / `check:generated`) and why generated files must never be hand-edited.
- Strict Zod ABI contracts, TypeScript `as const satisfies` narrowing (and its sharp edges in tests), and Vitest patterns for mocking `fetch`.
- Practical git hygiene: rebasing onto a moving integration branch, dropping unwanted commits, reconciling parallel changes, and keeping a PR diff scoped.

### Challenges Overcome

See the **Challenges Faced** section above — the most instructive were the Node 26 native-build failure, the incorrect endpoint-path assumption from protocol reuse, and the OpenAI-key fallback risk.

### What I'd Do Differently Next Time

- Run the package **`typecheck`** (not just Vitest) before every push — Vitest transpiles without type-checking, which let a `tsc`-only error reach CI.
- Branch off the **clean integration tip** from day one to avoid inheriting unrelated merge/docs churn.
- **Verify third-party API specifics early** (endpoint path, auth header, model IDs) rather than assuming an "OpenAI-compatible" API is byte-for-byte identical — the `/v1` path difference was the key example.

**Teachable insight for future cohorts:** "OpenAI-compatible" does not mean "OpenAI-identical." Reusing a shared protocol client is the right instinct, but always verify the *path*, *auth header scheme*, and *credential-fallback behavior* for the new vendor — and prefer making the shared client configurable over special-casing. A silent `OPENAI_API_KEY` fallback in a shared client is a real cross-vendor credential-leak footgun; new provider factories should **fail closed**.

---

## Resources Used

- [nous-core provider-adapter docs](https://docs.nue.orthg.nl/docs/development/provider-adapters) — quickstart, provider-leaf anatomy, schemas/ABI reference, generated-catalogs workflow, testing checklist.
- [Perplexity API reference — Chat Completions](https://docs.perplexity.ai/api-reference/chat-completions-post) — confirmed the `/chat/completions` (no `/v1`) endpoint.
- [Spring AI — Perplexity Chat](https://docs.spring.io/spring-ai/reference/api/chat/perplexity-chat.html) — cross-check of base URL / OpenAI-compatibility.
- `CONTRIBUTING.md` and `.architecture` in the repo — Node 22+ requirement and provider-leaf tiering.
- Related issues: **#390** (nativeToolUse capability), **#413** (provider-surface cleanup), **#414** (parallel-leaf catalog/roster merge churn).

---

### Also Flagged (Pre-existing, Unrelated)

`@nous/shared-server` typecheck fails in `bootstrap.ts` (`thoughtEmitter`) and `trpc/routers/chat.ts` (`sessionId`/`cards`/`empty_response_kind`/`thinking_unavailable`). Confirmed identical with my changes stashed, so it predates this work — flagged per the project's "may be stale/broken" caveat.
