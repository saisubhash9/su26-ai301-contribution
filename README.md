# Contribution: [Adapter: Perplexity Sonar Model Provider]

**Contribution Number:** 1  
**Student:** Sai Subhash Manam  
**Issue:** https://github.com/orthogonalhq/nous-core/issues/314  
**Pull Request:** https://github.com/orthogonalhq/nous-core/pull/411  
**Branch:** https://github.com/saisubhash9/nous-core/tree/feat/314-perplexity-adapter  
**Status:** Phase III [Week 3 — Implementation Complete, PR Submitted]

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

Crucially, Perplexity's API is **OpenAI Chat Completions–compatible**. The docs' Protocol Decision Table maps this exactly to *"same protocol, different defaults/credentials → keep vendor-specific metadata in the leaf and reuse shared protocol logic."* This means the leaf should **reuse** `self/subcortex/providers/src/protocols/openai-api/**` rather than implement a custom wire protocol — so no `implementation.ts`, no custom `invoke()`/`stream()`. Perplexity needs only Bearer-token auth and its own endpoint/model defaults.

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
| `provider.ts` | `providerFactory` → `new ChatCompletionsProvider(config, { apiKey })`. |
| `adapter.ts` | Re-exports `chatCompletionsAdapter` as `providerAdapter`. |
| `index.ts` | Leaf public exports (required by the generator). |

**Review:** Passed `pnpm --filter @nous/subcortex-providers run check:generated`, `... run typecheck`, and the full package test suite. Commits follow Conventional Commits:
`feat(providers): add Perplexity (Sonar) certified provider leaf` and a follow-up `fix(providers): typecheck Perplexity discovery assertions via leaf type`.

**Evaluate:** Verified end-to-end through the definition-driven path: the catalog hydrates Perplexity with a derived built-in id, the factory constructs a `ChatCompletionsProvider` with the resolved key/endpoint, and a `perplexity`-vendor config resolves to the `chat-completions` adapter. Live API verification with a real key is pending (covered below).

---

## Testing Strategy

### Unit Tests

- [x] `adapter.ts` resolves to the shared `chat-completions` adapter, and a `perplexity`-vendor config resolves to that adapter key.
- [x] `provider.ts` factory constructs a `ChatCompletionsProvider` with the resolved API key and Perplexity endpoint/model.
- [x] `definition.ts` passes the framework's strict Zod validation (`ProviderDefinitionSchema`), declares the correct `auth.header` (Bearer), and intentionally omits dynamic model discovery.

### Integration Tests

- [x] Catalog hydration: Perplexity is present in `PROVIDER_DEFINITIONS` with a stable built-in id derived from `vendorKey` (no hand-authored UUID).
- [x] Codegen freshness + public-export + provider-pipeline integration tests pass with the new vendor.
- [ ] Live `invoke()`/`stream()` against the real Perplexity API — pending a provisioned `PERPLEXITY_API_KEY`. (Streaming behavior is already exercised generically by the shared `ChatCompletionsProvider` suite.)

**Result:** `315 passed | 2 skipped` across the providers package, including a new dedicated `perplexity-provider.test.ts` (10 tests). `check:generated` and `typecheck` both clean.

### Manual Testing

Pending a real Perplexity API key. The shared chat-completions provider (which Perplexity reuses verbatim) already has live/mocked coverage for request shape, auth, error mapping (401/403/429/timeout/abort), and streaming.

---

## Implementation Notes

### Week 1 Progress

- Set up the `nous-core` monorepo.
- Diagnosed and fixed a `node-gyp` compilation failure caused by an incompatible Node.js v26 install, pinning to v22 LTS via Conda.
- Connected the fork to upstream and created `feat/314-perplexity-adapter`.
- Read `.architecture` and the provider-adapter docs to understand "certified provider leaves," the generated-catalog workflow, and the testing checklist.
- Established the file-structure blueprint, choosing the OpenAI-compatible (chat-completions) path over a custom protocol.

### Week 3 Progress (Today — Implementation & PR)

- Synced the working branch to the tip of the integration branch (`feat/contributor-friendly-inference-provider-surface`) — the maintainer's required PR target for Cloud API adapters (explicitly **not** `dev`).
- Built the 4-file Perplexity leaf (`definition.ts`, `provider.ts`, `adapter.ts`, `index.ts`) reusing the `openai-api` protocol; regenerated the three provider catalogs via `pnpm generate:providers` (never hand-edited).
- Extended five existing vendor-roster tests and added a dedicated `perplexity-provider.test.ts` (10 tests).
- Confirmed **zero core/bootstrap/runtime edits**: the runtime resolves `PERPLEXITY_API_KEY` from `auth.envVar` and the endpoint from `defaultEndpoint` automatically.
- Final verification: `check:generated` ✅, `typecheck` ✅, full package suite **315 passed | 2 skipped**.
- CI initially failed on a type error in the new test (reading optional discovery fields off the narrowed `as const` leaf type); fixed by widening to `ProviderDefinitionLeaf` for those assertions, then re-pushed.
- Opened **PR [#411](https://github.com/orthogonalhq/nous-core/pull/411)** against the integration branch with a detailed writeup, including the structural edge flagged below.

### Structural Edge Flagged to Maintainer

Perplexity is the **first second vendor to share the `chat-completions` adapterKey** (alongside OpenAI). `ADAPTER_MODULES` is a flat `[...CERTIFIED_PROVIDER_ADAPTER_MODULES, textAdapter]` with no dedup, so `chat-completions` now appears **twice** in the raw aggregate. Resolution is unaffected — `moduleByKey` dedupes and both leaves re-export the *same* adapter singleton. Per the docs ("do not edit `adapter-resolver.ts` for a normal provider addition"), I left the resolver untouched and instead updated the resolver test to assert *distinct* adapter keys. The maintainer may want the generated aggregate deduped by `adapterKey` at the codegen layer eventually. Raised in the PR description.

### Also Flagged (Pre-existing, Unrelated)

`@nous/shared-server` typecheck currently fails in `bootstrap.ts` (`thoughtEmitter`) and `trpc/routers/chat.ts` (`sessionId`/`cards`/`empty_response_kind`/`thinking_unavailable`). Confirmed identical with my changes stashed, so it predates this work — flagged per the project's "may be stale/broken" caveat.
