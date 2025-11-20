FROM registry.access.redhat.com/ubi10/nginx-126:10.0

# Use root during build so we can write into nginx dirs
USER 0

# Copy our app files into the image
COPY index.html /opt/app-root/src/index.html
COPY nginx.conf /etc/nginx/nginx.conf

# Drop back to non-root for running
USER 1001
