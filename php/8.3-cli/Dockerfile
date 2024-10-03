FROM php:8.3.12-cli-alpine3.20 AS base

LABEL maintainer="support@68publishers.io"
WORKDIR /var/www/html

# ENV
ENV COMPOSER_ALLOW_SUPERUSER=1
ENV LD_PRELOAD /usr/lib/preloadable_libiconv.so php

ENV INI_VERIABLES_ORDER "EGPCS"
ENV INI_DATE_TIMEZONE "UTC"
ENV INI_REALPATH_CACHE_SIZE "4096K"
ENV INI_REALPATH_CACHE_TTL "600"
ENV INIT_MEMORY_LIMIT "512M"

RUN set -ex \
    # Build dependencies
    && apk add --no-cache --virtual .build-deps  \
        $PHPIZE_DEPS \
        # intl
    	icu-dev \
    	icu-data-full \
    # Required dependencies
    && apk add --no-cache \
        # iconv
        gnu-libiconv=~1.17 \
        harfbuzz-icu \
        # tools
        git=~2.45 \
        jq=~1.7 \
        curl=~8.10 \
        # psql
        libpq-dev \
        # gd (+ imagick)
        freetype-dev \
        libjpeg-turbo-dev \
        libwebp-dev \
        libpng-dev \
    	libavif-dev \
        # zip
        libzip-dev \
        # ampq
        rabbitmq-c-dev \
    # Redis and Amqp extensions
    && pecl install redis-6.0.2 \
    && pecl install amqp-2.1.2 \
    # Configure & Install PHP extensions
    && docker-php-ext-configure gd --enable-gd --with-freetype --with-jpeg --with-webp --with-avif \
    && docker-php-ext-configure pgsql -with-pgsql=/usr/local/pgsql \
    && docker-php-ext-install \
        -j "$(nproc)" \
        gd \
        pgsql \
        pdo_pgsql \
        intl \
        zip \
        gd \
        exif \
        bcmath \
        opcache \
    && docker-php-ext-enable \
        redis \
        amqp \
    # Delete PHP build dependencies and cache/temp files
    && apk del .build-deps \
    && rm -rf \
        /tmp/pear

# Install Composer
COPY --from=composer:2.7.9 /usr/bin/composer /usr/bin/composer

# PHP custom ini values
COPY php-ini/common "$PHP_INI_DIR/conf.d/"

# Set default shell
SHELL ["/bin/ash", "-eo", "pipefail", "-c"]

# Verify composer
RUN ls -latr /usr/local/bin/ \
    && composer --version

FROM base AS dev

ENV INI_XDEBUG_MODE "develop,debug"
ENV INI_XDEBUG_START_WITH_REQUEST "yes"
ENV INI_XDEBUG_DISCOVER_CLIENT_HOST "0"
ENV INI_XDEBUG_CLIENT_HOST "host.docker.internal"
ENV INI_XDEBUG_OUTPUT_DIR "/tmp/docker-xdebug"
ENV INI_XDEBUG_USE_COMPRESSION "0"
ENV INI_XDEBUG_PROFILER_OUTPUT_NAME "cachegrind.out.%p"
ENV INI_XDEBUG_LOG "/var/log/xdebug.log"

ENV INI_OPCACHE_ENABLE="1"
ENV INI_OPCACHE_MEMORY_CONSUPTION="256"
ENV INI_OPCACHE_MAX_ACCELERATED_FILES="20000"
ENV INI_OPCACHE_VALIDATE_TIMESTAMPS="1"

RUN set -ex \
    # Build dependencies
    && apk add --no-cache --virtual .build-deps \
        linux-headers \
        $PHPIZE_DEPS \
    # Required dependencies
    && apk add --no-cache \
        imagemagick \
        imagemagick-dev \
    # Xdebug and Imagick
    && pecl install xdebug-3.3.2 \
    # Imagick - the imagick version in the pecl registry contains error and fix is still not released. See https://github.com/Imagick/imagick/issues/640
    && git clone https://github.com/Imagick/imagick.git tmp-imagick-repository \
    && cd tmp-imagick-repository \
    && git checkout 28f27044e435a2b203e32675e942eb8de620ee58 \
    && pecl install package.xml \
    && cd .. \
    && rm -rf tmp-imagick-repository \
    # Enable PHP extensions
    && docker-php-ext-enable \
        xdebug \
        imagick \
    # Delete build dependencies
    && apk del .build-deps \
    # Create xdebug log file
    && touch $INI_XDEBUG_LOG \
    && chmod 666 $INI_XDEBUG_LOG

# Copy ini configurations
COPY php-ini/xdebug "$PHP_INI_DIR/conf.d/"
COPY php-ini/opcache "$PHP_INI_DIR/conf.d/"

# Use development ini file
RUN mv "$PHP_INI_DIR/php.ini-development" "$PHP_INI_DIR/php.ini"

USER www-data

FROM base AS prod

ENV INI_OPCACHE_ENABLE="1"
ENV INI_OPCACHE_MEMORY_CONSUPTION="256"
ENV INI_OPCACHE_MAX_ACCELERATED_FILES="20000"
ENV INI_OPCACHE_VALIDATE_TIMESTAMPS="0"

ENV INI_POST_MAX_SIZE="32M"
ENV INI_UPLOAD_MAX_FILESIZE="32M"

# Copy ini configurations
COPY php-ini/opcache "$PHP_INI_DIR/conf.d/"
COPY php-ini/production "$PHP_INI_DIR/conf.d/"

# Use production ini file
RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"

USER www-data

FROM prod AS prod-imagick

USER root

RUN set -ex \
    # Build dependencies
    && apk add --no-cache --virtual .build-deps  \
        $PHPIZE_DEPS \
    # Required dependencies
    && apk add --no-cache \
        imagemagick \
        imagemagick-dev \
    # Imagick - the imagick version in the pecl registry contains error and fix is still not released. See https://github.com/Imagick/imagick/issues/640
    && git clone https://github.com/Imagick/imagick.git tmp-imagick-repository \
    && cd tmp-imagick-repository \
    && git checkout 28f27044e435a2b203e32675e942eb8de620ee58 \
    && pecl install package.xml \
    && cd .. \
    && rm -rf tmp-imagick-repository \
    # Enable PHP extensions
    && docker-php-ext-enable imagick \
    # Delete build dependencies
    && apk del .build-deps

USER www-data
