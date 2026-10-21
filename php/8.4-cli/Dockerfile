# Vendored from docker-library/php@f6048b5 (8.4/alpine3.23/cli): PHP 8.4.27 security release (CVE-2026-109618, CVE-2026-109619)
# was not yet published on Docker Hub. Replace with 'FROM php:8.4.27-cli-alpine3.23 AS base' once it is.
#
# NOTE: THIS DOCKERFILE IS GENERATED VIA "apply-templates.sh"
#
# PLEASE DO NOT EDIT IT DIRECTLY.
#

FROM alpine:3.23 AS php-src

# dependencies required for running "phpize"
# these get automatically installed and removed by "docker-php-ext-*" (unless they're already installed)
ENV PHPIZE_DEPS \
		autoconf \
		dpkg-dev dpkg \
		file \
		g++ \
		gcc \
		libc-dev \
		make \
		pkgconf \
		re2c

# persistent / runtime deps
RUN apk add --no-cache \
		ca-certificates \
		curl \
		openssl \
		tar \
		xz

# ensure www-data user exists
RUN set -eux; \
	adduser -u 82 -D -S -G www-data www-data
# 82 is the standard uid/gid for "www-data" in Alpine
# https://git.alpinelinux.org/aports/tree/main/apache2/apache2.pre-install?h=3.14-stable
# https://git.alpinelinux.org/aports/tree/main/lighttpd/lighttpd.pre-install?h=3.14-stable
# https://git.alpinelinux.org/aports/tree/main/nginx/nginx.pre-install?h=3.14-stable

ENV PHP_INI_DIR /usr/local/etc/php
RUN set -eux; \
	mkdir -p "$PHP_INI_DIR/conf.d"; \
# allow running as an arbitrary user (https://github.com/docker-library/php/issues/743)
	[ ! -d /var/www/html ]; \
	mkdir -p /var/www/html; \
	chown www-data:www-data /var/www/html; \
	chmod 1777 /var/www/html

# Apply stack smash protection to functions using local buffers and alloca()
# Make PHP's main executable position-independent (improves ASLR security mechanism, and has no performance impact on x86_64)
# Enable optimization (-O2)
# Enable linker optimization (this sorts the hash buckets to improve cache locality, and is non-default)
# https://github.com/docker-library/php/issues/272
# -D_LARGEFILE_SOURCE and -D_FILE_OFFSET_BITS=64 (https://www.php.net/manual/en/intro.filesystem.php)
ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64"
ENV PHP_CPPFLAGS="$PHP_CFLAGS"
ENV PHP_LDFLAGS="-Wl,-O1 -pie"

ENV GPG_KEYS AFD8691FDAEDF03BDF6E460563F15A9B715376CA 9D7F99A0CB8F05C8A6958D6256A97AF7600A39A6 0616E93D95AF471243E26761770426E17EBBB3DD

ENV PHP_VERSION 8.4.27
ENV PHP_URL="https://www.php.net/distributions/php-8.4.27.tar.xz" PHP_ASC_URL="https://www.php.net/distributions/php-8.4.27.tar.xz.asc"
ENV PHP_SHA256="dcf93923b0e38e0d5252b25b796520b5a47761a08b3488b6d0b5d1115764d3f0"

RUN set -eux; \
	\
	apk add --no-cache --virtual .fetch-deps gnupg; \
	\
	mkdir -p /usr/src; \
	cd /usr/src; \
	\
	curl -fsSL -o php.tar.xz "$PHP_URL"; \
	\
	if [ -n "$PHP_SHA256" ]; then \
		echo "$PHP_SHA256 *php.tar.xz" | sha256sum -c -; \
	fi; \
	\
	curl -fsSL -o php.tar.xz.asc "$PHP_ASC_URL"; \
	export GNUPGHOME="$(mktemp -d)"; \
	for key in $GPG_KEYS; do \
		gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key"; \
	done; \
	gpg --batch --verify php.tar.xz.asc php.tar.xz; \
	gpgconf --kill all; \
	rm -rf "$GNUPGHOME"; \
	\
	apk del --no-network .fetch-deps

COPY php-src/docker-php-source /usr/local/bin/

RUN set -eux; \
	apk add --no-cache --virtual .build-deps \
		$PHPIZE_DEPS \
		argon2-dev \
		coreutils \
		curl-dev \
		gnu-libiconv-dev \
		libsodium-dev \
		libxml2-dev \
		linux-headers \
		oniguruma-dev \
		openssl-dev \
		readline-dev \
		sqlite-dev \
	; \
	\
# make sure musl's iconv doesn't get used (https://www.php.net/manual/en/intro.iconv.php)
	rm -vf /usr/include/iconv.h; \
	\
	export \
		CFLAGS="$PHP_CFLAGS" \
		CPPFLAGS="$PHP_CPPFLAGS" \
		LDFLAGS="$PHP_LDFLAGS" \
# https://github.com/php/php-src/blob/d6299206dd828382753453befd1b915491b741c6/configure.ac#L1496-L1511
		PHP_BUILD_PROVIDER='https://github.com/docker-library/php' \
		PHP_UNAME='Linux - Docker' \
	; \
	docker-php-source extract; \
	cd /usr/src/php; \
	gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
	case "$gnuArch" in \
		'i686-linux-'*) \
			# php zend jit assumes SSE registers, xmm0-7, are available (and "Pentium4"/SSE2 is a reasonable baseline)
			# https://github.com/php/php-src/pull/22832
			# so, allow it and use SSE/SSE2 floating-point unit instead of the legacy stack
			export \
				CFLAGS="$CFLAGS -msse2 -mfpmath=sse" \
				CPPFLAGS="$CPPFLAGS -msse2 -mfpmath=sse" \
			; \
			;; \
	esac; \
	test "$PHP_INI_DIR" != "${PHP_INI_DIR%/php}"; \
	./configure \
		--build="$gnuArch" \
		--sysconfdir="${PHP_INI_DIR%/php}" \
		--with-config-file-path="$PHP_INI_DIR" \
		--with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
		\
# make sure invalid --configure-flags are fatal errors instead of just warnings
		--enable-option-checking=fatal \
		\
# https://github.com/docker-library/php/issues/439
		--with-mhash \
		\
# https://github.com/docker-library/php/issues/822
		--with-pic \
		\
# --enable-mbstring is included here because otherwise there's no way to get pecl to use it properly (see https://github.com/docker-library/php/issues/195)
		--enable-mbstring \
# --enable-mysqlnd is included here because it's harder to compile after the fact than extensions are (since it's a plugin for several extensions, not an extension in itself)
		--enable-mysqlnd \
# https://wiki.php.net/rfc/argon2_password_hash
		--with-password-argon2 \
# https://wiki.php.net/rfc/libsodium
		--with-sodium=shared \
# always build against system sqlite3 (https://github.com/php/php-src/commit/6083a387a81dbbd66d6316a3a12a63f06d5f7109)
		--with-pdo-sqlite=/usr \
		--with-sqlite3=/usr \
		\
		--with-curl \
		--with-iconv=/usr \
		--with-openssl \
		--with-readline \
		--with-zlib \
		\
# https://github.com/docker-library/php/pull/1259
		--enable-phpdbg \
		--enable-phpdbg-readline \
		\
# in PHP 7.4+, the pecl/pear installers are officially deprecated (requiring an explicit "--with-pear")
		--with-pear \
		\
	; \
	make -j "$(nproc)"; \
	find -type f -name '*.a' -delete; \
	make install; \
	find \
		/usr/local \
		-type f \
		-perm '/0111' \
		-exec sh -euxc ' \
			strip --strip-all "$@" || : \
		' -- '{}' + \
	; \
	make clean; \
	\
# https://github.com/docker-library/php/issues/692 (copy default example "php.ini" files somewhere easily discoverable)
	cp -v php.ini-* "$PHP_INI_DIR/"; \
	\
	cd /; \
	docker-php-source delete; \
	\
	runDeps="$( \
		scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
			| tr ',' '\n' \
			| sort -u \
			| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
	)"; \
	apk add --no-cache $runDeps; \
	\
	apk del --no-network .build-deps; \
	\
# update pecl channel definitions https://github.com/docker-library/php/issues/443
	pecl update-channels; \
	rm -rf /tmp/pear ~/.pearrc; \
	\
# smoke test
	php --version

COPY php-src/docker-php-ext-* php-src/docker-php-entrypoint /usr/local/bin/

# enable OPcache by default (https://wiki.php.net/rfc/make_opcache_required)
RUN docker-php-ext-enable opcache

# sodium was built as a shared module (so that it can be replaced later if so desired), so let's enable it too (https://github.com/docker-library/php/issues/598)
RUN docker-php-ext-enable sodium

ENTRYPOINT ["docker-php-entrypoint"]
CMD ["php", "-a"]

FROM php-src AS base

LABEL maintainer="support@68publishers.io"
WORKDIR /var/www/html

# ENV
ENV COMPOSER_ALLOW_SUPERUSER=1
ENV LD_PRELOAD="/usr/lib/preloadable_libiconv.so php"

ENV INI_VARIABLES_ORDER="EGPCS"
ENV INI_DATE_TIMEZONE="UTC"
ENV INI_REALPATH_CACHE_SIZE="4096K"
ENV INI_REALPATH_CACHE_TTL="600"
ENV INI_MEMORY_LIMIT="512M"

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
        gnu-libiconv \
        harfbuzz-icu \
        # tools
        git \
        jq \
        curl \
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
    && pecl install redis-6.2.0 \
    && pecl install amqp-2.1.2 \
    # Configure & Install PHP extensions
    && docker-php-ext-configure gd --enable-gd --with-freetype --with-jpeg --with-webp --with-avif \
    && docker-php-ext-configure pgsql --with-pgsql=/usr/local/pgsql \
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
        pcntl \
    && docker-php-ext-enable \
        redis \
        amqp \
    # Delete PHP build dependencies and cache/temp files
    && apk del .build-deps \
    && rm -rf \
        /tmp/pear

# Install Composer
COPY --from=composer:2.8.9 /usr/bin/composer /usr/bin/composer

# PHP custom ini values
COPY php-ini/common "$PHP_INI_DIR/conf.d/"

# Set default shell
SHELL ["/bin/ash", "-eo", "pipefail", "-c"]

# Verify composer
RUN ls -latr /usr/local/bin/ \
    && composer --version

FROM base AS dev

ENV INI_XDEBUG_MODE="develop,debug"
ENV INI_XDEBUG_START_WITH_REQUEST="yes"
ENV INI_XDEBUG_DISCOVER_CLIENT_HOST="0"
ENV INI_XDEBUG_CLIENT_HOST="host.docker.internal"
ENV INI_XDEBUG_OUTPUT_DIR="/tmp/docker-xdebug"
ENV INI_XDEBUG_USE_COMPRESSION="0"
ENV INI_XDEBUG_PROFILER_OUTPUT_NAME="cachegrind.out.%p"
ENV INI_XDEBUG_LOG="/var/log/xdebug.log"

ENV INI_OPCACHE_ENABLE="1"
ENV INI_OPCACHE_MEMORY_CONSUMPTION="256"
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
    && pecl install xdebug-3.4.3 \
    && pecl install imagick-3.8.0 \
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
ENV INI_OPCACHE_MEMORY_CONSUMPTION="256"
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
    # Imagick
    && pecl install imagick-3.8.0 \
    # Enable PHP extensions
    && docker-php-ext-enable imagick \
    # Delete build dependencies
    && apk del .build-deps

USER www-data
