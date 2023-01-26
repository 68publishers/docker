_IMAGE=68publishers/php
_BUILD_ARGS_TAG ?= 8.1-unit-dev
_BUILD_ARGS_TAG_VERSION ?= latest
_BUILD_ARGS_DIR ?= 8.1-unit
_BUILD_ARGS_XDEBUG ?= 0
_BUILD_ARGS_IMAGICK ?= 0

_builder:
	docker build --pull \
		--build-arg xdebug=${_BUILD_ARGS_XDEBUG} \
		--build-arg imagick=${_BUILD_ARGS_IMAGICK} \
		-t ${_IMAGE}:${_BUILD_ARGS_TAG}-${_BUILD_ARGS_TAG_VERSION} \
		./${_BUILD_ARGS_DIR}
	docker push ${_IMAGE}:${_BUILD_ARGS_TAG}-${_BUILD_ARGS_TAG_VERSION}

_builder_dev:
	@$(MAKE) _builder \
		-e _BUILD_ARGS_XDEBUG="1" \
		-e _BUILD_ARGS_IMAGICK="1"

_builder_prod:
	@$(MAKE) _builder \
		-e _BUILD_ARGS_XDEBUG="0" \
		-e _BUILD_ARGS_IMAGICK="0"

_builder_prod_imagick:
	@$(MAKE) _builder \
		-e _BUILD_ARGS_XDEBUG="0" \
		-e _BUILD_ARGS_IMAGICK="1"

build-8.1-unit-dev-%:
	@$(MAKE) _builder_dev \
		-e _BUILD_ARGS_TAG="8.1-unit-dev" \
		-e _BUILD_ARGS_TAG_VERSION="$*" \
		-e _BUILD_ARGS_DIR="8.1-unit"

build-8.1-unit-prod-%:
	@$(MAKE) _builder_prod \
		-e _BUILD_ARGS_TAG="8.1-unit-prod" \
		-e _BUILD_ARGS_TAG_VERSION="$*" \
		-e _BUILD_ARGS_DIR="8.1-unit"

build-8.1-unit-prod-imagick-%:
	@$(MAKE) _builder_prod_imagick \
		-e _BUILD_ARGS_TAG="8.1-unit-prod-imagick" \
		-e _BUILD_ARGS_TAG_VERSION="$*" \
		-e _BUILD_ARGS_DIR="8.1-unit"
