FROM php:8.1.21-zts-alpine3.18 as base

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

ARG unit_version="1.30.0"

# Fix iconv issue (https://github.com/docker-library/php/issues/240)
RUN set -ex \
    # Build dependencies
    && apk add --no-cache --virtual .build-deps  \
        $PHPIZE_DEPS \
        # intl
    	icu-dev \
    	icu-data-full \
        # nginx unit
        openssl-dev \
        musl-dev \
        linux-headers \
    # Required dependencies
    && apk add --no-cache \
        # iconv
        gnu-libiconv=~1.17 \
        harfbuzz-icu \
        # tools
        git=~2.40 \
        jq=~1.6 \
        yarn=~1.22 \
        curl=~8.1 \
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
        # nginx unit
        pcre-dev \
    # Nginx unit build
    && NCPU="$(getconf _NPROCESSORS_ONLN)" \
    && export UNITTMP=$(mktemp -d -p /tmp -t unit.XXXXXX) \
    && cd $UNITTMP \
    && curl -O "https://unit.nginx.org/download/unit-$unit_version.tar.gz" \
    && tar xzf unit-$unit_version.tar.gz \
    && cd unit-$unit_version \
    && echo '*self_spec:' > /tmp/no-pie-compile.specs \
    && echo '+ %{!r:%{!fpie:%{!fPIE:%{!fpic:%{!fPIC:%{!fno-pic:-fno-PIE}}}}}}' >> /tmp/no-pie-compile.specs \
    && ./configure \
        --modules=/usr/lib/unit/modules \
        --control="unix:/var/run/unit/control.unit.sock" \
        --openssl \
        --state=/var/lib/unit \
        --pid=/var/run/unit/unit.pid \
        --log=/var/log/unit.log \
        --tmp=/var/tmp \
        --user=unit \
        --group=unit \
        --cc-opt='-g -O2 -flto=auto -ffat-lto-objects -specs=/tmp/no-pie-compile.specs -fstack-protector-strong -Wformat -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -fPIC' \
    && ./configure php \
        --module=php \
        --lib-path=/usr/local/bin/php \
    && make -j $NCPU unitd \
    && install -pm755 build/sbin/unitd /usr/sbin/unitd \
    && make -j $NCPU php-install \
    && ldconfig / \
    && addgroup \
        --gid 1000 \
        --system \
        unit \
    && adduser \
        --uid 1000 \
        --system \
        --ingroup unit \
        --disabled-password \
         --no-create-home \
        --home /nonexistent \
        --shell /bin/false \
        unit \
    && mkdir /docker-entrypoint.d/ \
    && mkdir -p /var/lib/unit \
    && mkdir -p /usr/state \
    && mkdir -p /var/run/unit \
    && chown www-data:www-data /var/lib/unit \
    && chown www-data:www-data /usr/state \
    && chown www-data:www-data /var/run/unit \
    && ln -sf /dev/stdout /var/log/unit.log \
    && ln -sf /dev/stdout /var/log/access.log \
    # Redis and Amqp extensions
    && pecl install redis-5.3.7 \
    && pecl install amqp-1.11.0 \
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
        /tmp/pear \
        /tmp/no-pie-compile.specs \
        $UNITTMP

# Install Composer
COPY --from=composer:2.5.8 /usr/bin/composer /usr/bin/composer

# PHP custom ini values
COPY php-ini/common "$PHP_INI_DIR/conf.d/"

# Set default shell
SHELL ["/bin/ash", "-eo", "pipefail", "-c"]

# Verify composer and yarn
RUN ls -latr /usr/local/bin/ \
    && composer --version \
    && yarn --version

# Copy Unit entrypoint and the default webserver configuration
COPY unit/docker-entrypoint.sh /usr/local/bin/
COPY unit/settings.json /docker-entrypoint.d/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

STOPSIGNAL SIGTERM

ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
EXPOSE 8080
CMD ["unitd", "--no-daemon", "--control", "unix:/var/run/unit/control.unit.sock"]

FROM base as dev

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
    && pecl install xdebug-3.2.1 \
    && pecl install imagick-3.7.0 \
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

FROM base as prod

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

FROM prod as prod-imagick

USER root

RUN set -ex \
    # Build dependencies
    && apk add --no-cache --virtual .build-deps  \
        $PHPIZE_DEPS \
    # Required dependencies
    && apk add --no-cache \
        imagemagick \
        imagemagick-dev \
    # Imagick
    && pecl install imagick-3.7.0 \
    # Enable PHP extensions
    && docker-php-ext-enable imagick \
    # Delete build dependencies
    && apk del .build-deps

USER www-data
