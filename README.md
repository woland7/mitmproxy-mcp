# mitmproxy-mcp

A minimal [Model Context Protocol](https://modelcontextprotocol.io/) server for inspecting
[mitmproxy](https://mitmproxy.org/) dump files with an LLM agent.

`mitmproxy-mcp` loads `.mitm` trace dumps from disk, normalizes HTTP flows, and exposes
them over MCP stdio so an agent can search traffic, inspect requests and responses, and
generate reproducible Python `requests` snippets.

## Status

This project is intentionally small and early-stage. It currently focuses on offline
inspection of mitmproxy dump files, not live proxying or replay.

Supported today:

- list mitmproxy dump files on disk
- load a dump file into memory
- list normalized HTTP flow summaries
- filter flows by method, host, path fragment, and status code
- retrieve a full normalized flow
- export a single flow as a Python `requests.request(...)` snippet

Not included yet:

- live mitmproxy integration
- traffic replay
- persistence or database storage
- endpoint grouping or schema inference
- OpenAPI generation
- UI

## Requirements

- Python 3.11+
- [uv](https://docs.astral.sh/uv/)
- One or more mitmproxy dump files, usually ending in `.mitm`

## Installation

Clone the repository and install dependencies with `uv`:

```bash
git clone <repo-url>
cd mitmproxy-mcp
uv sync
```

If you want a default local trace directory, copy the example environment file:

```bash
cp .env.example .env
```

Then edit `.env` if needed:

```bash
MITM_DUMP_DIR=./data
```

## Usage

Run the MCP stdio server:

```bash
uv run mitmproxy-mcp
```

You can also run it as a Python module:

```bash
uv run python -m mitmproxy_mcp
```

For a quick manual check against a dump file:

```bash
uv run mitmproxy-mcp-inspect ./path/to/session.mitm --limit 10
```

## MCP Client Configuration

Add the server to an MCP client that supports stdio servers. Replace `/path/to/mitmproxy-mcp`
with your local checkout path:

```json
{
  "mcpServers": {
    "mitmproxy": {
      "command": "uv",
      "args": [
        "run",
        "--directory",
        "/path/to/mitmproxy-mcp",
        "mitmproxy-mcp"
      ]
    }
  }
}
```

A template is available in [`examples/mcp-server.json`](examples/mcp-server.json).

## MCP Tools

### `list_trace_files`

List mitmproxy dump files under a directory.

Arguments:

- `directory`: directory to search; defaults to `MITM_DUMP_DIR` or `./data`
- `pattern`: glob pattern; defaults to `*.mitm`

### `load_trace`

Load a mitmproxy dump file into memory and return metadata.

Arguments:

- `path`: path to the dump file

Returns:

- `trace_id`: absolute path used to identify the loaded trace
- `path`: absolute dump path
- `flow_count`: number of HTTP flows loaded
- `hosts`: distinct hosts in the trace
- `methods`: distinct HTTP methods in the trace

### `list_flows`

List compact normalized flow summaries from a loaded trace.

Arguments:

- `trace_id`: trace ID returned by `load_trace`
- `offset`: pagination offset; defaults to `0`
- `limit`: max flows to return; defaults to `20`
- `method`: optional HTTP method filter
- `host`: optional exact host filter
- `path_contains`: optional path substring filter
- `status_code`: optional response status code filter

### `get_flow`

Return the full normalized request and response for one flow.

Arguments:

- `trace_id`: trace ID returned by `load_trace`
- `flow_id`: flow ID returned by `list_flows`

The full flow includes query parameters, ordered header pairs, decoded body text when
available, and base64-encoded raw body content.

### `export_flow_as_python`

Generate a direct Python `requests.request(...)` snippet for one loaded flow.

Arguments:

- `trace_id`: trace ID returned by `load_trace`
- `flow_id`: flow ID returned by `list_flows`

The generated snippet preserves duplicate query parameters. Headers are emitted first as
ordered pairs, then converted to the mapping expected by `requests`.

Example tool calls are available in [`examples/tool-calls.json`](examples/tool-calls.json).

## Security

Mitmproxy dumps can contain cookies, access tokens, authorization headers, request bodies,
and personal data. Review or sanitize trace files before committing them to a public
repository.

## Project Layout

```text
docs/decisions/   architectural and technical decision records
docs/plans/       high-level project intent and scope
docs/tasks/       execution tracking
examples/         MCP client and tool-call examples
src/              Python package source
```

The repository is decision-led: cross-cutting technical choices should be captured in
`docs/decisions/` before they become implicit project convention.

## Development

Install dependencies:

```bash
uv sync
```

Run the server:

```bash
uv run mitmproxy-mcp
```

Run the inspection helper:

```bash
uv run mitmproxy-mcp-inspect ./path/to/session.mitm --limit 10
```

Build the package:

```bash
uv build
```

This repository currently does not define a pytest suite or linting toolchain. See
[`AGENTS.md`](AGENTS.md) and the decision records under [`docs/decisions/`](docs/decisions/)
for the current project standards.

## Validation

Manual validation notes are tracked in [`VALIDATION.md`](VALIDATION.md). The current
implementation has been validated against a real mitmproxy dump for loading, filtering,
full-flow retrieval, and Python snippet export.

## License

No license has been declared yet. Add a license before publishing if you want others to
use, copy, modify, or distribute this project.
