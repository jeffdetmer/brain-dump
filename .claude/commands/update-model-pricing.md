---
description: Update the AI model pricing catalog from provider pricing pages so ticket cost tracking stays accurate. Optionally pass provider names, model names, or pasted pricing as arguments.
---

# Update Model Pricing

Load the `update-model-pricing` skill and follow its workflow end to end:

1. Fetch current pricing from the provider docs (Anthropic, OpenAI, and any
   provider named in the arguments). If the user pasted pricing for models not
   yet on the pricing pages, treat that as the source.
2. Add/update entries in `DEFAULT_COST_MODELS` (`core/cost.ts`).
3. Update the registry tests in `core/__tests__/cost-explorer.test.ts`.
4. Run `pnpm test core/__tests__/cost-explorer.test.ts` and `pnpm check`.
5. Apply to the live database with
   `pnpm brain-dump telemetry recalculate-costs --pretty` and report the
   inserted/updated counts.

Arguments (optional): $ARGUMENTS
