# Quotation Factory CLI

A .NET CLI tool for LLM-based coding agents to interact with the **Quotation Factory API** via Bearer token authentication. Includes 87 agent skill documents covering the full quoting workflow, configuration, interactive UI guides, per-page Settings references, and 656 validated API endpoint references.

## Features

- **API Discovery** — `qfc api discover` lists all endpoints grouped by tag, with `--search`, `--tag`, `--method`, and `--compact` filters. Compact mode outputs one line per endpoint for minimal context window usage.
- **Endpoint Inspection** — `qfc api describe <path>` shows parameters, request/response schemas, idempotency hints, and a cross-platform example call. Use `--expand-refs` to inline referenced schema types.
- **Generic REST commands** — `qfc api get|post|put|delete <path>` with `--param`, `--query`, `--body`, and `--body-file` options. `--body-file` reads JSON from a file, avoiding PowerShell 5.1 quote-stripping issues with `--body`.
- **Response filtering** — `--fields a,b,c` keeps only selected fields, `--max-items N` truncates arrays with metadata, `--quiet` suppresses output on success.
- **Batch operations** — `qfc api batch --ops '[...]'` executes multiple API calls in a single invocation with optional `--continue-on-error`.
- **Nesting summary** — `qfc api nesting-summary` aggregates cutting plan utilization and remnant strategies for a project in one command. Auto-detects partyId/projectId from navigation context when `qfc serve` is running.
- **Dry-run validation** — `--dry-run` on POST/PUT validates request body against the OpenAPI schema without sending.
- **File uploads** — `--file` and `--form` options for multipart/form-data uploads (geometry files, engineering drawings).
- **Browser OAuth auth** — persistent access and refresh tokens via `qfc auth login`, renewed shortly before expiry. Manual non-refreshable tokens remain available via `qfc auth set-token` or `QFC_ACCESS_TOKEN`.
- **Unattended service auth** — `qfc auth login --client-id <id> --client-secret <secret>` persists a party-scoped service credential and reuses its access token across commands until near expiry. Environment-only `QFC_CLIENT_ID`/`QFC_CLIENT_SECRET` remains diskless.
- **Live UI preview** — `qfc serve` reverse-proxies the Rhodium24 web app through localhost with automatic Auth0 authentication. Factory mode (full UI) or portal mode (`--party-id` for self-service buyer view). Multi-layer caching (persistent disk cache, HTML cache, speculative API prefetch) — page loads in ~500ms with warm cache.
- **MCP 2.0 server** — `qfc mcp` exposes the API to any MCP client (Claude Desktop, Cursor, VS Code) via the official ModelContextProtocol C# SDK v2 (protocol revision 2026-07-28). Meta-tools (`discover_endpoints`, `describe_endpoint`, `get_schema`, `call_api`) are access-scoped to server surfaces (contributor / admin / system). Runs over **stdio** (stored token) or **HTTP as an OAuth 2.1 resource server** (`--http`) that validates callers' Auth0 JWTs and forwards them downstream.
- **Run as an OS service** — `qfc service install` registers a long-running server (`mcp --http` or `serve`) as a **Windows service** or **systemd unit**, starting on boot and restarting on failure. `--print` previews the definition for both platforms without elevation.
- **LLM-optimized output** — pretty-printed JSON to stdout, null fields omitted, no unicode escapes, auto-generated summaries for undocumented endpoints, `_next` guidance fields, error body truncation (500 chars), and fuzzy tag suggestions on empty results.
- **Cross-platform** — all output, examples, and commands work identically on Windows (PowerShell, cmd), macOS, Linux, and Git Bash. No platform-specific syntax in generated commands.
- **Global .NET tool** — install once, use everywhere as `qfc`.
- **87 agent skills** — workflow recipes, reference docs, and interactive UI guides in `.claude/commands/` (available as `/slash-commands`) covering the full quoting lifecycle, configuration, and troubleshooting. Skills include machine-readable `requires:`/`next:` dependency frontmatter for automated workflow planning.
- **Updatable skills** — `qfc skills update` installs verified skills from the latest GitHub release without reinstalling the CLI.

## AI Agent Workflow

The recommended workflow for an AI agent using this CLI:

```bash
# 1. Authenticate (persistent — survives across sessions, works on all platforms)
qfc auth login

# 2. Discover available endpoints (--compact saves context window tokens)
qfc api discover --compact                # one-line-per-endpoint text output
qfc api discover --search projects        # search by keyword
qfc api discover --tag Project            # filter by tag (exact tag name)
qfc api discover --method POST            # filter by HTTP method

# 3. Inspect an endpoint before calling it (includes idempotency hint)
qfc api describe /api/parties/{partyId}/projects --method POST
qfc api describe /api/parties/{partyId}/projects --method POST --expand-refs

# 4. Validate before sending (--dry-run checks body against schema)
qfc api post /api/parties/{partyId}/projects \
  --param partyId=<uuid> --body "{\"name\":\"Test\"}" --dry-run

# 5. Call the endpoint (double-quoted body works on all platforms)
qfc api get /api/parties/{partyId}/projects \
  --param partyId=550e8400-e29b-41d4-a716-446655440000 \
  --query Page=1 --query PageSize=25

# 6. Filter large responses to save context window
qfc api get /api/parties/{partyId}/projects/getProjectsPaged \
  --param partyId=<uuid> --query Page=1 \
  --max-items 5 --fields id,name,status

# 7. Batch multiple calls in one invocation
qfc api batch --ops '[{"method":"GET","path":"/api/parties/{partyId}/projects","params":{"partyId":"<uuid>"}}]'
```

## Getting Started

### Authenticate

**Persistent storage (recommended):**

```bash
qfc auth login
```

Browser login stores renewable credentials. Use `qfc auth set-token eyJ...` only when browser OAuth is unavailable; manually copied tokens cannot be refreshed.

For unattended service authentication, run `qfc auth login --client-id <id> --client-secret <secret>`. QFC caches the resulting access token across commands and obtains a new one only near expiry. Authentication files are encrypted with Windows DPAPI for the current user. On platforms where OS-backed protection is unavailable, QFC warns and applies owner-only file (`0600`) and profile-directory (`0700`) permissions.

**Or via environment variable (current session only):**

```powershell
# PowerShell
$env:QFC_ACCESS_TOKEN = "eyJ..."
```

```bash
# Bash
export QFC_ACCESS_TOKEN="eyJ..."
```

To get a token:
1. Open **https://rhodium24.io/app** in your browser
2. Open Developer Tools (F12) → Network tab
3. Search for `getProjectsPaged` in the filter
4. Click the request → Headers → copy the `Authorization` value (after "Bearer ")

Run `qfc auth login` for renewable browser credentials, or `qfc auth status` to check expiry and refresh availability.

### Configure

```bash
# Set the API base URL (default already points to https://api.rhodium24.io/backend)
qfc config set api-base-url https://api.rhodium24.io/backend

# (Optional) Point to a local swagger.json to override the auto-cached spec
qfc config set swagger-path /path/to/swagger.json

# Show current config
qfc config show
```

The API spec (`swagger.json`) is **fetched automatically** from the server on first use and cached locally for 24 hours. No manual configuration needed. If the server is unreachable, the stale cache is used as fallback.

Or use environment variables (also supported via a `.env` file in the project root):

| Variable | Description |
|----------|-------------|
| `QFC_ACCESS_TOKEN` | Non-refreshable Bearer token (takes precedence over stored OAuth credentials). Prefer `qfc auth login` for persistent interactive use. |
| `QFC_API_BASE_URL` | API base URL |
| `QFC_SWAGGER_PATH` | Path to a local `swagger.json` to override auto-caching |

Configuration precedence (highest to lowest): environment variables > `.env` file > `settings.json` > auto-cache > remote fetch.

### Discover and Describe Endpoints

```bash
# List all endpoints — compact mode for LLMs (3x smaller output)
qfc api discover --compact

# Full JSON output with all metadata
qfc api discover

# Search endpoints by keyword
qfc api discover --search quotation

# Filter by tag and/or HTTP method
qfc api discover --tag Project --method GET

# Get full details for a specific endpoint
qfc api describe /api/parties/{partyId}/projects --method GET

# Expand referenced types inline (shows AddressDto fields etc.)
qfc api describe /api/parties/{partyId}/projects --method POST --expand-refs

# The describe output includes:
# - Path and query parameters (null fields omitted)
# - Request body schema with required fields highlighted
# - Response schemas
# - Cross-platform example call (double-quoted, no single quotes)
# - _next field with a ready-to-run command
# - Auto-generated summary when the API spec has none
```

### Call the API

```bash
# GET with path parameter substitution
qfc api get /api/parties/{partyId}/projects \
  --param partyId=550e8400-e29b-41d4-a716-446655440000

# POST with body (double-quoted — works on bash, PowerShell, cmd, and Git Bash)
qfc api post /api/parties/{partyId}/projects \
  --param partyId=550e8400-e29b-41d4-a716-446655440000 \
  --body "{\"name\":\"New Project\"}"

# With query parameters
qfc api get /api/parties/{partyId}/projects \
  --param partyId=550e8400-e29b-41d4-a716-446655440000 \
  --query Page=1 --query PageSize=25 --query Search=steel

# File upload (multipart/form-data)
qfc api post /api/parties/{partyId}/projects/{projectId}/uploadGeometryWithDetails \
  --param partyId=<uuid> --param projectId=<uuid> \
  --file "geometryFiles[0].file=@bracket.step" \
  --form "geometryFiles[0].quantity=4"

# DELETE
qfc api delete /api/parties/{partyId}/projects/{projectId} \
  --param partyId=<uuid> --param projectId=<uuid>

# Suppress output on success (--quiet)
qfc api put /api/parties/{partyId}/projects/{projectId}/status \
  --param partyId=<uuid> --param projectId=<uuid> \
  --body "{\"status\":\"Quoted\"}" --quiet

# Truncate large arrays (--max-items) and filter fields (--fields)
qfc api get /api/parties/{partyId}/projects/getProjectsPaged \
  --param partyId=<uuid> --max-items 10 --fields id,name,status

# Validate body against schema without sending (--dry-run)
qfc api post /api/parties/{partyId}/projects \
  --param partyId=<uuid> --body "{\"name\":\"Test\"}" --dry-run

# Execute multiple operations in one invocation (batch)
qfc api batch --ops '[
  {"method":"GET","path":"/api/parties/{partyId}/projects","params":{"partyId":"<uuid>"}},
  {"method":"GET","path":"/api/user"}
]'
qfc api batch --ops '[...]' --continue-on-error

# Health check
qfc health
```

### Show Config

```bash
qfc config show
```

### MCP 2.0 Server

Expose the API to MCP clients (Claude Desktop, Cursor, VS Code) with `qfc mcp`. Built on the official ModelContextProtocol C# SDK v2 (protocol revision 2026-07-28 — stateless, discovery-first).

```bash
# stdio transport (default) — uses the stored qfc token; point your MCP client at this command
qfc mcp --access contributor

# Expose specific surfaces only (repeatable). Surfaces: rfq, quotation, order,
# procurement, platform-config, portal-config, system
qfc mcp --surface quotation --surface procurement

# Enable mutating calls (POST/PUT/DELETE) — read-only by default
qfc mcp --access contributor --allow-writes

# HTTP transport as an OAuth 2.1 resource server (integrates with Auth0)
# Validates each caller's Auth0 JWT and forwards it to the backend.
# Multi-tenant: each caller's scope is derived from their token roles (capped
# by --access/--surface), so one instance serves contributor/admin/system tiers.
qfc mcp --http --port 5200 --public-url https://mcp.example.com

# Local HTTP without auth (development only — never expose publicly)
qfc mcp --http --no-auth
```

Tools exposed: `whoami`, `find_party`, `list_surfaces`, `list_skills`, `get_skill`, `discover_endpoints`, `describe_endpoint`, `get_schema`, `call_api` — all filtered to the server's access scope. `list_skills`/`get_skill` let an agent pull a qf-* workflow playbook (the skills are embedded in the binary, so no `.claude/commands` folder is needed at runtime; the skills are also exposed as MCP prompts for human-driven clients). `whoami` returns the caller's identity and the parties (companies) they can act on (paginated; use `find_party` to search by name when the caller belongs to many). `call_api` refuses destructive or customer-facing actions (delete, cancel, send, ERP export) until re-called with `confirm: true`. On the HTTP transport the server advertises Auth0 via RFC 9728 Protected Resource Metadata at `/.well-known/oauth-protected-resource` and challenges unauthenticated calls with a `401`.

**Multi-tenant, per-caller scope.** On the authenticated HTTP transport, each caller's scope is derived from the roles in their Auth0 token (Reader/Contributor → contributor surfaces, Admin → +admin, SysAdmin → +system), capped by the instance's `--access`/`--surface` ceiling — so a single service instance serves different tiers to different callers, each acting as their own Auth0 identity.

It works with today's tokens without any Auth0 change: a token that carries **no role/permission claim** falls back to the instance scope (whatever `--access`/`--surface` the instance was launched with), and per-caller differentiation activates automatically once the tokens start carrying roles. To emit roles, add them to the Auth0 access token (e.g. via an Action) and set `Auth0RolesClaim` if they live under a namespaced claim. Use `--no-per-caller-scope` to force the fixed instance scope for everyone. (stdio always uses the fixed scope; the backend enforces real permissions on every call regardless.)

**Party isolation (multi-tenant).** On the authenticated HTTP transport, `call_api` refuses a request that targets a party the caller is not a member of (resolved from their `/api/user` profile), returning `PARTY_FORBIDDEN` before the request reaches the backend. So a single centralized instance — e.g. one Quotation Factory hosts for its Customer Success team and lists in the ChatGPT / Claude MCP directories — safely serves many callers: each customer reaches only their own party, while a cross-party Customer Success user reaches all of theirs. Agents call `whoami` to discover which parties they may use. Disable the MCP-layer check with `--no-party-check` (the backend still enforces access).

For hosting: `--bind 0.0.0.0` exposes the port directly (e.g. in a container behind an ingress; default is loopback), an unauthenticated `/health` endpoint backs orchestrator probes, and every `call_api` outcome is written to stderr as a structured audit line (caller, method, path, surface, status). `call_api` also accepts a `files` array (local `path` on stdio, base64 `contentBase64` when hosted) for CAD/drawing uploads, and returns binary responses (PDF/zip) base64-encoded rather than corrupting them.

The `qf-*` skills are also exposed as **MCP prompts** — an MCP client discovers them via `prompts/list` and pulls a workflow with `prompts/get`, then executes its steps through `call_api`. Prompts are **filtered to the server's access scope** (a skill is shown when it references an endpoint in an allowed surface; skills that reference no endpoints are always shown). Point `--skills-dir` at the skills folder (defaults to `.claude/commands`), or disable with `--no-prompts`.

### Run as an OS Service

Register a long-running server (`mcp --http` by default, or `serve`) with the OS service manager. Requires Administrator (Windows) or root (Linux).

```bash
# Preview the service definition for both platforms — no elevation needed
qfc service install --command "mcp --http --port 5200" --print

# Install (Windows service via sc.exe, or systemd unit on Linux)
qfc service install --name qfc-mcp --command "mcp --http --port 5200"

# Control it
qfc service start   --name qfc-mcp
qfc service status  --name qfc-mcp
qfc service stop    --name qfc-mcp
qfc service uninstall --name qfc-mcp
```

On Linux the unit is `Type=notify` with `Restart=on-failure`; use `--user` and `--env-file` (e.g. to supply `QFC_ACCESS_TOKEN` for a `serve` service). When installing via `dotnet run`, pass `--exe-path` to the published `qfc(.exe)` binary.

## Documentation

| Document | Location | Purpose |
|----------|----------|---------|
| Setup & Configuration | `dist/SETUP-PROMPT.md` | Installation, authentication, skill index — paste into Claude Desktop as a project prompt |
| Prompt Examples | `dist/PROMPT-EXAMPLES.md` | 25+ example prompts showing how to use each skill |
| Skill Map | `.claude/commands/qf-cli.md` | Central skill index with "I want to..." lookup table, quoting workflow steps, configuration prerequisites, and troubleshooting guide |
| Prerequisites | `.claude/commands/qf-prerequisites.md` | Shared warnings and conventions referenced by all skills |
| Glossary | `.claude/commands/qf-glossary.md` | Canonical definitions for BOM item, working step, resource, nesting, etc. |
| Enums | `.claude/commands/qf-enums.md` | Centralized project statuses, working step types (84), issue codes (28), rule editor models (13) |

## Architecture

```
src/QuotationFactory.Cli/
├── Authentication/         # Bearer token provider (persistent file + env var, 30s cache)
├── Commands/               # CLI command handlers
│   ├── ApiCommands.cs      # get/post/put/delete with --quiet, --max-items, --fields, --dry-run
│   ├── BatchCommand.cs     # qfc api batch — multi-operation sequential execution
│   ├── NestingSummaryCommand.cs # qfc api nesting-summary — cutting plan utilization + remnant settings
│   ├── DiscoverCommand.cs  # qfc api discover — endpoint listing with mtime-cached spec
│   ├── DescribeCommand.cs  # qfc api describe — endpoint detail with idempotency hints
│   └── ServeCommand.cs     # qfc serve — orchestrator for the reverse proxy server
├── Serve/                  # Reverse proxy server components
│   ├── InjectedScripts.cs  # Auth injection, context tracking, guide engine, PDF polyfill JS
│   ├── TtsHandler.cs       # TTS endpoints, cache, ElevenLabs/OpenAI integration
│   ├── GatewayHandler.cs   # Gateway API endpoints (discover, describe, tags, schema)
│   ├── GuideHandler.cs     # Guide overlay + selector validation HTTP endpoints
│   ├── WebSocketRelay.cs   # WebSocket bidirectional proxy with proper cleanup
│   └── AssetCache.cs       # Gzip compression, disk cache, CachedAsset/CachedHtml records
├── Configuration/          # Settings storage & environment overrides (.env walk capped to 5 levels)
├── Gateway/                # GatewayClient with persistent down-cache (60s TTL)
├── OpenApi/                # OpenAPI spec parsing
│   ├── OpenApiSpecProvider.cs  # Spec loader and endpoint extraction
│   ├── SchemaFlattener.cs      # JSON Schema to flat property list
│   └── EndpointInfo.cs         # Data records for endpoints, params, schemas
├── Output/                 # JSON output helpers (error body truncation at 500 chars)
└── Program.cs              # Entry point

.claude/commands/            # 87 slash commands (agent skill documents)
├── qf-cli.md               # Core CLI usage + Skill Map + "I want to..." selection table
├── qf-prerequisites.md     # Shared warnings & conventions (referenced by all skills)
├── qf-glossary.md          # Canonical domain term definitions
├── qf-enums.md             # Centralized enum values (statuses, working steps, issue codes)
├── qf-auth.md              # Authentication setup
├── qf-context.md           # Navigation context (partyId, projectId, bomItemId)
├── qf-create-project.md    # Project creation workflow
├── qf-upload-geometry.md   # Geometry file upload
├── qf-analyze-drawing.md   # Drawing analysis (OCR)
├── qf-detect-purchasing-part.md # Purchasing part detection
├── qf-set-material.md      # Material assignment
├── qf-add-working-step.md  # Working step addition
├── qf-lookup-customer.md   # Customer lookup
├── qf-projects.md          # Project management & status
├── qf-bom-item.md          # BOM item details & overrides
├── qf-quotations.md        # Quotations, RFQ, price requests
├── qf-business-rules.md    # 13 business rule models
├── qf-resources.md         # Machines & Excel estimators
├── qf-materials.md         # Articles, standards, keywords
├── qf-financial.md         # Currency, terms, calculation
├── qf-delivery.md          # Delivery calendar & units
├── qf-production.md        # Routes, nesting, machining
├── qf-suppliers.md         # Address book & suppliers
├── qf-portal.md            # Portal & email config
├── qf-party.md             # Company profile & agents
├── qf-party-pricing.md     # Party-level pricing rules
├── qf-users.md             # Users, roles, access
├── qf-imnoo.md             # Imnoo CNC integration
├── qf-analytics.md         # Segment.io CDP events
├── qf-guide-author.md      # Meta-skill: authoring interactive UI guides
├── qf-guide-create-project.md  # Guide: create a new project walkthrough
├── qf-guide-project-states.md  # Guide: project lifecycle states tour
├── qf-guide-upload-geometry.md # Guide: upload CAD files walkthrough
├── qf-guide-cad-drawings.md    # Guide: CAD + engineering drawings tour
├── qf-guide-dynamic.md     # Dynamic guide generation from user questions
└── understand*.md           # 8 codebase understanding skills
```

All 25 workflow/configuration skills include `requires:` and `next:` YAML frontmatter fields, forming a machine-readable dependency graph for automated workflow planning. The 6 guide skills define interactive UI walkthroughs with DOM selectors, narration text, and step-by-step playback instructions.

## Distribution

The cross-platform `qfc-cli.zip` contains:
- `bin/win-x64/qfc.exe` — Windows (x64) self-contained binary
- `bin/linux-x64/qfc` — Linux (x64) self-contained binary
- `bin/osx-x64/qfc` — macOS (Intel) self-contained binary
- `bin/osx-arm64/qfc` — macOS (Apple Silicon) self-contained binary
- `skills/` — all 87 agent skill documents (with dependency frontmatter)
- `README.md` — project overview, architecture, and LLM integration tips
- `SETUP-PROMPT.md` — installation and configuration instructions
- `PROMPT-EXAMPLES.md` — 25+ example prompts for using the skills
- `.env.example` — configuration template (copy to `.env` and fill in)

`CLAUDE.md` is intentionally **not** included — it is the internal development guide for working on the CLI, not end-user material. The zip is assembled by `build/pack.ps1`, which stages exactly this manifest; keep the script and this list in sync.

Releases also provide `qfc-skills.zip`. Install or refresh its contents in the current project with:

```bash
qfc skills update
qfc skills update --prerelease
qfc skills update --version 0.0.1-beta
qfc skills update --skills-dir .claude/commands
```

The updater verifies the release checksum and preserves files it did not install.

No .NET runtime required on any platform.

## Building

```bash
# Build
dotnet build

# Run tests
dotnet test

# Run locally (development)
dotnet run --project src/QuotationFactory.Cli -- auth status

# Publish self-contained binaries (all platforms)
dotnet publish src/QuotationFactory.Cli -c Release -r win-x64 --self-contained -o dist/bin/win-x64
dotnet publish src/QuotationFactory.Cli -c Release -r linux-x64 --self-contained -o dist/bin/linux-x64
dotnet publish src/QuotationFactory.Cli -c Release -r osx-x64 --self-contained -o dist/bin/osx-x64
dotnet publish src/QuotationFactory.Cli -c Release -r osx-arm64 --self-contained -o dist/bin/osx-arm64
```

## Resilience

The CLI is designed for unattended agent use where silent failures are worse than loud ones:

- **Atomic file writes** — config, token, and swagger cache files use write-to-temp-then-rename to prevent corruption from crashes or concurrent access.
- **Corrupted config recovery** — malformed `settings.json` or token files fall back to defaults with a warning instead of crashing.
- **Swagger cache validation** — the cached spec is validated on load (must be a JSON object with a `paths` property). Corrupted caches are auto-deleted and re-fetched. Cross-invocation mtime caching skips re-parsing when the file hasn't changed.
- **Circular `$ref` protection** — the OpenAPI schema flattener detects circular references and broken `$ref` pointers, preventing infinite loops on malformed specs.
- **HTTP timeouts** — all outbound HTTP calls have explicit timeouts (30s for spec fetch, 2min for proxy, 30s for TTS).
- **Gateway down cache** — when `qfc serve` is not running, the failed availability check is cached to disk for 60 seconds, skipping the 150ms probe on subsequent cold invocations.
- **Error body truncation** — error response bodies are capped at 500 characters to prevent context window overflow, with a `fullErrorTruncated` flag when content was trimmed.
- **Expired env var auto-fallback** — when `QFC_ACCESS_TOKEN` holds an expired token but a valid stored token exists, the CLI automatically uses the stored token. Prevents stale env vars (baked into Windows User environment or inherited by long-running parent processes) from permanently breaking auth.
- **Bounded `.env` search** — the upward directory walk for `.env` files is capped at 5 levels to prevent pathological traversal on deeply nested paths.
- **Gzip-compressed proxy responses** — HTML, JS, and CSS proxied from the upstream are gzip-compressed before serving. The upstream response is decompressed for URL rewriting and auth injection, then re-compressed. Reduces the 3.1 MB JS bundle to ~1 MB on the wire.
- **Persistent disk cache** — processed JS/CSS assets (gzip-compressed, URL-rewritten) are saved to `%LOCALAPPDATA%\QuotationFactory.Cli\asset-cache\` and loaded into memory on server startup. Assets survive server restarts — only the very first launch ever fetches from upstream.
- **Startup prefetch** — a background task fetches the SPA HTML page, discovers asset URLs, caches any missing assets, and pre-warms `/api/user` — all before the browser connects.
- **HTML page cache** — processed HTML (auth-injected, CSP-stripped) cached for 5 minutes with token-based invalidation. Eliminates ~1s upstream round-trip on repeated page loads.
- **Speculative API prefetch** — `/api/user` is fetched in background when HTML is first requested. By the time the SPA's JS loads and calls it (~1-2s later), the response is already cached (2ms instead of 1000ms).
- **Deferred server init** — OpenAPI spec loading and ElevenLabs voice resolution run in background tasks. The server starts accepting requests within ~900ms; gateway endpoints return 503 until spec loads (~200ms later).
- **Preview-compatible readiness** — root `/` returns 200 (with JS redirect) instead of 302, so the Claude Code preview UI's readiness poll detects the server immediately rather than timing out.
- **Unused script stripping** — Appcues script tags are stripped from proxied HTML to eliminate blocked network requests.
- **Resource leak prevention** — file streams in multipart uploads are tracked and disposed on validation errors; TTS audio cache is bounded (500 entries / 200MB).
- **Stable cache keys** — TTS cache uses SHA256 hashing instead of `GetHashCode()` for deterministic, collision-resistant keys.
- **PowerShell 5.1 quote-stripping detection** — `--body` arguments with nested JSON lose their inner double quotes in PS 5.1 (a known platform bug). The CLI detects mangled bodies (JSON without quotes) and returns a `BODY_QUOTES_STRIPPED` error pointing to `--body-file` as the fix.
- **`--body-file` option** — reads request body from a JSON file instead of a command-line argument, bypassing shell quoting issues entirely.
- **`--fields` empty-match warning** — when `--fields` matches zero properties, a stderr warning lists the available property names so the caller can correct without a round-trip.
- **WebSocket relay cleanup** — bidirectional WebSocket proxy properly cancels and awaits both relay directions before disposing sockets, preventing orphaned tasks.
- **Gateway decompression** — the gateway client handles gzip-compressed responses from the proxy, preventing raw binary output.

234 unit tests cover these behaviors, including dedicated robustness tests for each resilience mechanism. All tests use isolated temp directories — they never read or write real user data (tokens, config).

## LLM Tool Integration Tips

1. **Use `--compact` for discovery** — `qfc api discover --compact` outputs one line per endpoint (~90KB vs ~276KB full JSON). Use full JSON only when you need parameter details.
2. **Use `--expand-refs` for describe** — `qfc api describe <path> --expand-refs` inlines referenced schema types so you can see all fields without extra calls. The output includes an `idempotent` field so agents know which calls are safe to retry.
3. **Filter large responses** — `--max-items 10` truncates arrays with `totalCount`/`truncated` metadata. `--fields id,name,status` keeps only the fields you need. Both reduce context window usage dramatically.
4. **Use `--quiet` in pipelines** — suppress success output when you only care about errors (e.g. status updates, deletes).
5. **Validate before sending** — `--dry-run` on POST/PUT checks your JSON body against the OpenAPI schema and reports missing required fields without making the actual API call.
6. **Batch operations** — `qfc api batch --ops '[...]'` executes multiple API calls sequentially in one invocation. Use `--continue-on-error` to process all operations even if some fail. Returns an array of results with success/failure per operation.
7. **Follow the `_next` field** — `describe` and `health` output includes a `_next` field with a ready-to-run command.
8. **Use `--body-file` for JSON bodies** — PowerShell 5.1 strips inner double quotes from `--body` arguments. Write JSON to a temp file and pass `--body-file path.json` instead. The CLI detects mangled bodies and suggests this fix.
9. **Use `--param` for path parameters** — `{partyId}` in the path is replaced by `--param partyId=<value>`. Missing params produce a `MISSING_PATH_PARAMS` error with the exact param names needed.
10. **Check `didYouMean` on empty results** — if `--tag` returns nothing, the output includes fuzzy suggestions and the full `availableTags` list.
11. **Null fields are omitted** — output only includes fields with values. No `"format": null` or `"refName": null` noise.
12. **Errors are structured JSON on stderr** — pretty-printed with readable `<angle brackets>`, not unicode escapes. Error bodies are truncated to 500 chars to prevent context window overflow.
13. **Use skill dependency frontmatter** — each skill's YAML frontmatter has `requires:` and `next:` fields. Agents can parse these to build a workflow graph and determine prerequisites before executing a skill.
14. **Reference skills for shared knowledge** — `qf-prerequisites` (shared warnings), `qf-glossary` (domain terms), and `qf-enums` (project statuses, working step types, issue codes) are centralized references. Load them once instead of hunting through individual skills.
15. **Swagger auto-caching** — the API spec is fetched once and cached for 24 hours with mtime-based revalidation. Set `QFC_SWAGGER_PATH` only if you need to override this with a specific local file.
