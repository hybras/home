FROM node:lts-alpine3.15 as build

WORKDIR /app

RUN apk add --no-cache git && \
	git clone --depth 1 https://github.com/snuffyDev/Beatbump . && \
	npm install && \
	npm run build

FROM node:lts-alpine3.15 as app
COPY --from=build /app /app
WORKDIR /app

ENV VITE_DOMAIN="localhost"
ENV VITE_SITE_URL="http://localhost:9006"

EXPOSE 5000

CMD [ "npm", "run", "dev"]
