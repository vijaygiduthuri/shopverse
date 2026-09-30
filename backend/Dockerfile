FROM golang:1.24-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN go mod tidy && CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o /shopverse-api ./cmd/main.go

FROM gcr.io/distroless/static-debian12

COPY --from=builder /shopverse-api /shopverse-api

USER nonroot:nonroot

EXPOSE 8080

ENTRYPOINT ["/shopverse-api"]
