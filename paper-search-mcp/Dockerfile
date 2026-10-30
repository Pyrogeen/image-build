FROM python:3.12-slim-bookworm@sha256:623674a14ef93d052bb9b0de4dbf5f28eb578a35263838941cc8aa43213c1487 AS python-base

FROM python-base AS builder
COPY --from=ghcr.io/astral-sh/uv:0.12.9@sha256:8b940d3a9d65bed080436972241af2e21c84b5e8c9193f7014ed71479ee795ff /uv /uvx /usr/local/bin/
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy UV_PYTHON_DOWNLOADS=0
WORKDIR /app
COPY pyproject.toml uv.lock README.md LICENSE ./
RUN --mount=type=cache,target=/root/.cache/uv uv sync --frozen --no-install-project --no-dev
COPY paper_search_mcp/ paper_search_mcp/
RUN --mount=type=cache,target=/root/.cache/uv uv sync --frozen --no-dev --no-editable
# Patch: disable the MCP SDK DNS-rebinding Host/Origin check. FastMCP
# auto-enables it at construction (default host is 127.0.0.1) with
# allowed_hosts=["127.0.0.1:*", "localhost:*", …]; upstream main() then
# rebinds the host to 0.0.0.0 at runtime but never clears transport_security,
# so behind a Kubernetes Service (Host=<svc-name> or the gateway hostname)
# every request dies with 421 "Misdirected Request". Host validation is the
# gateway's job, not the in-pod MCP server's — same rationale as the t212-mcp
# patch.
# NOTE: patch the INSTALLED copy, not /app source — `uv sync --no-editable`
# COPIES the project into .venv, which is import-authoritative at runtime
# (/app/paper_search_mcp/ is dead weight at this point).
RUN python3 - <<'EOF'
import pathlib
p = pathlib.Path(
    "/app/.venv/lib/python3.12/site-packages/paper_search_mcp/server.py"
)
src = p.read_text()
needle = 'mcp = FastMCP("paper_search_server")'
assert needle in src, "FastMCP line not found in installed package — update patch"
assert "# PATCH:" not in src, "already patched"
p.write_text(src.replace(needle, needle + """
# PATCH: clear transport_security so the SDK Host check does not 421
# non-localhost Host headers (see Dockerfile comment).
if mcp.settings.transport_security is not None:
    mcp.settings.transport_security = None""", 1))
EOF

FROM python-base
RUN groupadd --gid 10001 app && useradd --uid 10001 --gid app --create-home app
COPY --from=builder --chown=10001:10001 /app/.venv /app/.venv
ENV PATH="/app/.venv/bin:$PATH" PYTHONUNBUFFERED=1
# Server mode is set at runtime (PAPER_SEARCH_MCP_TRANSPORT=streamable-http,
# PAPER_SEARCH_MCP_HOST=0.0.0.0, PAPER_SEARCH_MCP_PORT) — upstream defaults are
# stdio/127.0.0.1:8000, so the caller must set transport explicitly.
WORKDIR /home/app
USER 10001:10001
ENTRYPOINT ["paper-search-mcp"]