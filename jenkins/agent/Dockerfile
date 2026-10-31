FROM jenkins/inbound-agent:latest-jdk21

USER root

ARG DOCKER_GID=1001

RUN apt-get update && \
    apt-get install -y \
        docker.io \
        git \
        curl \
        unzip \
        less && \
    rm -rf /var/lib/apt/lists/*

RUN groupadd -g ${DOCKER_GID} docker-host && \
    usermod -aG docker-host jenkins

USER jenkins