# Cursor context metadata

The plugin no longer advertises the same 200,000-token context and 32,768-token
output limit for every model. Those were metadata constants, not a local history
truncation policy.

`model.for_auth` still obtains usable IDs from
`agent.v1.AgentService/GetUsableModels` and applies the account's disabled-model
list. It also queries `aiserver.v1.AiService/AvailableModels` using
`useModelParameters`, `useReactModelPicker`, and `excludeMaxNamedModels`. The
parameterized response carries `contextTokenLimit`; the legacy flat response
observed on 2026-09-28 did not.

The plugin matches exact model names, server names, explicit aliases and
non-MAX variant legacy slugs. It uses the default context limit because the
current executor does not enable MAX mode. A slug can occur in both default and
MAX variants; the larger MAX value must not overwrite its default capacity.
Display names such as "1M" are not authoritative numeric metadata.

Static discovery and models without a positive observed default limit omit
`ContextLength`. `MaxCompletionTokens` is omitted because neither inspected
catalog publishes a verified output ceiling. These omissions mean unknown,
not zero capacity or unlimited capacity. Metadata lookup failures are returned
as errors; the plugin does not silently replace them with invented limits.
Catalog reads have a 15-second deadline and a 4 MiB response bound. A new
account-model discovery fetches current metadata rather than persisting a
model-family table.

## Observed account metadata, 2026-09-28

| Model ID | Default context | MAX context, not enabled by this patch |
| --- | ---: | ---: |
| `grok-4.7-xhigh-fast` | 256,000 | 500,000 |
| `claude-fable-5-1-thinking-xhigh` | 300,000 | 1,000,000 |
| `gpt-5.6-sol-xhigh` | 272,000 | 1,000,000 |

These are authenticated live catalog observations, not hard-coded defaults or
exhaustive boundary/load tests. Account policies and upstream values can change.
Input sent to Cursor includes its own system/tool overhead; the declared context
window is not a promise that all of it is available for user text. Plugin usage
estimates are not an authoritative upstream tokenizer measurement.

The contract was cross-checked against the installed first-party Cursor 3.22.7
`AvailableModels` schema and request construction. Cursor's
[model documentation](https://cursor.com/docs/models-and-pricing) also distinguishes
default context from MAX mode. This patch neither enables MAX billing nor changes
CPA host code, account settings, client compaction settings, or request replay.

Regression coverage includes dynamic capacity refresh, default/MAX duplicate
slugs, explicit aliases, disabled IDs, unknown/invalid capacities, bounded
metadata responses, and upstream errors. Deployment and client model-catalog
refresh are separate from editing and validating this source.
