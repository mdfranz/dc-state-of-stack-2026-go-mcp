# MCP Server Patterns (2026)

This guide distills a practical pattern for building Go MCP servers around security and data systems. It is based on three implementations:

- [go-velociraptor-mcp](https://github.com/mdfranz/go-velociraptor-mcp): a shared Go library powering both a CLI and an MCP server over a gRPC/mTLS API.
- [osqueryi-mcp](https://github.com/mdfranz/osqueryi-mcp): a local stdio server that discovers schemas and runs `osqueryi` as a subprocess.
- [elastic-security-mcp](https://github.com/mdfranz/elastic-security-mcp): structured Elastic Security tools with raw-query and bulk-export escape hatches.

The central idea is simple: MCP is the adapter. The client, domain logic, safety rules, and output policy belong in a reusable core that can be exercised without an LLM.

## 1. The Shape of a Useful MCP Server

An MCP server should make an existing capability easier for an agent to use. It should not become a second implementation of the vendor API.

```text
                  +----------------------+
                  |  MCP client / agent  |
                  +----------+-----------+
                             |
                         stdio MCP
                             |
                  +----------v-----------+
                  |     MCP adapter      |
                  | schemas, tools,      |
                  | protocol errors      |
                  +----------+-----------+
                             |
                  +----------v-----------+
                  |    shared core       |
                  | config, auth, client,|
                  | validation, limits   |
                  +----------+-----------+
                             |
              +--------------+--------------+
              |                             |
       vendor API / SDK              local CLI / subprocess
```

### Recommended layout

Keep a small server in one command package. When the same capability also needs a CLI, extract the reusable logic first:

```text
internal/<app>/
  config.go       # configuration and validation
  client.go       # API, SDK, gRPC, or subprocess access
  schemas.go      # typed inputs and domain validation
  results.go      # limits, projection, and output formatting

cmd/<app>-mcp/
  main.go         # MCP bootstrap and stdio transport
  tools.go        # tool schemas and handlers

cmd/<app>-cli/    # optional: deterministic human/automation interface
tools/            # optional: smoke and agent-driven tests
```

The CLI is not mandatory. It is valuable when it gives the shared client a deterministic interface for development, debugging, and regression tests.

## 2. Start With the Existing Boundary

Before writing MCP code, identify the cleanest existing boundary:

- An SDK or HTTP client for a remote service.
- A gRPC client with an existing configuration file.
- A local command such as `osqueryi`.
- A CLI or library that already handles authentication, pagination, and response decoding.

Reuse that boundary. If it already handles retries, authentication, or pagination, do not implement a second version in the MCP adapter.

The MCP layer should call a small domain-facing method such as:

```go
type Service interface {
    ListTables(ctx context.Context) (string, error)
    DescribeTable(ctx context.Context, name string) (string, error)
    RunQuery(ctx context.Context, input QueryInput) (string, error)
}
```

For a remote API, the service may call an SDK. For a local tool, it may use `exec.CommandContext`. The MCP handler should not need to know which one it is.

## 3. Bootstrap the Smallest Working Server

The official [MCP Go SDK](https://github.com/modelcontextprotocol/go-sdk) provides the server and stdio transport. Pin the version in `go.mod`, but select the current compatible release from the SDK documentation rather than copying a stale version into a general guide.

The bootstrap has only a few responsibilities:

1. Load and validate configuration.
2. Initialize logging away from protocol `stdout`.
3. Build the shared client or service.
4. Register tools.
5. Run the server with a cancellable context.

```go
func main() {
    os.Exit(run())
}

func run() int {
    cfg, err := loadConfig()
    if err != nil {
        fmt.Fprintln(os.Stderr, err)
        return 1
    }

    logger, closeLog, err := newLogger(cfg)
    if err != nil {
        fmt.Fprintln(os.Stderr, err)
        return 1
    }
    defer closeLog()
    slog.SetDefault(logger)

    ctx, stop := signal.NotifyContext(
        context.Background(), syscall.SIGINT, syscall.SIGTERM,
    )
    defer stop()

    service, err := NewService(cfg)
    if err != nil {
        slog.Error("service initialization failed", "error", err)
        return 1
    }

    server := mcp.NewServer(
        &mcp.Implementation{Name: "example-mcp", Version: version},
        nil,
    )
    registerTools(server, service)

    if err := server.Run(ctx, &mcp.StdioTransport{}); err != nil &&
        !errors.Is(err, context.Canceled) {
        slog.Error("server stopped with error", "error", err)
        return 1
    }
    return 0
}
```

### Stdio discipline

With stdio transport, `stdout` is reserved for MCP protocol frames. A stray `fmt.Println`, subprocess banner, or debug message can corrupt the session.

- Send diagnostics to a file or `stderr`.
- Keep subprocess output captured and handled by the client layer.
- Never log credentials, authorization headers, or unnecessarily large payloads.

A file logger is useful for a long-running local server. A simple stderr logger is enough for development. Rotation, log levels, request IDs, and structured fields are operational improvements, not prerequisites for the first tool.

### Configuration

Keep configuration close to the existing client contract:

- Reuse the client’s URL, credential, certificate, and scope variables.
- Add MCP-specific variables only for MCP behavior, such as log path, lock path, result limits, or disabled tools.
- Validate required values and URLs before starting the transport.
- Use typed durations and sizes with clear defaults.
- Do not introduce a generic profile-file hierarchy unless the product already has one.

If a server has both a CLI and an MCP binary, both should use the same configuration loader and validation rules.

## 4. Design Tools as a Workflow

Agents perform better when tools expose a deliberate path through the data rather than an undifferentiated copy of every API endpoint.

### Progressive disclosure

A useful tool ladder is:

1. **Discovery** — list tables, indices, clients, artifacts, schemas, or capabilities.
2. **Structured operations** — common searches and lookups with typed filters.
3. **Advanced escape hatch** — raw SQL, VQL, Query DSL, or vendor-specific requests.
4. **Export or asynchronous work** — large result sets, collection jobs, or long-running operations.

For example, an osquery workflow can move from `list_tables` to `describe_table` to `run_query`. Elastic can guide an agent from index discovery to structured event search, then to raw Query DSL only when necessary. Velociraptor separates discovery, collection dispatch, flow inspection, and result retrieval.

### Tool descriptions are part of the interface

Descriptions should answer:

- What is this tool for?
- When should the agent use it?
- What should it call first?
- What are the important cost or safety limits?
- Which tool should it use for large or advanced work?

Keep descriptions concise and operational. A description can say “use structured search first; use raw DSL only when the structured filters cannot express the request.” It should not contain a complete tutorial or a large schema dump.

### Typed schemas and validation

Use JSON Schema to describe the contract and typed Go inputs to enforce it. Validate again in the server; prompts and schemas are guidance, not security controls.

```go
type QueryInput struct {
    Query  string `json:"query"`
    Limit  int    `json:"limit,omitempty"`
    Cursor string `json:"cursor,omitempty"`
}

func (in QueryInput) Validate() error {
    if strings.TrimSpace(in.Query) == "" {
        return errors.New("query is required")
    }
    if in.Limit != 0 && (in.Limit < 1 || in.Limit > 100) {
        return errors.New("limit must be between 1 and 100")
    }
    return nil
}
```

For every input, decide explicitly:

- Required fields and valid ranges.
- Allowed identifiers, enums, tables, columns, or artifact names.
- Maximum time windows, rows, bytes, or pages.
- Whether a continuation token disables caching.
- Whether the operation is read-only or can change state.

Reject unknown fields when strict decoding is appropriate. Use `json.Decoder` with `DisallowUnknownFields` for typed inputs that should not silently ignore misspellings. Preserve large integer values with `UseNumber` when IDs or timestamps can exceed exact `float64` precision.

### Handler boundary

The handler should decode, validate, call the service, and translate the result. Application failures belong in an MCP tool result with `IsError: true`; reserve the returned Go error for protocol or handler failures.

```go
func registerTools(server *mcp.Server, service Service) {
    server.AddTool(&mcp.Tool{
        Name:        "run_query",
        Description: "Run a bounded read-only query. Discover available fields before querying.",
        InputSchema: querySchema,
    }, func(ctx context.Context, req *mcp.CallToolRequest) (*mcp.CallToolResult, error) {
        var input QueryInput
        if err := decodeArgs(req.Params.Arguments, &input); err != nil {
            return errorResult(fmt.Errorf("invalid arguments: %w", err)), nil
        }
        if err := input.Validate(); err != nil {
            return errorResult(err), nil
        }

        output, err := service.RunQuery(ctx, input)
        if err != nil {
            return errorResult(err), nil
        }
        return textResult(output), nil
    })
}
```

Keep repetitive handler helpers small. A generic handler factory is useful after several tools share the same decode/validate/execute path; it is not required for the first tool.

## 5. Guardrails Belong in the Server

Security and performance controls should not depend on the model obeying instructions.

### Identity and scope

- Use a read-only service identity whenever possible.
- Apply tenant, account, organization, or site scope in the client/service layer.
- Do not rely on the model to remember a mandatory scope filter.
- Allow explicit scope overrides only when the deployment and authorization model permit them.
- Disable high-risk or irrelevant tools through configuration when the server is deployed in a restricted role.

Velociraptor’s organization scope, Elastic’s read-only search identity, and the domain-specific restrictions in the structured osquery tools are examples of this principle.

### Query and input safety

Validate identifiers and structured inputs before invoking the backend. For query languages, add domain-specific validation where it catches common mistakes cheaply:

- Reject unsupported commands or unsafe operations when the tool is intended to be read-only.
- Reject missing filters or unbounded time ranges for expensive searches.
- Validate known tables, columns, artifact names, and sort fields.
- Escape values when constructing a query from separate typed parameters.
- Return corrective errors that explain how to fix the request.

Do not add a generic regular-expression query translator to every MCP server. Query normalization is useful only when the target dialect and its failure modes justify it.

### Response limits

Every result-producing tool needs a bounded output policy:

- Limit rows, characters, bytes, pages, or event count.
- Prefer selected fields or snippets to full dashboard-shaped records.
- Include `truncated`, `next_cursor`, or equivalent metadata when data was capped.
- Return partial-result and timeout status explicitly when the backend supports it.
- Keep large exports out of model context.

The exact limit is product-specific. A limit of 100 rows or approximately 16 KiB may suit a local osquery helper; a security search may need a character cap and a separate export path.

### Large and long-running work

Use a separate tool or workflow for volume:

- Stream JSONL or another machine-readable format to a restricted local file.
- Roll files by size when results can be large.
- Use pagination or scroll APIs rather than loading everything into memory.
- Report progress for long-running exports or collection jobs.
- Return a manifest, file path, job ID, or continuation token rather than the entire dataset.

Velociraptor collection flows and Elastic bulk exports demonstrate why dispatch, polling, and result retrieval should be separate from small interactive searches.

## 6. Client and Performance Patterns

The MCP adapter should not own vendor-specific transport policy. Put these behaviors in the shared client or service so the CLI, tests, and MCP server behave identically.

### Reuse before replacement

If an SDK or CLI already provides authentication, retries, pagination, TLS, or response decoding, call it directly. Add a thin shim only when the MCP tool needs a narrower service interface.

If building a client from scratch, add features in this order:

1. Context-aware requests or subprocesses.
2. Authentication and response error normalization.
3. Request and output limits.
4. Retries only for known transient failures and idempotent operations.
5. Pagination and streaming where the backend requires it.
6. Caching and request coalescing only after observing repeated work.

Do not retry cancellations or deadlines. Do not retry mutations unless the operation is explicitly idempotent and the API supports safe replay.

### Caching

Cache only data whose freshness and identity are understood:

- Schema, table, index, artifact, or capability metadata is often a good candidate.
- Dynamic security events usually need short, tool-specific TTLs or no cache.
- Continuation-token pages should normally bypass a simple cache.
- Cache keys must include identity, scope, query parameters, and relevant configuration.
- Corrupted or stale cache data must fail open to a refresh, not silently change authorization or query scope.

A persistent cache, timestamp bucketing, checksums, singleflight, and negative caching are optional optimizations. They do not belong in the minimum server template.

### Locks and lifecycle

A PID lock can be useful when an MCP host accidentally starts duplicate local processes, and all three reference implementations use some form of local operational protection. Treat it as deployment policy rather than a protocol requirement:

- Make the path configurable and allow it to be disabled for tests.
- Handle stale locks carefully.
- Remove the lock on clean shutdown.
- Do not use a lock as a substitute for correct concurrent service behavior.

Use `signal.NotifyContext` and pass the resulting context through every backend call. Long-running tools should stop when the MCP session is cancelled.

## 7. Testing in Three Layers

An MCP server is not finished when one API request succeeds. Test the data path, the protocol boundary, and the investigation workflow.

### Layer 1: deterministic code tests

Test without an LLM or live backend:

- Configuration validation and defaults.
- Typed input validation and query construction.
- Identifier and scope handling.
- API/subprocess error translation.
- Response limits, projection, and truncation metadata.
- Pagination, cache behavior, and export naming when those features exist.

Use fixtures or a local fake service for backend responses.

### Layer 2: direct MCP smoke tests

Start the real binary over stdio and verify:

- Initialization succeeds.
- The expected tool catalog is advertised.
- Valid calls return usable content.
- Invalid calls return MCP tool errors instead of protocol corruption.
- Output limits and cancellation behavior work.

This can be a small Go, Python, or shell harness. It should not require a paid model.

### Layer 3: task-level agent tests

Use an MCP client and an LLM for a small set of fixed tasks. Record:

- Tool sequence.
- Tool arguments and errors.
- Duration and response size.
- Token usage when available.
- Whether the agent chose the intended structured path.

Examples include:

- Discovering a relevant table or index and querying it.
- Enriching an IP or domain through the appropriate lookup tool.
- Pivoting from an alert to a process or endpoint.
- Dispatching a collection and retrieving its results.

These are integration and evaluation checks, not deterministic unit tests. Credentials, data, model behavior, and provider availability can change their results.

## 8. Common Mistakes

- Writing logs or subprocess output to stdio `stdout`.
- Exposing every backend endpoint instead of a small tool ladder.
- Giving agents a raw query tool without discovery, limits, or safety guidance.
- Trusting tool descriptions or JSON Schema without server-side validation.
- Returning unbounded records or multi-megabyte exports in model context.
- Duplicating auth, retries, pagination, or config logic inside MCP handlers.
- Adding a cache before defining freshness, scope, and invalidation behavior.
- Treating a live LLM test as a substitute for deterministic tests.
- Making long-running collection or export operations look like fast request/response searches.
- Assuming every server needs HTTP, profiles, persistent caching, query rewriting, adaptive timeouts, or a CLI.

## 9. Reference Projects

Use the implementations as concrete examples of different backend boundaries:

- [go-velociraptor-mcp](https://github.com/mdfranz/go-velociraptor-mcp) — shared library, CLI/MCP adapters, gRPC/mTLS, scoped endpoint operations, asynchronous collections, exports, and live tests.
- [osqueryi-mcp](https://github.com/mdfranz/osqueryi-mcp) — local subprocess execution, runtime schema discovery, progressive disclosure, bounded results, schema caching, and direct MCP smoke tests.
- [elastic-security-mcp](https://github.com/mdfranz/elastic-security-mcp) — structured security search, raw Query DSL fallback, response caching, bulk export, optional integrations, and task-level agent testing.
- [MCP Go SDK](https://github.com/modelcontextprotocol/go-sdk) — official Go protocol implementation and current transport/API documentation.
