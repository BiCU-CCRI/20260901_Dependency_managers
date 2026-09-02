FROM mcr.microsoft.com/devcontainers/universal:6-noble

USER root

RUN apt-get update \
    && apt-get install --no-install-recommends -y \
        gfortran \
        libblas-dev \
        liblapack-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

USER vscode
