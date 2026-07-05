---
name: update-model-pricing
description: >
  Update Brain Dump's AI model pricing catalog (DEFAULT_COST_MODELS) when
  providers release new models or change prices. Use when the user asks to add
  a new model, refresh provider pricing, or fix ticket cost attribution for a
  model that shows $0 or wrong costs. Covers Anthropic, OpenAI, Google, Cursor,
  OpenCode, and open-source providers.
---

# Update Model Pricing

Brain Dump attributes AI costs to tickets using per-model pricing rows. This
skill is the repeatable workflow for keeping that pricing current.

## Where pricing lives

| Location                                   | Role                                                                                                                                                                                                            |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `core/cost.ts` → `DEFAULT_COST_MODELS`     | **Single source of truth.** Seeded into new databases and synced into existing ones.                                                                                                                            |
| `core/cost.ts` → `syncDefaultCostModels()` | Inserts missing defaults and refreshes rows still marked `is_default` in existing DBs. Runs as part of cost recalculation.                                                                                      |
| `core/__tests__/cost-explorer.test.ts`     | Registry-assert tests: per-provider model-name lists plus spot-check pricing assertions. Must be updated alongside the catalog.                                                                                 |
| `src/lib/launch-model-catalog.ts`          | Launch model picker. Reads cost-model rows at runtime — new models under an existing provider appear automatically. Only touch it for a **new provider id** or the Pi allowlist (`PI_MODEL_NAMES_BY_PROVIDER`). |

All prices are **USD per million tokens** (`inputCostPerMtok`, `outputCostPerMtok`,
`cacheReadCostPerMtok`, `cacheCreateCostPerMtok`).

## Workflow

### 1. Get authoritative prices

- **Anthropic**: WebFetch `https://platform.claude.com/docs/en/about-claude/pricing`
  (the model pricing table has base input, 5m cache writes, cache hits, output).
- **OpenAI**: WebFetch `https://developers.openai.com/api/docs/pricing`.
- If a model is too new to be on the pricing page, use figures the user
  provides (announcement posts) and note the source in the commit message.
- Never invent prices from memory — models newer than your training data are
  exactly the ones this skill exists for.

### 2. Derive cache prices when only input/output are published

- **Anthropic**: cache read = `0.1 × input`, 5-minute cache write = `1.25 × input`
  (the docs table lists them explicitly — prefer the table).
- **OpenAI (GPT-5.6 and later)**: cache read = `0.1 × input`; cache writes are
  billed at `1.25 × input`, so set `cacheCreateCostPerMtok`. Older OpenAI
  models have no cache-write charge — leave `cacheCreateCostPerMtok` unset.

### 3. Edit `DEFAULT_COST_MODELS` in `core/cost.ts`

- **New model** → add a new entry in the provider's section with `isDefault: true`.
- **Price change** → edit the existing entry in place. `syncDefaultCostModels()`
  updates any DB row still marked default; rows the user hand-edited are left
  alone unless listed in `legacyRowsToReplace`.
- **Do not delete old models** — historical token usage still resolves against
  them. Only `LEGACY_OPENAI_MODELS_TO_REMOVE` handles intentional removals.
- **Introductory pricing**: seed the price that is actually billed _today_ and
  leave a comment with the standard price and switchover date (see the
  `claude-sonnet-5` entry for the pattern). Rerun this skill after the date.
- Subscription-routed providers (`openai-codex` for Pi) use all-zero prices;
  if Pi exposes a new model, also add it to `PI_MODEL_NAMES_BY_PROVIDER` in
  `src/lib/launch-model-catalog.ts`.

### 4. Update the registry tests

In `core/__tests__/cost-explorer.test.ts`:

- Add the new model names to the provider's expected list (**alphabetically
  sorted** — `listCostModels` orders by provider, then model name).
- Update or add `toMatchObject` pricing assertions for changed/new models.
- The sync-reconciliation tests hardcode `inserted`/`removed` counts (e.g.
  `expect(result).toEqual({ inserted: N, ... })`) — bump `N` by the number of
  models added to the affected provider.

### 5. Validate

```bash
pnpm test core/__tests__/cost-explorer.test.ts
pnpm check
```

### 6. Apply to the live database

```bash
pnpm brain-dump telemetry recalculate-costs --pretty
```

This syncs the defaults into the user's database and re-prices all historical
`token_usage` rows. Use `deep-recalculate-costs` instead if provider-log
backfill is also wanted. Report the inserted/updated counts and any change in
total attributed cost.

## Sanity checks before finishing

- Cache read should be ~10% of input; a cache read _higher_ than input price is
  almost certainly a transcription error.
- Every new entry has `isDefault: true` (otherwise sync won't manage it).
- New Anthropic/OpenAI models automatically appear in the Claude/Codex launch
  pickers — no catalog change needed for existing providers.
