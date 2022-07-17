FROM alpine:3.16 as build

ARG CADDY_VERSION=v2.5.2

ENV PATH=/root/go/bin:${PATH}
RUN apk --no-cache add go git && \
    go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest && \
    /root/go/bin/xcaddy build ${CADDY_VERSION} \
    --with github.com/caddy-dns/duckdns --output /root/go/bin

FROM caddy:2.5.2 as app
COPY --from=build /root/go/bin/caddy /usr/bin/caddy
