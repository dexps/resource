# Model And Pricing Data

This directory maintains official-source model and pricing data for downstream
model selection, display, and billing estimates.

## Files

- `models.json`: model registry grouped by provider or product scope.
- `model-prices.json`: pricing table grouped by provider, with `_meta.currency` and
  `_meta.default_unit`.

Both files include a root `_meta` object with the update date, schema version,
source URLs, scope classification, and maintenance notes. Consumers should
ignore `_meta` or select only root values whose JSON type is `array` for
`models.json`.

Keep `_meta`. It is the contract for future updates: source URLs, scope type,
schema version, update date, and caveats live there so downstream consumers do
not need to hard-code provider assumptions.

## Scope Types

Root keys can mean different things:

- Direct model providers: OpenAI, Anthropic Claude, Google Gemini, Kimi,
  xAI, DeepSeek, Mistral, Cohere, Perplexity, Z.AI, MiniMax, AI21.
- Cloud or model platforms: Google Vertex, Alibaba Model Studio/Qwen,
  Baidu Qianfan, Tencent Hunyuan, AI Studio.
- Product entitlement scopes: Codex plans, Gemini CLI, Antigravity.

Do not mix aggregators into a direct-provider scope. If an aggregator is added,
add it as its own explicitly labeled root key.

## Source Policy

Use official sources only:

- OpenAI: Developers model list, pricing page, latest-model guide.
- Anthropic Claude: official model overview and pricing docs.
- Google Gemini: Gemini API model and pricing docs.
- Google Vertex: official Google Cloud Gemini/Vertex pricing docs.
- Kimi/Moonshot: official Kimi model and pricing docs.
- Antigravity: official pricing/plan page.
- xAI: official xAI models and pricing docs.
- DeepSeek: official API models and pricing docs.
- Mistral: official model docs and pricing page.
- Cohere: official model docs and pricing page.
- Perplexity: official Sonar pricing docs and OpenAPI schema.
- Alibaba/Qwen: official Alibaba Cloud Model Studio docs.
- Baidu Qianfan/ERNIE: official Baidu AI Cloud Qianfan docs.
- Tencent Hunyuan: official Tencent Cloud Hunyuan docs.
- Z.AI/BigModel: official Z.AI docs and pricing page.
- MiniMax: official MiniMax API docs and pricing pages.
- AI21: official Jamba docs and AI21 pricing page.

Do not invent model IDs, context windows, statuses, or prices. If an official
page lists a model but does not publish token pricing, keep the model in
`models.json` and omit token pricing or record the official plan entitlement.

## Update Workflow

1. Re-check the official source URLs in `_meta.sources`.
2. Update `models.json` first, preserving provider/product grouping.
3. Update `model-prices.json` next, using official units and `null` where an official
   table shows no cached-input price.
4. Keep deprecated, legacy, and snapshot models when official model or pricing
   pages still list them.
5. Do not remove a model just because it is old; remove it only when official
   sources no longer list it or clearly mark it retired beyond support.
6. Update `_meta.updated_at` in both JSON files.
7. Add an update note below when the change expands provider coverage, changes
   schema semantics, or changes pricing units/currencies.

## Validation

Run these checks after each update:

```sh
jq empty models.json model-prices.json

jq -r 'to_entries[] | select(.value|type=="array") | .key as $provider | [.value[].id] | group_by(.)[] | select(length>1) | "\($provider): \(.[0]) x\(length)"' models.json
```

Check that local price IDs resolve to local model IDs:

```sh
node - <<'NODE'
const fs = require('fs');
const models = JSON.parse(fs.readFileSync('models.json', 'utf8'));
const prices = JSON.parse(fs.readFileSync('model-prices.json', 'utf8'));
const byScope = Object.fromEntries(
  Object.entries(models)
    .filter(([, value]) => Array.isArray(value))
    .map(([scope, value]) => [scope, new Set(value.map((model) => model.id))])
);

for (const [scope, section] of Object.entries(prices)) {
  if (scope === '_meta' || !section || typeof section !== 'object') continue;
  const ids = new Set();
  for (const key of ['models', 'embeddings']) {
    if (section[key] && typeof section[key] === 'object') {
      for (const id of Object.keys(section[key])) ids.add(id);
    }
  }
  const known = byScope[scope];
  const missing = known ? [...ids].filter((id) => !known.has(id)) : [...ids];
  if (missing.length) console.log(`${scope}: ${missing.join(', ')}`);
}
NODE
```

For official coverage checks, compare provider docs against local IDs. Example:

```sh
comm -23 \
  <(curl -L --compressed -s https://developers.openai.com/api/docs/models/all | rg -o '/api/docs/models/[a-zA-Z0-9._-]+' | sed 's#^/api/docs/models/##' | rg -v '^(all|all\.png)$' | sort -u) \
  <(jq -r '.openai[].id' models.json | sort -u)

comm -23 \
  <(curl -L --compressed -s https://ai.google.dev/gemini-api/docs/models | rg -o '/gemini-api/docs/models/[a-zA-Z0-9._-]+' | sed 's#^/gemini-api/docs/models/##' | sort -u) \
  <(jq -r '.gemini[].id' models.json | sort -u)
```

Empty output means local coverage matches the official model links currently
visible on those pages.

Additional spot checks for providers with clean official machine-readable pages:

```sh
curl -L --compressed -s https://api-docs.deepseek.com/quick_start/pricing |
  rg 'deepseek-v4-flash|deepseek-v4-pro'

node - <<'NODE'
(async () => {
  const text = await (await fetch('https://docs.perplexity.ai/openapi.json')).text();
  const ids = [...new Set([...text.matchAll(/"(sonar(?:-[a-z0-9]+)*)"/g)].map((m) => m[1]))].sort();
  console.log(ids.join('\n'));
})();
NODE

curl -L --compressed -s https://platform.minimax.io/docs/llms.txt |
  rg 'pricing-paygo|text-generation|models-intro'
```

## Pricing Notes

Prices are USD unless a section states otherwise. The root currency in
`model-prices.json` is `mixed`; provider sections declare their own currency and unit.
Chinese cloud/platform pricing is recorded in CNY when the official source uses
CNY. Token prices are per 1M tokens unless a model price object states another
unit, such as per 1K tokens, per second, per image, per request, per minute, or
plan price.

Antigravity is recorded as plan entitlement pricing because the official page
does not publish token-level prices.

## Update Notes

### 2026-05-07

- Expanded provider/platform coverage using official sources only: xAI,
  DeepSeek, Mistral, Cohere, Perplexity, Alibaba/Qwen, Baidu Qianfan,
  Tencent Hunyuan, Z.AI/BigModel, MiniMax, and AI21.
- Bumped JSON schema metadata to `1.1.0` and added `_meta.scope_types`.
- Changed `model-prices.json` root currency to `mixed`; section-level currencies and
  units now carry the billing meaning.
- Added validation for duplicate model IDs and price IDs that do not resolve to
  local model entries.
