# LLM configuration notes

How `@cap-js/agents` (0.9.10) picks and configures the LLM behind `requires.llm`, and how this project sets it up.
Findings come from reading the plugin source (`cds-plugin.js`, `lib/config/local.js`, `lib/models/*.js`).

## Current setup (local LM Studio)

In [.cdsrc.yaml](.cdsrc.yaml):

```yaml
requires:
  llm:
    kind: openai
    model: qwen3.8-27b-turbo-fable-cold-fusion-735-882-heretic-uncensored-neo-coder-max-mtp
    temperature: 0.6
    maxTokens: 16384
    credentials:
      baseURL: http://localhost:1234/v1
      apiKey: lm-studio      # LM Studio ignores it, the OpenAI client needs a non-empty value
```

- Start LM Studio's local server and load a model that supports **tool calling**. The agents rely on the MCP tools `describe`, `query` and `call`.
- Check the result with `npm run watch`, which runs `DEBUG=agents cds watch` through `cross-env` so it also works in Windows `cmd`. The log should show `cds.connect.to 'llm' with: { kind: 'openai', ... }`.
- `maxTokens` caps the **reply length**. It does not set the context window. Set the context length in LM Studio's model load settings. Prompt and reply together must fit in that window.

## Default `requires.llm` per profile

Set by the plugin's own `package.json`. A value in the project overrides it.

| Profile | Default `llm` |
| --- | --- |
| `development` | `auto` |
| `with-claude` | `anthropic` |
| `hybrid` | `aicore` |
| `production` | `aicore` |

Available kinds and their implementations:

| `kind` | Implementation | Backed by |
| --- | --- | --- |
| `anthropic` | `lib/models/anthropic` | LangChain `ChatAnthropic` |
| `openai` | `lib/models/openai` | LangChain `ChatOpenAI` (OpenAI-compatible servers: LM Studio, Ollama, vLLM, ...) |
| `aicore` | `lib/models/aicore` | SAP AI SDK `OrchestrationClient` |
| `llm-mock` / `mock` | `lib/models/mock` | no real model |

Shorthand: `llm: <provider>/<model>` is expanded for `anthropic` and `aicore` only.

| Shorthand | `anthropic` | `aicore` |
| --- | --- | --- |
| `opus` | `claude-opus-latest` | `anthropic--claude-4.8-opus` |
| `sonnet` | `claude-sonnet-latest` | `anthropic--claude-4.6-sonnet` |
| `haiku` | `claude-haiku-latest` | `anthropic--claude-4.5-haiku` |

Any other text after the slash is used as the model name unchanged.

## `auto` in development

Runs once when the server is `served` and tries these in order:

1. **Anthropic:** env vars `ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL`. If no URL is set, the `env` block of `~/.claude/settings.json`, then `~/.config/opencode/opencode.json`.
2. **OpenAI:** env vars `OPENAI_BASE_URL`, `OPENAI_API_KEY`, `OPENAI_MODEL`. If no base URL is set, `~/.codex/config.toml`, then OpenCode's config.
3. **Mock:** logs `llm kind 'auto' falling back to 'mock'`.

Important for a **Claude Pro/Max subscription**: it logs in with OAuth and stores no API key in `settings.json`. Step 1 therefore finds nothing, and `auto` falls through to OpenAI (including a possible `~/.codex` config) or to mock. This is why the project sets `kind: openai` explicitly.

With explicit `kind: anthropic` or `kind: openai`, the same resolvers still run to fill gaps. Values you set (`model`, `credentials`) win over discovered ones.

## Where each setting is read from

The settings are **not** shared between providers.

| Setting | `openai` / `anthropic` | `aicore` |
| --- | --- | --- |
| Temperature | `requires.llm.temperature` | `cds.agents.params.temperature` (or per-agent `params`) |
| Reply cap | `requires.llm.maxTokens` | `cds.agents.params.max_tokens` |
| Endpoint and key | `requires.llm.credentials` | service binding, inline `credentials`, or a destination |

- `cds.agents.params` defaults to `{ max_tokens: 4096, temperature: 0 }` and is read **only** by the AI Core model class. It has no effect on `openai` or `anthropic`. The 4096 default is a reply cap, not a context length.
- `openai`: `credentials` becomes the LangChain `configuration` (`baseURL`, `apiKey`); everything else in `requires.llm` is passed to `ChatOpenAI` as options.
- `anthropic`: `credentials` and the other options are spread into `ChatAnthropic` (`anthropicApiUrl`, `apiKey`). Prompt-caching markers are added to messages.

## SAP AI Core differences

Used by default in `hybrid` and `production`. Example:

```yaml
requires:
  "[production]":
    llm:
      kind: aicore/sonnet       # model: anthropic--claude-4.6-sonnet
agents:
  params:
    max_tokens: 8192
    temperature: 0.6
```

Behavior that differs from the other kinds:

- **Credentials:** from the bound AI Core service instance (`VCAP_SERVICES`). Alternatively inline `credentials` (used as a service binding, skipping the lookup), or `destinationName` plus `resourceGroup` to go through a BTP destination. `resourceGroup` defaults to `default`.
- **Params:** passed to the orchestration config as `promptTemplating.model.params`. Resolution order: per-agent `params`, then `{ max_tokens: 4096, temperature: 0 }` for deep agents, otherwise `cds.agents.params`. Without explicit params, AI Core applies its own defaults (temperature around 1), which the plugin warns can cause truncation and extra ReAct iterations.
- **Temperature 0 is dropped** for models that reject it: `gpt-5.5`, `gpt-5.6-sol`, `gpt-5.6-luna`, `gpt-5.6-terra`, `gpt-6-astra`. Their temperature is left unset.
- **Prompt caching:** params and messages are adjusted for models that support it, and cache usage is normalized for some models.
- **Content filter:** an output content filter is built by default. Disable or replace it with the `contentFilter` option.
- **Streaming:** on by default. Disable it with `streaming: false`.
- **Resilience:** LangChain retries are off (`maxRetries: 0`). Retries are handled by AI Core (set to 2) and a circuit breaker (`cds.agents.circuitBreaker`: 50% errors over 10 requests, 30 s reset). Failures surface as `LLM_UNAVAILABLE`.
- **Tokens and quotas:** `cds.agents.quotas` (for example `maxLLMInvocationsPerTask`, `maxLLMTokensPerTask`, `maxLLMTokensPerDay`) apply across all kinds.

## How the agent graph is built

For `@agent` services the plugin answers A2A requests through a LangGraph graph. It is built **lazily on the first request** by sending the CDS event `buildGraph` to the service (`srv/langgraph-executor-srv.js`), then cached per service (FIFO, key = hash of the active feature toggles, size `cds.agents.graphCacheSize`, default 20).

The default `buildGraph` handler (`srv/handlers/index.js`) calls `agents.for(srv)`, which picks a harness from `cds.agents.harness` (default `langchain`, alternative `pi`):

| Situation | Graph |
| --- | --- |
| `AGENTS.md` found next to the `.cds` file, via `@agent.directory`, or in an `<service>-agent/` folder | **Deep agent**: `createDeepAgent` from `deepagents`, with `AGENTS.md` as memory and `skills/` as skills (`createAutoDeepAgent`) |
| No `AGENTS.md` | **ReAct agent**: `createAgent` from `langchain` |

Both are assembled from the same four building-block events, each of which you can override:

| Event | Default | Returns |
| --- | --- | --- |
| `buildTools` | tools generated from the CDS service, plus MCP and A2A connections to other `@mcp` / `@agent` services (`agents.connect`, `@agent.connect`) | array of LangChain tools |
| `buildModel` | connects to `cds.requires.llm` (or the name in `@agent.llm`) and instantiates its `impl` | LangChain chat model |
| `buildSystemPrompt` | generic prompt from the service definition | string |
| `buildMiddleware` | quotas, content filter, `agent_actions` metric | array of middleware |

After that, `GraphExecutor` wraps the graph. If the graph has no `checkpointer` it injects `CdsCheckpointSaver`, which stores state in the CDS database and enables multi-turn chats and human-in-the-loop (`@agent.hitl`). Streaming, quotas, time limits and A2A task handling come from the executor, so a custom graph keeps them.

## Defining your own graph (`buildGraph`)

Yes, you can. The right place is the **service implementation** of the agent, here `srv/travel-agent/service.js`, not a global `server.js`. The handlers are registered on the service, and a `server.js` has no service instance to attach them to.

### What you need

- `this.on("buildGraph", handler)` inside `init()`, **before** `super.init()`. The plugin registers its defaults when the service is served and the comments in its source say handlers registered earlier win (FIFO). I did not run this to confirm the order.
- The handler must return one of:
  - a **compiled LangGraph graph** (anything with `invoke()`), which gets the usual executor, checkpointer and streaming;
  - `{ harness: "pi", factory }` for the Pi harness;
  - an object with an `execute()` method, used as-is as the executor. `GraphExecutor` is **not** exported from the package (`package.json` `exports`), so you can't reuse it directly.
- `deepagents`, `langchain` and `@langchain/langgraph` already resolve from this project through the plugin's dependencies (checked with `import()`). Add them to your own `package.json` if you import them directly.
- The graph runs with the default input `{ messages: [HumanMessage] }` and the output is read from the **last message**, so a hand-written `StateGraph` should use `MessagesAnnotation`.
- Keep `AGENTS.md` and `skills/` if you still want the deep-agent behavior, since only your handler decides whether to load them.

### Example: reuse the default parts, change the graph

Untested sketch. It rebuilds what `createAutoDeepAgent` does, but adds a custom sub-agent and a recursion limit. Put it in `srv/travel-agent/service.js`.

```js
const cds = require('@sap/cds')

module.exports = class TravelAgentService extends cds.ApplicationService {
  async init() {

    // Registered before super.init(), so it runs instead of the plugin's default handler
    this.on('buildGraph', async () => {
      const { createDeepAgent } = await import('deepagents')

      // Reuse the plugin's building blocks (each can also be overridden separately)
      const tools        = await this.send('buildTools')
      const model        = await this.send('buildModel', { deepAgent: true, tools })
      const middleware   = await this.send('buildMiddleware', { model, tools })
      const systemPrompt = await this.send('buildSystemPrompt')

      return createDeepAgent({
        model, tools, systemPrompt, middleware,
        subagents: [{
          name: 'flight-researcher',
          description: 'Finds and compares flights for a given route and dates',
          systemPrompt: 'You only research flights and return a short comparison.',
          tools,
        }],
        // no checkpointer: GraphExecutor injects CdsCheckpointSaver
      }).withConfig({ recursionLimit: 60 })
    })

    return super.init()
  }
}
```

### Example: a hand-written LangGraph flow

```js
this.on('buildGraph', async () => {
  const { StateGraph, MessagesAnnotation, START, END } = await import('@langchain/langgraph')
  const { ToolNode, toolsCondition } = await import('@langchain/langgraph/prebuilt')

  const tools = await this.send('buildTools')
  const model = (await this.send('buildModel', { tools })).bindTools(tools)

  return new StateGraph(MessagesAnnotation)
    .addNode('agent', async state => ({ messages: [await model.invoke(state.messages)] }))
    .addNode('tools', new ToolNode(tools))
    .addEdge(START, 'agent')
    .addConditionalEdges('agent', toolsCondition)   // -> 'tools' or END
    .addEdge('tools', 'agent')
    .compile()                                      // checkpointer injected by the executor
})
```

This one has no system prompt, memory or skills. Add them as a `SystemMessage` in the `agent` node if you need them. `@agent.hitl` approval relies on LangGraph `interrupt()` inside the tool, so it keeps working only if your `tools` come from `buildTools`.

### Smaller overrides that often suffice

- **Different prompt only:** `this.on('buildSystemPrompt', () => '...')`.
- **Extra or filtered tools:** `this.on('buildTools', async (req, next) => [...(await next()), myTool])`. I did not verify that `next()` works for these custom events, so test it.
- **Another model for one agent:** set `@agent.llm: 'myLlm'` on the service and define `cds.requires.myLlm`.
- **Other harness:** `cds.agents.harness: pi`.

### Limits to know about

- Executor options such as `inputMapper`, `outputMapper`, `configMapper` and `recursionLimit` exist on `GraphExecutor`, but a graph returned from `buildGraph` is wrapped with no options. Set the recursion limit on the graph yourself with `.withConfig({ recursionLimit })`, as above. Otherwise you'd have to return your own `execute()` object.
- The CAP docs (`search_docs`) describe the `AGENTS.md` convention as the supported way to avoid custom handlers. `buildGraph` itself is documented only in the plugin's source comments, so treat it as less stable between plugin versions.

## Chat preview UI (`/agent/preview/`) on a deployed BTP app

Short answer: possibly, but it is **off in production by default** and the plugin is beta. Findings from the plugin source (`lib/index.js`, `lib/preview/*`, `lib/utils/inner-auth.js`). Not tested on a real deployment.

- **Mounting:** the router `<agent path>/preview` is added only when `cds.env.server.index` is truthy. CAP defaults that to `!production`, so in the `production` profile the preview is not served.
- **Enabling it:** set `cds.server.index: true` for the production profile. This also turns on CAP's index page, which lists all services and endpoints. I found no separate switch for the preview alone.
- **Authentication:** the page does not log in by itself. When it is served, the request's `Authorization` header is copied into a `<meta name="a2a-auth">` tag and reused for the agent calls (`fetch` to the agent URL and `.well-known/agent-card.json`). Behind an approuter with XSUAA or IAS, the approuter forwards the user's JWT, so this should work. The consequence is that the user's token sits in the page HTML, which matters if the page ever has an XSS bug.
- **Dev-only login prompt:** `previewAuthChallenge` only applies to `mocked` auth. With XSUAA or IAS it does nothing, and your real authentication takes over.
- **Approuter and CSRF:** add routes for `/agent/preview` and `/agent` in `xs-app.json`. The page first fetches a CSRF token from `/`, so the application root must be routable too.
- **LLM in production:** the default for `production` and `hybrid` is `aicore`. The LM Studio setup in this project is local only. A deployed app needs an AI Core binding, or an explicit `requires.llm` for the production profile.
- **Not a product UI:** it is a developer preview. For end users, consider a small Fiori or UI5 chat that talks to the A2A endpoint (`POST /agent`) with the user's token, and keep the preview off.

## Other facts

- `DEBUG=agents` also prints which config file was loaded, with secrets masked.
- API keys are never logged. The startup line shows `apiKey: '***'`.
- Sampling parameters other than the above, such as `top_p`, can be passed through `requires.llm` for `openai` and `anthropic` because the options go straight to the LangChain constructors. This is not verified for every option.
