FROM debian:jessie-backports

# explicitly set user/group IDs
RUN groupadd -r cassandra --gid=999 && useradd -r -g cassandra --uid=999 cassandra

# grab gosu for easy step-down from root
ENV GOSU_VERSION 1.7
RUN apt-get update
RUN set -x \
	&& apt-get update && apt-get install -y --no-install-recommends ca-certificates wget && rm -rf /var/lib/apt/lists/* \
	&& wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)" \
	&& wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc" \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& gpg --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
	&& gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
	&& rm -r "$GNUPGHOME" /usr/local/bin/gosu.asc \
	&& chmod +x /usr/local/bin/gosu \
	&& gosu nobody true \
	&& apt-get purge -y --auto-remove ca-certificates wget

RUN apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 514A2AD631A57A16DD0047EC749D6EEC0353B12C

RUN echo 'deb http://www.apache.org/dist/cassandra/debian 12x main' >> /etc/apt/sources.list.d/cassandra.list

ENV CASSANDRA_VERSION 1.2.19

RUN apt-get update \
	&& apt-get install -y cassandra="$CASSANDRA_VERSION" \
	&& rm -rf /var/lib/apt/lists/*

# https://issues.apache.org/jira/browse/CASSANDRA-11661
RUN sed -ri 's/^(JVM_PATCH_VERSION)=.*/\1=25/' /etc/cassandra/cassandra-env.sh

ENV CASSANDRA_CONFIG /etc/cassandra

RUN mkdir -p /var/lib/cassandra "$CASSANDRA_CONFIG" \
	&& chown -R cassandra:cassandra /var/lib/cassandra "$CASSANDRA_CONFIG" \
	&& chmod 777 /var/lib/cassandra "$CASSANDRA_CONFIG"

# ALA configs

COPY cassandra-files/conf/90-nproc.conf /etc/security/limits.d/
COPY cassandra-files/conf/limits.conf /etc/security/ 
COPY cassandra-files/templates/cassandra.yaml /etc/cassandra/

COPY biocache-db/cassandra-schema.txt /opt/

COPY wait-for-it.sh /opt/

RUN mkdir -p /data/cassandra/data /data/cassandra/commitlog /data/cassandra/saved_caches \
	&& chown -R cassandra:cassandra /data/cassandra \
	&& chmod 777 /data/cassandra

RUN cassandra && \
	./opt/wait-for-it.sh --timeout=30 localhost:9042 --strict -- \ 
	cassandra-cli -f /opt/cassandra-schema.txt && \
	cassandra stop

# if any build steps change the data within the volume 
# after it has been declared, those changes will be discarded.
# therefore the VOLUME below comes after the command above

VOLUME /var/lib/cassandra

EXPOSE 7000 7001 7199 9042 9160

ENV TINI_VERSION v0.10.0
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /tini
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini.asc /tini.asc
RUN gpg --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 595E85A6B1B4779EA4DAAEC70B588DFF0527A9B7 \
 && gpg --verify /tini.asc

RUN chmod +x /tini

COPY docker-entrypoint.sh /docker-entrypoint.sh
ENTRYPOINT ["/tini", "--", "/docker-entrypoint.sh"]
CMD ["cassandra", "-f"]
