FROM alpine AS build
ARG DROPBEAR_VERSION=2026.90
RUN apk add --no-cache build-base zlib-dev zlib-static linux-headers
WORKDIR /build
RUN wget -q https://matt.ucc.asn.au/dropbear/releases/dropbear-${DROPBEAR_VERSION}.tar.bz2 \
    && tar xjf dropbear-${DROPBEAR_VERSION}.tar.bz2 \
    && cd dropbear-${DROPBEAR_VERSION} \
    && ./configure \
    && make PROGRAMS="dropbear dropbearkey" LDFLAGS="-static" \
    && strip dropbear dropbearkey \
    && cp dropbear dropbearkey /usr/local/bin/

FROM alpine AS assemble
COPY --from=build /usr/local/bin/dropbear /output/usr/sbin/dropbear
COPY --from=build /usr/local/bin/dropbearkey /output/usr/bin/dropbearkey
COPY root/ /output/

FROM scratch
COPY --from=assemble /output/ /
