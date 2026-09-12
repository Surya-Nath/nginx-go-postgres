FROM golang:1.22-alpine AS builder
WORKDIR /code
ENV CGO_ENABLED=0
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o /backend main.go

FROM scratch
COPY --from=builder /backend /backend
CMD ["/backend"]
