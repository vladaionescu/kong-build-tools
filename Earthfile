ARG KONG_SOURCE_LOCATION=../kong+source
ARG KONG_REQUIREMENTS=../kong+requirements/.requirements
ARG KONG_VERSION_ARTIFACT=../kong+version/version
ARG RESTY_IMAGE_BASE="ubuntu"
ARG RESTY_IMAGE_TAG="bionic"
ARG EARTHLY_GIT_HASH
ARG DOCKER_BASE_SUFFIX="$EARTHLY_GIT_HASH"
ARG DOCKER_OPENRESTY_SUFFIX="$EARTHLY_GIT_HASH"
ARG DOCKER_KONG_SUFFIX="$EARTHLY_GIT_HASH"
ARG DOCKER_REPOSITORY="mashape/kong-build-tools"

package:  # based on Dockerfile.package and actual-package-kong
    FROM kong/fpm:0.0.1

    COPY +kong/build /tmp/build
    COPY fpm-entrypoint.sh /fpm-entrypoint.sh
    COPY after-install.sh /after-install.sh
    COPY .rpmmacros /root/.rpmmacros
    ARG PRIVATE_KEY_FILE=kong.private.gpg-key.asc
    COPY ${PRIVATE_KEY_FILE} /kong.private.asc 
    ARG PRIVATE_KEY_PASSPHRASE
    ENV PRIVATE_KEY_PASSPHRASE ${PRIVATE_KEY_PASSPHRASE}

    ARG RESTY_IMAGE_BASE="ubuntu"
    ARG RESTY_IMAGE_TAG="xenial"
    COPY ${KONG_VERSION_ARTIFACT} /kong-version
    ARG KONG_VERSION=$(cat /kong-version)
    ARG KONG_PACKAGE_NAME=kong
    ARG KONG_CONFLICTS=kong-enterprise-edition
    ARG BUILDPLATFORM=x/amd64

    RUN mkdir -p /tmp/build/lib/systemd/system/
    COPY kong.service /tmp/build/lib/systemd/system/kong.service
    COPY kong.logrotate /tmp/build/etc/kong/kong.logrotate

    RUN /fpm-entrypoint.sh
    SAVE ARTIFACT /output/* AS LOCAL output/

kong:  # based on Dockerfile.kong and actual-build-kong
    FROM +openresty
    COPY --dir ${KONG_SOURCE_LOCATION}/* /kong/
    WORKDIR /kong
    COPY kong /kong
    COPY id_rsa /root/id_rsa
    COPY build-kong.sh /build-kong.sh
    RUN /build-kong.sh
    SAVE ARTIFACT /tmp/build
    # Uncomment to export image.
    #SAVE IMAGE "${DOCKER_REPOSITORY}:kong-${RESTY_IMAGE_BASE}-${RESTY_IMAGE_TAG}-${DOCKER_KONG_SUFFIX}"

openresty:  # based on build-openresty Dockerfile.openresty
    ARG PACKAGE_TYPE="deb"
    FROM +openresty-base-${PACKAGE_TYPE}
    ARG EDITION="community"
    ENV EDITION $EDITION

    COPY "${KONG_REQUIREMENTS}" /req/

    ARG LIBYAML_VERSION=$(grep LIBYAML_VERSION /req/.requirements | awk -F"=" "'{print \$2}'")
    ENV LIBYAML_VERSION $LIBYAML_VERSION
    RUN curl -fsSLo /tmp/yaml-${LIBYAML_VERSION}.tar.gz https://pyyaml.org/download/libyaml/yaml-${LIBYAML_VERSION}.tar.gz \
        && cd /tmp \
        && tar xzf yaml-${LIBYAML_VERSION}.tar.gz \
        && ln -s /tmp/yaml-${LIBYAML_VERSION} /tmp/yaml \
        && cd /tmp/yaml \
        && ./configure \
        --libdir=/tmp/build/usr/local/kong/lib \
        --includedir=/tmp/yaml-${LIBYAML_VERSION} \
        && make install \
        && ./configure --libdir=/usr/local/kong/lib \
        && make install \
        && rm -rf /tmp/yaml-${LIBYAML_VERSION}

    ARG KONG_GMP_VERSION=$(grep KONG_GMP_VERSION /req/.requirements | awk -F"=" "'{print \$2}'")
    ENV KONG_GMP_VERSION $KONG_GMP_VERSION
    RUN if [ "$EDITION" = "enterprise" ] ; then curl -fsSLo /tmp/gmp-${KONG_GMP_VERSION}.tar.bz2 https://ftp.gnu.org/gnu/gmp/gmp-${KONG_GMP_VERSION}.tar.bz2 \
        && cd /tmp \
        && tar xjf gmp-${KONG_GMP_VERSION}.tar.bz2 \
        && ln -s /tmp/gmp-${KONG_GMP_VERSION} /tmp/gmp \
        && cd /tmp/gmp \
        && ./configure --build=x86_64-linux-gnu --enable-static=no --libdir=/tmp/build/usr/local/kong/lib \
        && make -j${RESTY_J}; fi

    ARG KONG_NETTLE_VERSION=$(grep KONG_NETTLE_VERSION /req/.requirements | awk -F"=" "'{print \$2}'")
    ENV KONG_NETTLE_VERSION $KONG_NETTLE_VERSION
    RUN if [ "$EDITION" = "enterprise" ] ; then curl -fsSLo /tmp/nettle-${KONG_NETTLE_VERSION}.tar.gz https://ftp.gnu.org/gnu/nettle/nettle-${KONG_NETTLE_VERSION}.tar.gz \
        && cd /tmp \
        && tar xzf nettle-${KONG_NETTLE_VERSION}.tar.gz \
        && ln -s /tmp/nettle-${KONG_NETTLE_VERSION} /tmp/nettle \
        && cd /tmp/nettle \
        && LDFLAGS="-Wl,-rpath,/usr/local/kong/lib" \
        ./configure --disable-static \
        --libdir=/tmp/build/usr/local/kong/lib \
        --with-include-path="/tmp/gmp-${KONG_GMP_VERSION}/" \
        --with-lib-path="/tmp/gmp-${KONG_GMP_VERSION}/.libs/" \
        && make -j${RESTY_J}; fi

    ARG KONG_DEP_PASSWDQC_VERSION="1.3.1"
    ENV KONG_DEP_PASSWDQC_VERSION $KONG_DEP_PASSWDQC_VERSION
    RUN if [ "$EDITION" = "enterprise" ] ; then curl -fsSLo /tmp/passwdqc-${KONG_DEP_PASSWDQC_VERSION}.tar.gz https://www.openwall.com/passwdqc/passwdqc-${KONG_DEP_PASSWDQC_VERSION}.tar.gz \
        && cd /tmp \
        && tar xzf passwdqc-${KONG_DEP_PASSWDQC_VERSION}.tar.gz \
        && ln -s /tmp/passwdqc-${KONG_DEP_PASSWDQC_VERSION} /tmp/passwdqc \
        && cd /tmp/passwdqc \
        && make libpasswdqc.so -j$BUILD_JOBS \
        && make \
        DESTDIR=/tmp/build/ \
        SHARED_LIBDIR=/usr/local/kong/lib \
        SHARED_LIBDIR_REL='.' \
        DEVEL_LIBDIR=/usr/local/kong/lib \
        INCLUDEDIR=/usr/local/kong/include/passwdqc \
        CONFDIR=/usr/local/etc/passwdqc \
        MANDIR=/usr/local/share/man \
        install_lib; fi

    ARG KONG_NGINX_MODULE=$(grep KONG_NGINX_MODULE /req/.requirements | awk -F"=" "'{print \$2}'")

    ARG RESTY_VERSION=$(grep RESTY_VERSION /req/.requirements | awk -F"=" "'{print \$2}'")
    LABEL resty_version="${RESTY_VERSION}"

    ARG RESTY_OPENSSL_VERSION=$(grep RESTY_OPENSSL_VERSION /req/.requirements | awk -F"=" "'{print \$2}'")
    LABEL resty_openssl_version="${RESTY_OPENSSL_VERSION}"

    ARG RESTY_PCRE_VERSION=$(grep RESTY_PCRE_VERSION /req/.requirements | awk -F"=" "'{print \$2}'")
    LABEL resty_pcre_version="${RESTY_PCRE_VERSION}"

    ARG RESTY_LUAROCKS_VERSION=$(grep RESTY_LUAROCKS_VERSION /req/.requirements | awk -F"=" "'{print \$2}'")
    LABEL resty_luarocks_version="${RESTY_LUAROCKS_VERSION}"

    COPY openresty-build-tools /tmp/openresty-build-tools
    COPY openresty-patches /tmp/openresty-patches
    COPY build-openresty.sh /tmp/build-openresty.sh

    ARG OPENRESTY_PATCHES=1
    ENV OPENRESTY_PATCHES="${OPENRESTY_PATCHES}"

    COPY kong-licensing /enterprise/kong-licensing
    COPY lua-kong-nginx-module /enterprise/lua-kong-nginx-module

    ARG DEBUG=1
    RUN DEBUG="${DEBUG}" /tmp/build-openresty.sh \
        && rm -rf /work

    WORKDIR /kong
    COPY --dir ${KONG_SOURCE_LOCATION}/* /kong/
    COPY build-kong.sh /build-kong.sh

    RUN /build-kong.sh && rm -rf /kong

    RUN sed -i 's/\/tmp\/build//' `grep -l -I -r '\/tmp\/build' /tmp/build/` || true

    # Uncomment to export image.
    #SAVE IMAGE "${DOCKER_REPOSITORY}:openresty-${RESTY_IMAGE_BASE}-${RESTY_IMAGE_TAG}-${DOCKER_OPENRESTY_SUFFIX}"

openresty-base-deb:  # based on Dockerfile.deb and build-base
    FROM ${RESTY_IMAGE_BASE}:${RESTY_IMAGE_TAG}
    RUN DEBIAN_FRONTEND=noninteractive apt-get update \
        && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
            build-essential \
            ca-certificates \
            curl \
            gettext-base \
            libgd-dev \
            libgeoip-dev \
            libncurses5-dev \
            libperl-dev \
            libreadline-dev \
            libxslt1-dev \
            make \
            perl \
            unzip \
            zlib1g-dev \
            libssl-dev \
            git \
            m4 \
            file \
            ssh \
            valgrind
    # Uncomment to export image.
    #SAVE IMAGE "${DOCKER_REPOSITORY}:${RESTY_IMAGE_BASE}-${RESTY_IMAGE_TAG}-${DOCKER_BASE_SUFFIX}"
