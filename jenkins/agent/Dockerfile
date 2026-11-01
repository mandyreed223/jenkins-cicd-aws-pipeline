FROM jenkins/inbound-agent:latest-jdk21

USER root

ARG DOCKER_GID=1001

# Install Docker CLI and supporting tools
RUN apt-get update && \
    apt-get install -y \
        docker.io \
        git \
        curl \
        unzip \
        less && \
    rm -rf /var/lib/apt/lists/*

# Install AWS CLI v2
RUN curl -fsSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "/tmp/awscliv2.zip" && \
    unzip -q /tmp/awscliv2.zip -d /tmp && \
    /tmp/aws/install && \
    rm -rf /tmp/aws /tmp/awscliv2.zip

# Give Jenkins access to the host Docker socket
RUN groupadd -g ${DOCKER_GID} docker-host && \
    usermod -aG docker-host jenkins

USER jenkins