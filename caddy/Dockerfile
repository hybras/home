ARG CADDY_VERSION=2.5.2
ARG CADDY=caddy:${CADDY_VERSION}

FROM alpine:3.16 as build
ARG CADDY_VERSION
RUN apk --no-cache add go git
RUN go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest
RUN /root/go/bin/xcaddy build v"${CADDY_VERSION}" \
    --with github.com/caddy-dns/duckdns --output /root/go/bin

FROM ${CADDY} as app
COPY --from=build /root/go/bin/caddy /usr/bin/caddy
