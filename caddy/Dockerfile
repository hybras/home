ARG CADDY_VERSION=2.10.0
FROM caddy:${CADDY_VERSION}-builder AS builder

RUN xcaddy build "${CADDY_VERSION}" \
    --with github.com/caddy-dns/duckdns

FROM caddy:${CADDY_VERSION}-alpine

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
