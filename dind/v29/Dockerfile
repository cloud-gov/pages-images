# This docker file is based on docker-cli https://github.com/docker-library/docker/blob/master/29/cli/Dockerfile
# and the docker-dind https://github.com/docker-library/docker/blob/master/29/dind/Dockerfile files
# to create a docker in docker image
ARG base_image

FROM ${base_image}

# Set to skip prompt during USG audit
ENV DEBIAN_FRONTEND=noninteractive


ENV DOCKER_COMPOSE_VERSION 5.2.0
ENV DOCKER_VERSION 29.6.1
ENV DOCKER_BUILDX_VERSION 0.35.0
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
	sha256='d41ece72044243b4f58b343441ae37446d9c29a7d6b5e11c61847bbcf8f7dfda'; \
	;; \
	'aarch64') \
	url="https://github.com/docker/buildx/releases/download/v$DOCKER_BUILDX_VERSION/buildx-v$DOCKER_BUILDX_VERSION.linux-arm64"; \
	sha256='c4248d6cbc4a619a7e0b4609c11e509ad4ac0b475e1c64817c0ac20c5d90c766'; \
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
	sha256='018f9612ecabc5f2d7aaa53d6f5f44453a87611e2d72c8ef84d7b1eca070e719'; \
	;; \
	'aarch64') \
	url="https://github.com/docker/compose/releases/download/v$DOCKER_COMPOSE_VERSION/docker-compose-linux-aarch64"; \
	sha256='739de570a0adf5eab12830db980f549fb5f44ad6b266e1e43e20f6f9df7cbcca'; \
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
