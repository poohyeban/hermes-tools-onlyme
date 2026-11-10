FROM python:3.14-slim-trixie@sha256:9aaa474097d094040649f2162a627c0049b408f52138d980916788d80f04802f AS build
COPY requirements.lock /build/requirements.lock
RUN python -m venv /opt/osb-mcp \
    && /opt/osb-mcp/bin/pip --isolated install --no-cache-dir --index-url https://pypi.org/simple \
        --require-hashes --only-binary=:all: -r /build/requirements.lock \
    && /opt/osb-mcp/bin/pip check

FROM python:3.14-slim-trixie@sha256:9aaa474097d094040649f2162a627c0049b408f52138d980916788d80f04802f
COPY --from=build /opt/osb-mcp /opt/osb-mcp
USER 10000:10000
ENTRYPOINT ["/opt/osb-mcp/bin/opensandbox-mcp"]
