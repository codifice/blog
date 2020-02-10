FROM debian:9

# Avoid warnings by switching to noninteractive
ENV DEBIAN_FRONTEND=noninteractive
ENV HUGO_VERSION="0.61.0"

# This Dockerfile adds a non-root 'vscode' user with sudo access. However, for Linux,
# this user's GID/UID must match your local user UID/GID to avoid permission issues
# with bind mounts. Update USER_UID / USER_GID if yours is not 1000. See
# https://aka.ms/vscode-remote/containers/non-root-user for details.
ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=$USER_UID

# Configure apt and install packages
RUN apt-get update \
    #
    # Configure apt
    && apt-get -y install --no-install-recommends apt-utils dialog wget 2>&1 \
    #
    # Verify git and needed tools are installed
    && apt-get -y install git iproute2 procps lsb-release \
    #
    # Create a non-root user to use if preferred - see https://aka.ms/vscode-remote/containers/non-root-user.
    && groupadd --gid $USER_GID $USERNAME \
    && useradd -s /bin/bash --uid $USER_UID --gid $USER_GID -m $USERNAME \
    # [Optional] Add sudo support for the non-root user
    && apt-get install -y sudo \
    && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME\
    && chmod 0440 /etc/sudoers.d/$USERNAME \
    #
    # Clean up
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*

# Download and install hugo
RUN cd /tmp \
    && wget https://github.com/gohugoio/hugo/releases/download/v"$HUGO_VERSION"/hugo_"$HUGO_VERSION"_Linux-64bit.deb \
    && wget https://github.com/gohugoio/hugo/releases/download/v"$HUGO_VERSION"/hugo_"$HUGO_VERSION"_checksums.txt \
    && grep " hugo_${HUGO_VERSION}_Linux-64bit.deb\$" hugo_${HUGO_VERSION}_checksums.txt | sha256sum -c - \
    && dpkg -i hugo_"$HUGO_VERSION"_Linux-64bit.deb \
    && rm /tmp/hugo*

# Switch back to dialog for any ad-hoc use of apt-get
ENV DEBIAN_FRONTEND=dialog