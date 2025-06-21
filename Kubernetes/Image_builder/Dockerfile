FROM alpine:3.18

LABEL maintainer="sayantan2k21"

# Install Docker and essential tools
RUN apk add --no-cache \
    docker \
    bash \
    sudo \
    git \
    curl \
    procps \
    openrc \
    shadow

# Create docker storage directory (for DinD)
RUN mkdir -p /var/lib/docker

# Create non-root user and give docker permissions
RUN adduser -D -s /bin/bash dockeruser && \
    echo "dockeruser ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers && \
    usermod -aG docker dockeruser

# Create dockerd-entrypoint.sh inside Docker image
RUN echo '#!/bin/bash' > /usr/local/bin/dockerd-entrypoint.sh && \
    echo 'echo "▶️ Starting Docker daemon..."' >> /usr/local/bin/dockerd-entrypoint.sh && \
    echo 'sudo dockerd --host=unix:///var/run/docker.sock &' >> /usr/local/bin/dockerd-entrypoint.sh && \
    echo 'sleep 5' >> /usr/local/bin/dockerd-entrypoint.sh && \
    echo 'echo "✅ Docker daemon started."' >> /usr/local/bin/dockerd-entrypoint.sh && \
    echo 'tail -f /dev/null' >> /usr/local/bin/dockerd-entrypoint.sh && \
    chmod +x /usr/local/bin/dockerd-entrypoint.sh

USER dockeruser
WORKDIR /home/dockeruser


# optional
EXPOSE 2375 


ENTRYPOINT [] 

# process will be run by jenkins and kubernetes plugin
