# This docker file is based on docker-cli https://github.com/docker-library/docker/blob/master/29/cli/Dockerfile
# and the docker-dind https://github.com/docker-library/docker/blob/master/29/dind/Dockerfile files
# to create a docker in docker image
ARG base_image

FROM ${base_image}

# Set to skip prompt during USG audit
ENV DEBIAN_FRONTEND=noninteractive


ENV DOCKER_COMPOSE_VERSION 5.0.0
ENV DOCKER_VERSION 29.1.2
ENV DOCKER_BUILDX_VERSION 0.30.1
ENV DOCKER_TLS_CERTDIR=/certs
# https://github.com/docker/docker/tree/master/hack/dind
ENV DIND_COMMIT 8d9e3502aba39127e4d12196dae16d306f76993d

RUN apt-get update && apt-get install -y --no-install-recommends \
	bash \
	wget \
	ca-certificates \
	openssh-client \
	grep

# pre-add a "docker" group for socket usage
RUN set -eux; \
	addgroup -gid 2375 --system docker

RUN set -eux; \
	\
	sysArch="$(uname -m)"; \
	case "$sysArch" in \
	'x86_64') \
	url="https://download.docker.com/linux/static/stable/x86_64/docker-$DOCKER_VERSION.tgz"; \
	;; \
	'aarch64') \
	url="https://download.docker.com/linux/static/stable/aarch64/docker-$DOCKER_VERSION.tgz"; \
	;; \
	*) echo >&2 "error: unsupported 'docker.tgz' architecture ($sysArch)"; exit 1 ;; \
	esac; \
	\
	wget -O 'docker.tgz' "$url"; \
	\
	tar --extract \
	--file docker.tgz \
	--strip-components 1 \
	--directory /usr/local/bin/ \
	--no-same-owner \
	'docker/docker' \
	; \
	rm docker.tgz; \
	\
	docker --version

RUN set -eux; \
	\
	sysArch="$(uname -m)"; \
	case "$sysArch" in \
	'x86_64') \
	url="https://github.com/docker/buildx/releases/download/v$DOCKER_BUILDX_VERSION/buildx-v$DOCKER_BUILDX_VERSION.linux-amd64"; \
	sha256='c37114fcd034025ec68e224657c8a5a850df472ded3ddcbca75ad3a7ebb9710d'; \
	;; \
	'aarch64') \
	url="https://github.com/docker/buildx/releases/download/v$DOCKER_BUILDX_VERSION/buildx-v$DOCKER_BUILDX_VERSION.linux-arm64"; \
	sha256='31d012d52d6df68aef4b55db62330967b562811f0de30cdfaa4505f314797c76'; \
	;; \
	*) echo >&2 "warning: unsupported 'docker-buildx' architecture ($sysArch); skipping"; exit 0 ;; \
	esac; \
	\
	wget -O 'docker-buildx' "$url"; \
	echo "$sha256 *"'docker-buildx' | sha256sum -c -; \
	\
	plugin='/usr/local/libexec/docker/cli-plugins/docker-buildx'; \
	mkdir -p "$(dirname "$plugin")"; \
	mv -vT 'docker-buildx' "$plugin"; \
	chmod +x "$plugin"; \
	\
	docker buildx version

RUN set -eux; \
	\
	sysArch="$(uname -m)"; \
	case "$sysArch" in \
	'x86_64') \
	url="https://github.com/docker/compose/releases/download/v$DOCKER_COMPOSE_VERSION/docker-compose-linux-x86_64"; \
	sha256='5091bac5729ce968c602d157c2f0b959b7b367d4efb70aa864eb9ae78eebe13e'; \
	;; \
	'aarch64') \
	url="https://github.com/docker/compose/releases/download/v$DOCKER_COMPOSE_VERSION/docker-compose-linux-aarch64"; \
	sha256='d68ba7053066a44a51ee33b7dcdc106d8c8745eb0eaf46dc59fbbeb22ec46392'; \
	;; \
	*) echo >&2 "warning: unsupported 'docker-compose' architecture ($sysArch); skipping"; exit 0 ;; \
	esac; \
	\
	wget -O 'docker-compose' "$url"; \
	echo "$sha256 *"'docker-compose' | sha256sum -c -; \
	\
	plugin='/usr/local/libexec/docker/cli-plugins/docker-compose'; \
	mkdir -p "$(dirname "$plugin")"; \
	mv -vT 'docker-compose' "$plugin"; \
	chmod +x "$plugin"; \
	\
	ln -sv "$plugin" /usr/local/bin/; \
	docker-compose --version; \
	docker compose version

# https://github.com/docker-library/docker/pull/166
#   dockerd-entrypoint.sh uses DOCKER_TLS_CERTDIR for auto-generating TLS certificates
#   docker-entrypoint.sh uses DOCKER_TLS_CERTDIR for auto-setting DOCKER_TLS_VERIFY and DOCKER_CERT_PATH
# (For this to work, at least the "client" subdirectory of this path needs to be shared between the client and server containers via a volume, "docker cp", or other means of data sharing.)

# also, ensure the directory pre-exists and has wide enough permissions for "dockerd-entrypoint.sh" to create subdirectories, even when run in "rootless" mode
RUN mkdir /certs /certs/client && chmod 1777 /certs /certs/client
# (doing both /certs and /certs/client so that if Docker does a "copy-up" into a volume defined on /certs/client, it will "do the right thing" by default in a way that still works for rootless users)

# https://github.com/docker/docker/blob/master/project/PACKAGERS.md#runtime-dependencies
RUN set -eux; \
	apt-get install -y --no-install-recommends \
	btrfs-progs \
	e2fsprogs \
	# e2fsprogs-extra \
	# ip6tables \
	iptables \
	openssl \
	# shadow-uidmap \
	uidmap \
	xfsprogs \
	# xz \
	xz-utils \
	# pigz: https://github.com/moby/moby/pull/35697 (faster gzip implementation)
	pigz \
	zfsutils-linux \
	zfs-dkms \
	zfs-zed \
	;

# set up subuid/subgid so that "--userns-remap=default" works out-of-the-box
RUN set -eux; \
	addgroup --system dockremap; \
	adduser --ingroup dockremap dockremap; \
	echo 'dockremap:165536:65536' >> /etc/subuid; \
	echo 'dockremap:165536:65536' >> /etc/subgid

RUN set -eux; \
	\
	sysArch="$(uname -m)"; \
	case "$sysArch" in \
	'x86_64') \
	url="https://download.docker.com/linux/static/stable/x86_64/docker-$DOCKER_VERSION.tgz"; \
	;; \
	'armhf') \
	url="https://download.docker.com/linux/static/stable/armel/docker-$DOCKER_VERSION.tgz"; \
	;; \
	'armv7') \
	url="https://download.docker.com/linux/static/stable/armhf/docker-$DOCKER_VERSION.tgz"; \
	;; \
	'aarch64') \
	url="https://download.docker.com/linux/static/stable/aarch64/docker-$DOCKER_VERSION.tgz"; \
	;; \
	*) echo >&2 "error: unsupported 'docker.tgz' architecture ($sysArch)"; exit 1 ;; \
	esac; \
	\
	wget -O 'docker.tgz' "$url"; \
	\
	tar --extract \
	--file docker.tgz \
	--strip-components 1 \
	--directory /usr/local/bin/ \
	--no-same-owner \
	# we exclude the CLI binary because we already extracted that over in the "docker:25-cli" image that we're FROM and we don't want to duplicate those bytes again in this layer
	--exclude 'docker/docker' \
	; \
	rm docker.tgz; \
	\
	dockerd --version; \
	containerd --version; \
	ctr --version; \
	runc --version

RUN set -eux; \
	wget -O /usr/local/bin/dind "https://raw.githubusercontent.com/docker/docker/$DIND_COMMIT/hack/dind"; \
	chmod +x /usr/local/bin/dind

COPY modprobe.sh /usr/local/bin/
COPY docker-entrypoint.sh /usr/local/bin/
COPY dockerd-entrypoint.sh /usr/local/bin/

RUN chmod +x /usr/local/bin/dockerd-entrypoint.sh; \
	chmod +x /usr/local/bin/docker-entrypoint.sh; \
	chmod +x /usr/local/bin/modprobe.sh; \
	export PATH="/usr/local/bin/:$PATH";

VOLUME /var/lib/docker
EXPOSE 2375 2376

ENTRYPOINT ["dockerd-entrypoint.sh"]
CMD []
