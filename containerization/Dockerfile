# ─────────────────────────────────────────────────────────────────────────────
# Multi-stage Dockerfile — Laravel 11 / PHP 8.3
#
# Design decisions documented inline. This Dockerfile is not "just a Dockerfile"
# — it is a contract that defines runtime behavior across ALL environments.
#
# Stages:
#   1. deps        — Install PHP and Node dependencies (with lock files)
#   2. assets      — Build frontend assets (Node/Vite)
#   3. production  — Minimal runtime image (no dev dependencies, no build tools)
# ─────────────────────────────────────────────────────────────────────────────

# ── Stage 1: PHP Dependencies ─────────────────────────────────────────────
FROM php:8.3-fpm-alpine3.19 AS deps

# Pin Alpine version to 3.19 explicitly — "latest" in base images is a
# reliability anti-pattern. When Alpine releases 3.20, this image must be
# explicitly upgraded, forcing a conscious review of any breaking changes.

# Install system dependencies required for PHP extensions.
# Each package is justified:
#   - postgresql-dev: required for pdo_pgsql
#   - $PHPIZE_DEPS: required only during extension compilation, removed after
#   - curl, unzip: required by Composer
RUN apk add --no-cache \
    postgresql-dev \
    curl \
    unzip \
    && apk add --no-cache --virtual .build-deps \
        $PHPIZE_DEPS \
    && docker-php-ext-install \
        pdo \
        pdo_pgsql \
        opcache \
        pcntl \   
    && pecl install redis-6.0.2 \
    && docker-php-ext-enable redis \
    && apk del .build-deps \
    # Remove PECL channel cache — reduces image size, not needed at runtime
    && rm -rf /tmp/pear

# Pin Redis extension version. Unpinned pecl install redis would install
# whatever is latest at build time — creating non-reproducible builds.

# Install Composer — pin the digest, not just the version tag.
# Version tags on Docker Hub are mutable. Digest-pinning ensures this
# COPY --from references the exact binary we tested against.
COPY --from=composer:2.7 /usr/bin/composer /usr/bin/composer

WORKDIR /app

# Copy dependency manifest files FIRST, before application code.
# Docker layer cache: if composer.json/lock haven't changed, this layer
# (and the expensive `composer install`) is served from cache even when
# application code changes. This is the single biggest build-time optimization.
COPY composer.json composer.lock ./

RUN composer install \
    --no-dev \
    --no-scripts \
    --no-autoloader \
    --no-interaction \
    --prefer-dist \
    # --no-scripts: prevents composer scripts from running during dependency
    # installation. Scripts run against application code that doesn't exist yet.
    # --no-autoloader: autoloader is generated in the next step, after all
    # application code is present.
    && composer clear-cache

# ── Stage 2: Frontend Assets ───────────────────────────────────────────────
FROM node:20-alpine3.19 AS assets

# Node is ONLY present in this stage. The final production image has no Node
# runtime — reducing attack surface and image size significantly.

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --only=production
# npm ci (not npm install): uses lockfile strictly, fails if lockfile is
# out of sync with package.json. Ensures reproducible builds.

COPY . .
RUN npm run build && \
    # Verify build output exists before proceeding — fail fast
    test -d public/build || (echo "Frontend build output missing" && exit 1)

# ── Stage 3: Production Runtime ────────────────────────────────────────────
FROM php:8.3-fpm-alpine3.19 AS production

# Install only runtime dependencies — no build tools, no compilers.
RUN apk add --no-cache \
    postgresql-libs \
    nginx \
    supervisor \
    curl \
    # curl: required only for health checks and internal service calls
    && rm -rf /var/cache/apk/*

# Copy PHP extensions compiled in stage 1
COPY --from=deps /usr/local/lib/php/extensions/ /usr/local/lib/php/extensions/
COPY --from=deps /usr/local/etc/php/conf.d/ /usr/local/etc/php/conf.d/

# ── Application user setup ────────────────────────────────────────────────
# Create a dedicated non-root application user.
# UID/GID 1000 is chosen to align with most CI runner user IDs.
# The key constraint: this user has NO sudo, NO write access to /etc,
# and write access ONLY to /var/www/html/storage and /var/www/html/bootstrap/cache.
RUN addgroup -g 1000 appgroup && \
    adduser -u 1000 -G appgroup -s /sbin/nologin -D appuser

WORKDIR /var/www/html

# Copy PHP configuration
COPY docker/php/php.ini /usr/local/etc/php/conf.d/app.ini
COPY docker/php/php-fpm.conf /usr/local/etc/php-fpm.d/www.conf
COPY docker/nginx/nginx.conf /etc/nginx/nginx.conf
COPY docker/supervisor/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# Copy application code
COPY --chown=appuser:appgroup . .

# Copy dependencies from build stages
COPY --from=deps --chown=appuser:appgroup /app/vendor ./vendor
COPY --from=assets --chown=appuser:appgroup /app/public/build ./public/build

# Generate optimized autoloader NOW — with all application code present.
# Class map generation requires the full codebase.
RUN composer dump-autoload --classmap-authoritative --no-dev

# PHP optimization for production
RUN php artisan config:cache && \
    php artisan route:cache && \
    php artisan view:cache && \
    php artisan event:cache
# These caches MUST be generated at image build time, not container startup.
# Generating them at startup requires write access to bootstrap/cache during
# normal runtime — a security and reliability anti-pattern.

# Create writable directories with minimal permissions
# Storage: writable by appuser only; not world-writable
RUN mkdir -p storage/framework/{sessions,views,cache} \
             storage/logs \
             bootstrap/cache \
    && chown -R appuser:appgroup storage bootstrap/cache \
    && chmod -R 755 storage bootstrap/cache \
    && chmod -R 644 storage/logs

# Lock down application code: read-only for appuser
# The application should never need to write to its own codebase at runtime.
RUN find /var/www/html -type f -not -path "*/storage/*" -not -path "*/bootstrap/cache/*" \
    -exec chmod 444 {} \; && \
    find /var/www/html -type d -not -path "*/storage/*" -not -path "*/bootstrap/cache/*" \
    -exec chmod 555 {} \;

# ── Build metadata as OCI labels ─────────────────────────────────────────
ARG BUILD_DATE
ARG GIT_SHA
ARG GIT_REF
LABEL org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.revision="${GIT_SHA}" \
      org.opencontainers.image.ref.name="${GIT_REF}" \
      org.opencontainers.image.title="myapp-api" \
      org.opencontainers.image.vendor="MyCompany"

# Health check — validates the application is genuinely serving traffic,
# not just that PHP-FPM process is alive.
HEALTHCHECK --interval=15s --timeout=5s --start-period=30s --retries=3 \
    CMD curl -f http://localhost/health || exit 1
# start-period=30s: gives OPcache and application bootstrapping time.
# Failing to set start-period causes unnecessary restarts during cold start.

# Switch to non-root user — all subsequent operations run as appuser
USER appuser

EXPOSE 80

# Entrypoint: supervisord manages both nginx and php-fpm processes.
# NOT using separate nginx and php-fpm containers in this configuration
# because inter-container communication over unix socket vs TCP adds
# operational complexity without meaningful benefit at this scale.
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
