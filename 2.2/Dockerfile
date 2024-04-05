FROM golang:alpine3.18 AS builder

ENV DATAPLANE_MINOR 2.9.2
ENV DATAPLANE_URL https://github.com/haproxytech/dataplaneapi.git

RUN apk add --no-cache ca-certificates git make && \
    git clone "${DATAPLANE_URL}" "${GOPATH}/src/github.com/haproxytech/dataplaneapi" && \
    cd "${GOPATH}/src/github.com/haproxytech/dataplaneapi" && \
    git checkout "v${DATAPLANE_MINOR}" && \
    make build && cp build/dataplaneapi /dataplaneapi

FROM alpine:3.18

MAINTAINER Dinko Korunic <dkorunic@haproxy.com>

LABEL Name HAProxy
LABEL Release Community Edition
LABEL Vendor HAProxy
LABEL Version 2.2.33
LABEL RUN /usr/bin/docker -d IMAGE

ENV HAPROXY_BRANCH 2.2
ENV HAPROXY_MINOR 2.2.33
ENV HAPROXY_SHA256 24f9eec04ee8d9e3652370be3db9852dec8aa650b3c8eeae969300c86b6fda5b
ENV HAPROXY_SRC_URL http://www.haproxy.org/download

ENV LIBSLZ_MINOR 1.2.0
ENV LIBSLZ_SHA256 723a8ef648ac5b30e5074c013ff61a5e5f54a5aafc9496f7dab9f6b02030bf24
ENV LIBSLZ_URL https://github.com/wtarreau/libslz/archive/refs/tags

ENV HAPROXY_UID haproxy
ENV HAPROXY_GID haproxy

COPY --from=builder /dataplaneapi /usr/local/bin/dataplaneapi

RUN apk add --no-cache ca-certificates && \
    apk add --no-cache --virtual build-deps gcc libc-dev \
    linux-headers lua5.3-dev make openssl openssl-dev pcre2-dev tar \
    zlib-dev curl shadow && \
    curl -sfSL "${LIBSLZ_URL}/v${LIBSLZ_MINOR}.tar.gz" -o libslz.tar.gz && \
    echo "$LIBSLZ_SHA256 *libslz.tar.gz" | sha256sum -c - && \
    mkdir -p /tmp/libslz && \
    tar -xzf libslz.tar.gz -C /tmp/libslz --strip-components=1 && \
    make -C /tmp/libslz static && \
    rm -f libslz.tar.gz && \
    curl -sfSL "${HAPROXY_SRC_URL}/${HAPROXY_BRANCH}/src/haproxy-${HAPROXY_MINOR}.tar.gz" -o haproxy.tar.gz && \
    echo "$HAPROXY_SHA256 *haproxy.tar.gz" | sha256sum -c - && \
    groupadd "$HAPROXY_GID" && \
    useradd -g "$HAPROXY_GID" "$HAPROXY_UID" && \
    mkdir -p /tmp/haproxy && \
    tar -xzf haproxy.tar.gz -C /tmp/haproxy --strip-components=1 && \
    rm -f haproxy.tar.gz && \
    make -C /tmp/haproxy -j"$(nproc)" TARGET=linux-musl CPU=generic USE_PCRE2=1 USE_PCRE2_JIT=1 \
                            USE_TFO=1 USE_LINUX_TPROXY=1 USE_GETADDRINFO=1 \
                            USE_LUA=1 LUA_LIB=/usr/lib/lua5.3 LUA_INC=/usr/include/lua5.3 \
                            EXTRA_OBJS="contrib/prometheus-exporter/service-prometheus.o" \
                            USE_SLZ=1 SLZ_INC=/tmp/libslz/src SLZ_LIB=/tmp/libslz \
                            USE_OPENSSL=1 USE_PTHREAD_EMULATION=1 \
                            all && \
    make -C /tmp/haproxy TARGET=linux2628 install-bin install-man && \
    ln -s /usr/local/sbin/haproxy /usr/sbin/haproxy && \
    mkdir -p /var/lib/haproxy && \
    chown "$HAPROXY_UID:$HAPROXY_GID" /var/lib/haproxy && \
    mkdir -p /usr/local/etc/haproxy && \
    ln -s /usr/local/etc/haproxy /etc/haproxy && \
    cp -R /tmp/haproxy/examples/errorfiles /usr/local/etc/haproxy/errors && \
    rm -rf /tmp/libslz && \
    rm -rf /tmp/haproxy && \
    chmod +x /usr/local/bin/dataplaneapi && \
    ln -s /usr/local/bin/dataplaneapi /usr/bin/dataplaneapi && \
    touch /usr/local/etc/haproxy/dataplaneapi.yml && \
    chown "$HAPROXY_UID:$HAPROXY_GID" /usr/local/etc/haproxy/dataplaneapi.yml && \
    apk del build-deps && \
    apk add --no-cache openssl zlib lua5.3-libs pcre2 && \
    rm -f /var/cache/apk/*

COPY haproxy.cfg /usr/local/etc/haproxy
COPY docker-entrypoint.sh /

STOPSIGNAL SIGUSR1

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["haproxy", "-f", "/usr/local/etc/haproxy/haproxy.cfg"]
