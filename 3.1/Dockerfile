FROM golang:alpine3.20 AS builder

ENV DATAPLANE_MINOR 3.0.3
ENV DATAPLANE_V2_MINOR 2.9.8
ENV DATAPLANE_URL https://github.com/haproxytech/dataplaneapi.git

RUN apk add --no-cache ca-certificates git make && \
    git clone "${DATAPLANE_URL}" "${GOPATH}/src/github.com/haproxytech/dataplaneapi" && \
    cd "${GOPATH}/src/github.com/haproxytech/dataplaneapi" && \
    git checkout "v${DATAPLANE_MINOR}" && \
    make build && cp build/dataplaneapi /dataplaneapi && \
    make clean && \
    git checkout "v${DATAPLANE_V2_MINOR}" && \
    make build && cp build/dataplaneapi /dataplaneapi-v2

FROM alpine:3.20

MAINTAINER Dinko Korunic <dkorunic@haproxy.com>

LABEL Name HAProxy
LABEL Release Community Edition
LABEL Vendor HAProxy
LABEL Version 3.1.0
LABEL RUN /usr/bin/docker -d IMAGE

ENV HAPROXY_BRANCH 3.1
ENV HAPROXY_MINOR 3.1.0
ENV HAPROXY_SHA256 56a1468574ab411dcabde837f96bea6cf3c2eb90e279469f75ed1dcdc70fce11
ENV HAPROXY_SRC_URL http://www.haproxy.org/download

ENV HAPROXY_UID haproxy
ENV HAPROXY_GID haproxy

COPY --from=builder /dataplaneapi /usr/local/bin/dataplaneapi
COPY --from=builder /dataplaneapi-v2 /usr/local/bin/dataplaneapi-v2

RUN apk add --no-cache ca-certificates jemalloc && \
    apk add --no-cache --virtual build-deps gcc libc-dev \
    linux-headers lua5.4-dev make openssl openssl-dev pcre2-dev tar \
    zlib-dev curl shadow jemalloc-dev && \
    curl -sfSL "${HAPROXY_SRC_URL}/${HAPROXY_BRANCH}/src/devel/haproxy-${HAPROXY_MINOR}.tar.gz" -o haproxy.tar.gz && \
    echo "$HAPROXY_SHA256 *haproxy.tar.gz" | sha256sum -c - && \
    groupadd "$HAPROXY_GID" && \
    useradd -g "$HAPROXY_GID" "$HAPROXY_UID" && \
    mkdir -p /tmp/haproxy && \
    tar -xzf haproxy.tar.gz -C /tmp/haproxy --strip-components=1 && \
    rm -f haproxy.tar.gz && \
    make -C /tmp/haproxy -j"$(nproc)" TARGET=linux-musl CPU=generic USE_PCRE2=1 USE_PCRE2_JIT=1 \
                            USE_TFO=1 USE_LINUX_TPROXY=1 USE_GETADDRINFO=1 \
                            USE_LUA=1 LUA_LIB=/usr/lib/lua5.4 LUA_INC=/usr/include/lua5.4 \
                            USE_PROMEX=1 USE_SLZ=1 \
                            USE_OPENSSL=1 USE_PTHREAD_EMULATION=1 \
                            USE_QUIC=1 USE_QUIC_OPENSSL_COMPAT=1 \
                            ADDLIB=-ljemalloc \
                            all && \
    make -C /tmp/haproxy TARGET=linux2628 install-bin install-man && \
    ln -s /usr/local/sbin/haproxy /usr/sbin/haproxy && \
    mkdir -p /var/lib/haproxy && \
    chown "$HAPROXY_UID:$HAPROXY_GID" /var/lib/haproxy && \
    mkdir -p /usr/local/etc/haproxy && \
    ln -s /usr/local/etc/haproxy /etc/haproxy && \
    cp -R /tmp/haproxy/examples/errorfiles /usr/local/etc/haproxy/errors && \
    rm -rf /tmp/haproxy && \
    chmod +x /usr/local/bin/dataplaneapi && \
    ln -s /usr/local/bin/dataplaneapi /usr/bin/dataplaneapi && \
    chmod +x /usr/local/bin/dataplaneapi-v2 && \
    ln -s /usr/local/bin/dataplaneapi-v2 /usr/bin/dataplaneapi-v2 && \
    touch /usr/local/etc/haproxy/dataplaneapi.yml && \
    chown "$HAPROXY_UID:$HAPROXY_GID" /usr/local/etc/haproxy/dataplaneapi.yml && \
    apk del build-deps && \
    apk add --no-cache openssl zlib lua5.4-libs pcre2 && \
    rm -f /var/cache/apk/*

COPY haproxy.cfg /usr/local/etc/haproxy
COPY docker-entrypoint.sh /

STOPSIGNAL SIGUSR1

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["haproxy", "-f", "/usr/local/etc/haproxy/haproxy.cfg"]
