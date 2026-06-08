# su26-ai301-contribution
This repo holds my progress and contribution to the open source projects, for the codepath AI-301 class
# Contribution: [Adapter: Perplexity Sonar Model Provider]

**Contribution Number:** 1  
**Student:** Sai Subhash Manam  
**Issue:** https://github.com/orthogonalhq/nous-core/issues/314  
**Status:** Phase I [Complete]

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
