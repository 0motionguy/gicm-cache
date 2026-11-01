# @gicm/cache

A three-layer cache for LLM agents: prompt-prefix blocks, a similarity-matched response cache and a tool-result cache.

The package gathers the static parts of a prompt into Anthropic `cache_control` blocks, answers repeated questions from a local response cache and remembers deterministic tool results. It is for developers running agents against metered APIs who want to stop paying twice for the same tokens. Everything is in process; there is no external store.

Part of [gICM](https://github.com/0motionguy/gICM), a family of OpenClaw agent packages. API reference and examples: [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md).

## What it does

- **L1 prefix cache:** collects system, skill, tool and context blocks in that order and emits them as content blocks marked `cache_control: ephemeral`, evicting the oldest blocks above a token cap (100,000 by default, estimated at 4 characters per token).
- **L2 response cache:** stores query and response pairs with a 64-dimension hashed-character embedding and returns a stored answer when cosine similarity reaches the threshold (0.85 by default). Defaults: 60-minute TTL, 1,000 entries.
- **L3 tool cache:** keys deterministic tool results by tool name and inputs, with a per-tool TTL through `registerTool`. Defaults: 30 minutes, 5,000 entries.
- **Manager:** `CacheManager` ties the layers together with hit, miss, evict and save events, aggregated stats and `evictExpired()` / `clear(layer?)`.
- **Savings estimate:** `calculateSavings(tokens, model)` prices cached input against uncached input from a small built-in table of three Claude models.

## Example

```ts
import { CacheManager } from "@gicm/cache";

const cache = new CacheManager({ l2: { similarityThreshold: 0.9 } });
cache.addPrefix("You are a TypeScript reviewer.", "system");
const system = cache.buildCacheControlMessages();   // blocks carrying cache_control

const hit = cache.lookupResponse(question);
if (!hit) cache.storeResponse(question, answer, "anthropic/claude-sonnet-4.5");
```

## How it's built

TypeScript (ESM) bundled with tsup and tested with vitest; zod validates the config. Each layer is a `Map`-backed class (`l1-prefix`, `l2-response`, `l3-tool`) behind `CacheManager`. The L2 embedding is a character-hashing function, not a learned model, so "similar" means similar wording.

## Repository layout

- `src/`: `cache-manager`, `l1-prefix`, `l2-response`, `l3-tool`, `embeddings`, `types`
- `src/__tests__/`: vitest suite (32 test cases)
- `skill/SKILL.md`: the OpenClaw skill definition

## Run it locally

```bash
pnpm install
pnpm build   # tsup -> dist/
pnpm test    # vitest (add --run for a single pass)
```

## Status

Beta. Version 1.0.0, a single extracted release (February 2026) with a test suite. Cached data is lost when the process exits, and the price table covers three models only.
