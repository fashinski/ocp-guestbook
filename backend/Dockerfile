# Build stage
FROM registry.access.redhat.com/ubi10/go-toolset:10.0 AS builder

# Use the standard writable workdir for UBI images
WORKDIR /opt/app-root/src

# Copy module files first (for better caching)
COPY go.mod ./
# If you have go.sum in the backend directory, uncomment this:
# COPY go.sum ./
RUN go mod download

# Copy the rest of the source code
COPY . .

# Build the binary
RUN go build -o /tmp/guestbook-api .

# Runtime stage
FROM registry.access.redhat.com/ubi10-minimal:10.0

COPY --from=builder /tmp/guestbook-api /guestbook-api

ENV PORT=8080
EXPOSE 8080

CMD ["/guestbook-api"]
