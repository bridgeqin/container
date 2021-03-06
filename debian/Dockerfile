FROM node:12.19.1-slim@sha256:69496565e889a612b11212889558048a68b899a40a35cf034c1737541646ef5c

# Build arguments to change source url, branch or tag
ARG HEDGEDOC_REPOSITORY=https://github.com/hedgedoc/hedgedoc.git
ARG VERSION=master
ARG UID=10000

ARG CODIMD_REPOSITORY
RUN if [ -n "${CODIMD_REPOSITORY}" ]; then echo "CODIMD_REPOSITORY is deprecated. Please use HEDGEDOC_REPOSITORY instead" && exit 1; fi

# Set some default config variables
ARG DEBIAN_FRONTEND=noninteractive
ENV DOCKERIZE_VERSION=v0.6.1
ENV NODE_ENV=production

RUN apt-get update && \
    apt-get install --no-install-recommends -y \
      ca-certificates \
      wget && \
    rm -r /var/lib/apt/lists/*

RUN wget https://github.com/jwilder/dockerize/releases/download/$DOCKERIZE_VERSION/dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz && \
    tar -C /usr/local/bin -xzvf dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz && \
    rm dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz

# Add configuraton files
COPY ["resources/config.json", "resources/.sequelizerc", "/files/"]

SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN export dev_apt=( \
      "bzip2" \
      "git" \
      "jq" \
    ) && \
    apt-get update && \
    apt-get install --no-install-recommends -y \
      "${dev_apt[@]}" \
      # Add fonts for PDF export
      fonts-noto \
      gosu && \
    \
    # Clone the source
    git clone --depth 1 --branch "$VERSION" "$HEDGEDOC_REPOSITORY" /hedgedoc && \
    ln -s /hedgedoc /codimd && \
    # Print the cloned version and clean up git files
    cd /hedgedoc && \
    git log --pretty=format:'%ad %h %d' --abbrev-commit --date=short -1 && echo && \
    git rev-parse HEAD > /tmp/gitref && \
    rm -rf /hedgedoc/.git && \
    \
    # Mime the git repository for fullversion
    mkdir /hedgedoc/.git && \
    mv /tmp/gitref /hedgedoc/.git/HEAD && \
    jq ".repository.url = \"${HEDGEDOC_REPOSITORY}\"" /hedgedoc/package.json > /hedgedoc/package.new.json && \
    mv /hedgedoc/package.new.json /hedgedoc/package.json && \
    \
    # Symlink configuration files
    rm -f /hedgedoc/config.json && \
    ln -s /files/config.json /hedgedoc/config.json && \
    rm -f /hedgedoc/.sequelizerc && \
    ln -s /files/.sequelizerc /hedgedoc/.sequelizerc && \
    \
    # Install NPM dependencies and build project
    yarn install --pure-lockfile && \
    yarn install --production=false --pure-lockfile && \
    npm run build && \
    \
    # Clean up this layer
    yarn install && \
    yarn cache clean && \
    apt-get autoremove --purge -qy "${dev_apt[@]}" && \
    rm -r /var/lib/apt/lists/* && \
    # Create hedgedoc user
    adduser --uid $UID --home /hedgedoc/ --disabled-password --system hedgedoc && \
    chown -R hedgedoc /hedgedoc/

WORKDIR /hedgedoc
EXPOSE 3000

COPY ["resources/docker-entrypoint.sh", "/usr/local/bin/docker-entrypoint.sh"]

ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]

CMD ["npm", "start"]
