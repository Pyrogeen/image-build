FROM python:3.14-slim-bookworm@sha256:82bc3c539b8813ada9d68c63b40158fa002f7f33de9bf3312a3dfdc0620dff56 AS python-base

FROM python-base AS builder
COPY --from=ghcr.io/astral-sh/uv:0.12.9@sha256:8b940d3a9d65bed080436972241af2e21c84b5e8c9193f7014ed71479ee795ff /uv /uvx /usr/local/bin/
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy UV_PYTHON_DOWNLOADS=0
WORKDIR /app
COPY pyproject.toml uv.lock README.md LICENSE ./
RUN --mount=type=cache,target=/root/.cache/uv uv sync --frozen --no-install-project --no-dev
COPY src/trading212_mcp/ src/trading212_mcp/
# Patch: bind to all interfaces (0.0.0.0) so the pod accepts traffic on port 8000.
# Upstream hardcodes host="127.0.0.1" in main() with no env var to override it.
RUN sed -i 's/host="127\.0\.0\.1"/host="0.0.0.0"/' src/trading212_mcp/server.py
# Patch: disable DNS-rebinding Host/Origin checks. Upstream hardcodes
# enable_dns_rebinding_protection=True with allowed_hosts=["127.0.0.1:8000",
# "localhost:8000"]; traffic via the Kubernetes Service carries Host=t212-mcp:8000,
# which would get a 421 "Invalid Host header". Behind an LB/Service the check is
# the wrong defence-in-depth layer (that's the proxy's job) — the SDK docs
# recommend switching it off in exactly this scenario.
RUN sed -i 's/enable_dns_rebinding_protection=True/enable_dns_rebinding_protection=False/' src/trading212_mcp/server.py
# Fail the build if either patch silently did nothing (e.g. upstream refactored the literal away).
RUN grep -q 'host="0.0.0.0"' src/trading212_mcp/server.py \
    && grep -q 'enable_dns_rebinding_protection=False' src/trading212_mcp/server.py
RUN --mount=type=cache,target=/root/.cache/uv uv sync --frozen --no-dev --no-editable

FROM python-base
RUN groupadd --gid 10001 app && useradd --uid 10001 --gid app --create-home app
COPY --from=builder --chown=10001:10001 /app/.venv /app/.venv
ENV PATH="/app/.venv/bin:$PATH" PYTHONUNBUFFERED=1
ENV TRADING212_CACHE_DIR=/home/app/.cache/trading212-v3
WORKDIR /home/app
USER 10001:10001
ENTRYPOINT ["trading212-mcp-server"]