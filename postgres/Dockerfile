ARG PG_MAJOR
ARG PGVECTOR_TAG

# Stage 1: pgvector image to copy extension from
FROM pgvector/pgvector:${PGVECTOR_TAG}-pg${PG_MAJOR} AS pgvector-source

# Stage 2: DHI hardened Postgres base + pgvector + VectorChord (use -dev for RUN/apt-get)
FROM dhi.io/postgres:${PG_MAJOR}-debian13-dev

ARG PG_MAJOR
ARG VECTORCHORD_TAG
ARG PGVECTORS_TAG
ARG TARGETARCH

# Copy pgvector extension from pgvector image into DHI base
COPY --from=pgvector-source /usr/share/postgresql/${PG_MAJOR}/extension/vector* /usr/share/postgresql/${PG_MAJOR}/extension/
COPY --from=pgvector-source /usr/lib/postgresql/${PG_MAJOR}/lib/vector.so /usr/lib/postgresql/${PG_MAJOR}/lib/

ENV DB_STORAGE_TYPE=SSD

# VectorChord/pgvecto.rs use amd64 in asset names; uname -m returns x86_64
# Strip "v" prefix from version tags for URL (Docker ARG doesn't expand ${var#pattern})
# DHI postgres base has no postgresql-14 apt package; extract .deb instead of apt-get install
RUN VCHORD_ARCH=$(echo "$TARGETARCH" | sed 's/x86_64/amd64/') && \
    VCHORD_VER=$(echo "$VECTORCHORD_TAG" | sed 's/^v//') && \
    PGVECTORS_VER=$(echo "$PGVECTORS_TAG" | sed 's/^v//') && \
    apt-get update && \
    apt-get install -y wget && \
    wget -nv -O /tmp/vchord.deb "https://github.com/tensorchord/VectorChord/releases/download/${VCHORD_VER}/postgresql-${PG_MAJOR%.*}-vchord_${VCHORD_VER}-1_${VCHORD_ARCH}.deb" && \
    dpkg-deb -x /tmp/vchord.deb /tmp/vchord && \
    cp -a /tmp/vchord/usr/lib/postgresql/${PG_MAJOR}/lib/vchord.so /usr/lib/postgresql/${PG_MAJOR}/lib/ && \
    cp -a /tmp/vchord/usr/share/postgresql/${PG_MAJOR}/extension/vchord* /usr/share/postgresql/${PG_MAJOR}/extension/ && \
    rm -rf /tmp/vchord.deb /tmp/vchord && \
    if [ -n "$PGVECTORS_TAG" ]; then \
        PGVECTORS_SUFFIX=$(if [ "$PGVECTORS_TAG" = '0.3.0' ]; then echo "_vectors"; fi) && \
        wget -nv -O /tmp/pgvectors.deb "https://github.com/tensorchord/pgvecto.rs/releases/download/v$PGVECTORS_TAG/vectors-pg${PG_MAJOR%.*}_${PGVECTORS_VER}_${VCHORD_ARCH}${PGVECTORS_SUFFIX}.deb"; \
        dpkg-deb -x /tmp/pgvectors.deb /tmp/pgvectors && \
        cp -a /tmp/pgvectors/usr/lib/postgresql/${PG_MAJOR}/lib/vectors.so /usr/lib/postgresql/${PG_MAJOR}/lib/ && \
        cp -a /tmp/pgvectors/usr/share/postgresql/${PG_MAJOR}/extension/vectors* /usr/share/postgresql/${PG_MAJOR}/extension/ && \
        rm -rf /tmp/pgvectors.deb /tmp/pgvectors; \
    fi && \
    cp -a /usr/lib/postgresql/${PG_MAJOR}/lib/vector.so /usr/lib/postgresql/${PG_MAJOR}/lib/vchord.so /opt/postgresql/${PG_MAJOR}/lib/ && \
    ([ -z "$PGVECTORS_TAG" ] || cp -a /usr/lib/postgresql/${PG_MAJOR}/lib/vectors.so /opt/postgresql/${PG_MAJOR}/lib/) && \
    cp -a /usr/share/postgresql/${PG_MAJOR}/extension/vector* /usr/share/postgresql/${PG_MAJOR}/extension/vchord* /opt/postgresql/${PG_MAJOR}/share/extension/ && \
    ([ -z "$PGVECTORS_TAG" ] || cp -a /usr/share/postgresql/${PG_MAJOR}/extension/vectors* /opt/postgresql/${PG_MAJOR}/share/extension/) && \
    apt-get remove -y wget && \
    apt-get autoremove -y && \
    apt-get clean -y && \
    rm -rf /var/lib/apt/lists/*

RUN mkdir -p /etc/postgresql && chmod 1777 /etc/postgresql

COPY postgresql.hdd.conf postgresql.ssd.conf /var/postgresql-conf-tpl/
COPY immich-docker-entrypoint.sh healthcheck.sh set-env.sh /usr/local/bin/

HEALTHCHECK --interval=5m --start-interval=5s --start-period=5m CMD /usr/local/bin/healthcheck.sh

RUN if [ -n "$PGVECTORS_TAG" ]; then \
        sed -i "s/vchord.so/vchord.so, vectors.so/" /var/postgresql-conf-tpl/postgresql.*.conf; \
        sed -i "s/, public/, public, vectors/" /var/postgresql-conf-tpl/postgresql.*.conf; \
    fi

ENTRYPOINT ["/usr/local/bin/immich-docker-entrypoint.sh"]

CMD ["-c", "config_file=/etc/postgresql/postgresql.conf"]
