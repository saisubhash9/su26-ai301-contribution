# Contribution: [Adapter: Perplexity Sonar Model Provider]

**Contribution Number:** 1  
**Student:** Sai Subhash Manam  
**Issue:** https://github.com/orthogonalhq/nous-core/issues/314  
**Status:** Phase III [In Progress]

---

## Why I Chose This Issue

I wanted to tackle an issue that bridges backend tool development with Large Language Model APIs. Because I frequently utilize LLM APIs to build automated tools and optimize complex workflows, building a dedicated model provider adapter is a natural extension of my current skill set. 

Working on the `nous-core` framework also gives me a great opportunity to look under the hood of a local-first AI agent architecture. By implementing the Perplexity Sonar provider, I will gain hands-on experience with how agentic frameworks standardize communication across diverse, external LLM endpoints using interface patterns.

---

## Understanding the Issue

### Problem Description

The `nous-core` project currently lacks native support for Perplexity's API, specifically their Sonar models. For the local agent to communicate with Perplexity, an adapter class that implements the core framework's `IModelProvider` (or `AgentAdapter`) interface needs to be created. Without this, users cannot route agent tasks through Perplexity's infrastructure.

### Expected Behavior

Developers and users of `nous-core` should be able to select Perplexity Sonar as their active model provider. Once the user supplies the necessary Perplexity API keys and configurations, the framework should seamlessly establish a connection and route prompts/responses through the Perplexity API just as it does for other supported models.

### Current Behavior

Attempting to use Perplexity Sonar models natively within `nous-core` is currently impossible. Because there is no dedicated provider adapter, the framework does not know how to format requests, handle authentication, or parse responses from the Perplexity API endpoint.

### Affected Components

This implementation will primarily touch the model provider/adapter directories in the codebase. It will likely involve:
* Creating a new adapter class file (e.g., `perplexity_provider`) that implements the `IModelProvider` or `AgentAdapter` interface.
* Updating configuration and initialization files to recognize Perplexity API keys.
* Modifying provider registries or factory patterns so the new adapter can be instantiated by the system.

---

## Reproduction Process

### Environment Setup

Setting up the local development environment for this monorepo presented a specific challenge regarding Node.js version compatibility with native C++ modules. 

1. **Initial Attempt:** I created a Conda environment (`nous`) and installed Node.js. It defaulted to v26.3.0.
2. **The Error:** Running `pnpm install` caused `node-gyp` to fail when compiling the `better-sqlite3` native module. The C++ V8 API in Node 26 introduced breaking changes (`GetPrototype` removed, `Value` deprecated), causing the build to crash.
3. **The Solution:** The `CONTRIBUTING.md` specifies Node 22+. I downgraded the environment strictly to Node 22 (LTS) using `conda install -c conda-forge nodejs=22`. I then cleared `node_modules`, re-enabled Corepack (`corepack enable && corepack prepare pnpm@latest --activate`), and ran `pnpm install` again. 
4. **Success:** The native compilation succeeded, and subsequent `pnpm build` and `pnpm test` commands passed without errors.

### Steps to Reproduce
*(For this specific issue, reproduction means setting up the codebase and verifying Perplexity is absent).*
1. Fork and clone the `nous-core` repository.
2. Search the codebase under `self/subcortex/providers/src/providers/`.
3. Observe that folders exist for Anthropic, OpenAI, etc., but Perplexity is missing.

### Reproduction Evidence

- **My findings:** The environment is now fully stable. I have configured my Git remotes (`origin` pointing to my fork, `upstream` pointing to `orthogonalhq/nous-core`) and created my working branch: `feat/314-perplexity-adapter`.

---

## Solution Approach

### Analysis

Based on the `CONTRIBUTING.md` guidelines, model provider adapters are treated as "certified provider leaves" under **Tier 1: Integrations**. The architecture requires a modular file structure rather than a single monolithic class. The Anthropic provider serves as the primary reference implementation for this structure. Perplexity's API shape is highly compatible with the OpenAI protocol, which simplifies the serialization payload, but requires specific configuration for Bearer token authentication and the `search_results` extension.

### Proposed Solution

I will create a new provider leaf for Perplexity in the `self/subcortex/providers/src/providers/perplexity/` directory. This will implement the required `IModelProvider` contract, utilizing the OpenAI-compatible protocol wrapper provided by the core framework while injecting Perplexity-specific configurations.

### Implementation Plan

Using the UMPIRE framework:

**Understand:** The system needs a strictly typed module to route agent requests to Perplexity's Sonar models while adhering to the core's schema and streaming boundaries.

**Match:** I will match the architectural pattern used in `self/subcortex/providers/src/providers/anthropic/`. I will also leverage the existing `openai-api` protocol wrapper (`self/subcortex/providers/src/protocols/openai-api/provider.ts`) since Perplexity supports this shape.

**Plan:** 
1. Scaffold the directory: `self/subcortex/providers/src/providers/perplexity/`.
2. Create `definition.ts`: Define the model IDs (e.g., `llama-3.1-sonar-large-128k-online`), capabilities, and input/output schemas.
3. Create `adapter.ts`: Map the generic `TextModelInputSchema` to Perplexity's expected JSON payload.
4. Create `provider.ts`: Handle initialization, Bearer token authentication, and implement `invoke()` and `stream()`.
5. Create `index.ts`: Export the module for the registry.
6. Create `__tests__/`: Write unit tests using Vitest to verify definition aggregation, adapter behavior, and construction.

**Implement:** *(In progress on branch `feat/314-perplexity-adapter`)*

**Review:** Ensure the code passes `pnpm typecheck`, `pnpm lint`, and `pnpm test`. The commit message must follow the project's strict Conventional Commits standard: `feat(subcortex-providers): add Perplexity model provider adapter (#314)`.

**Evaluate:** Run a local test script to verify that a prompt sent through the new Perplexity provider returns a properly parsed response and handles streaming chunks correctly.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: Verify that `adapter.ts` correctly maps standard system/user messages into the Perplexity API payload.
- [ ] Test case 2: Verify `provider.ts` constructs successfully when a valid API key is provided and throws an appropriate error when missing.
- [ ] Test case 3: Verify the schema definitions in `definition.ts` pass the framework's strict Zod runtime validation.

### Integration Tests

- [ ] Execute an `invoke()` call using a mock Perplexity API response to ensure the wrapper returns the standardized framework output.
- [ ] Execute a `stream()` call to ensure the asynchronous generator yields text chunks cleanly without terminating prematurely.

### Manual Testing

*To be completed after implementation.*

---

## Implementation Notes

### Week 1 Progress

- Successfully set up the `nous-core` monorepo.
- Diagnosed and fixed a Node-gyp compilation issue caused by an incompatible Node.js v26 installation, rolling back to the required v22 LTS via Conda.
- Connected the Git fork to the upstream repository and established a `feat/314-perplexity-adapter` branch.
- Read through the `.architecture` and `CONTRIBUTING.md` documents to understand the required directory structures for "certified provider leaves" and the strict testing/linting requirements (`oxlint`, `vitest`).
- Established the blueprint for the file structure mirroring the Anthropic provider.
