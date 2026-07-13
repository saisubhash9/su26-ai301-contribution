# Contribution 2: [Adapter: Issue 300 Model Provider]

**Contribution Number:** 2  
**Student:** Sai Subhash Manam  
**Issue:** https://github.com/orthogonalhq/nous-core/issues/300  
**Status:** Phase I [In Progress]

---

## Why I Chose This Issue

Building on my previous work integrating the Perplexity adapter (#314), I wanted to tackle another model provider integration to solidify my understanding of the `nous-core` framework. In my ongoing AI301 Open Source Capstone work, as well as my experience running large-scale data processing pipelines on high-performance Slurm clusters, standardizing communication across different toolsets is critical. 

Tackling Issue #300 allows me to apply the "certified provider leaf" pattern I learned in my first contribution more efficiently, while focusing on the unique authentication schema, endpoint routing, and capability mapping of a new external API.

---

## Understanding the Issue

### Problem Description

The `nous-core` project currently lacks a native certified provider leaf for the API requested in Issue #300. Without this dedicated adapter, the local agent cannot establish vendor identity or route tasks through this specific provider's models.

### Expected Behavior

Users should be able to select this model provider, supply their specific API credential, and have the framework seamlessly connect and route standard agent prompts through the provider's infrastructure.

### Current Behavior

Attempting to use the provider natively is currently impossible. The provider is absent from the `self/subcortex/providers/src/providers/` directory, meaning the framework has no metadata, endpoint defaults, or factory to instantiate it.

### Affected Components

This implementation will primarily touch the provider package:
* A new certified provider leaf under `self/subcortex/providers/src/providers/<vendor_name>/`.
* The generated provider catalogs (`provider-definitions.ts`, `provider-adapters.ts`, `provider-factories.ts`), which will be regenerated.
* Vendor-roster test suites to exhaustively enumerate the new provider.

---

## Reproduction Process

### Environment Setup

My local environment is already stabilized from my previous contribution. The active toolchain is pinned to **Node v22.22.3 / pnpm 10.6.2** using a Conda environment, which circumvents the native C++ module compilation errors (`better-sqlite3` and `node-gyp`) present in Node 26. 

### Steps to Reproduce

1. Sync the local fork with the `orthogonalhq/nous-core` upstream repository.
2. Checkout the integration branch `feat/contributor-friendly-inference-provider-surface`.
3. Inspect `self/subcortex/providers/src/providers/`.
4. Observe that the target provider's directory and configuration files are missing.

### Reproduction Evidence

- **My findings:** The environment is stable, but attempting to configure the agent to use the provider from Issue #300 fails to resolve an adapter key, confirming the lack of integration.

---

## Solution Approach

### Analysis

This integration requires creating a strictly-typed, metadata-only provider leaf. The primary architectural decision at this stage is determining whether this API is fully OpenAI-compatible (allowing me to reuse the shared `protocols/openai-api/provider.ts` as I did with Perplexity) or if it utilizes a distinct REST schema requiring a custom protocol implementation and response parser.

### Proposed Solution

Scaffold a new provider leaf under `self/subcortex/providers/src/providers/<vendor_name>/`. Define the vendor's identity, endpoint, and credentials as declarative metadata. If compatible, delegate execution to the shared `chat-completions` protocol; otherwise, build a custom implementation matching the `ProviderAdapter` contract.

### Implementation Plan

**Understand:** The system needs a certified provider leaf that routes agent requests to the target API, with credentials flowing in via definition-driven plumbing.

**Match:** Mirror the file shape of recently added leaves (like `perplexity` or `anthropic`).

**Plan:**
1. Scaffold the provider directory.
2. `definition.ts` — Define the metadata `ProviderDefinitionLeaf` with the correct `vendorKey`, `defaultEndpoint`, and `auth` schema.
3. `provider.ts` — Create the factory that resolves the specific API key and fails closed if missing.
4. `adapter.ts` — Export the appropriate protocol adapter.
5. Regenerate catalogs using `pnpm --filter @nous/subcortex-providers run generate:providers`.
6. Add dedicated unit tests and extend vendor-roster assertions.

**Implement:** [Pending — Branch setup in progress]

**Review:** [Pending]

**Evaluate:** [Pending]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: Factory correctly resolves the specific API key and fails closed (throws) if absent.
- [ ] Test case 2: Composed request URL routes to the correct provider endpoint.
- [ ] Test case 3: Adapter configuration resolves to the correct underlying protocol.

### Integration Tests

- [ ] Catalog hydration: Vendor is successfully added to `PROVIDER_DEFINITIONS` without manual UUIDs.
- [ ] Ensure `pnpm run check:generated` and `pnpm run typecheck` pass flawlessly.

### Manual Testing

[Pending live API key test to verify streaming and response parsing]

---

## Implementation Notes

### Week 1 Progress

- Started Phase I.
- Investigated the requirements for Issue #300 and planned the adapter structure based on learnings from PR #411.
- Updated local environment and synced with the upstream integration branch.
- Initialized local feature branch to begin scaffolding the leaf.

### Code Changes

- **Files modified:** [Pending]
- **Key commits:** [Pending]
- **Approach decisions:** Determining protocol compatibility to dictate the adapter routing pattern.

---

## Pull Request

**PR Link:** [Not yet submitted]

**PR Description:** [Drafting in progress]

**Maintainer Feedback:**
- [Pending]

**Status:** Phase I [In Progress]

---

## Learnings & Reflections

### Technical Skills Gained

[To be updated as implementation progresses]

### Challenges Overcome

[To be updated as implementation progresses]

### What I'd Do Differently Next Time

[To be updated as implementation progresses]

---

## Resources Used

- [nous-core provider-adapter docs](https://docs.nue.orthg.nl/docs/development/provider-adapters)
- [Target API Reference - Pending]


<br>

---

# Contribution 1: [Adapter: Perplexity Sonar Model Provider]

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

```bash
$ ls self/subcortex/providers/src/providers/
anthropic  codex-cli  github-copilot-cli  groq  llama-cpp  ollama  openai
# → no `perplexity/` directory
