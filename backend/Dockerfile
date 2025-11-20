# Build stage
FROM registry.access.redhat.com/ubi10/go-toolset:10.0 AS builder

# Run as root in the builder so we can modify go.mod/go.sum
USER 0

WORKDIR /opt/app-root/src

# Copy all backend source code
COPY . .

# Resolve and download dependencies, this may update go.mod and go.sum
RUN go mod tidy

# Build the binary
RUN go build -o /tmp/guestbook-api .

# Runtime stage
FROM registry.access.redhat.com/ubi10-minimal:10.0

COPY --from=builder /tmp/guestbook-api /guestbook-api

ENV PORT=8080
EXPOSE 8080

CMD ["/guestbook-api"]
