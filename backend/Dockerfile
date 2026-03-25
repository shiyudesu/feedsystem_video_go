#
# Multi-stage build for backend API and worker.
#
# Build:
#   docker build -f backend/Dockerfile -t feedsystem-backend:api --target api .
#   docker build -f backend/Dockerfile -t feedsystem-backend:worker --target worker .
#

ARG GO_VERSION=1.24.5
ARG GOPROXY=https://goproxy.cn,direct
ARG GOSUMDB=sum.golang.google.cn

FROM golang:${GO_VERSION} AS build
ARG GOPROXY=https://goproxy.cn,direct
ARG GOSUMDB=sum.golang.google.cn
WORKDIR /src/backend
ENV GOPROXY=${GOPROXY}
ENV GOSUMDB=${GOSUMDB}

COPY backend/go.mod backend/go.sum ./
RUN set -eux; \
		go env -w GOPROXY="${GOPROXY}"; \
		go env -w GOSUMDB="${GOSUMDB}"; \
		ok=0; \
		for i in 1 2 3; do \
			if go mod download; then \
				ok=1; \
				break; \
			fi; \
			echo "go mod download failed, retry ${i}/3"; \
			sleep $((i * 2)); \
		done; \
		[ "$ok" -eq 1 ]

COPY backend/ ./

ENV CGO_ENABLED=0
RUN go build -trimpath -ldflags="-s -w" -o /out/api ./cmd
RUN go build -trimpath -ldflags="-s -w" -o /out/worker ./cmd/worker

FROM alpine:3.21 AS base
RUN apk add --no-cache ca-certificates tzdata && adduser -D -H -s /sbin/nologin app
WORKDIR /app
COPY --from=build /src/backend/configs ./configs
RUN mkdir -p ./.run/uploads && chown -R app:app /app
USER app

FROM base AS api
COPY --from=build /out/api /app/api
EXPOSE 8080
ENTRYPOINT ["/app/api"]

FROM base AS worker
COPY --from=build /out/worker /app/worker
ENTRYPOINT ["/app/worker"]

