# adapted from
# https://github.com/zaproxy/zaproxy/blob/main/docker/Dockerfile-bare
# https://github.com/adoptium/containers/blob/main/11/jre/ubuntu/jammy/Dockerfile
ARG base_image

FROM ${base_image}

# Set to skip prompt during USG audit
ENV DEBIAN_FRONTEND=noninteractive

ENV JAVA_HOME /opt/java/openjdk
ENV PATH $JAVA_HOME/bin:$PATH


RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
    # curl required for historical reasons, see https://github.com/adoptium/containers/issues/255
    curl \
    wget \
    # java.lang.UnsatisfiedLinkError: libfontmanager.so: libfreetype.so.6: cannot open shared object file: No such file or directory
    # java.lang.NoClassDefFoundError: Could not initialize class sun.awt.X11FontManager
    # https://github.com/docker-library/openjdk/pull/235#issuecomment-424466077
    fontconfig \
    # utilities for keeping Ubuntu and OpenJDK CA certificates in sync
    # https://github.com/adoptium/containers/issues/293
    ca-certificates p11-kit \
    tzdata \
    # locales ensures proper character encoding and locale-specific behaviors using en_US.UTF-8
    locales \
    ; \
    echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen; \
    locale-gen en_US.UTF-8;

# Default to UTF-8 file.encoding (maybe duplicating above)
ENV LANG='en_US.UTF-8' LANGUAGE='en_US:en' LC_ALL='en_US.UTF-8'

ENV JAVA_VERSION=jdk-11.0.31+11

RUN set -eux; \
    ARCH="$(dpkg --print-architecture)"; \
    case "${ARCH}" in \
    amd64) \
    ESUM='a6af3d61851f57eb79ef0189837522329717458bf230ee284da2d26634f1ea3a'; \
    BINARY_URL='https://github.com/adoptium/temurin11-binaries/releases/download/jdk-11.0.31%2B11/OpenJDK11U-jre_x64_linux_hotspot_11.0.31_11.tar.gz'; \
    ;; \
    arm64) \
    ESUM='eabe05fb80626ad24db17cf1df137855e77fbacbc83c11aaf243cedd224467de'; \
    BINARY_URL='https://github.com/adoptium/temurin11-binaries/releases/download/jdk-11.0.31%2B11/OpenJDK11U-jre_aarch64_linux_hotspot_11.0.31_11.tar.gz'; \
    ;; \
    armhf) \
    ESUM='5d3e988cdc8291779068c957c01d339f26178ff65d13af4671107b169e80a69f'; \
    BINARY_URL='https://github.com/adoptium/temurin11-binaries/releases/download/jdk-11.0.31%2B11/OpenJDK11U-jre_arm_linux_hotspot_11.0.31_11.tar.gz'; \
    ;; \
    ppc64el) \
    ESUM='11e58bf1eeae10f0dc1a98cc43bf97af270a0b516f6ff9fb3189024c5e22550a'; \
    BINARY_URL='https://github.com/adoptium/temurin11-binaries/releases/download/jdk-11.0.31%2B11/OpenJDK11U-jre_ppc64le_linux_hotspot_11.0.31_11.tar.gz'; \
    ;; \
    s390x) \
    ESUM='4c311b19aa3922951be288076f0f41a831ab7af32284da9b3e21cdaa251a078a'; \
    BINARY_URL='https://github.com/adoptium/temurin11-binaries/releases/download/jdk-11.0.31%2B11/OpenJDK11U-jre_s390x_linux_hotspot_11.0.31_11.tar.gz'; \
    ;; \
    *) \
    echo "Unsupported arch: ${ARCH}"; \
    exit 1; \
    ;; \
    esac; \
    wget --progress=dot:giga -O /tmp/openjdk.tar.gz ${BINARY_URL}; \
    echo "${ESUM} */tmp/openjdk.tar.gz" | sha256sum -c -; \
    mkdir -p "$JAVA_HOME"; \
    tar --extract \
    --file /tmp/openjdk.tar.gz \
    --directory "$JAVA_HOME" \
    --strip-components 1 \
    --no-same-owner \
    ; \
    rm -f /tmp/openjdk.tar.gz ${JAVA_HOME}/lib/src.zip; \
    # https://github.com/docker-library/openjdk/issues/331#issuecomment-498834472
    find "$JAVA_HOME/lib" -name '*.so' -exec dirname '{}' ';' | sort -u > /etc/ld.so.conf.d/docker-openjdk.conf; \
    ldconfig; \
    # https://github.com/docker-library/openjdk/issues/212#issuecomment-420979840
    # https://openjdk.java.net/jeps/341
    java -Xshare:dump;

RUN set -eux; \
    echo "Verifying install ..."; \
    echo "java --version"; java --version; \
    echo "Complete."

RUN apt-get install -q -y --fix-missing \
    openjdk-11-jdk \
    xmlstarlet \
    unzip && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /zap

# Download and expand the latest stable release
RUN wget -qO- https://raw.githubusercontent.com/zaproxy/zap-admin/master/ZapVersions.xml | xmlstarlet sel -t -v //url |grep -i Linux | wget --content-disposition -i - -O - | tar zxv && \
    mv ZAP*/* . && \
    rm -R ZAP*

# Update add-ons
# RUN ./zap.sh -cmd -silent -addonupdate
# Copy them to installation directory
# RUN cp /root/.ZAP/plugin/*.zap plugin/ || :

RUN echo "zap2docker-bare" > /zap/container

RUN useradd --uid=1000 --home-dir=/home/zap --shell=/bin/bash zap
RUN chown -R 1000:1000 /zap
COPY --link --chown=1000:1000 policies /home/zap/.ZAP/policies/
RUN usermod --shell /bin/false zap

ENV PATH=$JAVA_HOME/bin:/zap/:$PATH
ENV ZAP_PATH=/zap/zap.sh
ENV HOME=/home/zap/
ENV ZAP_PORT=8080

RUN rm -rf /var/lib/apt/lists/*

HEALTHCHECK CMD curl --silent --output /dev/null --fail http://localhost:$ZAP_PORT/ || exit 1
