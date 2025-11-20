FROM registry.access.redhat.com/ubi10/nginx-126:10.0

# Use root while we change files and permissions
USER 0

# Make sure log and cache dirs exist and are writable
RUN mkdir -p /var/log/nginx /var/cache/nginx \
    && chown -R 1001:0 /var/log/nginx /var/cache/nginx

# Copy our app files
COPY index.html /opt/app-root/src/index.html
COPY nginx.conf /etc/nginx/nginx.conf

# Drop back to non-root for running
USER 1001

# Important: actually start nginx, not the S2I helper message
CMD ["nginx", "-g", "daemon off;"]
