# Production image built from the sources in this directory.
#
# Same layout as the official image (https://github.com/alexjustesen/docker-speedtest-tracker),
# but it copies the local sources instead of downloading a release, and builds amd64 and arm64
# from one file. Build from the project root:
#
#   docker build -f docker/production/Dockerfile .

#############################
# Install Composer dependencies (platform-independent)
#############################
FROM --platform=$BUILDPLATFORM serversideup/php:8.5-fpm-nginx-alpine-v4.5.0 AS dependencies

USER root

# Install the intl extension required by Composer dependencies
RUN install-php-extensions intl \
    && rm -rf /tmp/*

WORKDIR /app

COPY composer.json composer.lock /app/

RUN --mount=type=cache,target=/tmp/cache,uid=0,gid=0 \
    COMPOSER_CACHE_DIR=/tmp/cache COMPOSER_ALLOW_SUPERUSER=1 composer install --no-interaction --prefer-dist --optimize-autoloader --no-dev --no-scripts

COPY . /app

RUN --mount=type=cache,target=/tmp/cache,uid=0,gid=0 \
    COMPOSER_CACHE_DIR=/tmp/cache COMPOSER_ALLOW_SUPERUSER=1 composer install --no-interaction --prefer-dist --optimize-autoloader --no-dev \
    && chown -R www-data:www-data /app

USER www-data

#############################
# Build assets (platform-independent)
#############################
FROM --platform=$BUILDPLATFORM node:26-alpine AS assets

WORKDIR /app

COPY --from=dependencies /app /app

RUN --mount=type=cache,target=/root/.npm \
    npm ci && npm run build

#############################
# Production image
#############################
FROM serversideup/php:8.5-fpm-nginx-alpine-v4.5.0 AS production

ARG TARGETARCH

ARG LIBRESPEED_CLI_VERSION="1.0.12" \
    OOKLA_CLI_VERSION="1.2.0"

ENV AUTORUN_ENABLED="true" \
    PHP_OPCACHE_ENABLE="1" \
    SHOW_WELCOME_MESSAGE="false"

# Switch to root so we can do root things
USER root

# Install system dependencies, iperf3 is used by the iPerf3 speedtest service
RUN apk add --no-cache jq iperf3 iputils \
    && rm -rf /var/cache/apk/*

# Install CLI tools for the target architecture
RUN set -e; \
    case "${TARGETARCH}" in \
        amd64) LIBRESPEED_ARCH="amd64"; OOKLA_ARCH="x86_64" ;; \
        arm64) LIBRESPEED_ARCH="arm64"; OOKLA_ARCH="aarch64" ;; \
        *) echo "Unsupported architecture: ${TARGETARCH}"; exit 1 ;; \
    esac; \
    curl -o /tmp/librespeed-cli.tgz -L \
        "https://github.com/librespeed/speedtest-cli/releases/download/v${LIBRESPEED_CLI_VERSION}/librespeed-cli_${LIBRESPEED_CLI_VERSION}_linux_${LIBRESPEED_ARCH}.tar.gz" \
    && curl -o /tmp/speedtest-cli.tgz -L \
        "https://install.speedtest.net/app/cli/ookla-speedtest-${OOKLA_CLI_VERSION}-linux-${OOKLA_ARCH}.tgz" \
    && tar xzf /tmp/librespeed-cli.tgz -C /usr/bin \
    && tar xzf /tmp/speedtest-cli.tgz -C /usr/bin \
    && rm /tmp/librespeed-cli.tgz /tmp/speedtest-cli.tgz

# Install the intl extension with root permissions
RUN install-php-extensions intl \
    && rm -rf /tmp/*

# Copy s6 service definitions for Laravel scheduler and queue worker
COPY --chmod=755 docker/production/root /

# Drop back to our unprivileged user
USER www-data

# Set working directory
WORKDIR /var/www/html

# Copy source code and Composer dependencies from the dependencies stage
COPY --chown=www-data:www-data --from=dependencies /app /var/www/html

COPY --chown=www-data:www-data --from=assets /app/public/build /var/www/html/public/build
