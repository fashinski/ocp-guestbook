# Build stage
FROM registry.access.redhat.com/ubi10/go-toolset:10.0 AS builder

# Use a writable workdir
WORKDIR /opt/app-root/src

# Copy all backend source code into the image
COPY . .

# Let Go resolve and download all dependencies based on the source imports
RUN go mod tidy

# Build the binary
RUN go build -o /tmp/guestbook-api .

# Runtime stage
FROM registry.access.redhat.com/ubi10-minimal:10.0

COPY --from=builder /tmp/guestbook-api /guestbook-api

ENV PORT=8080
EXPOSE 8080

CMD ["/guestbook-api"]
