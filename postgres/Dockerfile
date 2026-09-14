ARG PG_MAJOR
ARG PGVECTOR_TAG

# Stage 1: pgvector image to copy extension from
FROM pgvector/pgvector:${PGVECTOR_TAG}-pg${PG_MAJOR} AS pgvector-source

# Stage 2: DHI hardened Postgres base + pgvector + VectorChord (use -dev for RUN)
FROM dhi.io/postgres:${PG_MAJOR}-debian13-dev

ARG PG_MAJOR
ARG VECTORCHORD_TAG
ARG PGVECTORS_TAG
ARG TARGETARCH
# pgvecto.rs 0.3.0 assets are named ..._amd64_vectors.deb; leave empty for 0.2.0
ARG PGVECTORS_SUFFIX=

# Copy pgvector extension from pgvector image into DHI base
COPY --from=pgvector-source /usr/share/postgresql/${PG_MAJOR}/extension/vector* /usr/share/postgresql/${PG_MAJOR}/extension/
COPY --from=pgvector-source /usr/lib/postgresql/${PG_MAJOR}/lib/vector.so /usr/lib/postgresql/${PG_MAJOR}/lib/

# BuildKit ADD follows GitHub 301 (tensorchord → supervc-stack); avoids DHI apt/wget
ADD https://github.com/tensorchord/VectorChord/releases/download/${VECTORCHORD_TAG}/postgresql-${PG_MAJOR}-vchord_${VECTORCHORD_TAG}-1_${TARGETARCH}.deb /tmp/vchord.deb
ADD https://github.com/tensorchord/pgvecto.rs/releases/download/v${PGVECTORS_TAG}/vectors-pg${PG_MAJOR}_${PGVECTORS_TAG}_${TARGETARCH}${PGVECTORS_SUFFIX}.deb /tmp/pgvectors.deb

# DHI default is /var/lib/postgresql/${PG_MAJOR}/data; Immich/homecloud mount /var/lib/postgresql/data
ENV PGDATA=/var/lib/postgresql/data
ENV DB_STORAGE_TYPE=SSD

# DHI postgres base has no postgresql-N apt package; extract .deb instead of apt-get install
# Canonical files stay under /usr. Current DHI already has directory-level
# /opt/postgresql/${PG_MAJOR}/{lib,share} -> /usr/lib/postgresql/${PG_MAJOR}/{lib,share},
# so per-file ln into /opt/lib would collide (same directory as the .so we just copied).
# Extension SQL/control live at Debian /usr/share; DHI sharedir is /usr/lib/.../share
# (also /opt/.../share). Link those into sharedir without duplicating bytes.
RUN mkdir -p \
        /usr/lib/postgresql/${PG_MAJOR}/lib \
        /usr/lib/postgresql/${PG_MAJOR}/share/extension \
        /usr/share/postgresql/${PG_MAJOR}/extension && \
    dpkg-deb -x /tmp/vchord.deb /tmp/vchord && \
    cp -a /tmp/vchord/usr/lib/postgresql/${PG_MAJOR}/lib/vchord.so /usr/lib/postgresql/${PG_MAJOR}/lib/ && \
    cp -a /tmp/vchord/usr/share/postgresql/${PG_MAJOR}/extension/vchord* /usr/share/postgresql/${PG_MAJOR}/extension/ && \
    if [ -n "$PGVECTORS_TAG" ]; then \
        dpkg-deb -x /tmp/pgvectors.deb /tmp/pgvectors && \
        cp -a /tmp/pgvectors/usr/lib/postgresql/${PG_MAJOR}/lib/vectors.so /usr/lib/postgresql/${PG_MAJOR}/lib/ && \
        cp -a /tmp/pgvectors/usr/share/postgresql/${PG_MAJOR}/extension/vectors* /usr/share/postgresql/${PG_MAJOR}/extension/; \
    fi && \
    for f in vector.so vchord.so vectors.so; do \
        src="/usr/lib/postgresql/${PG_MAJOR}/lib/$f"; \
        dest="/opt/postgresql/${PG_MAJOR}/lib/$f"; \
        if [ -e "$src" ] && [ ! -e "$dest" ]; then ln -s "$src" "$dest"; fi; \
    done && \
    ln -s /usr/share/postgresql/${PG_MAJOR}/extension/vector* \
          /usr/share/postgresql/${PG_MAJOR}/extension/vchord* \
          /usr/lib/postgresql/${PG_MAJOR}/share/extension/ && \
    rm -rf /tmp/vchord.deb /tmp/vchord /tmp/pgvectors.deb /tmp/pgvectors

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
