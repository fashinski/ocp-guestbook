# Build stage
FROM registry.access.redhat.com/ubi10/go-toolset:10.0 AS builder

WORKDIR /app

# Copy module files first (better caching)
COPY go.mod ./
# If you have go.sum in backend, uncomment the next line:
# COPY go.sum ./
RUN go mod download

# Copy the rest of the source code
COPY . .

# Build the binary
RUN go build -o /tmp/guestbook-api .

# Runtime stage
FROM registry.access.redhat.com/ubi10-minimal:10.0

# Copy compiled binary from builder stage
COPY --from=builder /tmp/guestbook-api /guestbook-api

ENV PORT=8080
EXPOSE 8080

CMD ["/guestbook-api"]
