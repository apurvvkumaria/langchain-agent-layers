# CLAUDE.md

A hands-on learning project for **agent runtime patterns** — built up in 29 deliberate
layers, each adding one runtime capability on top of the last. Three front doors (Click CLI
`agent.py`, FastAPI `api.py`, MCP server `mcp_integration/server.py`) over one shared ReAct
core (`core.py`). See `README.md` for the full layer table, dependency rationale,
file-by-file project structure, usage examples, and OpenShell sandbox setup — assume the
README is authoritative for those.

## Conventions for Claude Code

- **Audience:** deep distributed-systems background, experimenting with agent frameworks
  outside current work exposure. Skip generic programming explanations; lean into *why* an
  agent framework is designed a certain way and how it maps to systems concepts I already
  know (control loops, retries, orchestration, state). Don't dumb things down.
- **Explain the runtime, not just the code.** When changing code, say what the framework
  does behind the call (e.g., what `AgentExecutor.invoke` does per iteration). Understanding
  the runtime is the whole point of this project.
- **Use uv for everything** — `uv run python agent.py`, `uv add`, `uv remove`. Never
  suggest `pip` or hand-editing `pyproject.toml` deps.
- **Keep it minimal and readable.** Learning artifact; favor clear, well-commented code
  over abstraction or production hardening unless I ask.
- **Never commit secrets.** `.env` holds real keys and is git-ignored. Don't echo secret
  values or write them into tracked files.
- **Don't make billed API calls without asking** — running the agent live hits the
  Anthropic API. Smoke-test construction with a dummy key when verifying changes.

## Architectural rules (don't break these)

- **One-way imports:** `tools`/`hooks` ← `core` ← (`agent`, `api`). `agent` also imports
  `api` for `serve`. Nothing imports `agent`.
- **Match the agent to the command:** single-shot `ask` uses a memory-free builder so input
  tokens stay flat; `chat`/`research` use a memory-backed one. Don't unify them.
- **LangFuse callbacks must be passed per-call** via `config={"callbacks": [...]}`. The
  ones set on the `AgentExecutor` constructor do *not* propagate through `astream_events`.

## Package-shadowing gotchas (real traps, leave them alone)

- **`mcp_integration/`, not `mcp/`.** The MCP SDK package is named `mcp`; an `mcp/`
  directory would shadow `from mcp import ...`.
- **`openshell/` deliberately has no `__init__.py`** — it's docs+config only. If it became a
  Python package it would namespace-shadow the `openshell` SDK.
- **`skills/<name>/` packages are the OpenClaw shape** (`SKILL.md` + `skill.py` +
  `policy.yaml`). Don't flatten them into single-file tools.

## Dependency pins / framework-version traps

- **LangChain 1.x moved classic agents** to `langchain-classic`. `create_react_agent` +
  `AgentExecutor` are no longer in top-level `langchain.agents`.
- **LangFuse v4:** handler is `langfuse.langchain.CallbackHandler` (not `langfuse.callback`),
  auth via `LANGFUSE_*` env vars, not constructor args.
- **`duckduckgo-search` → `ddgs`** (deprecated rename; the community tool requires `ddgs` at
  runtime).
- **deepeval pinned to 4.x** because 2.x imports the removed `langchain.schema` and breaks
  pytest collection. 4.x needs `click<8.4`, so `click` is pinned `>=8.1,<8.4` to let both
  resolve.
- **Native arm64 required on macOS:** `torch` / `onnxruntime` have no macOS-x86_64 +
  Python-3.13 wheels, so `sentence-transformers` and `chromadb` won't install under a
  Rosetta venv. Project uses arm64 `uv` at `~/.local/bin/uv` + a uv-managed arm64 Python 3.13.
  `uv sync` will fail on x86_64 mac — call this out rather than silently working around it.

## OpenShell sandbox notes (Layer 21)

- The agent-in-sandbox path uses the **`openshell` CLI as a subprocess** from
  `sandbox_runner.py`, never `import openshell`. The installed SDK is sandbox-centric
  (`Sandbox`, `SandboxClient`, `ExecResult`), not the `@skill`/`SkillContext` decorator API
  some docs suggest.
- Sandbox policy is the **real v0.0.47 schema** (`filesystem_policy` / `landlock` /
  `process` / `network_policies`). Egress is **binary-keyed, default-deny** — unlisted = denied.
  No `deny:'*'`. Resources (`--cpu`/`--memory`) are `create` flags, not policy fields.
- `tls: skip` (pass-through), not `terminate` (MITMs the Python SDK and breaks cert
  validation).
- macOS gateway gotcha: needs `host.openshell.internal` in the cert SAN list, real Docker
  socket at `$HOME/.docker/run/docker.sock`, host:host bind mounts. See `openshell/setup.md`
  + `scripts/setup-openshell.sh`.
