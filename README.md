# countwerk skill

The agent skill for [countwerk](https://countwerk.ai): durable counters
your agent can't reset. Spend caps, step limits, fleet budgets,
blast-radius stops, and distinct counting, all over plain HTTP.

## Install

```bash
npx skills add countwerk/skill
```

## Configure

The skill expects the op token and namespace in the environment:

```bash
export COUNTWERK_TOKEN=cw_o_...
export COUNTWERK_NS=my-agent
# optional, defaults to https://countwerk.ai
export COUNTWERK_URL=https://countwerk.ai
```

No namespace yet? Funding is signup: `POST /v1/<ns>/fund` returns a 402
with x402 payment instructions; the first settled payment mints your
tokens. Details in the skill and at [countwerk.ai/llms.txt](https://countwerk.ai/llms.txt).

Prefer the model never touching a credential? Use the MCP server
instead: [countwerk/mcp](https://github.com/countwerk/mcp).

The skill is knowledge only: no scripts, no dependencies. Content lifts
deliberately from the served docs; prices are never hardcoded
(`GET /pricing` is authoritative).
