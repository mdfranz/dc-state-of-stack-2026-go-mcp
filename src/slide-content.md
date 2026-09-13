# Slide 1: During the next five minutes...

* **Why** you should consider writing a custom MCP server for data-intensive platforms (SIEM, observability)
* **How** to iteratively build, test, and run STDIO MCP servers — for Python security agents or Claude, Codex, OpenCode

# Slide 2: Who am I?

> built a dedicated dozen MCP servers in Go, Rust, or Python in 2026

* **Security & Compliance Lead, Pydantic** — Logfire, plus security agents with Pydantic AI
* **25 years** — Cisco, Tenable, Mandiant, Ping Identity, T. Rowe Price
* **Last 2.5 years** — building blue-team security agents
* **This talk** — lessons from CrowdStrike, Chronicle, Coralogix, and more

# Slide 3: Why MCP?

"USB for agents" with  well-understood challenges:
* **Token burn** — large JSON payloads overwhelming agent content
* **Agent thrashing** — hallucinated queries and API calls leading to agent delays
* **Security** — inherent vulnerabilities, RBAC challenges, and more

> Not all-or-nothing: MCP works alongside CLI tools and CodeMode. A well-designed local server mitigates most of this.

# Slide 4: Why Custom MCP?

Vendor servers exist, but:

* **No logging or caching** — black boxes
* **Generic tool calls** — tuned for everyone, not your telemetry
* **Sketchy MCP Markplaces** - do you really trust them with your API keys?
> You have to learn the API and the data anyway — package that knowledge in the server.

# Slide 5: Why Go?

* **Sure, Python/JS libraries are great** — but their dependencies, supply chain risk, and startup time
* **Single executable** — CLI/TUI and MCP server
* **Easier lift than Rust** — for Python developers, easier/faster Agentic coding

> Use other languages for time-expediency, or when trusted SDK is available in Python

# Slide 6: Lessons — Start with the CLI

1. **Build the CLI first** — reuse a client library, or generate from Swagger/OpenAPI (worst case: HTML/PDF)
2. **Explore the data** — list / get / describe
3. **Start with 3–5 tools** — enough to learn the shape and the use cases

# Slide 7: Lessons — Instrument and Iterate

4. **Instrument from day one** — logging, observability, test coverage
5. **Add an LLM harness** — Pydantic AI against common queries
6. **Explore, then specialize** — let telemetry tell you the next tool to build

# Slide 8: A Go Template

* **Official MCP Go SDK**
* **Cobra + Viper** — spf13

> See GO-MCP-GUIDE.md

# Slide 9: Testing Layers

* **Unit tests** — CLI and MCP server
* **JSON-RPC smoke test** — no LLM required
* **Pydantic AI harness** — real tools, real model
* **Interactive** — Explore with Claude, Codex, OpenCode, etc.

# Slide 10: Start with 3 Primitives

Simple example with OSquery

The minimum loop an agent needs:

* `list_tables` — what data exists?
* `describe_table` — which columns can I query?
* `run_query` — execute, return bounded JSON

> Proves protocol init, execution, and discovery — and lets you compare models.

# Slide 11: The Tool Ladder

Add tools from observed agent friction:

| Intent | Tool | Why |
|---|---|---|
| See the surface | `list_tables` | Cheapest inventory |
| Find relevant data | `search_tables` | Names + columns |
| Inspect one schema | `describe_table` | No row payloads |
| Learn shape + values | `preview_table` | Schema + sample |
| Query one table | `query_table` | Validated, bounded |
| Join or advanced SQL | `run_query` | Flexible, riskiest |

> Prefer the narrowest tool that answers the question.

# Slide 12: Tool Typing — Args

Typed args are a self-documenting schema:

```go
type SearchProcessesArgs struct {
  Executable  string `jsonschema:"Exact path match. No wildcards."`
  CommandLine string `jsonschema:"Token search (OR), not substring."`
  HashSHA256  string `jsonschema:"SHA256 for malware detection."`
  // + ProcessName, Host, Start/End
}
```

# Slide 13: Tool Typing — Registration

```go
mcp.AddTool(server, &mcp.Tool{
  Name: "search_processes",
  Description: "Search process events by executable,
    command line, hash, host, time range.",
})
```

> The tags *are* the docs the model reads — no separate prompt to keep in sync.

# Slide 14: Guardrails — Bound the Query

* **Require a filter** — and always bound the time range
* **Read-only identity** — restrict index and scope
* **Per-tool timeouts** — surface partial results

# Slide 15: Guardrails — Bound the Response

* **Cap characters** — return truncation metadata
* **Cache by freshness** — different TTLs per tool
* **Export separately** — paginate to files, never bulk JSON into context

> Search is for reasoning. Export is for volume.

# Slide 16: Pydantic E2E Testing

* **One shared client engine** — CLI, TUI, and MCP; test deterministically first
* **Normalize model queries** — catch hallucinated syntax, return corrective hints
* **Query APIs aren't read-only** — confirmed the hard way with DuckDB

# Slide 17: Implement Observability Early

* **Logs** — one agent builds, another observes and gives feedback
* **Traces** — exceptions and spans
* **Metrics** — query latency

> Don't roll your own: Logfire (SaaS) or Agentsview (local).

# Slide 18: Agentsview

Local agentic analytics for Claude, Codex, and OpenCode — tool sequence, arguments, and outcomes per task.

> https://github.com/kenn-io/agentsview

# Slide 19: Logfire

OpenTelemetry traces across model calls, tool calls, and spans — queryable after the fact, with a built-in MCP server.

> https://pydantic.dev/logfire

# Slide 20: For More Info

* **osqueryi-mcp** — starter server, smoke harnesses, framework examples
* **elastic-security-mcp** — structured tools, raw DSL fallback, caching, export
* **mcp-threat-hunt-aug-2026** — the long-form companion talk

> github.com/mdfranz — for another SIEM, bring its OpenAPI spec and rebuild the same ladder.
