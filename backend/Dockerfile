# syntax=docker/dockerfile:1

FROM golang:1.17-alpine AS builder

WORKDIR /src

RUN apk add --no-cache ca-certificates git \
    && addgroup -g 1000 app \
    && adduser -D -u 1000 -G app app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -trimpath -ldflags="-s -w" \
    -o /out/api ./cmd/api

FROM alpine:3.19

RUN apk add --no-cache ca-certificates wget \
    && addgroup -g 1000 app \
    && adduser -D -u 1000 -G app app \
    && rm -rf /var/cache/apk/*

WORKDIR /app

COPY --from=builder --chown=app:app /out/api ./api

USER app

EXPOSE 8081

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD wget -qO- http://127.0.0.1:8081/health || exit 1

CMD ["./api"]
