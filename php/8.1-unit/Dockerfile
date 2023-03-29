FROM alpine:3.17.1
LABEL maintainer="support@68publishers.io"
WORKDIR /var/www/html

# Base PHP Args
ARG date_timezone="UTC"
ARG memory_limit="512M"
ARG opcache_memory_consumption=256
ARG opcache_max_accelerated_files=20000
ARG opcache_validate_timestamps=1
ARG realpath_cache_size="4096K"
ARG realpath_cache_ttl=600
ARG variables_order="EGPCS"

# Xdebug Args & Envs
ARG xdebug=0
ARG xdebug_version="~3.2"
ARG xdebug_mode="develop,debug"
ARG xdebug_output_dir="/tmp/docker-xdebug"
ARG xdebug_use_compression=0
ARG xdebug_profiler_output_name="cachegrind.out.%p"

# Imagick
ARG imagick=0
ARG imagick_version="~3.7"

# Redis
ARG redis_version="~5.3.7"

# Envs
ENV PHP_VERSION 8.1
ENV NGINX_UNIT_VERSION 1.28

# Run bin/console cache:clear before image start
ENV CACHE_CLEAR_BOFORE_START false

# Fix iconv issue (https://github.com/docker-library/php/issues/240)
RUN apk add --no-cache --update gnu-libiconv=~1.17
ENV LD_PRELOAD /usr/lib/preloadable_libiconv.so php

RUN apk add -U harfbuzz-icu

# PHP & extensions
RUN set -ex \
    && apk add --no-cache --update \
        git=~2.38 \
        jq=~1.6 \
        php81=~8.1 \
        php81-intl=~8.1 \
        php81-zip=~8.1 \
        php81-exif=~8.1 \
        php81-fileinfo=~8.1 \
        php81-pdo_pgsql=~8.1 \
        php81-pdo_sqlite=~8.1 \
        php81-sqlite3=~8.1 \
        php81-session=~8.1 \
        php81-curl=~8.1 \
        php81-dom=~8.1 \
        php81-gd=~8.1 \
        php81-ctype=~8.1 \
        php81-iconv=~8.1 \
        php81-tokenizer=~8.1 \
        php81-xml=~8.1 \
        php81-simplexml=~8.1 \
        php81-xmlwriter=~8.1 \
        php81-xmlreader=~8.1 \
        php81-opcache=~8.1 \
        php81-bcmath=~8.1 \
        php81-pecl-amqp=~1.11 \
        php81-pecl-redis=$redis_version

# PHP .ini setup
RUN set -ex \
    && echo "date.timezone = $date_timezone" >> /etc/php81/conf.d/docker-timezone.ini \
    && echo "memory_limit = $memory_limit" >> /etc/php81/conf.d/docker-memory-limit-prod-only.ini \
    && echo "opcache.memory_consumption = $opcache_memory_consumption" >> /etc/php81/conf.d/docker-opcache.ini \
    && echo "opcache.max_accelerated_files = $opcache_max_accelerated_files" >> /etc/php81/conf.d/docker-opcache.ini \
    && echo "opcache.validate_timestamps = $opcache_validate_timestamps" >> /etc/php81/conf.d/docker-opcache.ini \
    && echo "realpath_cache_size = $realpath_cache_size" >> /etc/php81/conf.d/docker-realpath.ini \
    && echo "realpath_cache_ttl = $realpath_cache_ttl" >> /etc/php81/conf.d/docker-realpath.ini \
    && echo "variables_order = $variables_order" >> /etc/php81/conf.d/docker-env.ini

# Xdebug
RUN set -ex; \
    if [ "1" == "$xdebug" ]; then \
        apk add --no-cache --update php81-pecl-xdebug=$xdebug_version; \
        echo "zend_extension=xdebug.so" >> /etc/php81/conf.d/docker-xdebug.ini; \
        echo "xdebug.mode=$xdebug_mode" >> /etc/php81/conf.d/docker-xdebug.ini; \
        echo "xdebug.start_with_request=yes" >> /etc/php81/conf.d/docker-xdebug.ini; \
        echo "xdebug.discover_client_host=0" >> /etc/php81/conf.d/docker-xdebug.ini; \
        echo "xdebug.client_host=host.docker.internal" >> /etc/php81/conf.d/docker-xdebug.ini; \
        echo "xdebug.output_dir=$xdebug_output_dir" >> /etc/php81/conf.d/docker-xdebug.ini; \
        echo "xdebug.use_compression=$xdebug_use_compression" >> /etc/php81/conf.d/docker-xdebug.ini; \
        echo "xdebug.profiler_output_name=$xdebug_profiler_output_name" >> /etc/php81/conf.d/docker-xdebug.ini; \
    fi

# Imagick
RUN set -ex; \
    if [ "1" == "$imagick" ]; then \
        apk add --no-cache --update php81-pecl-imagick=$imagick_version; \
        apk add --no-cache ${PHPIZE_DEPS}; \
        apk add --no-cache imagemagick imagemagick-dev; \
        apk del --no-cache ${PHPIZE_DEPS}; \
    fi

# Composer
ENV COMPOSER_ALLOW_SUPERUSER=1
RUN set -ex \
    && apk add --no-cache --update composer=~2.4.4 \
    # && composer global require hirak/prestissimo \
    && rm -rf ~/.composer/cache/*

# Yarn
RUN set -ex \
    && apk add --no-cache --update \
        yarn=~1.22

# NGINX Unit
RUN set -ex \
    && apk add --no-cache --update \
        curl=~7.88 \
        unit=~1.29 \
        unit-php81=~1.29 \
    && mkdir -m777 -p /usr/tmp

COPY config/ /docker-entrypoint.d/
COPY run-nginx-unit /usr/local/bin/

RUN chmod +x /usr/local/bin/run-nginx-unit

SHELL ["/bin/ash", "-eo", "pipefail", "-c"]

RUN ls -latr /usr/local/bin/
RUN composer --version
RUN yarn --version

CMD ["run-nginx-unit"]
