# Contribution 3: Adapter: Replicate Model Provider

**Contribution Number:** 3  
**Student:** Sai Subhash Manam  
**Issue:** https://github.com/orthogonalhq/nous-core/issues/323  
**Status:** Phase I [In Progress]

> **Note on PR base branch:** Per the issue's `2026-06-18` update, this work targets the upstream integration branch `feat/contributor-friendly-inference-provider-surface` on `orthogonalhq/nous-core` — **not** `dev`/`main`. API-backed providers are implemented as certified provider leaves under `self/subcortex/providers/src/providers/<vendor>/`, using `ProviderDefinitionLeaf`, with built-in provider IDs derived from `vendorKey` (no hand-authored `wellKnownProviderId`). The issue's historical note also confirms the direct `IModelProvider` path is superseded and should not be used.

---

## Why I Chose This Issue

After completing the Perplexity Sonar adapter (#411, merged) and beginning a second provider integration on #300 — which was subsequently assigned to another contributor — I moved to Issue #323, the **Replicate** model provider. This was a deliberate choice: Replicate is a meaningfully *different* integration from Perplexity, and that difference is exactly what I wanted next.

Perplexity is OpenAI Chat Completions–compatible, so its leaf simply reused the shared `chat-completions` protocol. Replicate is **not** — it exposes a prediction-based, asynchronous API (create a prediction, then poll or stream for the result). That means this leaf almost certainly needs a **custom protocol implementation** rather than a metadata-only reuse. Working through the "distinct REST schema" branch of the framework's Protocol Decision Table is the natural next step in deepening my understanding of the `nous-core` provider architecture, and it maps directly onto my AI301 Open Source Capstone goals and my experience standardizing communication across heterogeneous toolchains.

---

## Understanding the Issue

### Problem Description

`nous-core` currently has no certified provider leaf for Replicate. Without it, the local agent cannot establish Replicate as a vendor, resolve its credentials, or route tasks through Replicate-hosted models.

### Expected Behavior

Users should be able to select Replicate as their active model provider, supply their `REPLICATE_API_TOKEN`, and have the framework connect and route agent prompts/responses through Replicate's infrastructure — including streaming output where the underlying model supports it.

### Current Behavior

Using Replicate natively is currently impossible. The provider is absent from `self/subcortex/providers/src/providers/`, so the framework has no vendor identity, endpoint defaults, credential metadata, or factory to instantiate it.

### Affected Components

* A new certified provider leaf under `self/subcortex/providers/src/providers/replicate/`.
* **Likely** a custom protocol implementation (a Replicate-specific `implementation.ts`, or a new protocol under `self/subcortex/providers/src/protocols/`) to handle the prediction create → poll/stream → parse lifecycle. *This is the key open question — see Analysis.*
* The generated provider catalogs (`provider-definitions.ts`, `provider-adapters.ts`, `provider-factories.ts`), which will be regenerated — never hand-edited.
* Vendor-roster test suites, extended to exhaustively enumerate the new provider.

---

## Reproduction Process

### Environment Setup

My local environment is carried over and stable from the previous contributions. The active toolchain is pinned to **Node v22.22.3 / pnpm 10.6.2** via a Conda environment, which avoids the native C++ module compilation errors (`better-sqlite3` / `node-gyp`) seen on Node 26.

### Steps to Reproduce

1. Sync the local fork with the `orthogonalhq/nous-core` upstream repository.
2. Update to the tip of the integration branch `feat/contributor-friendly-inference-provider-surface` (the issue's `2026-06-18` update explicitly requires updating the fork/branch before continuing work).
3. Inspect `self/subcortex/providers/src/providers/`.
4. Observe that no `replicate/` directory or configuration exists.

### Reproduction Evidence

- **Commit showing reproduction:** [Pending — feature branch to be created. For a "missing provider" issue, reproduction means verifying Replicate is absent before scaffolding.]
- **Screenshots/logs:** [N/A at this stage]
- **My findings:** The environment is stable and synced to the current integration branch. Attempting to configure the agent to use Replicate fails to resolve an adapter key, confirming the absence of any integration.

---

## Solution Approach

### Analysis

The primary architectural decision for this leaf is protocol shape, and preliminary review of Replicate's API indicates it is **not** OpenAI Chat Completions–compatible:

* Replicate is **prediction-based**. Running a model means creating a *prediction* object via `POST https://api.replicate.com/v1/predictions` with a `version`/`model` and an `input` object (the input schema is model-specific, exposed via each model's `openapi_schema`).
* Results are **asynchronous**. The output is not returned inline; the caller must either poll `GET /v1/predictions/{id}` until `status` is `succeeded`/`failed`, register a webhook, or connect to the prediction's SSE `stream` URL for progressive output.
* Auth uses a token via the `REPLICATE_API_TOKEN` environment variable, sent in the `Authorization` header. (Header scheme to confirm against the schema reference — classic docs use the `Token` prefix; the newer client uses Bearer.)

This places Replicate squarely in the **"distinct REST schema → custom protocol implementation and response parser"** branch of the Protocol Decision Table — the *opposite* of the Perplexity case, which reused the shared `chat-completions` protocol. **Working hypothesis:** Replicate needs a custom protocol/`implementation.ts` exposing `invoke()`/`stream()` that (1) creates a prediction, (2) polls or consumes the SSE stream, and (3) maps the prediction result into the framework's expected response shape. I will confirm this definitively against the updated `schemas-abi-reference` and `provider-leaf-anatomy` docs before locking the file shape.

### Proposed Solution

Scaffold a new provider leaf at `self/subcortex/providers/src/providers/replicate/` that declares Replicate's identity/endpoint/credentials as metadata, and delegates execution to a Replicate protocol implementation that models the prediction create → poll/stream → parse lifecycle. Regenerate the catalogs so the vendor hydrates with a `vendorKey`-derived built-in id (no hand-authored UUID or `wellKnownProviderId`).

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The system needs a strictly-typed leaf that routes agent requests to Replicate-hosted models. Because Replicate is async and prediction-based, the leaf must own (or delegate to) a custom protocol that handles the prediction lifecycle and streaming, with credentials/endpoint flowing in via the framework's definition-driven plumbing.

**Match:** Use the existing OpenAI-compatible leaves (`anthropic`, `perplexity`, `openai`) as the reference for leaf conventions, exports, and tests. For the protocol itself, review any leaf that ships a custom `implementation.ts` (rather than reusing `chat-completions`) as the closest structural template for the prediction/poll/stream pattern.

**Plan:**
1. Scaffold the `replicate/` provider directory.
2. `definition.ts` — metadata-only `ProviderDefinitionLeaf`: `vendorKey: 'replicate'`, `displayName`, `defaultEndpoint: 'https://api.replicate.com'`, `defaultModelId` (a streaming-capable default, e.g. a Llama/instruct model), `auth` (env `REPLICATE_API_TOKEN`, `Authorization` header, `required`, `purpose: 'api_key'`), `capabilities.streaming`. **No** `wellKnownProviderId`.
3. `implementation.ts` (custom protocol) — `invoke()`/`stream()` implementing create → poll/stream → parse against the prediction API, mapping results to the framework response contract.
4. `provider.ts` — `providerFactory` that resolves `REPLICATE_API_TOKEN`, **fails closed** if missing, and constructs the Replicate provider.
5. `adapter.ts` — export the Replicate `ProviderAdapter`.
6. `index.ts` — export the leaf's public surface (required by the generator).
7. Regenerate catalogs: `pnpm --filter @nous/subcortex-providers run generate:providers`.
8. Add a dedicated test file plus extend the vendor-roster assertions.

**Implement:** [Pending — leaf scaffolding to begin once protocol shape is confirmed against the docs.]

**Review:** [Pending — will run `check:generated`, `typecheck`, and the full package test suite; confirm Conventional Commit message and correct integration-branch target.]

**Evaluate:** [Pending — verify catalog hydration, factory construction/fail-closed, adapter resolution, prediction parsing, and streaming, then live API verification with a provisioned key.]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: Factory resolves `REPLICATE_API_TOKEN` and **fails closed** (throws) when absent — no silent fallback to another vendor's key.
- [ ] Test case 2: Composed request URL routes to `https://api.replicate.com/v1/predictions`.
- [ ] Test case 3: Adapter configuration resolves to the correct (custom Replicate) protocol adapter.
- [ ] Test case 4: `definition.ts` passes the framework's strict Zod validation (`ProviderDefinitionSchema`) and declares the correct `auth.header`.
- [ ] Test case 5: Prediction-lifecycle parsing — a mocked prediction response maps correctly to the framework's response shape (non-streaming path).
- [ ] Test case 6: Streaming path — a mocked SSE stream yields the expected progressive output events.

### Integration Tests

- [ ] Catalog hydration: Replicate is present in `PROVIDER_DEFINITIONS` with a stable built-in id derived from `vendorKey` (no hand-authored UUID).
- [ ] `pnpm run check:generated` and `pnpm run typecheck` pass; codegen freshness + public-export + provider-pipeline integration tests pass with the new vendor.

### Manual Testing

[Pending — live `invoke()`/`stream()` against the real Replicate API once a `REPLICATE_API_TOKEN` is provisioned, verifying prediction creation, polling/streaming, and response parsing.]

---

## Implementation Notes

### Week 1 Progress

- Started Phase I after #300 was reassigned to another contributor.
- Claimed Issue #323 — commented on the issue and reached out to the maintainer to confirm before starting (the issue was unassigned when I picked it up).
- Updated the local fork and synced to the tip of `feat/contributor-friendly-inference-provider-surface` per the issue's `2026-06-18` update.
- Reviewed Replicate's API and determined it is prediction-based / async and **not** Chat Completions–compatible — flagging a custom protocol implementation as the likely path (to be confirmed against the updated schema/anatomy docs).

### Code Changes

- **Files modified:** [Pending]
- **Key commits:** [Pending]
- **Approach decisions:** Confirming the prediction/poll/stream protocol shape before locking the leaf file layout (custom `implementation.ts` vs. shared protocol reuse).

---

## Pull Request

**PR Link:** [Not yet submitted]

**PR Description:** [Drafting in progress]

**Maintainer Feedback:**
- Outreach sent to claim the (unassigned) issue — awaiting confirmation.

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

- [Provider adapters — Quickstart](https://docs.nue.orthg.nl/docs/development/provider-adapters/quickstart)
- [Provider leaf anatomy](https://docs.nue.orthg.nl/docs/development/provider-adapters/provider-leaf-anatomy)
- [Schemas & ABI reference](https://docs.nue.orthg.nl/docs/development/provider-adapters/schemas-abi-reference)
- [Testing checklist](https://docs.nue.orthg.nl/docs/development/provider-adapters/testing-checklist)
- [Replicate — How does Replicate work?](https://replicate.com/docs/reference/how-does-replicate-work)
- [Replicate — HTTP API / Create a prediction](https://replicate.com/docs/reference/http)
- [Replicate — Streaming output](https://replicate.com/docs/topics/predictions/streaming)


<br>

---

# Contribution 2: Adapter: Issue 300 Model Provider

**Contribution Number:** 2  
**Student:** Sai Subhash Manam  
**Issue:** https://github.com/orthogonalhq/nous-core/issues/300  
**Status:** Phase I [Closed — Not Pursued]

> **Why this contribution was closed:** Issue #300 was assigned to another contributor while I was still in Phase I (branch setup / scaffolding). Rather than duplicate work, I discontinued it and moved to Issue #323 (Replicate) as **Contribution 3**. The record below is preserved as evidence of the work done before reassignment; the approach and learnings carried directly into Contribution 3.

---

## Why I Chose This Issue

Building on my Perplexity adapter (#314), I wanted to tackle another model provider integration to solidify my understanding of the `nous-core` framework and apply the "certified provider leaf" pattern more efficiently, focusing on a new provider's authentication schema, endpoint routing, and capability mapping.

---

## Understanding the Issue

### Problem Description

`nous-core` lacked a native certified provider leaf for the API requested in Issue #300. Without it, the local agent could not establish vendor identity or route tasks through that provider's models.

### Expected Behavior

Users should be able to select the provider, supply their API credential, and have the framework connect and route standard agent prompts through the provider's infrastructure.

### Current Behavior

The provider was absent from `self/subcortex/providers/src/providers/`, so the framework had no metadata, endpoint defaults, or factory to instantiate it.

### Affected Components

* A new certified provider leaf under `self/subcortex/providers/src/providers/<vendor_name>/`.
* The generated provider catalogs, to be regenerated.
* Vendor-roster test suites.

---

## Reproduction Process

### Environment Setup

My local environment was carried over and stable from Contribution 1 (Node v22.22.3 / pnpm 10.6.2 via Conda), so no additional setup was required.

### Steps to Reproduce

1. Sync the local fork with the `orthogonalhq/nous-core` upstream repository.
2. Checkout the integration branch `feat/contributor-friendly-inference-provider-surface`.
3. Inspect `self/subcortex/providers/src/providers/`.
4. Observe that the target provider's directory and configuration files are missing.

### Reproduction Evidence

- **Commit showing reproduction:** N/A — discontinued before a reproduction commit was pushed.
- **Screenshots/logs:** N/A
- **My findings:** The environment was stable, but configuring the agent to use the provider failed to resolve an adapter key, confirming the lack of integration. Work was halted when the issue was reassigned.

---

## Solution Approach

### Analysis

The open architectural question was whether the target API was fully OpenAI-compatible (allowing reuse of the shared `protocols/openai-api/provider.ts`, as with Perplexity) or required a distinct REST schema with a custom protocol implementation.

### Proposed Solution

Scaffold a metadata-only leaf declaring the vendor's identity/endpoint/credentials; delegate to the shared `chat-completions` protocol if compatible, otherwise build a custom implementation matching the `ProviderAdapter` contract.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The system needs a certified provider leaf that routes agent requests to the target API, with credentials flowing in via definition-driven plumbing.

**Match:** Mirror the file shape of recently added leaves (like `perplexity` or `anthropic`).

**Plan:**
1. Scaffold the provider directory.
2. `definition.ts` — metadata `ProviderDefinitionLeaf` (`vendorKey`, `defaultEndpoint`, `auth`).
3. `provider.ts` — factory resolving the API key; fail closed if missing.
4. `adapter.ts` — export the appropriate protocol adapter.
5. Regenerate catalogs.
6. Add unit tests and extend vendor-roster assertions.

**Implement:** Discontinued during scaffolding — issue reassigned to another contributor.

**Review:** N/A — not reached.

**Evaluate:** N/A — not reached.

---

## Testing Strategy

### Unit Tests

- [ ] N/A — not implemented (contribution discontinued).

### Integration Tests

- [ ] N/A — not implemented (contribution discontinued).

### Manual Testing

N/A — not reached.

---

## Implementation Notes

### Week 1 Progress

- Started Phase I; investigated Issue #300 requirements and planned the adapter structure based on learnings from PR #411.
- Updated the local environment and synced with the upstream integration branch.
- Initialized a local feature branch to begin scaffolding — **halted when the issue was reassigned.**

### Code Changes

- **Files modified:** None merged (early scaffolding only, discontinued).
- **Key commits:** N/A.
- **Approach decisions:** Determining protocol compatibility to dictate the adapter routing pattern; this analysis carried forward into Contribution 3 (Replicate, #323).

---

## Pull Request

**PR Link:** N/A — not pursued.

**PR Description:** N/A.

**Maintainer Feedback:**
- Issue was assigned to another contributor; I confirmed and stepped aside to avoid duplicated effort.

**Status:** Closed — Not Pursued (issue reassigned).

---

## Learnings & Reflections

### Technical Skills Gained

Reinforced the certified-provider-leaf mental model from PR #411 — identity/endpoint/credentials as declarative metadata, catalogs generated rather than hand-edited.

### Challenges Overcome

The main challenge was process rather than technical: coordination on open-source issues. I learned to confirm ownership before investing in scaffolding.

### What I'd Do Differently Next Time

Claim the issue and reach out to the maintainer *before* starting any scaffolding — which is exactly what I did for Contribution 3.

---

## Resources Used

- [nous-core provider-adapter docs](https://docs.nue.orthg.nl/docs/development/provider-adapters)


<br>

---

# Contribution 1: Adapter: Perplexity Sonar Model Provider

**Contribution Number:** 1  
**Student:** Sai Subhash Manam  
**Issue:** https://github.com/orthogonalhq/nous-core/issues/314  
**Status:** Phase IV [Complete — Merged ✅]

> **Note on PR base branch:** This PR targets the upstream integration branch `feat/contributor-friendly-inference-provider-surface` on `orthogonalhq/nous-core`, **not** `main`. The maintainer explicitly required Cloud API provider adapters to stack against that integration branch (not `dev`/`main`) until the provider-surface work is fully integrated. It is a real PR against the upstream repository (not a draft against my fork) and has been **merged**.
>
> **PR:** https://github.com/orthogonalhq/nous-core/pull/411 · **Branch:** https://github.com/saisubhash9/nous-core/tree/feat/314-perplexity-adapter

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
* The generated provider catalogs (`provider-definitions.ts`, `provider-adapters.ts`, `provider-factories.ts`), which are regenerated — never hand-edited.
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

- **Commit showing reproduction:** Working branch [`feat/314-perplexity-adapter`](https://github.com/saisubhash9/nous-core/tree/feat/314-perplexity-adapter), synced to the tip of the integration branch before scaffolding (reproduction = Perplexity verified absent).
- **Screenshots/logs:**
  ```bash
  $ ls self/subcortex/providers/src/providers/
  anthropic  codex-cli  github-copilot-cli  groq  llama-cpp  ollama  openai
  # → no `perplexity/` directory
  ```
- **My findings:** The environment is fully stable, git remotes are configured (`origin` → my fork, `upstream` → `orthogonalhq/nous-core`), and Perplexity is confirmed absent from the provider roster — so a new certified leaf is required.

---

## Solution Approach

### Analysis

Per the live [provider-adapter docs](https://docs.nue.orthg.nl/docs/development/provider-adapters), model provider adapters are "certified provider leaves" with a fixed file shape. The **Anthropic leaf** is the reference for structure/naming/exports/tests.

Crucially, Perplexity's API is **OpenAI Chat Completions–compatible**. The docs' Protocol Decision Table maps this exactly to *"same protocol, different defaults/credentials → keep vendor-specific metadata in the leaf and reuse shared protocol logic."* This means the leaf should **reuse** `self/subcortex/providers/src/protocols/openai-api/**` rather than implement a custom wire protocol — so no `implementation.ts`, no custom `invoke()`/`stream()`. Perplexity needs only Bearer-token auth and its own endpoint/model/path defaults.

### Proposed Solution

Create a new provider leaf at `self/subcortex/providers/src/providers/perplexity/` that declares Perplexity's identity/endpoint/credentials as metadata and delegates request execution and response parsing to the shared `chat-completions` protocol and adapter.

### Implementation Plan

Using UMPIRE framework (adapted):

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

### Manual Testing

After scaffolding, I confirmed via the test suite (mocked fetch) that a `perplexity`-vendor config resolves the correct adapter and composes the exact request URL `https://api.perplexity.ai/chat/completions`. Live `invoke()`/`stream()` verification against the real API is pending a provisioned key.

**After** — the certified `perplexity` leaf is present and the suite is green:

```bash
$ ls self/subcortex/providers/src/providers/
anthropic  codex-cli  github-copilot-cli  groq  llama-cpp  ollama  openai  perplexity
# → `perplexity/` leaf now present

$ pnpm --filter @nous/subcortex-providers run check:generated   # catalogs up to date (no diff)
$ pnpm --filter @nous/subcortex-providers run typecheck          # no type errors
$ pnpm --filter @nous/subcortex-providers test                   # all tests pass
```

---

## Implementation Notes

### Week 1 Progress

- Resolved the Node 26 native-module build failure by pinning to Node 22 LTS via Conda and re-priming Corepack; got `pnpm build`/`pnpm test` green.
- Synced `feat/314-perplexity-adapter` to the tip of the integration branch (the maintainer's required PR target for Cloud API adapters).
- Scaffolded the `perplexity/` leaf mirroring the `openai/` leaf; added the backward-compatible `completionsPath` override to the shared provider; regenerated catalogs; wrote the leaf tests and extended the vendor-roster assertions.

### Week 2 Progress

- Opened PR #411 against the integration branch, addressed the maintainer's minor review fixes, folded them in on rebase, and the PR was **merged**.

### Code Changes

- **Files modified:** `providers/perplexity/{definition,provider,adapter,index}.ts` (new); `protocols/openai-api/provider.ts` (optional `completionsPath`, default unchanged); regenerated catalogs (`provider-definitions.ts`, `provider-adapters.ts`, `provider-factories.ts`); new perplexity test file + extended vendor-roster assertions.
- **Key commits:** `feat(providers): add Perplexity (Sonar) certified provider leaf` (PR [#411](https://github.com/orthogonalhq/nous-core/pull/411)).
- **Approach decisions:** Reuse the shared `chat-completions` protocol (Perplexity is OpenAI-compatible) with only a `completionsPath` override; keep the change fully definition-driven with a fail-closed credential factory.

---

## Pull Request

**PR Link:** https://github.com/orthogonalhq/nous-core/pull/411 (merged ✅)

**PR Description:** Adds a certified Perplexity (Sonar) provider leaf that reuses the shared `chat-completions` protocol, plus a small backward-compatible `completionsPath` override on the shared provider (default unchanged). Fully definition-driven — no runtime/bootstrap/`@nous/shared` changes. Catalogs regenerated; new leaf tests and vendor-roster assertions added.

**Maintainer Feedback:**
- Requested minor review fixes — folded in on rebase.
- Confirmed the PR correctly targets the integration branch (not `main`); approved and merged.

**Status:** Merged ✅

---

## Learnings & Reflections

### Technical Skills Gained

Learned how `nous-core`'s definition-driven provider architecture lets a whole vendor be added **without touching the runtime** — identity, endpoint, credentials, and capabilities are declarative metadata, and catalogs are generated, never hand-edited. Also learned to distinguish "reuse the shared protocol" vs. "custom protocol" via the Protocol Decision Table, and to design a **fail-closed** credential factory so a missing key throws rather than silently falling back to another vendor.

### Challenges Overcome

Resolved the Node 26 native-module build failure (`better-sqlite3` / `node-gyp`) by pinning to Node 22 LTS via Conda and re-priming Corepack. Kept the shared-provider change strictly backward-compatible (default path unchanged) so no existing vendor was affected.

### What I'd Do Differently Next Time

Provision the live API key earlier so end-to-end `invoke()`/`stream()` verification isn't deferred, and confirm the PR's target integration branch with the maintainer at the very start (which I did), since it's not `main`.

---

## Resources Used

- [nous-core provider-adapter docs](https://docs.nue.orthg.nl/docs/development/provider-adapters)
- [Perplexity API Reference](https://docs.perplexity.ai/)
