FROM debian:bookworm-slim AS build
RUN apt-get update && apt-get install -y dropbear-bin

FROM scratch
COPY --from=build /usr/sbin/dropbear /usr/sbin/dropbear
COPY --from=build /usr/bin/dropbearkey /usr/bin/dropbearkey
COPY root/ /
